# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**nanobot** is an ultra-lightweight personal AI assistant framework (~4,000 lines of core agent code) built on Python 3.11+. It's a 99% smaller alternative to Clawdbot while delivering full agent functionality: tool use, multi-channel chat, memory, skills, scheduled tasks, and subagent spawning.

Package name: `nanobot-ai` (PyPI). CLI entry point: `nanobot`.

## Build & Development Commands

```bash
# Install for development
pip install -e .
# Or with dev dependencies
pip install -e ".[dev]"

# Run interactive chat
nanobot agent

# Run one-off message
nanobot agent -m "Hello!"

# Start multi-channel gateway
nanobot gateway

# Initialize config/workspace
nanobot onboard

# Show status
nanobot status
```

### Testing

```bash
# Run all tests
pytest

# Run a single test file
pytest tests/test_tool_validation.py

# Run a specific test
pytest tests/test_tool_validation.py::test_name -v
```

Tests use `pytest-asyncio` with `asyncio_mode = "auto"` — async test functions are automatically detected.

### Linting

```bash
# Check
ruff check nanobot/

# Fix auto-fixable issues
ruff check --fix nanobot/

# Format
ruff format nanobot/
```

Ruff config: line-length 100, target py311, rules `E F I N W` (ignores E501).

## Architecture

### Message Flow

```
Channels (Telegram/Discord/Slack/...) → MessageBus (async queues) → AgentLoop → LLM + Tools → MessageBus → Channels
```

The **MessageBus** (`bus/queue.py`) decouples channels from the agent via two async queues: inbound (channels→agent) and outbound (agent→channels). All channel implementations inherit `BaseChannel` and publish/subscribe through the bus.

### Core Agent Loop (`agent/loop.py`)

The `AgentLoop` is the processing engine:
1. Consumes `InboundMessage` from the bus
2. Loads/creates session (keyed by `{channel}:{chat_id}`)
3. Builds context via `ContextBuilder` (system prompt + memory + skills + history)
4. Iteratively calls the LLM and executes tool calls (max 20 iterations per message)
5. Publishes `OutboundMessage` back to the bus
6. Consolidates memory when session exceeds `memory_window` (default 50 messages)

`process_direct()` allows CLI and cron to invoke the agent without going through the bus.

### Tool System (`agent/tools/`)

Tools inherit from abstract `Tool` base class with `name`, `description`, `parameters` (JSON Schema). The `ToolRegistry` validates parameters against schema and converts tools to OpenAI function calling format. Built-in tools:
- `filesystem.py` — read_file, write_file, edit_file, list_dir
- `shell.py` — exec (with dangerous command guards)
- `web.py` — web_search (Brave API), web_fetch (HTTP + readability)
- `message.py` — send message to channel
- `spawn.py` — launch background subagent
- `cron.py` — schedule/list/remove cron jobs
- `mcp.py` — Model Context Protocol server integration

### Skills System (`agent/skills.py`)

Skills are markdown files (`SKILL.md`) with YAML frontmatter in directories under `nanobot/skills/` (built-in) or `~/.nanobot/workspace/skills/` (user). Workspace skills override built-in ones. Skills can declare requirements (`requires.bins`, `requires.env`) that are validated at load time. Always-loaded skills appear in full in the system prompt; others show metadata only and are lazy-loaded via read_file.

### Provider System (`providers/`)

LLM providers go through `LiteLLMProvider` which wraps LiteLLM for multi-provider support. The `ProviderRegistry` (`providers/registry.py`) is the single source of truth — each provider is a `ProviderSpec` with keywords, env vars, LiteLLM prefix, and detection rules. To add a new provider: add a `ProviderSpec` to `PROVIDERS` in `registry.py` and a field to `ProvidersConfig` in `config/schema.py`.

### Channels (`channels/`)

Each channel (Telegram, Discord, Slack, WhatsApp, Feishu, DingTalk, Email, QQ, Mochat) implements `BaseChannel` with async `start()`, `stop()`, and message handling. `ChannelManager` coordinates startup of all enabled channels. Most use WebSocket/long-polling; Email uses IMAP polling.

### Session & Memory

- Sessions stored as append-only JSONL in `~/.nanobot/sessions/`
- Memory consolidation: when history exceeds window, old messages are summarized by the LLM into `MEMORY.md` (facts) and `HISTORY.md` (events) in the workspace
- `MemoryStore` (`agent/memory.py`) handles read/write of memory files

### Subagents (`agent/subagent.py`)

`SubagentManager` spawns isolated background asyncio tasks with limited tools (file ops, exec, web — no message/spawn to prevent recursion). Results are published back to the origin channel.

### Configuration (`config/`)

Pydantic v2 models in `schema.py` define the full config structure. Config lives at `~/.nanobot/config.json` (camelCase JSON keys). Workspace files (SOUL.md, AGENTS.md, USER.md, TOOLS.md, HEARTBEAT.md) are in `~/.nanobot/workspace/`.

## Key Conventions

- **Async throughout** — all I/O uses async/await
- **OpenAI-compatible message format** — role/content dicts with tool_calls
- **Session keys** — `{channel}:{chat_id}` format
- **Logging** — loguru (`from loguru import logger`)
- **Config field naming** — camelCase in JSON, snake_case in Python (Pydantic aliases handle conversion)
- **No if-elif chains for providers** — everything goes through the registry pattern

## WhatsApp Bridge

The `bridge/` directory contains a TypeScript/Node.js WhatsApp bridge using Baileys. It communicates with the Python side via WebSocket. Requires Node.js >= 18.
