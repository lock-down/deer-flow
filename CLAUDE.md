# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DeerFlow is a full-stack "super agent harness" that orchestrates sub-agents, memory, and sandboxes to execute complex tasks via extensible skills.

**Stack**:
- Backend: Python 3.12, LangGraph + FastAPI Gateway, sandbox/tool system, MCP integration
- Frontend: Next.js 16 + React 19 + TypeScript 5.8 + Tailwind CSS 4 + pnpm 10.26.2
- Local dev entrypoint: `make dev` starts all services on `http://localhost:2026`

**Architecture**:
- **LangGraph Server** (port 2024): Agent runtime and workflow execution
- **Gateway API** (port 8001): REST API for models, MCP, skills, memory, artifacts, uploads
- **Frontend** (port 3000): Next.js web interface
- **Nginx** (port 2026): Unified reverse proxy entry point

## Commands

### Root Directory (Full Application)

```bash
make check      # Check system requirements (Node 22+, pnpm, uv, nginx)
make install    # Install all dependencies (frontend + backend)
make config     # Generate config.yaml from template (first-time setup only)
make dev        # Start all services with hot-reloading
make stop       # Stop all running services
make clean      # Stop services and clean temporary files
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
├── config.yaml              # Main application config (models, tools, sandbox, memory)
├── extensions_config.json   # MCP servers and skills config
├── backend/
│   ├── packages/harness/deerflow/   # Agent framework (deerflow.* imports)
│   │   ├── agents/                  # Lead agent, middlewares, memory, thread state
│   │   ├── sandbox/                 # Sandbox execution (local/docker)
│   │   ├── subagents/               # Subagent delegation system
│   │   ├── tools/                   # Built-in and community tools
│   │   ├── mcp/                     # MCP integration
│   │   ├── skills/                  # Skills loading and management
│   │   └── client.py                # Embedded Python client
│   └── app/                         # Application layer (app.* imports)
│       ├── gateway/                 # FastAPI Gateway API
│       └── channels/                # IM integrations (Feishu, Slack, Telegram)
├── frontend/src/
│   ├── app/                 # Next.js routes
│   ├── components/          # UI components (ui/, workspace/, landing/)
│   └── core/                # Business logic (threads, api, artifacts, skills)
└── skills/                  # Agent skills (public/, custom/)
```

### Service Routing (Nginx)

- `/api/langgraph/*` → LangGraph Server (2024)
- `/api/*` (other) → Gateway API (8001)
- `/` (non-API) → Frontend (3000)

## Development Workflow

### Test-Driven Development (TDD) — MANDATORY

Every new feature or bug fix MUST have unit tests. No exceptions.

```bash
cd backend && make test   # Run before committing
```

### Pre-Commit Validation

Before submitting changes:

1. Backend: `cd backend && make lint && make test`
2. Frontend (if touched): `cd frontend && pnpm lint && pnpm typecheck`
3. Frontend build (if UI/auth changes): `BETTER_AUTH_SECRET=local-dev pnpm build`

### Configuration

- `config.yaml`: Models, sandbox, memory, tools, channels. Copy from `config.example.yaml`
- `extensions_config.json`: MCP servers and skills enabled state
- Values starting with `$` are resolved as environment variables (e.g., `$OPENAI_API_KEY`)

## Key Patterns

### Backend

- Middlewares execute in strict order (see `backend/CLAUDE.md` for full list)
- Config caching with mtime-based auto-reload
- Sandbox virtual paths: `/mnt/user-data/` → `backend/.deer-flow/threads/{thread_id}/user-data/`

### Frontend

- Server Components by default, `"use client"` for interactive components
- Thread hooks (`useThreadStream`, `useSubmitThread`) are the primary API
- LangGraph client singleton via `getAPIClient()` in `core/api/`

## Environment Requirements

- Node.js >=22
- pnpm 10.26.2+
- Python >=3.12
- uv (Python package manager)
- nginx (for unified local endpoint)

## Important Notes

- `make config` aborts if `config.yaml` already exists — use only for first-time setup
- Proxy env vars can break frontend `pnpm install` — unset them if errors occur
- Frontend `pnpm check` is unreliable; use `pnpm lint && pnpm typecheck` instead
- In Docker Compose, IM channels use container service names (`http://langgraph:2024`)

## Detailed Documentation

- `backend/CLAUDE.md` — Backend architecture, middlewares, sandbox, subagents, memory
- `frontend/CLAUDE.md` — Frontend architecture, data flow, components, code style
- `.github/copilot-instructions.md` — Verified command sequences and validation flows
- `backend/docs/CONFIGURATION.md` — Configuration options
- `CONTRIBUTING.md` — Development setup and workflow