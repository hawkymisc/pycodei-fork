# Architecture

## Overview

PYCODEI is a CLI-based AI agent that uses the **ReAct** (Reasoning + Acting) pattern to perform data analysis, visualization, and computation. The user provides a natural language task; the agent iteratively calls an LLM, executes generated Python code in a persistent Jupyter kernel, and feeds results back to the LLM until the task is complete.

---

## Components

| Module | Lines | Responsibility |
|--------|-------|----------------|
| `python_code_interpreter.py` | 613 | CLI entry point, configuration loading, LLM client creation, ReAct conversation loop, tool approval system |
| `python_code_notebook.py` | 165 | Jupyter notebook creation and code execution via Papermill; result capture |
| `mcp_client_manager.py` | 416 | MCP server discovery, lifecycle management, and tool execution for all transports |
| `set_matplotlib_japanese_font.py` | 53 | One-time helper: configure matplotlib for Japanese character rendering |
| `papermill_enhancement/` | submodule | Fork of [nteract/papermill](https://github.com/nteract/papermill) with enhanced exception handling |

---

## Component Dependencies

```mermaid
graph TD
    CLI["python_code_interpreter.py<br/>CLI / Orchestrator"]
    NB["python_code_notebook.py<br/>Notebook Execution"]
    MCP["mcp_client_manager.py<br/>MCP Tool Manager"]
    PM["papermill_enhancement/<br/>Papermill Fork"]
    LLM["LLM API<br/>OpenAI / Azure OpenAI"]
    MCPSRV["External MCP Servers<br/>stdio / SSE / WebSocket"]
    CFG["~/.pycodei/config.json<br/>Configuration"]
    GUIDE["PYCODEI.md<br/>Agent Instructions"]

    CLI -->|"calls run_all()"| NB
    CLI -->|"get_openai_tools()"| MCP
    CLI -->|"chat.completions.create()"| LLM
    CLI -->|"reads"| CFG
    CLI -->|"reads"| GUIDE
    NB -->|"execute_notebook()"| PM
    MCP -->|"stdio/SSE/WS"| MCPSRV
```

---

## ReAct Conversation Loop

The core of PYCODEI is `PythonCodeInterpreter.run_conversation()` (`python_code_interpreter.py:389`). It runs for up to `CONVERSATION_LOOP_MAX_CYCLES` iterations (default: 100).

```mermaid
sequenceDiagram
    actor User
    participant CLI as python_code_interpreter.py
    participant LLM as LLM API
    participant NB as python_code_notebook.py
    participant MCP as mcp_client_manager.py

    User->>CLI: pycodei "Analyze ./data.csv"
    CLI->>CLI: Load config, build system prompt, register tools
    CLI->>LLM: chat.completions.create(messages, tools)

    loop ReAct Loop (up to 100 iterations)
        LLM-->>CLI: finish_reason = "tool_calls"
        CLI->>User: Show tool details + approval prompt
        alt Approved
            alt run_python tool
                CLI->>NB: run_all(python_code, messages)
                NB-->>CLI: execution results
            else MCP tool
                CLI->>MCP: callable(arguments, messages)
                MCP-->>CLI: JSON result
            end
            CLI->>LLM: Append tool result, call again
        else Denied
            CLI->>LLM: Append "execution skipped" result
        end

        LLM-->>CLI: finish_reason = "stop"
        CLI->>NB: write_messages_in_notebook (narrative only)
        CLI->>User: Print LLM response
        User->>CLI: Follow-up message (or "exit")
        CLI->>LLM: Append user message, call again
    end

    CLI->>CLI: Save logs/{result_name}.json
```

**Loop exit conditions:**
- User types `exit`
- `finish_flag` is set after a `stop` response and user exits
- `max_loops` iterations reached
- LLM API error

---

## Startup Sequence

```mermaid
sequenceDiagram
    participant main as main()
    participant CLI as PythonCodeInterpreter
    participant CFG as config.json
    participant MCP as MCPClientManager

    main->>CFG: load_user_config()
    CFG-->>main: config dict (or creates template + exits)
    main->>main: apply_config_to_env(config)
    main->>main: MCPClientManager(config["mcpServers"])
    main->>CLI: PythonCodeInterpreter(deployment_name)
    CLI->>CLI: create_llm_client() → AzureOpenAI or OpenAI
    CLI->>CLI: Build system prompt + load PYCODEI.md
    CLI->>CLI: Register run_python tool
    CLI->>MCP: get_openai_tools() [lazy: builds cache on first call]
    MCP-->>CLI: (mcp_tools, mcp_function_map)
    CLI->>CLI: Merge MCP tools into self.tools
    main->>CLI: initialize_messages(messages)
    main->>CLI: initialize_notebook(messages)
    main->>CLI: run_conversation()
```

---

## Class Structure

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

    PythonCodeInterpreter --> MCPClientManager : uses
    MCPClientManager --> MCPServerConfig : owns
    MCPClientManager --> MCPToolBinding : creates
    MCPToolBinding --> MCPServerConfig : references
```

---

## Tool Approval System

Every LLM-requested tool call goes through an approval gate before execution (`python_code_interpreter.py:447`). Approvals are cached in-process to avoid repeated prompts.

```mermaid
stateDiagram-v2
    [*] --> CheckCache: Tool call received from LLM

    CheckCache --> AutoApproved: function in tool_function_auto_permissions
    CheckCache --> AutoApproved: approval_key in tool_auto_permissions
    CheckCache --> PromptUser: No cached permission

    PromptUser --> Deny: User chooses [n]
    PromptUser --> AllowOnce: User chooses [y]
    PromptUser --> CacheKey: User chooses [a] always function+args
    PromptUser --> CacheFunction: User chooses [f] always function

    CacheKey --> AllowOnce: Store approval_key → tool_auto_permissions
    CacheFunction --> AllowOnce: Add function_name → tool_function_auto_permissions

    AllowOnce --> Execute: call available_functions[name]
    AutoApproved --> Execute: call available_functions[name]

    Execute --> AppendResult: Append tool result to messages
    Deny --> AppendResult: Append "execution skipped"
    AppendResult --> [*]
```

**Approval key format:** `"{function_name}:{json_args_sorted_keys}"` (`python_code_interpreter.py:272`)

---

## Configuration Layering

Settings are resolved with this priority order (highest to lowest):

```mermaid
graph LR
    ENV["Environment Variables<br/>(highest priority)"]
    CLI_ARG["CLI Arguments<br/>--deployment-name"]
    CFG["~/.pycodei/config.json"]
    DEF["DEFAULT_CONFIG dict<br/>(lowest priority)"]

    ENV --> CLI_ARG --> CFG --> DEF
```

The `PYCODEI_CONFIG_DIR` environment variable overrides the default config directory (`~/.pycodei`).

---

## Runtime Artifacts

Running `pycodei` generates these files in the current working directory:

```
./ (current working directory)
├── notebooks/
│   └── {YYYYMMDDHHMMSS}/
│       └── notebook.ipynb          # Live Jupyter notebook updated during session
├── logs/
│   └── log_{YYYYMMDD}-{HHMMSS}.json  # Full API request/response transcript
└── ai_workspace/                   # Persistent data storage for generated code
```

**Notebook structure:** Conversation messages are appended as Markdown cells; Python code executions are appended as Code cells with captured outputs.

**Log file structure:**
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

Log files can be reloaded with `--load-message` to resume or extend a previous conversation (memory rewind).

---

## Data Structures

### Messages Array (OpenAI Chat Format)

```python
messages = [
    {"role": "system", "content": "<system prompt + PYCODEI.md>"},
    {"role": "user",   "content": "<user prompt>"},
    {
        "role": "assistant",
        "content": None,  # None when tool_calls present
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
        "content": "<execution result string>"
    }
]
```

### Tool Definition (OpenAI Function Calling Format)

```python
{
    "type": "function",
    "function": {
        "name": "run_python",
        "description": "...",
        "parameters": {
            "type": "object",
            "properties": {
                "python_code": {"type": "string", "description": "..."}
            },
            "required": ["python_code"]
        }
    }
}
```

### Notebook Execution Result

`python_code_notebook.run_all()` returns a list of per-cell result lists:

```python
[
    # Cell 1 results
    [
        {"text/plain": "42", "output_type": "execute_result"},
        {"text/plain": "<Figure size 640x480>", "output_type": "display_data"},
    ],
    # Cell 2 results
    [
        {"text/plain": "Hello, world!\n", "output_type": "stream"},
    ],
]
```

| `output_type` | Source |
|---------------|--------|
| `execute_result` | Last expression value in a cell |
| `display_data` | `matplotlib` figures, `IPython.display` objects |
| `stream` | `print()` output (stdout only; stderr is silently dropped) |
| `error` | Exception traceback (abbreviated to last frame + exception line if > 3 lines) |
| `pycode_system_error` | Papermill-level exception (not a kernel error) |
