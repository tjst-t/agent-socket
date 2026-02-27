# agent-socket 開発ガイド

## プロジェクト概要

agent-socket は Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`) を薄くラップした Node.js プロセス。
UNIX domain socket 上の JSON Lines プロトコルで外部クライアント（Palmux 等）と通信する。

設計の詳細は `docs/design.md` を必ず読むこと。

## 技術スタック

- ランタイム: Node.js
- 言語: TypeScript (strict mode)
- SDK: @anthropic-ai/claude-agent-sdk
- 通信: UNIX domain socket + JSON Lines (改行区切り JSON)
- パッケージマネージャ: pnpm

## アーキテクチャ原則

- **1セッション = 1プロセス**: OOM 時の障害隔離、ライフサイクルの単純化
- **薄いラッパー**: Agent SDK のイベントを JSON Lines に変換するだけ。独自のロジックは最小限にする
- **クライアント非依存**: socket プロトコルさえ実装すれば任意の言語からクライアントを書ける

## ディレクトリ構成（想定）

```
agent-socket/
├── CLAUDE.md              # このファイル
├── README.md
├── docs/
│   └── design.md          # 設計ドキュメント
├── src/
│   ├── index.ts           # エントリポイント（CLI引数パース → サーバー起動）
│   ├── server.ts          # UNIX socket サーバー
│   ├── session.ts         # Agent SDK セッション管理
│   ├── protocol.ts        # JSON Lines メッセージの型定義
│   └── approval.ts        # canUseTool 承認フロー（Promise ベースの待機）
├── package.json
├── tsconfig.json
└── tests/
```

## 開発の進め方

### フェーズ1: SDK 検証（最優先）

Agent SDK の実際の挙動を確認する。プロトコル設計はこの結果に依存する。

1. 最小限のスクリプトで `query()` を呼び、イベントストリームの形式を記録する
2. `canUseTool` コールバックの動作を確認する（引数の型、戻り値の効果）
3. `resume` オプションでセッション再開が機能するか確認する
4. `settingSources` で CLAUDE.md やスキルが正しく読み込まれるか確認する
5. エラー発生時の挙動を確認する（無効な prompt、ネットワークエラー等）

結果を `docs/sdk-findings.md` に記録すること。

### フェーズ2: コア実装

SDK 検証の結果を踏まえてプロトコルを確定し、実装する。

1. TypeScript プロジェクトの初期化（tsconfig.json, package.json）
2. `protocol.ts`: JSON Lines メッセージの型定義
3. `server.ts`: UNIX domain socket サーバー（接続受付、JSON Lines 読み書き）
4. `session.ts`: Agent SDK セッション管理（query 呼び出し、イベント変換）
5. `approval.ts`: canUseTool の承認フロー（タイムアウト付き Promise）
6. `index.ts`: CLI エントリポイント（引数パース、サーバー起動）

### フェーズ3: 堅牢化

1. エラーハンドリング（SDK エラー、socket エラー、クライアント切断）
2. グレースフルシャットダウン（SIGTERM 処理、socket ファイル削除）
3. ログ出力（stderr、構造化ログ）
4. MCP サーバー設定の自動読み込み（settings.json パース）

## コーディング規約

- エラーは握りつぶさない。復旧できないエラーは `process.exit(1)` で即死させる（1セッション1プロセスなので安全）
- 外部入力（socket から受信した JSON）は必ずバリデーションする
- Agent SDK の型定義を最大限活用し、`any` は使わない
- canUseTool の承認待ちには必ずタイムアウトを設ける（デッドロック防止）
- コメントは日本語で書いてよい

## CLI インターフェース

```bash
agent-socket --cwd <dir> --socket <path> [--model <model>] [--resume <session-id>] [--permission <mode>]
```

| オプション | 必須 | デフォルト | 説明 |
|-----------|------|-----------|------|
| --cwd | yes | - | 作業ディレクトリ |
| --socket | yes | - | UNIX socket パス |
| --model | no | claude-sonnet-4-6 | モデル指定 |
| --resume | no | - | セッション再開用 session ID |
| --permission | no | default | パーミッションモード |

## テスト方法

Socket に直接 JSON を送って動作確認できる:

```bash
# 別ターミナルで agent-socket を起動
agent-socket --cwd . --socket /tmp/test.sock

# JSON Lines を送信
echo '{"type":"message","text":"hello"}' | socat - UNIX-CONNECT:/tmp/test.sock
```

## 注意事項

- Agent SDK はまだ初期段階のため、API が変わる可能性がある。SDK のバージョンは package.json で固定すること
- Claude Code CLI のセッションと agent-socket のセッションに互換性はない
- MCP サーバー設定は settingSources 経由では読み込まれないため、別途対応が必要
