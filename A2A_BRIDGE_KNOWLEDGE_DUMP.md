# A2A Bridge — Full Knowledge Dump

Everything from a deep-dive session studying `aegra/aegra#261` (A2A protocol support for Aegra) with the goal of building a **temporary, standalone A2A bridge** in front of a production Aegra deployment, until #261 actually merges. Paste this whole file as the first message into a fresh Claude Code session opened inside the real production repo.

---

## 1. Goal

Aegra is used in production to deploy LangGraph agents. We want to expose those agents over the **A2A (Agent-to-Agent) protocol** to an external system, but Aegra doesn't support A2A yet — `aegra/aegra#261` adds it, but is stuck in review (opened 2026-03-28, last commit 2026-03-30, still under active review as of 2026-04-26, with unresolved architectural concerns from maintainers). It's not landing soon.

**Decision: build a small, standalone bridge service** (separate from the production app, Python/FastAPI) that sits in front of Aegra, translates A2A JSON-RPC calls into calls against **Aegra's existing, stable, already-shipped public REST API** (not PR #261's code — that code has bugs, detailed below, and isn't merged anyway). Once #261 lands in a real release, delete the bridge and point A2A clients at Aegra directly.

**Scope decided:** Python, standalone service (not embedded in the main app), start with `message/send` only (simplest — request in, Task back, no SSE plumbing) and expand later if needed.

---

## 2. A2A Protocol — Core Concepts

A2A solves a different problem than MCP: MCP connects *one* agent to tools/data sources; A2A lets two **opaque** agents (different frameworks, different vendors, no shared code/memory) collaborate by exchanging messages over a standard interface.

Core vocabulary:

- **AgentCard** — a JSON document describing an agent's capabilities, published by convention at `/.well-known/agent-card.json`. A client fetches this before talking to the agent, to learn its URL/skills/auth.
- **Message** — one turn of conversation (from client or agent), made of one or more **Parts** (`text`, `file`, or `data` — discriminated by a `kind` field).
- **Task** — the unit of work spun up to handle something non-trivial (e.g., running a graph). Has a lifecycle:
  `submitted → working → completed / failed / input-required / canceled / rejected / auth-required` (full enum: `submitted, working, input_required, completed, canceled, failed, rejected, auth_required, unknown`).
- **Context** (`contextId`) — groups multiple related Tasks together (e.g., a whole multi-turn conversation). Maps naturally onto Aegra's `thread_id`.
- **Artifact** — the output an agent produces, itself a bundle of Parts (same shape as a Message's parts).
- Not every `message/send` needs to become a Task — the spec allows a trivial exchange to just return an immediate Message instead. **Aegra's own implementation (and your bridge) doesn't need this fork**: every Aegra "run" is already a full DB-persisted execution, so every `message/send` should just always become a Task, unconditionally. There's no cheaper code path to fall back to.

Transport: **JSON-RPC 2.0 over HTTP** is A2A's default (and the only one worth implementing here). One URL, one HTTP verb (`POST`), the actual operation is named inside the body via a `method` field — architecturally different from REST, which is why FastAPI has **no built-in JSON-RPC support** (FastAPI's whole design — one route per URL+verb pair, auto OpenAPI docs — doesn't fit a "one endpoint, N operations selected by a body field" shape). Everything has to be hand-rolled: manual body parsing, manual method dispatch, manual JSON-RPC error envelopes. (Third-party libs like `fastapi-jsonrpc` exist for this, but Aegra's PR hand-rolls it, and the bridge should follow the same shape.)

JSON-RPC 2.0 reserved error codes (from the spec itself, not made up per-project):

| Code | Meaning |
|---|---|
| -32700 | Parse error |
| -32600 | Invalid Request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32000 to -32099 | Reserved for implementation-defined server errors |

A2A itself adds two more reserved codes on top, for task-specific errors: **`-32001` `TaskNotFoundError`**, **`-32002` `TaskNotCancelableError`**. (Aegra's PR never emits either — everything collapses into the generic JSON-RPC codes above. Worth doing better in the bridge.)

---

## 3. The Real A2A Schema (verified against `a2a-sdk==0.3.25` source, not just spec prose)

Installed `a2a-sdk==0.3.25` (the exact version PR #261 pins) into an isolated scratch venv and read `a2a/types.py` directly — these are the actual Pydantic model definitions, not paraphrase. **Important:** the *latest* `a2a-sdk` on PyPI has been restructured heavily (mostly gRPC protobuf types now) — pin to `0.3.25` specifically (or whatever `<2.0.0` version you settle on) to get these Pydantic model shapes; this is exactly why PR reviewers demanded an upper bound (`a2a-sdk>=0.3.25,<2.0.0`) on the dependency.

All four methods share the same top-level JSON-RPC envelope, with `method` as a `Literal` discriminator field:

### `message/send` → `SendMessageRequest`
```json
{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "message/send",
  "params": {
    "message": {
      "messageId": "msg-1",
      "role": "user",
      "parts": [{"kind": "text", "text": "Hello"}],
      "contextId": "thread-abc",
      "taskId": "run-xyz"
    }
  }
}
```
`MessageSendParams` fields: `message` (required, a full `Message` object), `configuration` (optional), `metadata` (optional). **`contextId`/`taskId` only ever live nested inside `message` — never at the top level of `params`.**

`Message` required fields: `messageId`, `role` (`"user"` | `"agent"`), `parts` (list, required). Optional: `contextId`, `taskId`, `extensions`, `metadata`, `referenceTaskIds`. Fixed: `kind: "message"`.

`Part` = discriminated union `TextPart | FilePart | DataPart`, keyed by `kind`. `TextPart`: `{kind: "text", text: <required str>, metadata?}`.

Response → `SendMessageSuccessResponse.result: Task` (a single Task object).

### `message/stream` → `SendStreamingMessageRequest`
**Identical `params` shape** to `message/send` (same `MessageSendParams` type) — only `method: "message/stream"` differs. Response: a stream of `TaskStatusUpdateEvent` / `TaskArtifactUpdateEvent` objects over SSE, not one JSON-RPC response.

### `tasks/get` → `GetTaskRequest`
```json
{"jsonrpc": "2.0", "id": "2", "method": "tasks/get", "params": {"id": "run-xyz", "historyLength": 10}}
```
`TaskQueryParams` fields: `id` (required — **not** `taskId**), `historyLength` (optional, int), `metadata` (optional).
Response → `GetTaskSuccessResponse.result: Task` — **a single Task, not a list.** (`tasks/get` is "give me the state of exactly one task," not "list all tasks in a context." Maps to `GET /threads/{thread_id}/runs/{run_id}`, not `GET /threads/{thread_id}/runs`.) `Task.history: list[Message] | None` is what `historyLength` is meant to control — bring back N recent messages within that one task.

### `tasks/cancel` → `CancelTaskRequest`
```json
{"jsonrpc": "2.0", "id": "3", "method": "tasks/cancel", "params": {"id": "run-xyz"}}
```
`TaskIdParams` fields: `id` (required, the **only** required field — again `id`, not `taskId`), `metadata` (optional).
Response → `CancelTaskSuccessResponse.result: Task`.

### The `Task` object itself
```python
class Task(A2ABaseModel):
    artifacts: list[Artifact] | None = None
    context_id: str          # -> "contextId" on the wire (by_alias=True)
    history: list[Message] | None = None
    id: str
    kind: Literal['task'] = 'task'
    metadata: dict[str, Any] | None = None
    status: TaskStatus
```
Note the snake_case Python attrs ↔ camelCase JSON aliasing (`context_id` ↔ `contextId`) — this is why the code calls `.model_dump(by_alias=True)` everywhere; without `by_alias=True` you'd get snake_case keys on the wire, which a real A2A client wouldn't recognize.

---

## 4. Aegra's Real REST API — what the bridge should actually call

Confirmed directly from `libs/aegra-api/src/aegra_api/api/runs.py` and `libs/aegra-api/src/aegra_api/models/runs.py`. **Do not call Aegra's internal Python functions directly** (that's PR #261's mistake, see bug list below) — call the real HTTP endpoints, exactly as `langgraph_sdk`'s client would.

### `RunCreate` request model (`POST` body) — all fields:
`assistant_id` (required str), `input` (dict, optional — omit only when resuming from a checkpoint), `config`, `context`, `checkpoint`, `stream` (bool), `stream_mode` (str or list), `on_disconnect`, `on_completion` (`"delete"|"keep"`), `multitask_strategy`, `command` (for resuming interrupted runs), `interrupt_before`, `interrupt_after`, `stream_subgraphs`, `metadata`.

For a text message, standard LangGraph input shape: `{"messages": [{"role": "user", "content": "..."}]}` (plain dicts over the wire — the graph's reducer deserializes them, no need to construct LangChain message objects client-side since you're calling over HTTP, not importing Python objects in-process).

### `Run` response model — fields:
`run_id`, `thread_id`, `assistant_id`, `status` (`pending|running|error|success|timeout|interrupted`), `input`, `output`, `error_message`, `config`, `context`, `user_id`, `created_at`, `updated_at`.

### Relevant endpoints (from `runs.py`):
- `POST /threads/{thread_id}/runs` — create a run (fire and forget / poll later)
- `POST /threads/{thread_id}/runs/stream` — create + stream SSE (`create_and_stream_run`)
- `GET /threads/{thread_id}/runs/{run_id}` — **fetch one run's current state** (this is what `tasks/get` should call)
- `GET /threads/{thread_id}/runs` — list all runs on a thread (NOT what `tasks/get` needs)
- `PATCH /threads/{thread_id}/runs/{run_id}` — used for interrupt/cancel handling on Aegra's side
- `GET /threads/{thread_id}/runs/{run_id}/join` — join/wait alternate path
- **`POST /threads/{thread_id}/runs/wait`** — create a run, execute it, and wait for completion in one call (`wait_for_run`) — **this is the one `message/send` should call.**
- `GET /threads/{thread_id}/runs/{run_id}/stream` — SSE stream for an existing run

### Critical detail: `POST /threads/{thread_id}/runs/wait`'s actual response shape

Read `services/run_waiters.py` in full. The response is **not** a plain JSON body — it's a chunked `application/json` response:
```python
async def heartbeat_wait_body(run_id, thread_id, user_id, *, timeout) -> AsyncIterator[bytes]:
    # yields b"\n" heartbeat bytes every KEEPALIVE_INTERVAL_SECS while waiting
    # (keeps the HTTP connection alive through proxies/load balancers)
    ...
    output = await read_run_output(run_id, thread_id, user_id)  # reads RunORM.output from DB
    yield encode_output(output)   # json.dumps(output).encode() — the ONLY real content
```
Leading whitespace/newlines are ignored by JSON parsers, so **a normal HTTP client just needs to read the whole response body and `json.loads()` it (or call `.json()` on an `httpx`/`requests` response) — it works out of the box.** The body's JSON content is just `run.output` (e.g. `{"messages": [...]}`) — **it does NOT include `status`.**

**To get `run_id` and `status` too (needed to build a correct A2A Task):**
1. The response headers on `wait_for_run` include `Content-Location: /threads/{thread_id}/runs/{run_id}` — **parse the run_id straight out of this header.** This is more reliable than PR #261's approach (which queries the DB for "the most recently created run on this thread," which races if two runs land on the same thread concurrently).
2. Then call `GET /threads/{thread_id}/runs/{run_id}` to get the authoritative `status` (and `output`, redundantly but harmlessly) in one clean response — this endpoint returns the full `Run` model, no chunking, no heartbeat weirdness.

So the correct bridge flow for `message/send`:
```
POST /threads/{thread_id}/runs/wait   (body: {assistant_id, input})
  -> consume/discard the heartbeat body (or just await the full response)
  -> read run_id from Content-Location header
GET /threads/{thread_id}/runs/{run_id}
  -> get status + output, both authoritative, no raciness
```//

### Status → A2A TaskState mapping (used by the PR, reusable, but with a bug — see finding below)
```python
RUN_STATUS_TO_TASK_STATE = {
    "pending": "submitted",
    "running": "working",
    "success": "completed",
    "error": "failed",
    "timeout": "failed",
    "interrupted": "input-required",   # <- BUG, see finding #6 below. Aegra's cancel path
                                        #    sets status "interrupted", but a user-initiated
                                        #    cancel should map to A2A's "canceled" state, not
                                        #    "input-required" (which means "agent is waiting on you").
                                        #    Consider distinguishing "cancelled by user" from
                                        #    "actually interrupted/paused" if Aegra's status
                                        #    model allows it, or just accept this nuance and
                                        #    map interrupted -> canceled instead if you don't
                                        #    need Aegra's human-in-the-loop interrupt semantics
                                        #    over the A2A surface.
}
```

---

## 5. PR #261 — Full File Reference

Two files fully read and materialized locally to study line-by-line: `libs/aegra-api/src/aegra_api/adapters/a2a_adapter.py` (357 lines) and `a2a_service.py` (459 lines). Everything below is annotated from having traced every branch.

### `a2a_adapter.py` — the HTTP-facing layer

```python
"""A2A (Agent-to-Agent) protocol adapter.

Exposes agents via A2A JSON-RPC at /a2a/{assistant_id} with agent card
discovery at /.well-known/agent-card.json.
Uses the `a2a-sdk` library.
"""

from collections.abc import Awaitable, Callable
from typing import Any

import structlog
from fastapi import Depends, FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse, StreamingResponse
from sqlalchemy.ext.asyncio import AsyncSession
from starlette.responses import Response

from aegra_api.adapters.a2a_service import A2AService, get_a2a_service
from aegra_api.core.auth_deps import get_current_user
from aegra_api.core.orm import get_session
from aegra_api.models.auth import User
from aegra_api.services.streaming_service import streaming_service

logger = structlog.get_logger(__name__)


def _rpc_ok(result: Any, rpc_id: Any) -> dict[str, Any]:
    return {"jsonrpc": "2.0", "result": result, "id": rpc_id}


def _rpc_error(code: int, message: str, rpc_id: Any) -> dict[str, Any]:
    return {"jsonrpc": "2.0", "error": {"code": code, "message": message}, "id": rpc_id}


def _make_well_known_handler(service: A2AService) -> Callable[..., Awaitable[JSONResponse]]:
    async def _well_known_agent_card(
        request: Request, assistant_id: str | None = None
    ) -> JSONResponse:
        registry: dict[str, Any] = (
            service._langgraph_service._graph_registry if service._langgraph_service else {}
        )
        if not registry:
            raise HTTPException(status_code=404, detail="No agents registered")
        if assistant_id is not None:
            if assistant_id not in registry:
                raise HTTPException(status_code=404, detail=f"Unknown assistant_id: {assistant_id!r}")
            target_id = assistant_id
        else:
            target_id = next(iter(registry))
        graph_meta: Any = registry[target_id]
        name: str = graph_meta.get("name", target_id) if isinstance(graph_meta, dict) else target_id
        description: str = (
            graph_meta.get("description", f"Agent: {target_id}")
            if isinstance(graph_meta, dict) else f"Agent: {target_id}"
        )
        base_url: str = str(request.base_url).rstrip("/")
        card: dict[str, Any] = service.build_agent_card(
            assistant_id=target_id, name=name, description=description, base_url=base_url,
        )
        return JSONResponse(content=card)
    return _well_known_agent_card


def _make_agent_cards_handler(service: A2AService) -> Callable[..., Awaitable[JSONResponse]]:
    async def _list_agent_cards(request: Request) -> JSONResponse:
        registry: dict[str, Any] = (
            service._langgraph_service._graph_registry if service._langgraph_service else {}
        )
        base_url: str = str(request.base_url).rstrip("/")
        cards: list[dict[str, Any]] = []
        for agent_id, graph_meta in registry.items():
            name: str = graph_meta.get("name", agent_id) if isinstance(graph_meta, dict) else agent_id
            description: str = (
                graph_meta.get("description", f"Agent: {agent_id}")
                if isinstance(graph_meta, dict) else f"Agent: {agent_id}"
            )
            card: dict[str, Any] = service.build_agent_card(
                assistant_id=agent_id, name=name, description=description, base_url=base_url,
            )
            cards.append(card)
        return JSONResponse(content=cards)
    return _list_agent_cards


def _make_rpc_handler(service: A2AService) -> Callable[..., Awaitable[Response]]:
    async def _a2a_rpc(
        request: Request,
        assistant_id: str,
        user: User = Depends(get_current_user),
        session: AsyncSession = Depends(get_session),
    ) -> Response:
        try:
            body: dict[str, Any] = await request.json()
        except Exception:
            return JSONResponse(content=_rpc_error(-32700, "Parse error: invalid JSON", None))

        rpc_id: Any = body.get("id")
        method: str = body.get("method", "")
        params: dict[str, Any] = body.get("params") or {}

        if method == "message/send":
            message: dict[str, Any] = params.get("message") or {}
            parts: list[dict[str, Any]] = message.get("parts") or []
            context_id: str | None = params.get("contextId") or message.get("contextId")
            task_id: str | None = params.get("taskId") or message.get("taskId")
            try:
                result: dict[str, Any] = await service.send_message(
                    assistant_id=assistant_id, parts=parts, user=user,
                    context_id=context_id, task_id=task_id,
                )
            except ValueError as exc:
                return JSONResponse(content=_rpc_error(-32602, str(exc), rpc_id))
            except Exception as exc:
                logger.error("A2A send_message error", error=str(exc), assistant_id=assistant_id)
                return JSONResponse(content=_rpc_error(-32603, "Internal server error", rpc_id))
            return JSONResponse(content=_rpc_ok(result, rpc_id))

        if method == "tasks/get":
            task_id_param: str | None = (params.get("id") or params.get("taskId")) if params else None
            if not task_id_param:
                return JSONResponse(content=_rpc_error(-32602, "Missing required param: id", rpc_id))
            try:
                task_result: dict[str, Any] = await service.get_task(task_id_param, user)
            except ValueError as exc:
                return JSONResponse(content=_rpc_error(-32602, str(exc), rpc_id))
            except Exception as exc:
                logger.error("A2A get_task error", error=str(exc), task_id=task_id_param)
                return JSONResponse(content=_rpc_error(-32603, "Internal server error", rpc_id))
            return JSONResponse(content=_rpc_ok(task_result, rpc_id))

        if method == "tasks/cancel":
            cancel_task_id: str | None = (params.get("id") or params.get("taskId")) if params else None
            if not cancel_task_id:
                return JSONResponse(content=_rpc_error(-32602, "Missing required param: id", rpc_id))
            try:
                await streaming_service.cancel_run(cancel_task_id)          # <- BUG: fires before ownership check
                cancelled_task: dict[str, Any] = await service.get_task(cancel_task_id, user)  # <- ownership checked too late
            except ValueError as exc:
                return JSONResponse(content=_rpc_error(-32602, str(exc), rpc_id))
            except Exception as exc:
                logger.error("A2A tasks/cancel error", error=str(exc), task_id=cancel_task_id)
                return JSONResponse(content=_rpc_error(-32603, "Internal server error", rpc_id))
            return JSONResponse(content=_rpc_ok(cancelled_task, rpc_id))

        if method == "message/stream":
            message = params.get("message") or {}
            parts = message.get("parts") or []
            context_id = params.get("contextId") or message.get("contextId")
            task_id = params.get("taskId") or message.get("taskId")
            try:
                event_stream = service.stream_message(
                    assistant_id=assistant_id, parts=parts, user=user, session=session,
                    context_id=context_id, task_id=task_id,
                )
                return StreamingResponse(
                    event_stream, media_type="text/event-stream",
                    headers={"Cache-Control": "no-cache", "Connection": "keep-alive"},
                )
            except ValueError as exc:
                return JSONResponse(content=_rpc_error(-32602, str(exc), rpc_id))
            except Exception as exc:
                logger.error("A2A message/stream error", error=str(exc), assistant_id=assistant_id)
                return JSONResponse(content=_rpc_error(-32603, "Internal server error", rpc_id))

        return JSONResponse(content=_rpc_error(-32601, f"Method not found: {method!r}", rpc_id))
    return _a2a_rpc


def mount_a2a(app: FastAPI) -> None:
    service: A2AService = get_a2a_service()
    # Discovery endpoints are intentionally unauthenticated per the A2A spec.
    app.add_api_route(
        "/.well-known/agent-card.json", _make_well_known_handler(service),
        methods=["GET"], include_in_schema=False,
    )
    # /a2a/agent-cards MUST be registered before /a2a/{assistant_id} so the static path wins.
    app.add_api_route(
        "/a2a/agent-cards", _make_agent_cards_handler(service),
        methods=["GET"], include_in_schema=False,
    )
    app.add_api_route(
        "/a2a/{assistant_id}", _make_rpc_handler(service),
        methods=["POST"], include_in_schema=False, response_model=None,
    )
```

**Key mechanics explained (worth re-deriving in the bridge):**

- **Why `app.add_api_route(...)` instead of `@router.post(...)` decorators:** decorators register routes at import time, unconditionally. A2A must be conditionally mountable (`disable_a2a` config flag), which is only known at `create_app()` runtime — so routes are registered imperatively inside a function, not via decorator.
- **Closures over `Depends()`:** `_make_*_handler(service)` returns a handler that *closes over* `service`, captured once at mount time — vs. `user`/`session` which use `Depends()`, resolved fresh per request. This exists purely for testability: tests `patch("...get_a2a_service", return_value=fake_service)` then call `mount_a2a(app)` — the patch only needs to be active for that one call, not for the whole test's request-making phase, because the closures snapshot the fake service permanently at that instant.
- **Static route registered before dynamic route:** `/a2a/agent-cards` before `/a2a/{assistant_id}` — otherwise the path-parameter route could swallow requests meant for the static one (`{assistant_id}="agent-cards"`). General rule: always register static paths before parameterized ones in any router matching by registration order.
- **`include_in_schema=False`** on all three routes: JSON-RPC-over-HTTP doesn't map onto OpenAPI's per-operation schema model.
- **`response_model=None`** on the RPC route only: the handler returns fully-formed `Response` objects directly (mixed `JSONResponse`/`StreamingResponse`), so FastAPI shouldn't try response validation.
- **The docstring above `_a2a_rpc` is stale** — claims `message/stream` "Not yet implemented," but it's fully implemented below it. Don't trust docstrings over code in this file.

### `a2a_service.py` — the business logic layer

```python
"""A2A (Agent-to-Agent) service layer.

Maps A2A protocol concepts to Agent Protocol concepts:
- contextId -> thread_id
- taskId -> run_id
- A2A Message parts -> LangGraph messages
- Run status -> A2A TaskState
"""

from __future__ import annotations
import json
from collections.abc import AsyncIterator
from typing import Any
from uuid import uuid4

import structlog
from a2a.types import (
    AgentCapabilities, AgentCard, AgentSkill, Artifact, Part, Task,
    TaskArtifactUpdateEvent, TaskState, TaskStatus, TaskStatusUpdateEvent, TextPart,
)
from langchain_core.messages import AIMessage, HumanMessage
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from aegra_api.api.runs import create_and_stream_run, wait_for_run   # <- BUG: internal route-handler coupling
from aegra_api.core.orm import Run as RunORM
from aegra_api.core.orm import _get_session_maker
from aegra_api.models import RunCreate, User
from aegra_api.services.langgraph_service import LangGraphService
from aegra_api.utils.assistants import resolve_assistant_id

RUN_STATUS_TO_TASK_STATE: dict[str, str] = {
    "pending": "submitted", "running": "working", "success": "completed",
    "error": "failed", "timeout": "failed", "interrupted": "input-required",
}

def convert_parts_to_langchain(parts: list[dict[str, Any]]) -> list[HumanMessage]:
    messages: list[HumanMessage] = []
    for part in parts:
        kind = part.get("kind", "text")
        if kind != "text":
            raise ValueError(f"Unsupported part kind {kind!r}: only 'text' parts are supported")
        messages.append(HumanMessage(content=part.get("text", "")))
    return messages

def convert_output_to_parts(output: dict[str, Any]) -> list[dict[str, Any]]:
    messages: list[Any] = output.get("messages", [])
    parts: list[dict[str, Any]] = []
    for msg in messages:
        if isinstance(msg, AIMessage):
            content = msg.content
        elif isinstance(msg, dict) and msg.get("type") in ("ai", "AIMessage", "AIMessageChunk"):
            content = msg.get("content", "")
        else:
            continue
        if isinstance(content, str):
            parts.append({"kind": "text", "text": content})
        elif isinstance(content, list):
            for block in content:
                if isinstance(block, dict) and block.get("type") == "text":
                    parts.append({"kind": "text", "text": block.get("text", "")})
    return parts


class A2AService:
    def __init__(self) -> None:
        self._langgraph_service: LangGraphService | None = None

    def set_langgraph_service(self, service: LangGraphService) -> None:
        self._langgraph_service = service

    def build_agent_card(self, *, assistant_id, name, description, base_url) -> dict[str, Any]:
        card = AgentCard(
            name=name, description=description, url=f"{base_url}/a2a/{assistant_id}",
            version="1.0.0", capabilities=AgentCapabilities(),
            defaultInputModes=["text"], defaultOutputModes=["text"],
            skills=[AgentSkill(id="default", name="Default", description=description, tags=[])],
        )
        return card.model_dump(by_alias=True)

    async def send_message(self, *, assistant_id, parts, user, context_id=None, task_id=None) -> dict[str, Any]:
        if not self._langgraph_service:
            raise RuntimeError("LangGraph service not initialized")
        registry: dict[str, Any] = self._langgraph_service._graph_registry
        resolved_assistant_id = resolve_assistant_id(assistant_id, registry)
        langchain_messages = convert_parts_to_langchain(parts)
        input_data: dict[str, Any] = {"messages": langchain_messages}
        thread_id = context_id if context_id is not None else str(uuid4())
        request = RunCreate(assistant_id=resolved_assistant_id, input=input_data)

        output = await wait_for_run(thread_id, request, user)   # <- BUG: returns StreamingResponse now, not a dict

        maker = _get_session_maker()
        async with maker() as db_session:
            latest_run: RunORM | None = await db_session.scalar(
                select(RunORM).where(RunORM.thread_id == thread_id, RunORM.user_id == user.identity)
                .order_by(RunORM.created_at.desc()).limit(1)   # <- BUG: racy, could get wrong run
            )
        if latest_run:
            actual_task_state = TaskState(RUN_STATUS_TO_TASK_STATE.get(latest_run.status or "pending", "unknown"))
            effective_task_id = latest_run.run_id
        else:
            actual_task_state = TaskState.completed
            effective_task_id = task_id or str(uuid4())

        output_parts = convert_output_to_parts(output)   # <- BUG: should use latest_run.output, not broken `output`
        a2a_parts = [Part(root=TextPart(text=p["text"])) for p in output_parts]
        artifacts = [Artifact(artifactId="output", parts=a2a_parts)] if a2a_parts else None
        task = Task(id=effective_task_id, contextId=thread_id, status=TaskStatus(state=actual_task_state), artifacts=artifacts)
        return task.model_dump(by_alias=True)

    async def stream_message(self, *, assistant_id, parts, user, session, context_id=None, task_id=None) -> AsyncIterator[str]:
        if not self._langgraph_service:
            raise RuntimeError("LangGraph service not initialized")
        registry: dict[str, Any] = self._langgraph_service._graph_registry
        resolved_assistant_id = resolve_assistant_id(assistant_id, registry)
        langchain_messages = convert_parts_to_langchain(parts)
        input_data: dict[str, Any] = {"messages": langchain_messages}
        thread_id = context_id if context_id is not None else str(uuid4())
        effective_task_id = task_id or str(uuid4())   # <- BUG: never reconciled with the REAL run_id Aegra assigns

        request = RunCreate(assistant_id=resolved_assistant_id, input=input_data, stream_mode=["messages"])
        streaming_response = await create_and_stream_run(thread_id, request, user, session)

        yield f"data: {TaskStatusUpdateEvent(taskId=effective_task_id, contextId=thread_id, status=TaskStatus(state=TaskState.working), final=False).model_dump_json(by_alias=True)}\n\n"

        accumulated_text = ""
        stream_ended = False
        async for chunk in streaming_response.body_iterator:
            if stream_ended:
                break
            if not isinstance(chunk, str):
                chunk = chunk.decode("utf-8") if isinstance(chunk, bytes) else str(chunk)
            for line in chunk.strip().split("\n"):
                if line.startswith("data: "):
                    try:
                        event_data = json.loads(line[6:])
                    except (json.JSONDecodeError, TypeError):
                        continue
                    if isinstance(event_data, list):
                        for item in event_data:
                            if isinstance(item, dict):
                                content = item.get("content", "")
                                if content and isinstance(content, str):
                                    accumulated_text += content
                                    yield f"data: {TaskArtifactUpdateEvent(taskId=effective_task_id, contextId=thread_id, artifact=Artifact(artifactId='output', parts=[Part(root=TextPart(text=content))]), append=True, lastChunk=False).model_dump_json(by_alias=True)}\n\n"
                elif line.startswith("event: end"):
                    stream_ended = True
                    break

        if accumulated_text:
            yield f"data: {TaskArtifactUpdateEvent(taskId=effective_task_id, contextId=thread_id, artifact=Artifact(artifactId='output', parts=[Part(root=TextPart(text=accumulated_text))]), append=False, lastChunk=True).model_dump_json(by_alias=True)}\n\n"

        yield f"data: {TaskStatusUpdateEvent(taskId=effective_task_id, contextId=thread_id, status=TaskStatus(state=TaskState.completed), final=True).model_dump_json(by_alias=True)}\n\n"

    async def get_task(self, task_id: str, user: User) -> dict[str, Any]:
        maker = _get_session_maker()
        async with maker() as session:
            run_record: RunORM | None = await session.scalar(
                select(RunORM).where(RunORM.run_id == task_id, RunORM.user_id == user.identity)   # <- correct: ownership-safe, single query
            )
        if run_record is None:
            raise ValueError(f"Task not found: {task_id!r}")
        task_state = TaskState(RUN_STATUS_TO_TASK_STATE.get(run_record.status or "pending", "unknown"))
        output_parts = convert_output_to_parts(run_record.output or {})   # <- correct: uses the real DB row's output
        a2a_parts = [Part(root=TextPart(text=p["text"])) for p in output_parts]
        artifacts = [Artifact(artifactId="output", parts=a2a_parts)] if a2a_parts else None
        task = Task(id=run_record.run_id, contextId=run_record.thread_id, status=TaskStatus(state=task_state), artifacts=artifacts)
        return task.model_dump(by_alias=True)


_a2a_service: A2AService | None = None

def get_a2a_service() -> A2AService:
    global _a2a_service
    if _a2a_service is None:
        _a2a_service = A2AService()
    return _a2a_service
```

---

## 6. Consolidated Findings — 19 issues found tracing this PR (DO NOT replicate these in the bridge)

### Functional bugs (the feature doesn't work as claimed)
1. **`send_message`'s `output = await wait_for_run(...)`** — `wait_for_run` now returns a `StreamingResponse` (post-2026-04-04 heartbeat refactor in `run_waiters.py`); the PR's last commit was 2026-03-30, predating this refactor. `convert_output_to_parts(output)` then calls `.get()` on a `StreamingResponse` → `AttributeError` → every `message/send` fails with `-32603`. **A `git rebase` alone would not fix this** — it would only surface the incompatibility (make CI fail loudly); a human still has to rewrite the call site. Root cause: calling a FastAPI *route handler* directly as a plain internal function, coupling to an HTTP-transport implementation detail with no stability guarantee.
2. Compounding #1: even with the DB lookup for `latest_run`, the code still builds artifacts from the broken `output` variable instead of `latest_run.output`, which was already fetched and correct.
3. **`stream_message`'s `effective_task_id`** (`task_id or str(uuid4())`) is invented locally and never reconciled with the real `run_id` Aegra generates server-side inside `_prepare_run` (`run_id = str(uuid4())`, confirmed in `run_preparation.py:206` — `RunCreate` has no field to influence this). A later `tasks/get` on a task created via `message/stream` returns "Task not found" even though it ran fine. **The bridge must capture the real run_id (e.g. from the `Content-Location` header) and use that as the reported `taskId`, always.**

### Security
4. **`tasks/cancel` ownership-check-after-side-effect bug** — `streaming_service.cancel_run(cancel_task_id)` fires *before* any check that the run belongs to the calling user; the ownership-safe lookup (`get_task`, which filters by `user_id`) only happens afterward. Any authenticated user can cancel another user's run by supplying its run_id. Originally flagged by PR reviewers; independently confirmed by tracing the code. **The bridge must check ownership (fetch + verify the run belongs to the user) BEFORE issuing any cancel call.**
5. Discovery endpoints (`/.well-known/agent-card.json`, `/a2a/agent-cards`) are intentionally unauthenticated — exposes graph names/descriptions to anyone. Deliberate per A2A spec convention, but conflicts with Aegra's own security model per reviewer pushback. Decide deliberately for the bridge rather than copying default-on.

### Correctness / semantic mismatch
6. `RUN_STATUS_TO_TASK_STATE` never produces A2A's `canceled` state — Aegra's cancel path sets status `"interrupted"` → mapped to `"input-required"`, so a deliberately-cancelled task is reported as "agent is waiting for your input," which is backwards.
7. `send_message`'s "latest run for thread" DB lookup (`order_by(created_at.desc()).limit(1)`) is racy under concurrent runs on the same thread — use the exact run_id (from `Content-Location`), never "most recent."

### Spec-compliance gaps (incomplete, not wrong)
8. `historyLength` (valid per real `TaskQueryParams`) is never read or honored anywhere; `Task.history` is never populated.
9. A2A's dedicated error codes `TaskNotFoundError` (-32001) / `TaskNotCancelableError` (-32002) are never emitted — everything collapses to generic `-32602`/`-32603`.
10. Several `params.get("contextId")` / `params.get("taskId")` / `params.get("id") or params.get("taskId")` fallbacks check locations the real schema never actually populates (`MessageSendParams`/`TaskQueryParams`/`TaskIdParams` never carry those fields at the top level of `params`) — dead code, not real client-compatibility handling. Don't bother replicating these fallbacks in the bridge.
11. The Message-vs-Task spec fork (some sends could skip Task creation) is never used — fine, since Aegra's execution model makes every run a persisted Task anyway; the bridge should do the same (always return a Task).

### Architecture / design smells
12. Root cause of #1: calling FastAPI route handlers (`wait_for_run`, `create_and_stream_run`) directly as plain Python functions instead of through the actual public REST API. **The bridge inherently avoids this by construction — it's external and must go over HTTP anyway.**
13. Reaches into another service's "private" (`_`-prefixed) attribute (`self._langgraph_service._graph_registry`) from outside its class, twice.
14. `message/stream`'s `try/except` around `service.stream_message(...)` can never catch anything that function raises — it's an async generator (contains `yield`), so **none of its body executes until iteration begins**, which happens after the `200 OK` headers are already sent by `StreamingResponse`. A stronger version of the reviewer's "mid-stream exceptions after 200 sent" comment — it's not just mid-stream errors, it's *every* error path in that function, including line one.
15. `session` (`Depends(get_session)`) is held open for the entire `message/stream` SSE duration — the literal mechanism behind "connection pool starvation" under concurrent streaming clients.
16. Docstring on `_a2a_rpc` falsely claims `message/stream` is "Not yet implemented."

### Minor / inert
17. `tasks/get`'s `(...) if params else None` ternary is redundant — `params` is already normalized to `{}` earlier in `_a2a_rpc`, can never be falsy here.
18. `message/stream`'s `stream_ended` flag only breaks the inner loop; the outer loop's check runs one iteration late — harmless since the underlying generator naturally ends right after the `end` event anyway.
19. `message/send` has no required-field guard on `parts` (unlike `tasks/get`/`tasks/cancel`'s explicit checks) — empty/missing `parts` silently proceeds instead of being rejected up front.

---

## 7. Recommended Bridge Design

- **Stack:** Python, FastAPI (mirrors the target shape so migrating to native Aegra support later is a near-zero-diff cutover), standalone service — NOT embedded in the main production app, so it can be deleted cleanly once #261 ships.
- **Dependency:** `a2a-sdk` pinned to a known-good version (`0.3.25` confirmed to have the Pydantic model shapes used above; check for the same `<2.0.0`-style upper bound reviewers required, since newer versions restructure `a2a.types` into gRPC protobuf and drop these Pydantic classes). Use its real types (`AgentCard`, `Task`, `TaskStatus`, `TaskState`, `Artifact`, `Part`, `TextPart`, `TaskStatusUpdateEvent`, `TaskArtifactUpdateEvent`) rather than hand-rolling dicts — guarantees spec-correct serialization via `.model_dump(by_alias=True)`.
- **v1 scope: `message/send` only.** No `message/stream` SSE plumbing needed yet — simplest possible slice, matches what was decided mid-session. Add `tasks/get`/`tasks/cancel`/`message/stream` later if the actual use case needs them.
- **Auth:** forward whatever `Authorization` header the caller sends straight through to Aegra's REST calls — Aegra's own REST endpoints already enforce per-user ownership (404 on thread/run mismatch), so the bridge doesn't need to reimplement any of that logic itself; it inherits Aegra's ownership checks for free by going through the real API instead of raw DB queries.
- **`message/send` implementation outline:**
  1. Parse the JSON-RPC envelope by hand (same reasons FastAPI doesn't do this automatically apply here too: JSON-RPC's error contract — 200 OK + `{jsonrpc, error, id}` — doesn't match FastAPI's default 422 validation-error shape, and overriding that globally would break other unrelated routes if this bridge ever serves more than just A2A).
  2. Extract `message.parts` (reject/convert non-text parts — raise something mappable to `-32602`), `message.contextId` → `thread_id` (or mint a fresh UUID), `message.taskId` (bridge should NOT trust this for anything beyond echoing it back optionally — the *real* task/run id always comes from Aegra).
  3. `POST {AEGRA_BASE_URL}/threads/{thread_id}/runs/wait` with `{"assistant_id": ..., "input": {"messages": [{"role": "user", "content": text} for each text part]}}`, forwarding the `Authorization` header.
  4. Read `run_id` from the response's `Content-Location` header (format: `/threads/{thread_id}/runs/{run_id}`).
  5. `GET {AEGRA_BASE_URL}/threads/{thread_id}/runs/{run_id}` for authoritative `status` + `output`.
  6. Map `status` via a corrected version of `RUN_STATUS_TO_TASK_STATE` (fix the `interrupted → canceled` question deliberately, per finding #6).
  7. Convert `output["messages"]` AI-message content into `TextPart`s, wrap in an `Artifact`, build a real `a2a.types.Task(id=run_id, contextId=thread_id, status=..., artifacts=...)`, return `.model_dump(by_alias=True)` wrapped in the JSON-RPC success envelope.
- **Discovery endpoint:** include a minimal `/.well-known/agent-card.json` too (cheap, and likely necessary for any real external A2A client to find the bridge at all) — hardcode name/description/skills via env vars, point `url` at the bridge's own `/a2a/{assistant_id}` path.
- **Do not implement `tasks/cancel` without fixing the ownership-check ordering** (finding #4) — check the run belongs to the user first, then cancel.
- **Do not implement `tasks/get`/`tasks/cancel` echoing task IDs that were never validated against Aegra's real run_id** — always resolve through Aegra's actual API, never trust a client-supplied ID as authoritative.

---

## 8. Misc Python/Web Concepts Covered (useful background, not bridge-specific)

- **Closures as a "handler factory" pattern:** a function that builds and returns another function, which "remembers" variables from the enclosing scope even after the outer function has returned. Used throughout `a2a_adapter.py` for `service` (resolved once, at mount time) vs. `Depends()` (resolved fresh, per request) — different tools for "constant for the process's lifetime" vs. "different every call."
- **`await request.json()` — why async when JSON parsing is synchronous:** confirmed via Starlette's actual source (`starlette/requests.py`). `json.loads(body)` itself is plain sync code. The `await` exists because `body()` has to first collect the full request body off the network (`await self._receive()` — genuine I/O, waiting on bytes from a potentially slow client) before parsing can even start. Async marks "this might wait on something outside the CPU" (network, disk, DB) — not "this is a slow computation."
- **`model_dump()` vs `request.json()` — not interchangeable:** `model_dump()` is a Pydantic `BaseModel` method turning an *already-parsed, already-validated* model instance back into a dict. `request.json()` is Starlette's method turning *raw, unparsed bytes off the network* into a dict. Starlette's `Request` class has no `model_dump()` method at all — confirmed by grepping the actual installed source.
- **Why FastAPI has no built-in JSON-RPC support:** FastAPI's whole model is REST/OpenAPI-shaped (distinct URL+verb per operation); JSON-RPC's model is one URL, one verb, operation named inside the body — architecturally incompatible routing philosophies.
- **Pydantic discriminated unions** (`Annotated[Union[...], Field(discriminator="method")]`) are a real alternative to manual `if method == "..."` dispatch — FastAPI supports them as request body types. Not used here because JSON-RPC dictates its own error envelope (200 OK + `{jsonrpc, error, id}`), which doesn't match FastAPI's default `RequestValidationError` → 422 behavior; overriding that globally would affect every other route in the same app, not just this one.
- **Async generators are lazy:** calling a function containing `yield` executes *none* of its body — it just builds a generator object. The body only runs once something iterates it. This is why `message/stream`'s `try/except` around `service.stream_message(...)` can't catch anything that function would raise — none of it has run yet at that point.
- **JSON-RPC 2.0 error codes are a fixed, spec-defined vocabulary** (`-32700`/`-32600`/`-32601`/`-32602`/`-32603`), not something to memorize from scratch — same category as knowing common HTTP status codes.

---

## 9. Reference Links

- PR: https://github.com/aegra/aegra/pull/261 (opened 2026-03-28, last commit 2026-03-30, still under review as of 2026-04-26 — likely stale/unmergeable as-is)
- A2A spec: https://google.github.io/A2A/
- `a2a-sdk` on PyPI — pin to `0.3.25` (or whatever exact version has the Pydantic `types.py` shapes documented above; newer releases restructure around gRPC protobuf)
- Aegra docs added by the PR (still useful as a description of intended behavior even though unmerged): `docs/a2a.md` in the PR diff
