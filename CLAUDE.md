# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight personal AI assistant framework (~4,000 lines of core agent code). It supports 10+ chat channels, 20+ LLM providers, and a tool/skill system with MCP support. Published on PyPI as `nanobot-ai`.

## Development Commands

```bash
# Install (development)
pip install -e .
pip install -e ".[dev]"        # includes pytest, pytest-asyncio, ruff

# Run the agent (CLI mode)
nanobot agent

# Run onboarding setup
nanobot onboard

# Run tests
pytest                                              # all tests
pytest tests/test_cron_service.py                   # single file
pytest tests/test_commands.py::test_onboard_fresh_install  # single test

# Lint
ruff check nanobot/
ruff check --fix nanobot/      # auto-fix

# Count core lines
bash core_agent_lines.sh
```

## Architecture

**Message-driven design**: Channels receive messages → `MessageBus` (async queues) → `AgentLoop` processes with LLM → responses routed back through bus.

### Core Modules (`nanobot/`)

- **`agent/`** — Core agent loop, context building, memory, skills, subagents
  - `loop.py` — Main agent loop: receives messages, calls LLM with tools, executes tool calls
  - `context.py` — Builds system prompt + history + memory + skills for LLM calls
  - `memory.py` — Two-layer memory: `MEMORY.md` (long-term facts) + `HISTORY.md` (searchable log), with LLM-driven consolidation
  - `tools/` — Tool implementations: exec, read_file, write_file, edit_file, list_dir, message, web_search, web_fetch, cron, spawn (subagents), MCP
- **`bus/`** — `MessageBus` with async inbound/outbound queues decoupling channels from agent
- **`channels/`** — Chat platform adapters (Telegram, Discord, WhatsApp, Feishu, Slack, DingTalk, Email, QQ, Matrix, Mochat). Each extends `base.py`
- **`providers/`** — LLM provider adapters with registry-based auto-detection. Uses LiteLLM as unified interface. Add new providers via `registry.py`
- **`config/`** — Pydantic models with camelCase aliasing. Config lives at `~/.nanobot/config.json`
- **`session/`** — Conversation session management with message history
- **`cron/`** — Scheduled task system using croniter
- **`heartbeat/`** — Periodic wake-up tasks (virtual tool-call heartbeat for prompt cache optimization)
- **`skills/`** — YAML frontmatter + Markdown instruction files (weather, github, tmux, summarize, clawhub, cron, memory, skill-creator). Compatible with OpenClaw format
- **`cli/`** — typer-based CLI (`nanobot` command)
- **`templates/`** — Markdown templates for AGENTS.md, SOUL.md, USER.md, TOOLS.md, HEARTBEAT.md

### Bridge (`bridge/`)

Node.js/TypeScript WhatsApp bridge using whatsapp-web.js.

## Key Patterns

- **All async**: Core uses `asyncio` throughout. Tests use `pytest-asyncio` with `asyncio_mode = "auto"`
- **Registry pattern**: Providers and channels register themselves for auto-discovery
- **Pydantic config**: All configuration validated via Pydantic models with camelCase/snake_case aliasing
- **Tool system**: Abstract `Tool` base class with JSON schema validation. Tools registered in agent loop
- **Workspace**: User workspace at `~/.nanobot/workspace/` for skills, memory, and agent configuration

## Code Style

- Python 3.11+ with type hints throughout
- Ruff linter: rules E, F, I, N, W (E501 line-length ignored)
- Line length: 100 chars
- Commit format: `type(scope): description` (e.g., `fix(web): use self.api_key`)
