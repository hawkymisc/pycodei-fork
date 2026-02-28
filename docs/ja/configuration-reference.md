# 設定リファレンス

PYCODEI の設定は `~/.pycodei/config.json` で管理します。本ドキュメントはすべての設定キーの完全なリファレンスです。

---

## 設定ファイルの場所

| 場所 | 設定方法 |
|------|---------|
| `~/.pycodei/config.json` | デフォルト |
| `$PYCODEI_CONFIG_DIR/config.json` | `PYCODEI_CONFIG_DIR` 環境変数で上書き |

初回起動時（`pycodei --help` など）にファイルが存在しない場合、組み込みのデフォルト値からテンプレートが自動生成されます（`python_code_interpreter.py:41` の `DEFAULT_CONFIG` 参照）。その後、認証情報の記入を促すエラーで終了します。

---

## 設定の優先順位

設定は以下の優先順位で解決されます（上位が優先）:

1. **環境変数** — `config.json` のすべてのキーは同名の環境変数でも設定可能（例: `DEPLOYMENT_NAME=gpt-4o pycodei`）
2. **`--deployment-name` CLI フラグ** — `DEPLOYMENT_NAME` のみを上書き
3. **`~/.pycodei/config.json`** — メインの設定ファイル
4. **組み込みデフォルト** — ソースコード内の `DEFAULT_CONFIG` 辞書

> **実装メモ:** `apply_config_to_env()`（`python_code_interpreter.py:91`）は起動時に dict・list 以外のすべての設定値を `os.environ` に書き込みます。起動前に環境変数として設定されていた値は config の値で **上書きされます**。

---

## 基本設定

| キー | 型 | 必須 | デフォルト | 説明 |
|-----|-----|------|-----------|------|
| `DEPLOYMENT_NAME` | 文字列 | **必須** | `"gpt-5-mini"` | LLM API に渡すモデル名またはデプロイ名。Azure では deployment 名、OpenAI では モデル ID（例: `"gpt-4o"`）。 |
| `PYCODEI_CLIENT` | 文字列 | **必須** | `"azure"` | LLM プロバイダ。有効値: `"azure"`、`"openai"`（大文字小文字不問）。 |
| `CONVERSATION_LOOP_MAX_CYCLES` | 整数 | 任意 | `100` | 1セッションあたりの ReAct ループ最大回数。この回数を超えると強制終了。 |
| `Title` | 文字列 | 任意 | `"PYCODEI"` | 起動時に表示される ASCII バナーのテキスト。 |
| `TitleFont` | 文字列 | 任意 | `"slant"` | バナーに使う [pyfiglet](https://github.com/pwaller/pyfiglet) フォント名。 |

---

## Azure OpenAI 設定

`PYCODEI_CLIENT` が `"azure"` のとき必須。

| キー | 型 | 必須 | デフォルト | 説明 |
|-----|-----|------|-----------|------|
| `AZURE_OPENAI_API_KEY` | 文字列 | 必須 | `""` | Azure OpenAI API キー。 |
| `AZURE_OPENAI_ENDPOINT` | 文字列 | 必須 | プレースホルダー | Azure エンドポイント URL（例: `"https://my-resource.openai.azure.com/"`）。 |
| `OPENAI_API_VERSION` | 文字列 | 必須 | `"2024-10-01-preview"` | Azure OpenAI API バージョン文字列。 |

---

## OpenAI 設定

`PYCODEI_CLIENT` が `"openai"` のとき必須。

| キー | 型 | 必須 | デフォルト | 説明 |
|-----|-----|------|-----------|------|
| `OPENAI_API_KEY` | 文字列 | 必須 | `""` | OpenAI API キー。 |

---

## MCP サーバー設定

`mcpServers` はキーがユーザー定義のサーバー名、値がサーバー設定オブジェクトの JSON オブジェクトです。MCP サーバーはすべて任意です。

```json
{
  "mcpServers": {
    "<サーバー名>": { ... }
  }
}
```

### サーバーごとのフィールド

| フィールド | 型 | 必須 | デフォルト | 説明 |
|----------|-----|------|-----------|------|
| `disabled` | 真偽値 | 任意 | `false` | `true` にするとこのサーバーをスキップ（設定を残したまま無効化）。 |
| `transport` | 文字列 | 任意 | `"stdio"` | トランスポート種別。[トランスポート種別](#トランスポート種別) 参照。`type` でも指定可能。 |
| `command` | 文字列 | 条件付き | — | 実行ファイル。`stdio` トランスポートで必須。パス形式（`~`・`.`・`/`・`\` で始まる）なら自動展開。 |
| `args` | 文字列配列 | 任意 | `[]` | `command` に渡す引数。 |
| `env` | オブジェクト | 任意 | `{}` | サーバープロセスの追加環境変数。値は文字列に変換されます。 |
| `cwd` | 文字列 | 任意 | — | サーバープロセスの作業ディレクトリ。相対パスは `~/.pycodei/` を基準に解決。 |
| `url` | 文字列 | 条件付き | — | エンドポイント URL。`sse`・`http`・`https`・`websocket`・`ws`・`wss` トランスポートで必須。 |
| `headers` | オブジェクト | 任意 | `{}` | SSE・WebSocket リクエストに付加する HTTP ヘッダー（例: `Authorization`）。 |
| `encoding` | 文字列 | 任意 | `"utf-8"` | stdio 通信の文字エンコーディング。 |
| `encoding_error_handler` | 文字列 | 任意 | `"strict"` | エンコーディングエラーのハンドラ。`encodingErrorHandler` でも指定可能。 |

### トランスポート種別

| 値 | プロトコル | 必須フィールド |
|----|----------|--------------|
| `"stdio"`（デフォルト） | ローカルサブプロセス（stdin/stdout） | `command` |
| `"sse"` | HTTP Server-Sent Events | `url` |
| `"http"`、`"https"` | HTTP（`sse` のエイリアス） | `url` |
| `"websocket"`、`"ws"`、`"wss"` | WebSocket | `url` |

---

## 完全な設定例（コメント付き）

```json
{
  "DEPLOYMENT_NAME": "gpt-4o",
  "PYCODEI_CLIENT": "azure",

  "AZURE_OPENAI_API_KEY": "your-azure-api-key",
  "AZURE_OPENAI_ENDPOINT": "https://my-resource.openai.azure.com/",
  "OPENAI_API_VERSION": "2024-10-01-preview",

  "OPENAI_API_KEY": "sk-...",

  "CONVERSATION_LOOP_MAX_CYCLES": 50,

  "Title": "MyAgent",
  "TitleFont": "banner3-D",

  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"],
      "transport": "stdio"
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env": {"FS_ROOT": "/workspace"},
      "cwd": "/workspace"
    },
    "remote-api": {
      "transport": "sse",
      "url": "https://api.example.com/mcp",
      "headers": {"Authorization": "Bearer secret-token"}
    },
    "disabled-server": {
      "disabled": true,
      "command": "npx",
      "args": ["some-mcp-server"]
    }
  }
}
```

---

## PYCODEI.md — エージェント指示書ファイル

`config.json` に加えて、PYCODEI はシステムプロンプトを拡張するオプションの Markdown 指示書ファイルをサポートしています。ソースコードを変更せずに、ドメイン固有のコンテキスト、安全制約、プロジェクト固有の要件を追加できます。

### 検索順序

`load_pycodei_guide()`（`python_code_interpreter.py:109`）は以下の場所を順に検索し、**最初に見つかった空でないファイル** を使用します:

1. `{カレントディレクトリ}/PYCODEI.md`
2. `~/.pycodei/PYCODEI.md`
3. `{インストールディレクトリ}/PYCODEI.md`

### プロンプトへの挿入方法

ファイルの内容はベースシステムプロンプトにそのまま追記されます:

```
{base_system_content}

Additional instructions from PYCODEI.md:
{PYCODEI.md の内容}
```

### 記述例

```markdown
## プロジェクトコンテキスト

このエージェントは販売データを格納した PostgreSQL データベースを扱います。

### スキーマ
- `orders(id, customer_id, total, created_at)`
- `customers(id, name, email, region)`

### ルール
- ユーザーの明示的な確認なしに DELETE・DROP 文を実行してはならない。
- 金額は常に円（JPY）で小数点以下 0 桁で出力すること。
```

---

## プロンプト履歴

PYCODEI は CLI のプロンプト履歴を `~/.pycodei/prompt_history` に保存します（`prompt_toolkit` の `FileHistory` ファイル）。`config.json` での設定は不要です。
