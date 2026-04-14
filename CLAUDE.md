# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DeerFlow is a full-stack "super agent harness" that orchestrates sub-agents, memory, and sandboxes to execute complex tasks via extensible skills.

**Stack**:
- Backend: Python 3.12, LangGraph + FastAPI Gateway, sandbox/tool system, MCP integration
- Frontend: Next.js 16 + React 19 + TypeScript 5.8 + Tailwind CSS 4 + pnpm 10.26.2
- Local dev entrypoint: `make dev` starts all services on `http://localhost:2026`

**Architecture**:
- **LangGraph Server** (port 2024): Agent runtime and workflow execution (standard mode)
- **Gateway API** (port 8001): REST API for models, MCP, skills, memory, artifacts, uploads, agents, channels
- **Frontend** (port 3000): Next.js web interface
- **Nginx** (port 2026): Unified reverse proxy entry point

**Runtime Modes**:
- **Standard mode** (`make dev`): LangGraph Server handles agent execution. 4 processes total.
- **Gateway mode** (`make dev-pro`, experimental): Agent runtime embedded in Gateway via `RunManager` + `StreamBridge`. 3 processes, no LangGraph Server.

## Commands

### Root Directory (Full Application)

```bash
make check          # Check system requirements (Node 22+, pnpm, uv, nginx)
make install        # Install all dependencies (frontend + backend)
make config         # Generate config.yaml from template (first-time setup only)
make config-upgrade # Merge new fields from config.example.yaml into config.yaml
make setup          # Interactive setup wizard
make doctor         # Check config and system status
make dev            # Start all services (standard mode) with hot-reloading
make dev-pro        # Start all services (Gateway mode, experimental)
make stop           # Stop all running services
make clean          # Stop services and clean temporary files
# Docker commands:
make docker-start   # Start dev environment in Docker
make docker-start-pro # Start dev in Docker (Gateway mode)
make up             # Production Docker Compose deployment
make up-pro         # Production Docker Compose (Gateway mode)
```

### Backend Directory

```bash
cd backend
make install    # Install backend dependencies via uv
make dev        # Run LangGraph server only (port 2024)
make gateway    # Run Gateway API only (port 8001)
make test       # Run all backend tests (pytest)
make lint       # Lint with ruff
make format     # Format code with ruff
```

Run single test:
```bash
cd backend
PYTHONPATH=. uv run pytest tests/test_<feature>.py -v
```

### Frontend Directory

```bash
cd frontend
pnpm dev        # Dev server with Turbopack (port 3000)
pnpm build      # Production build (requires BETTER_AUTH_SECRET)
pnpm test       # Run unit tests with Vitest
pnpm lint       # ESLint
pnpm typecheck  # TypeScript check
```

**Note**: `pnpm build` fails without `BETTER_AUTH_SECRET`. Set it or use `SKIP_ENV_VALIDATION=1`.

## Architecture

### Harness / App Split (Backend)

The backend has two layers with strict dependency direction:

- **Harness** (`packages/harness/deerflow/`): Publishable agent framework. Import prefix: `deerflow.*`
- **App** (`app/`): Application code (Gateway API, IM channels). Import prefix: `app.*`

**Rule**: App imports deerflow, but deerflow NEVER imports app. Enforced by `tests/test_harness_boundary.py` in CI.

### Key Directories

```
deer-flow/
├── config.yaml              # Main application config (models, tools, sandbox, memory, channels)
├── extensions_config.json   # MCP servers and skills config
├── backend/
│   ├── packages/harness/deerflow/   # Agent framework (deerflow.* imports)
│   │   ├── agents/                  # Lead agent, middlewares, memory, thread state, checkpointer, factory
│   │   ├── sandbox/                 # Sandbox execution (local/docker), security, file locks, search tools
│   │   ├── subagents/               # Subagent delegation system, per-agent model config
│   │   ├── tools/                   # Built-in tools, ACP agent, tool search, setup agents
│   │   ├── mcp/                     # MCP integration with OAuth support
│   │   ├── skills/                  # Skills loading, management, security scanning, validation
│   │   ├── models/                  # Model factory, providers (OpenAI, Claude, vLLM, DeepSeek, etc.), credential loader
│   │   ├── guardrails/              # Guardrail middleware and providers (AllowlistProvider, OAP)
│   │   ├── runtime/                 # Gateway mode runtime (RunManager, StreamBridge, store)
│   │   ├── uploads/                 # Upload manager (PDF/PPT/Excel/Word conversion)
│   │   ├── tracing/                 # Tracing factory
│   │   ├── config/                  # Configuration modules (21 config schemas)
│   │   ├── community/               # Community tools (tavily, jina_ai, firecrawl, ddg_search, exa, infoquest, image_search, aio_sandbox)
│   │   └── client.py                # Embedded Python client (DeerFlowClient)
│   └── app/                         # Application layer (app.* imports)
│       ├── gateway/                 # FastAPI Gateway API (14 routers)
│       └── channels/                # IM integrations (Feishu, Slack, Telegram, Discord, WeChat, WeCom)
├── frontend/src/
│   ├── app/                 # Next.js routes (workspace/chats, workspace/agents, blog, [lang]/docs)
│   ├── components/          # UI components (ui/, ai-elements/, workspace/, landing/)
│   ├── content/             # Blog/docs MDX content (en/, zh/)
│   ├── core/                # Business logic (threads, api, agents, uploads, streamdown, ...)
│   └── typings/             # TypeScript type definitions
└── skills/                  # Agent skills (public/, custom/)
```

### Service Routing (Nginx)

Standard mode:
- `/api/langgraph/*` → LangGraph Server (2024)
- `/api/*` (other) → Gateway API (8001)
- `/` (non-API) → Frontend (3000)

Gateway mode:
- `/api/langgraph/*` → Gateway embedded runtime (8001)
- `/api/*` (other) → Gateway API (8001)
- `/` (non-API) → Frontend (3000)

## Development Workflow

### Test-Driven Development (TDD) — MANDATORY

Every new feature or bug fix MUST have unit tests. No exceptions.

```bash
cd backend && make test   # Run backend tests before committing
cd frontend && pnpm test  # Run frontend Vitest tests
```

### Pre-Commit Validation

Before submitting changes:

1. Backend: `cd backend && make lint && make test`
2. Frontend (if touched): `cd frontend && pnpm lint && pnpm typecheck`
3. Frontend build (if UI/auth changes): `BETTER_AUTH_SECRET=local-dev pnpm build`

### Configuration

- `config.yaml`: Models, sandbox, memory, tools, channels, guardrails, circuit breaker, checkpointer. Copy from `config.example.yaml`
- `extensions_config.json`: MCP servers and skills enabled state
- Values starting with `$` are resolved as environment variables (e.g., `$OPENAI_API_KEY`)
- `config_version` tracks schema version; bump in `config.example.yaml` when changing config schema
- Run `make config-upgrade` to merge missing fields from `config.example.yaml` into existing `config.yaml`

## Key Patterns

### Backend

- Middlewares execute in strict order (18 middlewares, see `backend/CLAUDE.md` for full list)
- Config caching with mtime-based auto-reload; `config_version` tracking with `make config-upgrade`
- Sandbox virtual paths: `/mnt/user-data/` → `backend/.deer-flow/threads/{thread_id}/user-data/`
- Sandbox tools: `bash`, `ls`, `read_file`, `write_file`, `str_replace`, `glob`, `grep`
- Circuit breaker prevents persistent LLM rate-limit bans (`circuit_breaker` config)
- Guardrails system with pluggable `GuardrailProvider` (AllowlistProvider, OAP policy providers)
- Per-subagent model override in `config.yaml` (`subagents.agents[].model_name`)
- Gateway mode (`make dev-pro`) embeds agent runtime in Gateway via `RunManager` + `StreamBridge`
- Sandbox security module gates bash execution; `file_operation_lock.py` uses `WeakValueDictionary` to prevent memory leaks

### Frontend

- Server Components by default, `"use client"` for interactive components
- Thread hooks (`useThreadStream`, `useSubmitThread`) are the primary API
- LangGraph client singleton via `getAPIClient()` in `core/api/`
- Vitest test infrastructure (`pnpm test`, tests under `tests/unit/`)
- Custom agents pages at `/workspace/agents/` with `core/agents/` API/hooks
- Blog and i18n docs at `/blog/` and `/[lang]/docs/`
- Streamdown (`core/streamdown/`) for streaming Markdown rendering

## Environment Requirements

- Node.js >=22
- pnpm 10.26.2+
- Python >=3.12
- uv (Python package manager)
- nginx (for unified local endpoint)

## Important Notes

- `make config` aborts if `config.yaml` already exists — use `make config-upgrade` to merge new fields
- `make setup` provides an interactive setup wizard for first-time configuration
- `make doctor` checks config and system status
- Proxy env vars can break frontend `pnpm install` — unset them if errors occur
- Frontend `pnpm check` is unreliable; use `pnpm lint && pnpm typecheck` instead
- In Docker Compose, IM channels use container service names (`http://langgraph:2024`)
- Gateway mode (`make dev-pro`) runs 3 processes instead of 4 (no LangGraph Server)
- Supported IM channels: Feishu, Slack, Telegram, Discord, WeChat, WeCom
- CLI authentication (`~/.claude`, `~/.codex`) auto-mounted in Docker for credential loader

## Detailed Documentation

- `backend/CLAUDE.md` — Backend architecture, middlewares, sandbox, subagents, memory
- `frontend/CLAUDE.md` — Frontend architecture, data flow, components, code style
- `.github/copilot-instructions.md` — Verified command sequences and validation flows
- `backend/docs/CONFIGURATION.md` — Configuration options
- `CONTRIBUTING.md` — Development setup and workflow