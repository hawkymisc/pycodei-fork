# Configuration Reference

PYCODEI is configured via `~/.pycodei/config.json`. This document is the authoritative reference for every configuration key.

---

## Config File Location

| Location | How to Set |
|----------|-----------|
| `~/.pycodei/config.json` | Default location |
| `$PYCODEI_CONFIG_DIR/config.json` | Override via `PYCODEI_CONFIG_DIR` environment variable |

On first run (`pycodei --help`), if the file does not exist it is auto-created from the built-in defaults (see `DEFAULT_CONFIG` in `python_code_interpreter.py:41`). The run then exits with an error asking you to fill in credentials.

---

## Configuration Precedence

Settings are resolved in this priority order (highest wins):

1. **Environment variables** — any key in `config.json` can also be set as an env var with the same name (e.g., `DEPLOYMENT_NAME=gpt-4o pycodei`)
2. **`--deployment-name` CLI flag** — overrides `DEPLOYMENT_NAME` only
3. **`~/.pycodei/config.json`** — primary config file
4. **Built-in defaults** — hard-coded `DEFAULT_CONFIG` dict in the source

> **Implementation note:** `apply_config_to_env()` (`python_code_interpreter.py:91`) writes every non-dict, non-list config value into `os.environ` at startup. Values already in the environment before launch are **overwritten** by config values.

---

## Core Settings

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `DEPLOYMENT_NAME` | string | **Yes** | `"gpt-5-mini"` | Model or deployment name passed to the LLM API. For Azure this is the deployment name; for OpenAI this is the model ID (e.g., `"gpt-4o"`). |
| `PYCODEI_CLIENT` | string | **Yes** | `"azure"` | LLM provider. Accepted values: `"azure"`, `"openai"`. Case-insensitive. |
| `CONVERSATION_LOOP_MAX_CYCLES` | integer | No | `100` | Maximum number of ReAct loop iterations per session before forceful exit. |
| `Title` | string | No | `"PYCODEI"` | Text rendered as the ASCII banner on startup. |
| `TitleFont` | string | No | `"slant"` | [pyfiglet](https://github.com/pwaller/pyfiglet) font name for the banner. |

---

## Azure OpenAI Settings

Required when `PYCODEI_CLIENT` is `"azure"`.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `AZURE_OPENAI_API_KEY` | string | Yes | `""` | Azure OpenAI API key. |
| `AZURE_OPENAI_ENDPOINT` | string | Yes | placeholder | Azure endpoint URL, e.g., `"https://my-resource.openai.azure.com/"`. |
| `OPENAI_API_VERSION` | string | Yes | `"2024-10-01-preview"` | Azure OpenAI API version string. |

---

## OpenAI Settings

Required when `PYCODEI_CLIENT` is `"openai"`.

| Key | Type | Required | Default | Description |
|-----|------|----------|---------|-------------|
| `OPENAI_API_KEY` | string | Yes | `""` | OpenAI API key. |

---

## MCP Server Settings

`mcpServers` is a JSON object whose keys are user-chosen server names and whose values are server configuration objects. All MCP servers are optional.

```json
{
  "mcpServers": {
    "<server-name>": { ... }
  }
}
```

### Per-Server Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `disabled` | boolean | No | `false` | Set to `true` to skip this server without removing its config. |
| `transport` | string | No | `"stdio"` | Transport type. See [Transport Types](#transport-types) below. Aliases: `type` is also accepted. |
| `command` | string | Conditional | — | Executable to run. Required for `stdio` transport. Path-expanded if it looks like a path (`~`, `.`, `/`, `\`). |
| `args` | array of string | No | `[]` | Arguments passed to `command`. |
| `env` | object | No | `{}` | Extra environment variables for the server process. Values are coerced to strings. |
| `cwd` | string | No | — | Working directory for the server process. Relative paths are resolved against `~/.pycodei/`. |
| `url` | string | Conditional | — | Endpoint URL. Required for `sse`, `http`, `https`, `websocket`, `ws`, `wss` transports. |
| `headers` | object | No | `{}` | HTTP headers sent with SSE/WebSocket requests (e.g., `Authorization`). |
| `encoding` | string | No | `"utf-8"` | Character encoding for stdio communication. |
| `encoding_error_handler` | string | No | `"strict"` | Error handler for encoding issues. Also accepted as `encodingErrorHandler`. |

### Transport Types

| Value | Protocol | Required Fields |
|-------|----------|----------------|
| `"stdio"` (default) | Local subprocess via stdin/stdout | `command` |
| `"sse"` | HTTP Server-Sent Events | `url` |
| `"http"`, `"https"` | HTTP (alias for `sse`) | `url` |
| `"websocket"`, `"ws"`, `"wss"` | WebSocket | `url` |

---

## Complete Annotated Example

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

## PYCODEI.md — Agent Instruction File

In addition to `config.json`, PYCODEI supports an optional Markdown instruction file that extends the system prompt. This lets you add domain-specific context, safety guardrails, or project requirements without modifying source code.

### Search Order

`load_pycodei_guide()` (`python_code_interpreter.py:109`) searches these locations and uses the **first non-empty file** found:

1. `{current_working_directory}/PYCODEI.md`
2. `~/.pycodei/PYCODEI.md`
3. `{install_directory}/PYCODEI.md`

### Content Injection

The file content is appended verbatim to the base system prompt:

```
{base_system_content}

Additional instructions from PYCODEI.md:
{content of PYCODEI.md}
```

### Example

```markdown
## Project Context

This agent is working with a PostgreSQL database containing sales data.

### Schema
- `orders(id, customer_id, total, created_at)`
- `customers(id, name, email, region)`

### Rules
- Never execute DELETE or DROP statements without explicit user confirmation.
- Always output monetary values in USD with 2 decimal places.
```

---

## Prompt History

PYCODEI stores CLI prompt history at `~/.pycodei/prompt_history` (a `FileHistory` file used by `prompt_toolkit`). This file is not part of `config.json` and does not need to be configured.
