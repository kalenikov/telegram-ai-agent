# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository discipline

Follow `AGENTS.md`. This is a **public, generic** Telegram bot runtime, not a
personal assistant repository. Do not copy private prompts, private project
knowledge, local runtime configs, secrets, real IDs/tokens, or machine-specific
deployment files into this repo. Keep edits scoped; inspect the relevant files
first.

Skills carry the operational context (read these instead of guessing):

- `.claude/skills/project-knowledge/SKILL.md` — architecture, release safety.
- `.claude/skills/bot-setup/SKILL.md` — install, systemd autostart, troubleshooting.
- `.claude/skills/topic-setup/SKILL.md` — Telegram forum topic wiring.

## Commands

```bash
uv sync                                              # install deps (Python 3.12+)
uv run telegram-bot                                  # run the bot foreground (reads .env)
uv run pytest                                        # full test suite
uv run pytest tests/test_public_runtime.py::test_public_settings_default_cwd_is_generic  # single test
uv run ruff check .                                  # lint
uv run ruff format --check .                         # format check
uv run mypy src/ mcp-servers/bot/server.py           # strict type check
```

Run all four checks (ruff check, ruff format --check, mypy, pytest) before final
handoff. mypy runs in `strict` mode; tests use `pytest-asyncio` in auto mode.

**Production (systemd):**
```bash
sudo systemctl start telegram-bot      # start
sudo systemctl stop telegram-bot       # stop
sudo systemctl restart telegram-bot    # restart after config/code changes
sudo systemctl status telegram-bot     # check status
journalctl -u telegram-bot -f          # follow logs
```

The unit file lives at `/etc/systemd/system/telegram-bot.service` (generated from
`docs/systemd/telegram-bot.service.template`). `KillMode=process` lets tmux sessions
survive a bot restart.

**Proxy:** set `PROXY_URL=http://127.0.0.1:10808` in `.env` to route all Telegram API
traffic through a local HTTP proxy (Xray/tinyproxy). Requires `aiohttp-socks` (already
in dependencies). Leave empty to connect directly.

## Architecture

A single aiogram bot process turns Telegram into a remote control surface for
agent CLIs (Claude Code / Codex) running on the same machine. Telegram is only
the front end; the agent and its working directories live locally.

**Entry point** (`src/telegram_bot/__main__.py`) wires everything: it builds the
aiogram `Dispatcher`, installs `AuthMiddleware` (allowlist by Telegram user ID),
registers routers in a deliberate order, and injects long-lived services
(`SessionManager`, `TmuxManager`, `TopicConfig`, `MessageQueue`, `Transcriber`,
`ForwardBatcher`) into the dispatcher context. Router order matters and is
documented inline — `forum_topic` first (updates topic config before any handler
reads it), then `commands → cancel → mode → forward → voice → photo → text`;
forward runs before voice/photo so batched media isn't handled twice.

**Two independent axes** define how a message is executed (do not conflate them):

- **engine** (`claude` | `codex`) — *which* agent CLI. Abstracted behind provider
  adapters in `core/services/providers.py` (command construction, event parsing,
  availability detection / fallback).
- **exec_mode** (`subprocess` | `tmux`) — *how* it runs. `subprocess` is a
  one-shot `claude -p` per message (zero warm-up); `tmux` is a persistent TUI
  session that survives bot restarts and supports the live `/tui` interface.

**Per-topic configuration** is the core abstraction. A `ChannelKey` is
`(chat_id, thread_id)` (`core/types.py`); each Telegram forum topic maps to its
own `TopicSettings` — `cwd`, `mode` (prompt mode), `engine`, `exec_mode`,
`stream_mode`, `mcp_config`, `model`. `TopicConfig` (`core/services/topic_config.py`)
reads `topic_config.json` with mtime-based caching; prompt modes resolve to
`src/telegram_bot/prompts/<mode>.md`, scanned lazily so new modes drop in without
a restart.

**Request flow**: a handler in `core/handlers/` validates and normalizes input
(downloading media, transcribing voice via Deepgram, batching forwarded
messages), then funnels through `handlers/_dispatch.py::enqueue_prompt` into the
`MessageQueue` (serializes per-channel work). The queue calls back into
`__main__.process_queue_item`, which handles session switching and delegates to
`handlers/streaming.py::send_streaming_response`. Streaming reads `stream_mode`
to decide how intermediate agent events surface in Telegram: `verbose` (one
message per event), `live` (one editable batched "thinking" message), or
`minimal` (placeholder + final results only).

**Session lifecycle**: `SessionManager` (`core/services/claude.py`) owns
subprocess agent runs, the session-id mapping (`session_mapping.json`, enabling
reply-to-resume and `/resume`), prompt-mode/tool gating, and timeouts.
`TmuxManager` (`core/services/tmux_manager.py`) is a facade over a cluster of
`tmux_*` helpers managing persistent TUI sessions: spawn, atomic JSON state,
modal-detection watchdog (alerts when the agent is waiting on input), and
startup recovery (`restore_all` / resume tails). The `core/tui/` package handles
terminal capture, modal detection, and key-sending for the `/tui` snapshot UI.
All tmux subprocess calls use list-args (never `shell=True`) and async paths
wrap blocking calls in `asyncio.to_thread`.

**Bot MCP server** (`mcp-servers/bot/server.py`) is a separate stdio MCP process
launched by the agent (via `.mcp.bot.json`). It lets the agent send messages,
images, and documents *back* to Telegram, routed by `TELEGRAM_CHAT_ID` /
`TELEGRAM_THREAD_ID` env vars. It is type-checked alongside `src/`.

**Config & i18n**: `core/config.py` loads env via pydantic-settings (cached
`get_settings()`). User-facing strings go through `t()` in `core/messages.py`
(en/ru); Cyrillic there and in the TUI glyph files is intentional and ruff-exempted
per `pyproject.toml`.
