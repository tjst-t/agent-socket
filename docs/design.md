# agent-socket 設計ドキュメント

## 概要

agent-socket は、Claude Agent SDK を薄くラップした Node.js プロセス。
Claude Code の TUI を UNIX domain socket 経由の JSON プロトコルに置き換え、外部 UI（Palmux 等）から構造化データで Claude とやり取りできるようにする。

## 背景・動機

Palmux（Go 製ターミナルクライアント）に Claude Desktop 風のチャット UI タブを追加したい。
しかし以下の制約がある：

| 方式 | 問題 |
|------|------|
| Claude Code CLI `stream-json` + `--resume` | ターンごとにプロセス再起動。起動コストが高い |
| Agent SDK を Palmux に組み込む | Node.js ランタイム依存で Go シングルバイナリが崩れる |
| Anthropic API を Go から直接呼ぶ | ツール実行（Bash, Read, Edit 等）の再実装が必要 |

**解決策**: Agent SDK をラップした独立プロセス（agent-socket）を作り、UNIX domain socket で通信する。
Palmux はシングルバイナリのまま、socket クライアントとして接続するだけ。

## アーキテクチャ

```
Palmux (Go, シングルバイナリ)
  ├── ターミナルタブ: tmux + Claude Code CLI（従来通り）
  └── チャットタブ: agent-socket に UNIX socket 接続

agent-socket (Node.js, 別プロジェクト)
  ├── 1セッション = 1プロセス（障害隔離）
  ├── Agent SDK (@anthropic-ai/claude-agent-sdk)
  ├── UNIX domain socket (/tmp/agent-socket-<session-id>.sock)
  └── settingSources で Claude Code の設定を引き継ぐ
```

### 1セッション1プロセスの理由

- OOM 時に他セッションに影響しない（障害隔離）
- ライフサイクルが単純（プロセス起動 = セッション開始、終了 = セッション終了）
- Claude Code と同じモデル（tmux が管理する1ウィンドウ1プロセス）

## 通信プロトコル

UNIX domain socket 上で改行区切り JSON (JSON Lines) をやり取りする。

### クライアント → agent-socket

```jsonl
{"type":"message","text":"バグを直して"}
{"type":"approve","request_id":"req_123"}
{"type":"deny","request_id":"req_123","reason":"危険なコマンド"}
{"type":"abort"}
```

### agent-socket → クライアント

```jsonl
{"type":"init","session_id":"sess_abc"}
{"type":"text_delta","text":"ファイルを確認します..."}
{"type":"tool_use","request_id":"req_123","tool":"Bash","input":{"command":"git status"}}
{"type":"approval_request","request_id":"req_124","tool":"Edit","input":{"file_path":"...","old_string":"...","new_string":"..."}}
{"type":"tool_result","request_id":"req_124","output":"..."}
{"type":"done","usage":{"input_tokens":1234,"output_tokens":567}}
{"type":"error","message":"..."}
```

**注意**: 上記メッセージ型は暫定。Agent SDK のイベント形式を実際に調査した上で確定させること（フェーズ1）。

### フロー例

```
Client                              agent-socket
  │                                      │
  ├── {"type":"message",                 │
  │    "text":"バグ直して"} ───────────→ │
  │                                      ├── Agent SDK に prompt 送信
  │                                      │
  │  ←─── {"type":"text_delta",          │
  │        "text":"調査します"} ─────────┤
  │                                      │
  │  ←─── {"type":"approval_request",    │
  │        "request_id":"req_1",         │
  │        "tool":"Bash",                │
  │        "input":{"command":           │
  │         "grep -r 'bug' src/"}} ──────┤
  │                                      │
  ├── {"type":"approve",                 │
  │    "request_id":"req_1"} ──────────→ │
  │                                      ├── canUseTool 解決、ツール実行
  │                                      │
  │  ←─── {"type":"tool_result", ...} ───┤
  │  ←─── {"type":"text_delta", ...} ────┤
  │  ←─── {"type":"done", ...} ──────────┤
  │                                      │
```

## Agent SDK の設定

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const session = query({
  prompt: userMessage,
  options: {
    cwd: workingDirectory,
    settingSources: ["user", "project", "local"],
    // ↑ これで以下が読み込まれる:
    //   ~/.claude/settings.json (ユーザー設定)
    //   .claude/settings.json (プロジェクト設定)
    //   .claude/settings.local.json (ローカル設定)
    //   CLAUDE.md (プロジェクト指示)
    //   .claude/skills/ (スキル)

    // MCP サーバーは settingSources 経由では読まれないため別途指定が必要
    // mcpServers: { ... },

    // セッション再開
    // resume: previousSessionId,
  }
});
```

### settingSources で引き継がれるもの

| 設定 | 引き継ぎ |
|------|----------|
| ~/.claude/settings.json | ✅ (`"user"`) |
| .claude/settings.json | ✅ (`"project"`) |
| .claude/settings.local.json | ✅ (`"local"`) |
| CLAUDE.md | ✅ (`"project"`) |
| .claude/skills/ | ✅ (`allowedTools` に `"Skill"` 追加が必要) |
| パーミッションモード | ✅ |
| MCP サーバー | ⚠️ `mcpServers` オプションで別途指定 |

### canUseTool コールバック（承認フローの核心）

```typescript
canUseTool: async (toolName, input) => {
  const requestId = generateId();

  // クライアントに承認リクエストを送信
  socket.write(JSON.stringify({
    type: "approval_request",
    request_id: requestId,
    tool: toolName,
    input: input,
  }) + "\n");

  // クライアントが approve/deny するまで待つ
  // ⚠️ タイムアウト必須（デッドロック防止）
  const decision = await waitForApproval(requestId, { timeoutMs: 300_000 });

  if (decision.approved) {
    return { behavior: "allow", updatedInput: input };
  } else {
    return { behavior: "deny", message: decision.reason || "User denied" };
  }
}
```

## セッション管理

### 起動

```bash
agent-socket --cwd /path/to/project --socket /tmp/agent-socket-<id>.sock
```

### 再開

```bash
agent-socket --cwd /path/to/project --socket /tmp/agent-socket-<id>.sock --resume <session-id>
```

### 終了

- クライアントが切断 → プロセスは待機（再接続可能、ただし SDK セッションの生存は要検証）
- SIGTERM → グレースフルシャットダウン（ソケットファイル削除）
- SIGKILL / OOM → OS がプロセス回収（他セッションに影響なし）

## セッション互換性

**Claude Code CLI と agent-socket のセッションに互換性はない。**

- Claude Code で始めた会話を agent-socket で resume → 不可
- agent-socket で始めた会話を Claude Code で resume → 不可
- 同じワーキングディレクトリでの作業は可能（ファイルは共有される）

これは Agent SDK の仕様上の制約であり、受け入れる。

## Palmux 側の統合（参考）

### Go バックエンド

- agent-socket プロセスの起動・管理（tmux window 内）
- UNIX socket への接続・JSON Lines の読み書き
- WebSocket 経由でフロントエンドにイベントを中継

### フロントエンド（チャットタブ）

- チャット UI の描画（Markdown レンダリング、コードブロック、diff 表示）
- ツール承認ダイアログ（Bash コマンド表示、ファイル編集 diff 表示）
- ストリーミングテキスト表示

## 未決定事項

- [ ] Agent SDK のイベント形式の詳細調査（フェーズ1 で実施）
- [ ] MCP サーバー設定の自動読み込み方法（settings.json からパースして渡す？）
- [ ] エラーハンドリング方針（Agent SDK のエラー種別と復旧戦略）
- [ ] ログ出力方針（デバッグ時の可観測性）
- [ ] クライアント切断時の SDK セッション生存確認
