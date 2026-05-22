# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Install & run the CLI

```bash
uv sync
uv tool install -e .
ml-intern                          # interactive mode
ml-intern "your prompt"            # headless mode (auto-approves all tools)
ml-intern --model anthropic/claude-opus-4-6 --max-iterations 50 "prompt"
```

### Python tests

```bash
uv run pytest                                          # all tests
uv run pytest tests/unit/test_agent_loop.py            # single file
uv run pytest -k "test_doom_loop"                      # single test by name
```

Tests require `dev` extras (`uv sync --extra dev`). `asyncio_mode = "auto"` is set in `pyproject.toml`, so `async def test_*` functions work without decorators.

### Backend (web API)

```bash
cd backend && uvicorn main:app --host 0.0.0.0 --port 7860 --reload
# or: bash backend/start.sh
```

### Frontend

```bash
cd frontend && npm install && npm run dev     # dev server (Vite, port 5173)
cd frontend && npm run build                 # production build → static/
cd frontend && npm run lint                  # ESLint
```

## Architecture

The project has three deployment surfaces sharing a common `agent/` core:

1. **CLI** (`agent/main.py → ml-intern`) — interactive terminal chat or headless single-prompt execution.
2. **Backend** (`backend/`) — FastAPI server that exposes the same agent over HTTP/SSE for the web frontend.
3. **Frontend** (`frontend/`) — React/TypeScript SPA that talks to the backend via SSE for streaming.

### Async message-passing core

All three surfaces funnel user input through the same async architecture in `agent/core/agent_loop.py`:

- **`submission_queue`** — caller pushes `Submission` objects (typed `OpType` operations: `USER_INPUT`, `EXEC_APPROVAL`, `UNDO`, `COMPACT`, `SHUTDOWN`).
- **`event_queue`** — the agent loop pushes typed `Event` objects back to the caller for display.
- **`submission_loop()`** — the single entry point that drives everything. Creates a `Session`, then dequeues submissions and dispatches to `Handlers`.

The agentic loop inside `Handlers.run_agent()` calls `litellm.acompletion` (via `_call_llm_streaming` / `_call_llm_non_streaming`), accumulates streamed tool calls, checks approvals, then calls `ToolRouter.call_tool()` — running non-approval tools concurrently via `asyncio.gather`. On approval-required tools, the loop pauses and emits `approval_required`; execution resumes when an `EXEC_APPROVAL` submission arrives.

### Session & ContextManager

`agent/core/session.py` — `Session` owns:
- `ContextManager` (message history, auto-compaction at ~170k tokens, session upload to HF datasets)
- `ToolRouter` reference
- cancellation state (`_cancelled: asyncio.Event`)
- per-model effective reasoning effort cache (`model_effective_effort`)

Context compaction calls the LLM on the history and replaces it with a summary. Sessions are serialized and uploaded to a HF dataset repo for telemetry (`save_sessions = true` in `Config`).

### ToolRouter & tools

`agent/core/tools.py` — `ToolRouter` registers tools at startup and routes `call_tool()` calls:
- **Built-in tools** (`create_builtin_tools()`) — Python handlers in `agent/tools/`. Key tools: `research`, `hf_docs_*`, `hf_papers`, `hf_inspect_dataset`, `hf_jobs`, `hf_repo_files`, `hf_repo_git`, `github_*`, `plan_tool`, sandbox/local shell tools.
- **MCP tools** — loaded dynamically from `mcpServers` in the active config JSON via `fastmcp.Client`.

The CLI always uses `local_mode=True` (direct shell access via local tools). The web backend uses `local_mode=False` (sandboxed execution).

Tools requiring user approval before execution (`sandbox_create`, `hf_jobs` run/uv, `hf_repo_files` upload/delete, `hf_repo_git` destructive ops) are gated in `_needs_approval()`. `yolo_mode = True` bypasses all approvals.

### Config

`agent/config.py` — `Config` is a Pydantic model loaded from a JSON file. `${VAR_NAME}` placeholders in the JSON are substituted from the environment / `.env`. Two configs exist:
- `configs/cli_agent_config.json` — used by `ml-intern` CLI
- `configs/frontend_agent_config.json` — used by the web backend

Key config fields: `model_name`, `mcpServers`, `yolo_mode`, `max_iterations` (default 300), `reasoning_effort` (default `"max"`), `save_sessions`.

### Adding a built-in tool

Edit `agent/core/tools.py` → `create_builtin_tools()`, adding a `ToolSpec` with `name`, `description`, `parameters` (JSON Schema), and an async `handler`. The handler signature must be `async (arguments: dict, *, session=None, tool_call_id=None) -> tuple[str, bool]`. If the tool needs approval, add a check in `_needs_approval()` in `agent_loop.py`.

### Frontend–backend contract

The backend (`backend/routes/agent.py`) exposes SSE streams. Each `Event` emitted by the agent loop is forwarded as a JSON-encoded SSE event. The frontend (`frontend/src/hooks/useAgentChat.ts` and `frontend/src/lib/sse-chat-transport.ts`) consumes these events and drives the Zustand stores (`agentStore`, `sessionStore`).

### Doom-loop detection

`agent/core/doom_loop.py` inspects the recent tool-call history. When a repeated pattern of identical tool calls is detected, a corrective system prompt is injected into the conversation instead of calling the LLM again. A separate check in `agent_loop.py` catches repeated malformed-JSON tool arguments for the same tool.
