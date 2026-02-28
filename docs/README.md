# PYCODEI — Design Documentation

This directory contains design documentation intended for **external maintainers and contributors** of the PYCODEI project.

## What is PYCODEI?

PYCODEI is an AI-powered Python code interpreter and analytics agent. It combines a large language model (OpenAI or Azure OpenAI) with a stateful Jupyter notebook execution environment using the **ReAct** (Reasoning + Acting) loop pattern. Users interact with it through a CLI, directing the agent to perform data analysis, machine learning, visualization, and other computational tasks.

## Documents

| Document | Description |
|----------|-------------|
| [architecture.md](./architecture.md) | System architecture: components, data flow, class structure, ReAct loop, and tool approval system |
| [configuration-reference.md](./configuration-reference.md) | Complete reference for all configuration keys, environment variables, MCP server config, and the `PYCODEI.md` guide file |
| [mcp-integration.md](./mcp-integration.md) | How MCP (Model Context Protocol) servers are integrated: transports, tool discovery, execution flow, and adding new servers |
| [contributing.md](./contributing.md) | Development setup, coding conventions, testing approach, commit/PR guidelines, and common extension patterns |

## Quick Orientation

```
pycodei-fork/
├── python_code_interpreter.py   # Main CLI entry point and ReAct orchestrator
├── python_code_notebook.py      # Jupyter notebook creation and execution
├── mcp_client_manager.py        # MCP server discovery and tool execution
├── set_matplotlib_japanese_font.py  # Japanese font configuration helper
├── papermill_enhancement/       # Git submodule: enhanced Papermill fork
├── sample_data/                 # Example datasets for demos
├── sample_results/              # Pre-generated example notebooks
└── docs/                        # This documentation directory
```

The project entry point is the `pycodei` CLI command, registered via `pyproject.toml`. Start with [architecture.md](./architecture.md) for a comprehensive understanding of how the system works.
