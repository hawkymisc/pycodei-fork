# pycodei スラッシュコマンド・状態管理 実装方針の調査

調査日: 2026-02-28

## 現状の把握

### スラッシュコマンド

**未実装。** ユーザー入力は生テキストとしてそのままLLMに渡されるだけ
（`python_code_interpreter.py` L495–505）。唯一の特殊コマンドは `exit` のみ。

### 状態管理

最小限のもののみ存在。

| 変数 | 内容 |
|---|---|
| `self.messages` | 会話履歴 |
| `self.tool_auto_permissions` | ツール承認記憶（関数名+引数単位） |
| `self.tool_function_auto_permissions` | ツール承認記憶（関数名単位） |
| `self.current_messages_index` | メッセージリスト内の現在位置 |

CLIフラグ `--load-message` 経由のメモリリワインドのみサポート。

---

## 既存コードの主な技術的負債

### 1. モジュールレベルの副作用（L126–128）

```python
# ファイルをインポートした瞬間に設定ファイル読み込みとMCP起動が走る
CONFIG = initialize_configuration()
deployment_name = os.getenv("DEPLOYMENT_NAME")
MCP_MANAGER = MCPClientManager(CONFIG.get("mcpServers"), base_dir=CONFIG_DIR)
```

テストが書けない、DI（依存性注入）が不可能、モジュール単体テストで副作用が発生する。

### 2. モノリシックな `run_conversation()`（L389–528、約140行）

ループ管理・入力処理・ツール実行・ログ保存・トークン集計が1メソッドに混在。
ここにスラッシュコマンドのdispatchを追加するほど改修コストが上がる。

### 3. テストハーネスが存在しない

`AGENTS.md` L22 に明記:
> There is no formal test harness yet—verification relies on running realistic prompts

### 4. LLMクライアントがOpenAI/Azure固定

`create_llm_client()` は `AzureOpenAI` または `OpenAI` のみをサポート。
他LLMへの切り替えには設計変更が必要。

---

## 選択肢の比較

| 観点 | 拡張（スラッシュ+状態管理実装） | ゼロから構築 |
|---|---|---|
| **初期速度** | 速い（既存インフラ活用） | 遅い（全部再実装） |
| **改修速度（中長期）** | 中〜低（既存の負債が足を引く） | 高（設計から綺麗にできる） |
| **技術的負債** | 既存の負債が残る＋増える | ゼロスタート |
| **活かせる既存資産** | Notebookインテグレーション、MCP、ツール承認UI | 参照・移植する形で取り込める |
| **テスタビリティ** | 低（テストハーネスなし、モジュールレベル副作用） | 設計次第で高くできる |
| **スラッシュコマンドの自然な追加箇所** | L495周辺にif分岐を差し込む（局所的） | 設計段階から組み込み可能 |

---

## 推奨: ゼロから構築

### 理由

1. **テストなしの機能追加は負債の純増** — `run_conversation()` に機能を積み上げるほど後の改修が辛くなる
2. **モノリシック構造の改修コストとゼロ構築のコストが大差ない** — どうせ `run_conversation()` を分割するなら、設計から直す方が早い
3. **スラッシュコマンド・状態管理は設計段階から組み込む方が結果的に速い**
4. **LLM切り替えの自由度** — Claudeを含む任意のLLMに対応できる設計にできる

### 活かすべき既存の知見

ゼロ構築でも以下はそのまま移植・参照する。

- **`python_code_notebook.py`** — Papermillを使ったNotebook実行の仕組み
- **`mcp_client_manager.py`** — MCPサーバー管理
- **`_prompt_tool_execution()` のUX設計** — y/a/f/nのツール承認インタラクション

---

## 例外: 既存拡張が有利なケース

スラッシュコマンドが `/exit`、`/load` 程度の**1〜2個で済み**、
かつ**中長期の改修予定がない**場合は、`run_conversation()` への局所的な追加で十分。

```python
# run_conversation() L495 付近への最小限の追加イメージ
user_input = prompt(...)
if user_input.startswith("/"):
    result = self._handle_slash_command(user_input)
    # ...
```
