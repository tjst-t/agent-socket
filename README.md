# agent-socket

Claude Agent SDK を UNIX domain socket 経由で操作するための薄いラッパープロセス。

外部クライアント（Go, Python 等）から JSON Lines プロトコルで Claude Code 相当の機能を利用できる。

## 動機

Claude Agent SDK は Node.js 専用だが、Go 製アプリケーション等から利用したい場合に、SDK を直接組み込むとシングルバイナリが崩れる。agent-socket は独立した Node.js プロセスとして動作し、UNIX domain socket で通信することでこの問題を解決する。

## ドキュメント

- [設計ドキュメント](docs/design.md) - アーキテクチャ、プロトコル仕様、SDK 設定
- [CLAUDE.md](CLAUDE.md) - 開発ガイド（Claude Code 向け）

## ステータス

開発準備中。まず Agent SDK の挙動検証（フェーズ1）から着手する。
