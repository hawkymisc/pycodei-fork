# Contributing Guide

This document covers the development setup, coding conventions, testing approach, and common extension patterns for PYCODEI maintainers.

---

## Development Setup

### Prerequisites

- Python >= 3.12
- `pip`
- An active OpenAI or Azure OpenAI account with a deployed model
- (Optional) Node.js, if you want to test stdio MCP servers like `@playwright/mcp`

### Clone and Install

```bash
git clone https://github.com/KentaroAOKI/pycodei.git
cd pycodei

# Initialize the Papermill submodule
git submodule update --init --recursive

# Install in editable mode — registers the `pycodei` console script
pip install -e .
```

### Configure

```bash
# Create the config directory and let PYCODEI write a template
pycodei --help
# → Error: Created a config template at ~/.pycodei/config.json. Update it with...

# Edit the template
nano ~/.pycodei/config.json
```

Populate at minimum:
```json
{
  "DEPLOYMENT_NAME": "gpt-4o",
  "PYCODEI_CLIENT": "openai",
  "OPENAI_API_KEY": "sk-..."
}
```

See [configuration-reference.md](./configuration-reference.md) for all options.

### Verify

```bash
pycodei --version
# pycodei 0.1.9

pycodei "print('hello from PYCODEI')"
# Runs the ReAct loop with a trivial Python task
```

---

## Repository Structure

```
pycodei-fork/
├── python_code_interpreter.py   # Main CLI + ReAct loop + config + tool approval
├── python_code_notebook.py      # Jupyter notebook creation and code execution
├── mcp_client_manager.py        # MCP server lifecycle and tool dispatch
├── set_matplotlib_japanese_font.py  # One-shot Japanese font setup
├── papermill_enhancement/       # Git submodule: Papermill fork with better exceptions
├── pyproject.toml               # Package metadata, dependencies, entry point
├── requirements.txt             # Direct dependency list (used for dev installs)
├── README.md                    # User-facing README
├── AGENTS.md                    # Short developer guidelines (source of truth for style)
├── .env.sample                  # Sample environment variable template
├── sample_data/                 # Example input datasets
│   └── diagnosis.csv            # Breast cancer features dataset
├── sample_results/              # Pre-generated reference notebooks
│   ├── sample_01.ipynb          # Stock price prediction (English)
│   ├── sample_02.ipynb          # Cancer classification (English)
│   └── ...
└── docs/                        # This documentation directory
```

**Runtime directories (git-ignored):**

```
ai_workspace/    # Persistent data storage for generated code
notebooks/       # Timestamped Jupyter notebooks from each session
logs/            # JSON conversation transcripts
```

---

## Coding Conventions

Follow the conventions established in `AGENTS.md` and the existing source code:

- **Style:** PEP 8 — 4-space indentation, snake_case for functions and variables, CapWords for classes
- **Imports:** Explicit only. No wildcard imports (`from x import *`). Standard library first, then third-party, then local modules.
- **Constants:** Place path prefixes and config constants near the top of the module. Wrap env lookups with `os.getenv()`.
- **Config keys:** Any new key added to `DEFAULT_CONFIG` must be documented in [configuration-reference.md](./configuration-reference.md).
- **Artifact naming:** Maintain the existing timestamp formats:
  - Notebook dirs: `YYYYMMDDHHMMSS` (no separators)
  - Log files: `log_YYYYMMDD-HHMMSS.json` (hyphen separator)
- **Logging:** New modules should use `logging.getLogger("pycodei.<module>")`. Existing code uses `print()` for user-facing output — maintain this pattern in `python_code_interpreter.py`.
- **Error handling:**
  - Config errors: raise `RuntimeError`, let `initialize_configuration()` convert to `SystemExit`
  - Tool errors: return an error string to the LLM (do not raise), so the agent can recover
  - MCP server errors: `logger.warning(...)` and skip the server

---

## Testing Approach

PYCODEI has no formal test harness (see `AGENTS.md`). Verification relies on:

1. **Run a realistic prompt** through the interpreter:
   ```bash
   pycodei "Analyze ./sample_data/diagnosis.csv and determine which features are most predictive of malignancy"
   ```

2. **Review the console output** for:
   - Correct tool invocation traces
   - Expected function arguments
   - Token usage counters incrementing
   - No unexpected errors

3. **Inspect the generated notebook** (`notebooks/{timestamp}/notebook.ipynb`) in VS Code or Jupyter Lab:
   - Check that plots render correctly
   - Verify that error cells show expected output when testing error handling
   - Confirm that the narrative markdown cells reflect the conversation

4. **Check the conversation log** (`logs/log_{timestamp}.json`):
   - All message roles are present and correctly structured
   - Tool call IDs match tool results

5. **For regressions:** Add a prompt + expected output pair to `sample_results/` so reviewers can compare.

### Useful Test Prompts

| Scenario | Prompt |
|----------|--------|
| Basic execution | `"Print the first 10 prime numbers"` |
| Data loading | `"Load ./sample_data/diagnosis.csv and show the first 5 rows"` |
| Visualization | `"Create a histogram of sepal lengths from the iris dataset"` |
| Multi-step | `"Load iris data, train a classifier, and report accuracy"` |
| Error recovery | `"import nonexistent_library"` (expect the LLM to handle the ImportError) |
| Memory rewind | Run one session, then: `pycodei "Continue the previous analysis" --load-message logs/log_*.json` |

---

## Common Extension Patterns

### Adding a New Built-in Tool

Built-in tools are registered in `PythonCodeInterpreter.__init__()` (`python_code_interpreter.py:203`).

1. Add an OpenAI tool spec to `self.tools`:
   ```python
   self.tools.append({
       "type": "function",
       "function": {
           "name": "my_new_tool",
           "description": "What this tool does.",
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

2. Add a callable to `self.available_functions`:
   ```python
   self.available_functions["my_new_tool"] = self._my_new_tool_impl
   ```

3. Implement the method. The signature must match:
   ```python
   def _my_new_tool_impl(self, function_arguments: str, messages: list) -> str:
       args = json.loads(function_arguments)
       # ... do work ...
       return "result string sent to LLM"
   ```

4. `_register_tool_descriptions()` is called automatically for tools in `self.tools` — no extra step needed.

### Adding a New MCP Server

See [mcp-integration.md](./mcp-integration.md#adding-a-new-mcp-server) for the full walkthrough. Summary:

1. Add an entry under `mcpServers` in `~/.pycodei/config.json`
2. Restart PYCODEI — discovery runs automatically at startup

### Supporting a New LLM Provider

The LLM client is created in `create_llm_client()` (`python_code_interpreter.py:139`). Any provider whose SDK is compatible with the OpenAI Python SDK interface (`client.chat.completions.create(model, messages, tools)`) can be added:

1. Add a new `elif provider == "my_provider":` branch in `create_llm_client()`
2. Add the provider name to the accepted values in `resolve_client_provider()`
3. Add the new `PYCODEI_CLIENT` value and any new credential keys to `DEFAULT_CONFIG`
4. Document the new keys in [configuration-reference.md](./configuration-reference.md)

### Customizing the System Prompt

Two approaches:

- **Runtime (no code change):** Create `~/.pycodei/PYCODEI.md` or `./PYCODEI.md`. Content is appended to the base system prompt. See [configuration-reference.md](./configuration-reference.md#pycodeimd--agent-instruction-file).
- **Structural (code change):** Edit `base_system_content` in `PythonCodeInterpreter.__init__()` (`python_code_interpreter.py:166`).

### Adding a New Configuration Key

1. Add the key with its default value to `DEFAULT_CONFIG` in `python_code_interpreter.py:41`
2. Read it via `os.getenv("MY_KEY")` in the relevant code (it is written to env by `apply_config_to_env()`)
3. Document it in [configuration-reference.md](./configuration-reference.md)
4. Mention the new key in the PR description (per `AGENTS.md`)

---

## Commit and PR Guidelines

From `AGENTS.md`:

- **Commit messages:** Short imperative summaries, e.g., `"add tool approval"`, `"feature: initialize_notebook"`, `"fix: handle empty MCP tool list"`
- **PR description must include:**
  - Goal-oriented description of the change
  - Any new/changed `config.json` keys
  - Reproduction steps: `pycodei "<your-test-prompt>"`
  - Screenshots or notebook snippets if UI or output changes
  - Link to related issues
  - Note any breaking changes or manual migration steps

- **Do not commit:**
  - Personal notebooks (generated runtime files)
  - API keys or credentials
  - Files in `ai_workspace/`, `notebooks/`, `logs/` (all git-ignored)
  - Large binary files

---

## The `papermill_enhancement` Submodule

`python_code_notebook.py` imports from `papermill_enhancement.papermill` (line 4), a custom fork of [nteract/papermill](https://github.com/nteract/papermill) with enhanced exception handling.

**Initialize it:**
```bash
git submodule update --init --recursive
```

**If the submodule is not initialized**, PYCODEI will fail at startup with `ModuleNotFoundError: No module named 'papermill_enhancement'`.

**Fallback to upstream Papermill:** If you cannot initialize the submodule, change line 4 of `python_code_notebook.py`:
```python
# From:
import papermill_enhancement.papermill as pm
# To:
import papermill as pm
```
The upstream `papermill==2.6.0` is installed via `requirements.txt`. Exception handling quality may degrade slightly.

**Updating the submodule:**
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

## Useful Reference Points in the Source

| Topic | File | Lines |
|-------|------|-------|
| Default config values | `python_code_interpreter.py` | 41–52 |
| Config loading | `python_code_interpreter.py` | 67–106 |
| System prompt construction | `python_code_interpreter.py` | 166–202 |
| Tool registration | `python_code_interpreter.py` | 203–235 |
| Tool approval gate | `python_code_interpreter.py` | 447–471 |
| ReAct loop | `python_code_interpreter.py` | 389–528 |
| CLI argument parsing | `python_code_interpreter.py` | 530–609 |
| Notebook execution | `python_code_notebook.py` | 53–143 |
| Traceback abbreviation | `python_code_notebook.py` | 31–50 |
| MCP server config parsing | `mcp_client_manager.py` | 151–216 |
| MCP tool discovery | `mcp_client_manager.py` | 218–246 |
| MCP tool name generation | `mcp_client_manager.py` | 248–269 |
| MCP transport dispatch | `mcp_client_manager.py` | 393–416 |
| MCP tool result format | `mcp_client_manager.py` | 328–338 |
