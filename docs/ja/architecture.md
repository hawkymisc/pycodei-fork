# アーキテクチャ

## 概要

PYCODEI は **ReAct**（推論 + 実行）パターンを用いた CLI ベースの AI エージェントです。ユーザーが自然言語でタスクを入力すると、エージェントは LLM を繰り返し呼び出し、生成された Python コードをステートフルな Jupyter カーネルで実行し、その結果を LLM に返すサイクルを、タスクが完了するまで繰り返します。

---

## コンポーネント一覧

| モジュール | 行数 | 責務 |
|----------|------|-----|
| `python_code_interpreter.py` | 613 | CLI エントリポイント、設定ロード、LLM クライアント生成、ReAct 会話ループ、ツール承認システム |
| `python_code_notebook.py` | 165 | Jupyter ノートブック生成・Papermill によるコード実行・結果キャプチャ |
| `mcp_client_manager.py` | 416 | MCP サーバーの発見・ライフサイクル管理・全トランスポートにおけるツール実行 |
| `set_matplotlib_japanese_font.py` | 53 | matplotlib の日本語フォント設定ヘルパー（一回実行型） |
| `papermill_enhancement/` | サブモジュール | [nteract/papermill](https://github.com/nteract/papermill) のフォーク（例外処理強化版） |

---

## コンポーネント依存関係

```mermaid
graph TD
    CLI["python_code_interpreter.py<br/>CLI / オーケストレーター"]
    NB["python_code_notebook.py<br/>ノートブック実行"]
    MCP["mcp_client_manager.py<br/>MCP ツールマネージャー"]
    PM["papermill_enhancement/<br/>Papermill フォーク"]
    LLM["LLM API<br/>OpenAI / Azure OpenAI"]
    MCPSRV["外部 MCP サーバー<br/>stdio / SSE / WebSocket"]
    CFG["~/.pycodei/config.json<br/>設定ファイル"]
    GUIDE["PYCODEI.md<br/>エージェント指示書"]

    CLI -->|"run_all() を呼び出し"| NB
    CLI -->|"get_openai_tools()"| MCP
    CLI -->|"chat.completions.create()"| LLM
    CLI -->|"読み込み"| CFG
    CLI -->|"読み込み"| GUIDE
    NB -->|"execute_notebook()"| PM
    MCP -->|"stdio/SSE/WS"| MCPSRV
```

---

## ReAct 会話ループ

中核となる `PythonCodeInterpreter.run_conversation()`（`python_code_interpreter.py:389`）は、`CONVERSATION_LOOP_MAX_CYCLES`（デフォルト: 100）回まで繰り返します。

```mermaid
sequenceDiagram
    actor ユーザー
    participant CLI as python_code_interpreter.py
    participant LLM as LLM API
    participant NB as python_code_notebook.py
    participant MCP as mcp_client_manager.py

    ユーザー->>CLI: pycodei "./data.csv を分析して"
    CLI->>CLI: 設定ロード・システムプロンプト構築・ツール登録
    CLI->>LLM: chat.completions.create(messages, tools)

    loop ReAct ループ（最大 100 回）
        LLM-->>CLI: finish_reason = "tool_calls"
        CLI->>ユーザー: ツール詳細を表示 + 承認プロンプト
        alt 承認
            alt run_python ツール
                CLI->>NB: run_all(python_code, messages)
                NB-->>CLI: 実行結果
            else MCP ツール
                CLI->>MCP: callable(arguments, messages)
                MCP-->>CLI: JSON 結果
            end
            CLI->>LLM: ツール結果を追加して再呼び出し
        else 拒否
            CLI->>LLM: "実行スキップ" の結果を追加して再呼び出し
        end

        LLM-->>CLI: finish_reason = "stop"
        CLI->>NB: メッセージをノートブックに書き込み
        CLI->>ユーザー: LLM の応答を表示
        ユーザー->>CLI: 追加の指示（または "exit"）
        CLI->>LLM: ユーザーメッセージを追加して再呼び出し
    end

    CLI->>CLI: logs/{result_name}.json に保存
```

**ループ終了条件:**
- ユーザーが `exit` と入力した
- `stop` 応答後にユーザーが終了した
- `max_loops` に達した
- LLM API エラー

---

## 起動シーケンス

```mermaid
sequenceDiagram
    participant main as main()
    participant CLI as PythonCodeInterpreter
    participant CFG as config.json
    participant MCP as MCPClientManager

    main->>CFG: load_user_config()
    CFG-->>main: 設定 dict（ファイルなければテンプレート生成 + 終了）
    main->>main: apply_config_to_env(config)
    main->>main: MCPClientManager(config["mcpServers"])
    main->>CLI: PythonCodeInterpreter(deployment_name)
    CLI->>CLI: create_llm_client() → AzureOpenAI または OpenAI
    CLI->>CLI: システムプロンプト構築 + PYCODEI.md 読み込み
    CLI->>CLI: run_python ツール登録
    CLI->>MCP: get_openai_tools()（遅延実行：初回呼び出し時にキャッシュ構築）
    MCP-->>CLI: (mcp_tools, mcp_function_map)
    CLI->>CLI: MCP ツールを self.tools にマージ
    main->>CLI: initialize_messages(messages)
    main->>CLI: initialize_notebook(messages)
    main->>CLI: run_conversation()
```

---

## クラス構造

```mermaid
classDiagram
    class PythonCodeInterpreter {
        +client: AzureOpenAI | OpenAI
        +deployment_name: str
        +messages: list
        +tools: list
        +available_functions: dict
        +tool_auto_permissions: dict
        +tool_function_auto_permissions: set
        +tool_descriptions: dict
        +ipynb_dir: str
        +ipynb_file: str
        +persistent_data_dir: str
        +run_conversation() list
        +run_python_code_in_notebook(code, messages) str
        +initialize_messages(messages, overwrite_system) void
        +initialize_notebook(messages) void
        +_prompt_tool_execution(fn_name, fn_args) str
        +_tool_approval_key(fn_name, raw_args) str
        +print_title() void
    }

    class MCPClientManager {
        +base_dir: Path
        +client_info: Implementation
        +has_servers: bool
        +get_openai_tools() tuple
        -_servers: dict
        -_openai_tools: list
        -_function_map: dict
        -_bindings: dict
        -_build_tool_cache() void
        -_parse_servers(raw_servers) dict
        -_list_tools_async(server_config) list
        -_call_tool_async(server_config, tool_name, args) str
        -_create_callable(binding) Callable
        -_assign_function_names(bindings) void
    }

    class MCPServerConfig {
        +name: str
        +transport: str
        +command: str
        +args: list
        +env: dict
        +cwd: str
        +url: str
        +headers: dict
        +encoding: str
        +encoding_errors: str
        +disabled: bool
        +stdio_parameters() StdioServerParameters
    }

    class MCPToolBinding {
        +server: MCPServerConfig
        +tool_name: str
        +description: str
        +input_schema: dict
        +output_schema: dict
        +function_name: str
    }

    PythonCodeInterpreter --> MCPClientManager : 使用
    MCPClientManager --> MCPServerConfig : 保持
    MCPClientManager --> MCPToolBinding : 生成
    MCPToolBinding --> MCPServerConfig : 参照
```

---

## ツール承認システム

LLM からのツール呼び出しリクエストは、実行前に必ず承認ゲートを通過します（`python_code_interpreter.py:447`）。承認はプロセス内でキャッシュされ、同じツール・同じ引数への繰り返しプロンプトを防ぎます。

```mermaid
stateDiagram-v2
    [*] --> キャッシュ確認: LLM からツール呼び出し受信

    キャッシュ確認 --> 自動承認: tool_function_auto_permissions に関数名あり
    キャッシュ確認 --> 自動承認: tool_auto_permissions に承認キーあり
    キャッシュ確認 --> ユーザー確認: キャッシュなし

    ユーザー確認 --> 拒否: [n] 実行しない
    ユーザー確認 --> 一回許可: [y] 今回のみ
    ユーザー確認 --> キーキャッシュ: [a] 常に許可（関数名 + 引数）
    ユーザー確認 --> 関数キャッシュ: [f] 常に許可（関数名のみ）

    キーキャッシュ --> 一回許可: approval_key を tool_auto_permissions に保存
    関数キャッシュ --> 一回許可: function_name を tool_function_auto_permissions に追加

    一回許可 --> 実行: available_functions[name] を呼び出し
    自動承認 --> 実行: available_functions[name] を呼び出し

    実行 --> 結果追加: ツール結果を messages に追加
    拒否 --> 結果追加: "実行スキップ" を messages に追加
    結果追加 --> [*]
```

**承認キーの形式:** `"{function_name}:{json_args_sorted_keys}"` (`python_code_interpreter.py:272`)

---

## 設定の優先順位

設定は以下の優先順位で解決されます（上位が優先）:

```mermaid
graph LR
    ENV["環境変数<br/>（最高優先度）"]
    CLI_ARG["CLI 引数<br/>--deployment-name"]
    CFG["~/.pycodei/config.json"]
    DEF["DEFAULT_CONFIG 辞書<br/>（最低優先度）"]

    ENV --> CLI_ARG --> CFG --> DEF
```

`PYCODEI_CONFIG_DIR` 環境変数で設定ディレクトリのデフォルト（`~/.pycodei`）を上書きできます。

---

## 実行時アーティファクト

`pycodei` を実行すると、カレントディレクトリに以下のファイルが生成されます:

```
./ （カレントディレクトリ）
├── notebooks/
│   └── {YYYYMMDDHHMMSS}/
│       └── notebook.ipynb          # セッション中にリアルタイム更新される Jupyter ノートブック
├── logs/
│   └── log_{YYYYMMDD}-{HHMMSS}.json  # LLM との完全なリクエスト/レスポンス記録
└── ai_workspace/                   # 生成コードが使う永続データストレージ
```

**ノートブック構造:** 会話メッセージは Markdown セルとして、Python コード実行はキャプチャされた出力付きの Code セルとして追記されます。

**ログファイル構造:**
```json
{
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "...", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "...", "name": "...", "content": "..."}
  ],
  "model": "deployment-name",
  "tools": [...]
}
```

ログファイルは `--load-message` で再ロードすることで、過去の会話を再開・継続できます（メモリ巻き戻し機能）。

---

## データ構造リファレンス

### メッセージ配列（OpenAI Chat 形式）

```python
messages = [
    {"role": "system", "content": "<システムプロンプト + PYCODEI.md>"},
    {"role": "user",   "content": "<ユーザーの指示>"},
    {
        "role": "assistant",
        "content": None,  # tool_calls がある場合は None
        "tool_calls": [
            {
                "id": "call_abc123",
                "type": "function",
                "function": {"name": "run_python", "arguments": "{\"python_code\": \"...\"}"}
            }
        ]
    },
    {
        "role": "tool",
        "tool_call_id": "call_abc123",
        "name": "run_python",
        "content": "<実行結果の文字列>"
    }
]
```

### ノートブック実行結果

`python_code_notebook.run_all()` はセルごとの結果リストを返します:

```python
[
    # セル 1 の結果
    [
        {"text/plain": "42", "output_type": "execute_result"},
        {"text/plain": "<Figure size 640x480>", "output_type": "display_data"},
    ],
    # セル 2 の結果
    [
        {"text/plain": "Hello, world!\n", "output_type": "stream"},
    ],
]
```

| `output_type` | 発生元 |
|---------------|-------|
| `execute_result` | セル最後の式の戻り値 |
| `display_data` | `matplotlib` 図・`IPython.display` オブジェクト |
| `stream` | `print()` の出力（stdout のみ。stderr は無視） |
| `error` | 例外のトレースバック（3行超の場合、最終フレーム + 例外行に省略） |
| `pycode_system_error` | Papermill レベルの例外（カーネルエラーではない） |
