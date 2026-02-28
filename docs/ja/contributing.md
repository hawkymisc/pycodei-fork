# コントリビュートガイド

本ドキュメントでは、PYCODEI のメンテナー向けに開発環境のセットアップ、コーディング規約、テスト方法、よく使う拡張パターンを説明します。

---

## 開発環境のセットアップ

### 必要な環境

- Python >= 3.12
- `pip`
- 有効な OpenAI または Azure OpenAI アカウントとデプロイ済みモデル
- （任意）`stdio` 型 MCP サーバー（例: `@playwright/mcp`）をテストする場合は Node.js

### クローンとインストール

```bash
git clone https://github.com/KentaroAOKI/pycodei.git
cd pycodei

# Papermill サブモジュールを初期化
git submodule update --init --recursive

# 編集可能モードでインストール（`pycodei` コンソールスクリプトを登録）
pip install -e .
```

### 設定

```bash
# config ディレクトリを作成し、PYCODEI にテンプレートを生成させる
pycodei --help
# → Error: Created a config template at ~/.pycodei/config.json. Update it with...

# テンプレートを編集
nano ~/.pycodei/config.json
```

最低限以下を入力してください:
```json
{
  "DEPLOYMENT_NAME": "gpt-4o",
  "PYCODEI_CLIENT": "openai",
  "OPENAI_API_KEY": "sk-..."
}
```

全設定オプションは [configuration-reference.md](./configuration-reference.md) を参照してください。

### 動作確認

```bash
pycodei --version
# pycodei 0.1.9

pycodei "print('PYCODEI から hello')"
# 簡単な Python タスクで ReAct ループを動作確認
```

---

## リポジトリ構造

```
pycodei-fork/
├── python_code_interpreter.py       # CLI + ReAct ループ + 設定 + ツール承認
├── python_code_notebook.py          # Jupyter ノートブック生成・コード実行
├── mcp_client_manager.py            # MCP サーバーのライフサイクル・ツール実行
├── set_matplotlib_japanese_font.py  # 日本語フォント設定（一回実行型）
├── papermill_enhancement/           # Git サブモジュール：例外処理強化版 Papermill
├── pyproject.toml                   # パッケージメタデータ・依存関係・エントリポイント
├── requirements.txt                 # 直接依存パッケージ一覧（開発インストール用）
├── README.md                        # ユーザー向け README
├── AGENTS.md                        # 開発ガイドライン（スタイルの根拠となるドキュメント）
├── .env.sample                      # 環境変数テンプレートのサンプル
├── sample_data/                     # サンプル入力データセット
│   └── diagnosis.csv                # 乳がん特徴量データセット
├── sample_results/                  # 参照用サンプルノートブック
│   ├── sample_01.ipynb              # 株価予測（英語）
│   ├── sample_02.ipynb              # がん分類（英語）
│   └── ...
└── docs/                            # ドキュメントディレクトリ
    ├── ja/                          # 日本語ドキュメント（このディレクトリ）
    └── *.md                         # 英語ドキュメント
```

**実行時ディレクトリ（.gitignore 対象）:**

```
ai_workspace/    # 生成コードが使う永続データストレージ
notebooks/       # セッションごとのタイムスタンプ付き Jupyter ノートブック
logs/            # JSON 形式の会話ログ
```

---

## コーディング規約

`AGENTS.md` と既存ソースコードに従ってください:

- **スタイル:** PEP 8 — インデント 4 スペース、関数・変数は snake_case、クラスは CapWords
- **インポート:** 明示的なインポートのみ（`from x import *` 禁止）。標準ライブラリ → サードパーティ → ローカルモジュールの順
- **定数:** パスプレフィックスや設定定数はモジュールの先頭近くに配置。環境変数の取得は `os.getenv()` でラップ
- **設定キー:** `DEFAULT_CONFIG` に追加した新しいキーは必ず [configuration-reference.md](./configuration-reference.md) に記載
- **アーティファクトの命名:** 既存のタイムスタンプ形式を維持:
  - ノートブックディレクトリ: `YYYYMMDDHHMMSS`（区切りなし）
  - ログファイル: `log_YYYYMMDD-HHMMSS.json`（ハイフン区切り）
- **ログ出力:** 新しいモジュールは `logging.getLogger("pycodei.<モジュール名>")` を使用。`python_code_interpreter.py` はユーザー向け出力に `print()` を使用しており、このパターンを維持すること
- **エラー処理:**
  - 設定エラー: `RuntimeError` を raise し、`initialize_configuration()` が `SystemExit` に変換
  - ツールエラー: エージェントがリカバリできるよう、raise せずにエラー文字列を LLM に返す
  - MCP サーバーエラー: `logger.warning(...)` を出力してそのサーバーをスキップ

---

## テスト方法

PYCODEI には正式なテストハーネスがありません（`AGENTS.md` 参照）。検証は手動テストで行います:

1. **実際のプロンプトを実行する:**
   ```bash
   pycodei "sample_data/diagnosis.csv を分析し、悪性腫瘍の予測に最も有効な特徴量を特定してください"
   ```

2. **コンソール出力を確認:**
   - ツール呼び出しのトレースが正しいか
   - 関数の引数が期待どおりか
   - トークン使用量カウンターが増加しているか
   - 予期しないエラーが出ていないか

3. **生成されたノートブックを確認**（`notebooks/{timestamp}/notebook.ipynb`）を VS Code または Jupyter Lab で開く:
   - グラフが正しくレンダリングされているか
   - エラーハンドリングのテスト時にエラーセルが正しい出力を示しているか
   - Markdown セルが会話の流れを正しく反映しているか

4. **会話ログを確認**（`logs/log_{timestamp}.json`）:
   - すべてのメッセージロールが存在し、正しく構造化されているか
   - ツール呼び出し ID とツール結果が一致しているか

5. **リグレッション確認:** 新しいプロンプトと期待される出力のペアを `sample_results/` に追加すると、レビュアーが比較できます

### 役立つテストプロンプト集

| シナリオ | プロンプト |
|---------|-----------|
| 基本実行 | `"最初の 10 個の素数を出力してください"` |
| データ読み込み | `"sample_data/diagnosis.csv を読み込み、最初の 5 行を表示してください"` |
| 可視化 | `"iris データセットの萼片長のヒストグラムを作成してください"` |
| 複数ステップ | `"iris データを読み込み、分類器を学習させ、精度を報告してください"` |
| エラーリカバリ | `"import nonexistent_library"` （LLM が ImportError を処理することを確認） |
| メモリ巻き戻し | セッションを 1 つ実行後: `pycodei "前の分析を続けてください" --load-message logs/log_*.json` |

---

## よく使う拡張パターン

### 新しい組み込みツールの追加

組み込みツールは `PythonCodeInterpreter.__init__()`（`python_code_interpreter.py:203`）で登録します。

1. `self.tools` に OpenAI ツールスペックを追加する:
   ```python
   self.tools.append({
       "type": "function",
       "function": {
           "name": "my_new_tool",
           "description": "このツールの説明。",
           "parameters": {
               "type": "object",
               "properties": {
                   "param_name": {"type": "string", "description": "..."}
               },
               "required": ["param_name"]
           }
       }
   })
   ```

2. `self.available_functions` に callable を追加する:
   ```python
   self.available_functions["my_new_tool"] = self._my_new_tool_impl
   ```

3. メソッドを実装する。シグネチャは以下に合わせること:
   ```python
   def _my_new_tool_impl(self, function_arguments: str, messages: list) -> str:
       args = json.loads(function_arguments)
       # ... 処理 ...
       return "LLM に返す結果の文字列"
   ```

4. `_register_tool_descriptions()` は `self.tools` のツールに対して自動的に呼び出されるため、追加の手順は不要です。

### 新しい MCP サーバーの追加

詳しい手順は [mcp-integration.md](./mcp-integration.md#新規-mcp-サーバーの追加手順) を参照してください。概要:

1. `~/.pycodei/config.json` の `mcpServers` にエントリを追加する
2. PYCODEI を再起動 — 起動時に自動的に発見が実行される

### 新しい LLM プロバイダのサポート

LLM クライアントは `create_llm_client()`（`python_code_interpreter.py:139`）で生成されます。OpenAI Python SDK のインターフェース（`client.chat.completions.create(model, messages, tools)`）と互換性があるプロバイダであれば追加できます:

1. `create_llm_client()` に `elif provider == "my_provider":` ブランチを追加する
2. `resolve_client_provider()` の有効値に新しいプロバイダ名を追加する
3. `DEFAULT_CONFIG` に新しい `PYCODEI_CLIENT` 値と必要な認証キーを追加する
4. [configuration-reference.md](./configuration-reference.md) に新しいキーを記載する

### システムプロンプトのカスタマイズ

2 通りのアプローチがあります:

- **実行時（コード変更不要）:** `~/.pycodei/PYCODEI.md` または `./PYCODEI.md` を作成する。内容はベースシステムプロンプトに追記されます。詳細は [configuration-reference.md](./configuration-reference.md#pycodeimd--エージェント指示書ファイル) を参照。
- **構造的（コード変更）:** `PythonCodeInterpreter.__init__()`（`python_code_interpreter.py:166`）の `base_system_content` を直接編集する。

### 新しい設定キーの追加

1. `python_code_interpreter.py:41` の `DEFAULT_CONFIG` にデフォルト値付きでキーを追加する
2. 対象のコードで `os.getenv("MY_KEY")` で参照する（`apply_config_to_env()` が env に書き込む）
3. [configuration-reference.md](./configuration-reference.md) に記載する
4. PR の説明に新しいキーを明記する（`AGENTS.md` の規約）

---

## コミット・PR ガイドライン

`AGENTS.md` より:

- **コミットメッセージ:** 短い命令形の要約（例: `"add tool approval"`、`"feature: initialize_notebook"`、`"fix: handle empty MCP tool list"`）
- **PR の説明に必須の記載事項:**
  - 変更の目的を説明する記述
  - 新規・変更した `config.json` のキー（あれば）
  - 再現手順: `pycodei "<テストプロンプト>"`
  - UI や出力が変わる場合はスクリーンショットやノートブックの抜粋
  - 関連する Issue へのリンク
  - 破壊的変更や手動マイグレーション手順（あれば）

- **コミットしてはいけないもの:**
  - 個人のノートブック（実行時生成ファイル）
  - API キーや認証情報
  - `ai_workspace/`・`notebooks/`・`logs/` 以下のファイル（すべて .gitignore 対象）
  - 大きなバイナリファイル

---

## `papermill_enhancement` サブモジュール

`python_code_notebook.py` は冒頭（4行目）で `papermill_enhancement.papermill` をインポートしています。これは [nteract/papermill](https://github.com/nteract/papermill) の例外処理強化フォークです。

**初期化コマンド:**
```bash
git submodule update --init --recursive
```

**サブモジュールが未初期化の場合**、起動時に `ModuleNotFoundError: No module named 'papermill_enhancement'` が発生します。

**アップストリーム Papermill へのフォールバック:** サブモジュールを初期化できない場合は `python_code_notebook.py` の 4 行目を変更します:
```python
# 変更前:
import papermill_enhancement.papermill as pm
# 変更後:
import papermill as pm
```
アップストリームの `papermill==2.6.0` は `requirements.txt` からインストール済みです。ただし例外処理の品質が若干低下する場合があります。

**サブモジュールの更新:**
```bash
cd papermill_enhancement
git fetch origin
git checkout enhancement-exception-handling
git pull
cd ..
git add papermill_enhancement
git commit -m "update papermill_enhancement submodule"
```

---

## ソースコード参照ポイント一覧

| トピック | ファイル | 行 |
|---------|---------|---|
| デフォルト設定値 | `python_code_interpreter.py` | 41–52 |
| 設定ロード | `python_code_interpreter.py` | 67–106 |
| システムプロンプト構築 | `python_code_interpreter.py` | 166–202 |
| ツール登録 | `python_code_interpreter.py` | 203–235 |
| ツール承認ゲート | `python_code_interpreter.py` | 447–471 |
| ReAct ループ | `python_code_interpreter.py` | 389–528 |
| CLI 引数パース | `python_code_interpreter.py` | 530–609 |
| ノートブック実行 | `python_code_notebook.py` | 53–143 |
| トレースバック省略 | `python_code_notebook.py` | 31–50 |
| MCP サーバー設定解析 | `mcp_client_manager.py` | 151–216 |
| MCP ツール発見 | `mcp_client_manager.py` | 218–246 |
| MCP ツール名生成 | `mcp_client_manager.py` | 248–269 |
| MCP トランスポート分岐 | `mcp_client_manager.py` | 393–416 |
| MCP 実行結果フォーマット | `mcp_client_manager.py` | 328–338 |
