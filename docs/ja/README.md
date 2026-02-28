# PYCODEI — 設計ドキュメント

このディレクトリは **外部メンテナー・コントリビューター向け** の設計ドキュメントを収録しています。

> English documentation is available in [docs/](../).

## PYCODEI とは

PYCODEI は AI 駆動の Python コードインタープリタ兼アナリティクスエージェントです。大規模言語モデル（OpenAI または Azure OpenAI）と、ステートフルな Jupyter ノートブック実行環境を **ReAct**（推論 + 実行）ループで組み合わせています。ユーザーは CLI から自然言語でタスクを指示するだけで、データ分析・機械学習・可視化などの計算処理をエージェントが自律的に実行します。

## ドキュメント一覧

| ドキュメント | 内容 |
|------------|------|
| [architecture.md](./architecture.md) | システムアーキテクチャ：コンポーネント構成、データフロー、クラス構造、ReAct ループ、ツール承認システム |
| [configuration-reference.md](./configuration-reference.md) | 全設定キーのリファレンス、環境変数、MCP サーバー設定、`PYCODEI.md` ガイドファイル |
| [mcp-integration.md](./mcp-integration.md) | MCP（Model Context Protocol）サーバーの統合方法：トランスポート、ツール発見、実行フロー、新規サーバー追加手順 |
| [contributing.md](./contributing.md) | 開発環境のセットアップ、コーディング規約、テスト方法、コミット・PR ガイドライン、拡張パターン集 |

## ソースコード概要

```
pycodei-fork/
├── python_code_interpreter.py       # CLI エントリポイント・ReAct ループ・設定・ツール承認
├── python_code_notebook.py          # Jupyter ノートブック生成・コード実行
├── mcp_client_manager.py            # MCP サーバーのライフサイクル管理・ツール実行
├── set_matplotlib_japanese_font.py  # 日本語フォント設定ヘルパー
├── papermill_enhancement/           # Git サブモジュール：Papermill フォーク（例外処理強化版）
├── sample_data/                     # サンプルデータセット
├── sample_results/                  # 参照用サンプルノートブック
└── docs/                            # ドキュメントディレクトリ
    ├── ja/                          # 日本語ドキュメント（このディレクトリ）
    └── *.md                         # 英語ドキュメント
```

まずは [architecture.md](./architecture.md) を読むとシステム全体の構造を把握できます。
