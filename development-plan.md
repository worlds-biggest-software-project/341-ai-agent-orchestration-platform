# AI Agent Orchestration Platform — Phased Development Plan

> Project: 341-ai-agent-orchestration-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three
`data-model-suggestion-*.md` files. The canonical database schema adopts **Data Model Suggestion 1
(Entity-Centric Normalized Relational)** because the platform's positioning targets regulated,
multi-tenant enterprise buyers who need queryable compliance and per-agent cost attribution. The
event-sourcing ideas from Suggestion 3 are folded into the durable-execution engine (Phase 3) as an
append-only `events` stream that drives crash recovery and time-travel, layered on top of the
normalized schema rather than replacing it.

---

## Core Requirements (synthesis)

- **What it does**: An open, LLM-native platform to define, execute, and govern multi-agent
  workflows — bundling orchestration, durable execution, a tool registry, and observability in one
  product, so teams no longer bolt a framework (LangGraph/CrewAI) onto a separate durable engine
  (Temporal/Inngest) and a third tracing tool (LangSmith/Langfuse).
- **Primary users**: platform engineering teams, AI-native product teams, enterprise IT leaders,
  and AI/ML engineers moving agents from PoC to production.
- **Key differentiators**: LLM calls as first-class primitives (token budgets, cost attribution,
  multi-provider fallback chains); unified orchestration + observability; built-in governance
  (audit, PII redaction, policy enforcement); native MCP and A2A; hybrid code + no-code builder.
- **MVP scope** (features.md "Must-have"): multi-agent orchestration with message passing/context
  sharing; tool registry (OpenAPI/JSON Schema); state persistence with basic crash recovery; LLM
  integration (OpenAI, Anthropic, open-source); execution via API/webhook/cron; real-time traces +
  logging; RBAC; configurable retries.
- **Post-MVP** (v1.1): visual builder; durable at-least-once execution; time-travel debugging;
  token budgeting/cost attribution; MCP; governance + PII redaction; cost dashboards.
- **Backlog**: autonomous prompt learning; intelligent fallback chains; cross-org A2A delegation;
  anomaly detection; code sandboxes; agent specialisation discovery.
- **Deployment**: self-hosted (Docker Compose) first, cloud/hybrid later — mirrors Temporal/n8n.
- **Integration surface**: LLM provider APIs; MCP servers; A2A agents; webhooks in/out; OTLP
  export; OAuth2/OIDC for delegated access.
- **Standards to implement**: OpenAPI 3.1 + JSON Schema 2020-12 (tool defs), OpenTelemetry GenAI
  semantic conventions (traces), CloudEvents 1.0.2 (event envelope), MCP 2025-11-25 (tool handoff),
  A2A (cross-org), OAuth2 (RFC 6749) + JWT (RFC 7519) (auth), OWASP LLM/Agentic Top 10 (security),
  ISO/IEC 42001 + NIST AI RMF + EU AI Act (governance surfaces).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The agent/LLM ecosystem (anthropic, openai, MCP SDK, mcp, opentelemetry, litellm) is Python-first; CrewAI and LangGraph set the integration expectations in Python. Async-native via asyncio fits long-running LLM I/O. |
| API framework | **FastAPI** | Native async, Pydantic v2 models, and automatic OpenAPI 3.1 generation — the platform's own API and its tool schemas share one schema language (standards.md: OAS 3.1). |
| Data validation / models | **Pydantic v2** | Single source of truth for request/response, config, workflow graph, and tool schemas; emits JSON Schema 2020-12 directly. |
| Database | **PostgreSQL 16** | Suggestion 1's normalized schema needs JSONB + GIN, partitioning for traces/events/audit, and rich relational joins for cost/compliance KPIs. SQLite supported only for local single-node dev. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Async engine matches FastAPI; Alembic gives the versioned migrations each phase requires. |
| Task queue / durable engine | **Custom durable executor backed by Postgres + Redis Streams** | The thesis is that off-the-shelf engines are not LLM-native; we build a step-based executor with Postgres-persisted state + an append-only `events` stream (Redis Streams for the work queue, Postgres for the durable log). Workers claim executions with `SELECT ... FOR UPDATE SKIP LOCKED`. |
| Cache / queue transport | **Redis 7** | Work queue (Streams + consumer groups), rate-limit token buckets, idempotency keys, and pub/sub for real-time trace streaming to the UI. |
| LLM provider abstraction | **LiteLLM** (provider router) + native **anthropic** / **openai** SDKs | LiteLLM gives a uniform call surface across OpenAI, Anthropic, Azure, Bedrock, Ollama (CrewAI uses the same), enabling fallback chains and unified token accounting. Native SDKs used where streaming/tool-use fidelity matters. |
| MCP | **`mcp` Python SDK** (Linux Foundation) | First-class MCP client for tool/resource handoff per spec 2025-11-25. |
| Observability | **OpenTelemetry Python SDK + OTLP exporter** | Emit GenAI semantic-convention spans (`gen_ai.*`, `invoke_agent`) so any backend (Grafana/Datadog/Honeycomb) works without custom glue. |
| Auth | **OAuth2 password + client-credentials, JWT (RS256), API keys (hashed)** | RFC 6749 / 7519. API keys for programmatic execution; JWT sessions for the UI; OAuth2 client-credentials for cross-service and A2A. |
| Frontend | **Next.js 15 (App Router) + TypeScript + React Flow + shadcn/ui + Tailwind** | Dashboard for traces/cost + a drag-and-drop graph builder (React Flow) for the hybrid no-code surface; server components for fast trace lists. |
| Real-time transport | **WebSocket (FastAPI) + SSE for LLM streaming** | Live trace/step updates and token-by-token agent output. |
| Containerisation | **Docker + docker-compose** | Self-hosted-first deployment (Postgres, Redis, API, worker, UI in one compose file). |
| Testing | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers** | Unit + integration against ephemeral Postgres/Redis; `respx`/`vcr` to mock LLM provider HTTP. Frontend: Vitest + Playwright. |
| Code quality | **ruff (lint+format), mypy (strict), pyright in CI** | Fast, standard Python toolchain. |
| Package / build | **uv + pyproject.toml** (backend); **pnpm** (frontend) | Fast, reproducible installs. |
| SDK | **TypeScript + Python client SDKs** | Per README multi-language pattern (Temporal-style); generated from the OpenAPI spec. |

### Project Structure

```
ai-agent-orchestration-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile                      # API + worker image
├── docker-compose.yml              # postgres, redis, api, worker, ui
├── alembic.ini
├── .env.example
├── README.md
├── migrations/                     # Alembic versions
│   └── versions/
├── src/
│   └── orchestrator/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory
│       ├── config.py               # Pydantic Settings
│       ├── db/
│       │   ├── engine.py           # async engine + session factory
│       │   ├── models.py           # SQLAlchemy ORM (Suggestion 1 schema)
│       │   └── repositories/       # data-access per aggregate
│       ├── domain/
│       │   ├── schemas.py          # Pydantic DTOs (agent, tool, workflow graph)
│       │   ├── graph.py            # WorkflowGraph parse/validate/topo
│       │   └── events.py           # CloudEvents envelopes, event types
│       ├── llm/
│       │   ├── router.py           # LiteLLM wrapper, fallback chains
│       │   ├── providers.py        # provider configs, pricing table
│       │   └── streaming.py        # SSE token streaming
│       ├── tools/
│       │   ├── registry.py         # tool CRUD + schema validation
│       │   ├── invoker.py          # http/function/mcp tool execution
│       │   └── mcp_client.py       # MCP client integration
│       ├── engine/
│       │   ├── executor.py         # durable step executor
│       │   ├── worker.py           # queue consumer loop
│       │   ├── checkpoints.py      # snapshot + replay
│       │   ├── retry.py            # backoff policy
│       │   └── nodes/              # agent_call, tool_call, human_gate, conditional, parallel
│       ├── governance/
│       │   ├── policies.py         # policy evaluation
│       │   ├── redaction.py        # PII detection/redaction
│       │   └── audit.py            # audit-log writer
│       ├── observability/
│       │   ├── otel.py             # OTel setup + span helpers
│       │   └── tracing.py          # trace persistence + live stream
│       ├── api/
│       │   ├── deps.py             # auth, db session, rbac deps
│       │   ├── auth.py             # OAuth2/JWT/API-key
│       │   └── routers/            # agents, tools, workflows, executions, traces, costs, webhooks, governance, mcp, a2a
│       ├── a2a/
│       │   ├── card.py             # Agent Card generation
│       │   └── server.py           # A2A endpoints
│       └── analytics/
│           ├── cost_rollup.py      # cost_records aggregation job
│           ├── anomaly.py          # drift/anomaly detection
│           └── optimizer.py        # prompt/fallback suggestions
├── sdk/
│   ├── python/
│   └── typescript/
├── ui/                             # Next.js app
│   ├── package.json
│   └── app/
└── tests/
    ├── conftest.py                 # testcontainers fixtures
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                   # sample workflows, tool schemas, LLM cassettes
```

The structure is grouped by concern (db, llm, tools, engine, governance, observability, api), so
every phase below adds modules/routers/tables without restructuring existing ones.

---

## Phase 1: Foundation — Project Skeleton, Data Layer, Auth

### Purpose
Establish the runnable backbone: configuration, the Postgres schema (Suggestion 1), async DB access,
auth (API keys + JWT), RBAC, and a health-checked FastAPI app in Docker Compose. After this phase a
developer can authenticate, and every later phase has tables and a session to write against.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Create the `uv` project, FastAPI app factory, settings, Docker Compose, and CI.

**Design**:
- `pyproject.toml` with deps: fastapi, uvicorn[standard], pydantic, pydantic-settings, sqlalchemy[asyncio], asyncpg, alembic, redis, litellm, anthropic, openai, mcp, opentelemetry-sdk, opentelemetry-exporter-otlp, python-jose[cryptography], passlib[argon2], httpx. Dev: pytest, pytest-asyncio, respx, testcontainers, ruff, mypy.
- `config.py` Pydantic `Settings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://localhost:6379/0"
      jwt_private_key_path: str
      jwt_public_key_path: str
      jwt_ttl_seconds: int = 3600
      otel_exporter_otlp_endpoint: str | None = None
      default_model: str = "claude-sonnet-4-6"
      env: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="AOP_", env_file=".env")
  ```
- `main.py`: `create_app() -> FastAPI` mounting routers, exception handlers, and `GET /healthz` returning `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml`: services `postgres:16`, `redis:7`, `api`, `worker` (Phase 3), `ui` (Phase 6).

**Testing**:
- `Unit: Settings loads from env with AOP_ prefix → correct typed values; missing database_url → ValidationError`.
- `Integration: GET /healthz with live testcontainer Postgres+Redis → 200, db=true, redis=true`.
- `Integration: GET /healthz with Redis down → 200, redis=false (degraded, not crash)`.
- `CI: ruff check, ruff format --check, mypy --strict all pass`.

#### 1.2 — Database schema & migrations (Suggestion 1)

**What**: Implement the 17-table normalized schema as SQLAlchemy ORM + the initial Alembic migration.

**Design**:
- Translate Suggestion 1 DDL directly. Tables: `organisations, users, api_keys, agents, tools,
  workflows, executions, steps, llm_calls, tool_calls, checkpoints, traces, events,
  governance_policies, policy_violations, cost_records, audit_log`.
- UUID PKs (`uuid_generate_v4`/`gen_random_uuid`). Enums implemented as CHECK constraints (portable)
  with matching Python `StrEnum`s in `domain/schemas.py`.
- `traces`, `events`, `audit_log` are `PARTITION BY RANGE (created_at)`; create monthly partitions
  via a helper and a migration that pre-creates the current + next month.
- Versioned registries: `agents`/`tools`/`workflows` keyed `UNIQUE (org_id, slug, version)`.
- GIN index on `traces.attributes`; standard indexes per Suggestion 1.

**Testing**:
- `Integration (real PG): alembic upgrade head → all 17 tables + indexes exist (query information_schema)`.
- `Integration: insert execution referencing non-existent workflow → ForeignKeyViolation`.
- `Integration: insert duplicate (org_id, slug, version) agent → UniqueViolation`.
- `Integration: insert trace into current-month partition → row lands in correct partition`.
- `Unit: every CHECK enum value has a matching Python StrEnum member (parametrised)`.

#### 1.3 — Authentication, API keys, and RBAC

**What**: API-key and JWT auth with role-based access control across four roles.

**Design**:
- API keys: generated as `aop_<prefix8>_<secret>`; store `key_hash = argon2(secret)`, `key_prefix`,
  `scopes TEXT[]` (`execute`, `read`, `admin`). `POST /v1/api-keys` returns the secret once.
- JWT: RS256, claims `{sub, org_id, role, scopes, exp, iat}`. `POST /v1/auth/token` (OAuth2 password grant).
- RBAC roles: `owner > admin > developer > viewer`. FastAPI deps:
  ```python
  async def require_auth(request) -> Principal: ...   # resolves api-key or JWT
  def require_role(min_role: Role) -> Callable: ...   # raises 403 if below
  def require_scope(scope: str) -> Callable: ...
  ```
  `Principal = {actor_id, actor_type: 'user'|'api_key', org_id, role, scopes}`.
- All queries are org-scoped via `Principal.org_id` (multi-tenancy enforced at the repository layer).

**Testing**:
- `Unit: api key verify — correct secret → True; tampered secret → False`.
- `Unit: JWT mint+verify roundtrip → claims preserved; expired token → 401`.
- `Integration: request with viewer role to POST /v1/agents → 403`.
- `Integration: request with valid api-key scope=read to a write endpoint → 403`.
- `Integration: developer in org A requesting org B's agent → 404 (tenant isolation)`.

### Definition of Done
Tables migrate cleanly; `/healthz` green in Docker Compose; auth + RBAC enforced; lint/type/tests pass.

---

## Phase 2: Agent & Tool Registry + LLM Runtime

### Purpose
Build the definition layer (agents, tools) and the LLM-native call primitive. After this phase a
single agent can be defined and invoked once against a real (or mocked) model, with token/cost
accounting recorded — the smallest unit of value the platform delivers.

### Tasks

#### 2.1 — Agent registry CRUD

**What**: Versioned CRUD for agents.

**Design**:
- Pydantic `AgentCreate`/`AgentRead` mirror the `agents` table (name, slug, system_prompt, model,
  model_provider, fallback_models, temperature, max_tokens, token_budget, tool_ids, role, goal,
  backstory, mcp_config, a2a_card, risk_classification).
- Endpoints: `POST /v1/agents` (creates v1 or auto-increments version on existing slug),
  `GET /v1/agents`, `GET /v1/agents/{slug}` (latest), `GET /v1/agents/{slug}/versions/{n}`,
  `PATCH` (creates new version, never mutates a referenced version), `DELETE` (soft `is_active=false`).
- Validation: `model_provider` ∈ enum; `fallback_models` non-empty entries; `risk_classification`
  enum (EU AI Act).

**Testing**:
- `Unit: AgentCreate with temperature 3.0 → ValidationError`.
- `Integration: POST same slug twice → version 1 then 2; GET latest → version 2`.
- `Integration: PATCH agent → new version row; old version unchanged`.
- `Integration: DELETE agent → is_active=false, excluded from list`.

#### 2.2 — Tool registry with OpenAPI/JSON Schema validation

**What**: Versioned tool registry where `schema_json` is OpenAPI 3.1 / JSON Schema 2020-12.

**Design**:
- `tool_type` ∈ `function|api|mcp|code_sandbox|browser|file_system|database|custom`.
- On create, validate `schema_json` is a valid JSON Schema 2020-12 (using `jsonschema` with the
  2020-12 validator); reject invalid schemas with the failing keyword/path.
- Fields: `endpoint_url`, `mcp_server_url`, `auth_config_json`, `timeout_ms` (default 30000).
- Endpoints mirror 2.1 (`/v1/tools`).

**Testing**:
- `Unit: valid 2020-12 schema → accepted`.
- `Unit: schema with unknown keyword / malformed type → 422 with JSON path`.
- `Integration: register api tool then GET → schema_json roundtrips intact`.
- `Integration: agent.tool_ids referencing a non-existent tool → 422`.

#### 2.3 — LLM router with provider abstraction & cost accounting

**What**: Uniform LLM call surface over OpenAI/Anthropic/open-source with token + cost capture.

**Design**:
- `llm/providers.py` holds a pricing table: `{model -> (input_cents_per_1k, output_cents_per_1k, cache_read_cents_per_1k)}`.
- `router.py`:
  ```python
  @dataclass
  class LLMRequest:
      model: str
      provider: str
      messages: list[dict]
      tools: list[dict] | None
      temperature: float
      max_tokens: int | None
      stream: bool = False

  @dataclass
  class LLMResult:
      content: str
      tool_calls: list[ToolCallRequest]
      stop_reason: Literal["end_turn","max_tokens","tool_use","stop_sequence","error"]
      tokens_input: int
      tokens_output: int
      tokens_cache_read: int
      cost_cents: int
      latency_ms: int
      model: str
      is_fallback: bool

  async def call_llm(req: LLMRequest) -> LLMResult: ...   # via LiteLLM
  ```
- Cost = `(tokens_input/1000)*in_rate + (tokens_output/1000)*out_rate + cache`; rounded to cents.
- Streaming variant yields tokens via SSE (`streaming.py`); final chunk carries usage.
- Each call persisted to `llm_calls` (mapped to the active `step` once the engine exists; in this
  phase a synthetic step is created for the direct-invoke endpoint).
- `POST /v1/agents/{slug}/invoke` — one-shot agent call: `{input, stream?}` → `LLMResult` + persisted
  `execution`/`step`/`llm_call` rows (no graph yet).

**Testing**:
- `Unit (mocked respx): Anthropic-style response → LLMResult with correct token counts and cost`.
- `Unit: cost calc for known token counts → expected cents (parametrised across models)`.
- `Integration (mocked): POST /invoke → 200, llm_calls row written with cost > 0`.
- `Integration (mocked): streaming invoke → SSE token chunks then a usage event`.
- `E2E (optional, real key, marked @pytest.mark.live): invoke against real model → stop_reason set`.

### Definition of Done
Agents and tools can be created/versioned; `/invoke` returns a costed result and writes `llm_calls`;
OpenAPI spec includes new routes.

---

## Phase 3: Durable Workflow Engine

### Purpose
The heart of the product. Turn a workflow graph of agent/tool/control nodes into a durable, crash-
recoverable execution with retries, checkpoints, and an append-only event log. After this phase
multi-agent workflows run reliably and can be replayed.

### Tasks

#### 3.1 — Workflow definition & graph validation

**What**: Versioned workflow CRUD storing a validated graph in `graph_json`.

**Design**:
- `domain/graph.py` parses `graph_json` into a `WorkflowGraph`:
  ```python
  NodeType = Literal["agent","tool","human_gate","conditional","parallel_fork","parallel_join","subworkflow"]
  class Node(BaseModel):
      id: str; type: NodeType
      agent_slug: str | None = None
      tool_slug: str | None = None
      prompt: str | None = None           # human_gate
      condition: str | None = None        # conditional (safe expression)
      subworkflow_slug: str | None = None
  class Edge(BaseModel):
      from_: str; to: str; condition: str | None = None
  class WorkflowGraph(BaseModel):
      nodes: list[Node]; edges: list[Edge]; entry_node: str
  ```
- Validation: entry_node exists; all edge endpoints reference real nodes; agent/tool nodes
  reference existing registry slugs; **no unreachable nodes**; cycles allowed only through a node
  with a termination guard; `parallel_fork` has a matching `parallel_join`.
- `trigger_type` ∈ `api|webhook|event|cron|manual`; `cron_expression` required when `cron`.
- Endpoints under `/v1/workflows` (versioned, same pattern as agents).

**Testing**:
- `Unit: graph with edge to unknown node → ValidationError naming the node`.
- `Unit: graph with unreachable node → ValidationError`.
- `Unit: fork without matching join → ValidationError`.
- `Integration: create workflow referencing missing agent slug → 422`.

#### 3.2 — Event log, execution state, and checkpoints

**What**: Append-only `events` stream (CloudEvents) as the durable source of truth, plus checkpoint
snapshots for fast resume.

**Design**:
- Event types (CloudEvents `ce_type`): `execution.created|started|completed|failed|cancelled`,
  `step.started|completed|failed|retrying`, `llm.call.completed`, `tool.call.completed`,
  `human_gate.requested|resolved`, `checkpoint.created`, `policy.violated`.
- Envelope per CloudEvents 1.0.2: `{ce_source, ce_type, ce_specversion:"1.0", ce_time, subject:execution_id, data}` → `events` table.
- `checkpoints.create(execution_id, step_id, state_json, version)` persists the workflow `state_json`
  after each successfully completed node. Resume reads the latest checkpoint and replays only events
  after it (checkpoint-resume, per Trigger.dev pattern).
- `executions.state_json` holds the live merged state (node outputs keyed by node id).

**Testing**:
- `Integration: completing a step writes one checkpoint and a step.completed event`.
- `Unit: resume from checkpoint v2 → state equals snapshot v2 + replayed events after v2`.
- `Integration: events for an execution are strictly time-ordered and append-only (no updates)`.

#### 3.3 — Step executor, node handlers, and retry policy

**What**: The async executor that advances an execution node-by-node with retries and backoff.

**Design**:
- `engine/executor.py`: `async def advance(execution_id)`:
  1. Load execution + workflow graph + latest checkpoint.
  2. Determine ready node(s) from graph topology and completed nodes.
  3. For each node, dispatch to a handler in `engine/nodes/`:
     - `agent_call`: builds messages from agent system_prompt + upstream state, calls `call_llm`,
       loops tool-use until `stop_reason != tool_use` (bounded by `max_tool_iterations`, default 10).
     - `tool_call`: invokes via `tools/invoker.py` (http/function/mcp), validates output against schema.
     - `human_gate`: sets execution `status='waiting_human'`, emits `human_gate.requested`, suspends.
     - `conditional`: evaluates `condition` against state via a sandboxed expression evaluator (no eval; use `simpleeval`).
     - `parallel_fork`/`parallel_join`: fan out ready branches concurrently with `asyncio.gather`, join on all.
  4. Persist `steps`, `llm_calls`, `tool_calls`; update `state_json`; write checkpoint + events.
  5. On terminal node → `status='completed'`, output assembled from output_schema.
- Retry policy from `workflows.retry_policy_json`:
  ```python
  class RetryPolicy(BaseModel):
      max_retries: int = 3
      backoff: Literal["fixed","exponential"] = "exponential"
      initial_delay_ms: int = 1000
      max_delay_ms: int = 60000
  ```
  Failed step → `status='retrying'`, re-enqueued with computed delay; exhausted → step `failed`,
  execution `failed` unless node marked `continue_on_error`.

**Testing**:
- `Unit: topology resolver — diamond graph yields correct ready set after each completion`.
- `Unit: exponential backoff delays = [1000,2000,4000] capped at max_delay_ms`.
- `Integration (mocked LLM): linear 2-agent workflow → both steps completed, state chained`.
- `Integration: agent node with tool_use loop → tool invoked, result fed back, final answer`.
- `Integration: tool raises → retried per policy, then step failed → execution failed`.
- `Integration: parallel_fork of 3 branches → all run concurrently, join waits for all`.

#### 3.4 — Worker, queue, and crash recovery

**What**: Background worker that consumes a Redis Streams queue and drives executions; recovers
in-flight executions after a crash.

**Design**:
- `engine/worker.py`: consumer-group loop on Redis Stream `aop:executions`. Claims with
  `SELECT ... FOR UPDATE SKIP LOCKED` on `executions` to guarantee single-worker ownership per execution.
- Enqueue on: `POST /v1/workflows/{slug}/run` (API trigger), webhook, cron tick, or step re-enqueue.
- Idempotency: `executions.idempotency_key UNIQUE`; duplicate run with same key returns the existing execution.
- Crash recovery on startup: find executions in `running` with a stale `updated_at` (heartbeat older
  than `lease_timeout`, default 60s) → re-enqueue; executor resumes from latest checkpoint (3.2).
- At-least-once delivery; node handlers must be idempotent (tool calls keyed by step id).

**Testing**:
- `Integration: run a workflow → execution reaches completed via worker`.
- `Integration: duplicate idempotency_key → same execution_id, no second run`.
- `Integration: kill worker mid-execution, restart → execution resumes from checkpoint, completes`.
- `Integration: two workers, one execution → only one claims it (SKIP LOCKED)`.

### Definition of Done
A multi-node workflow runs to completion through the worker, survives a simulated crash, retries
transient failures, and persists a full step/event/checkpoint trail. Migrations created.

---

## Phase 4: Triggers, Webhooks, Scheduling & Real-Time Streaming

### Purpose
Connect the engine to the outside world: API runs, signed inbound webhooks, cron schedules, and
live execution streaming. After this phase workflows fire automatically and clients watch progress
in real time. Can be developed in parallel with Phase 5.

### Tasks

#### 4.1 — Execution API & run lifecycle

**What**: Endpoints to start, inspect, cancel, and resume executions.

**Design**:
- `POST /v1/workflows/{slug}/run` `{input, idempotency_key?}` → `202 {execution_id, status}`.
- `GET /v1/executions/{id}` → execution with steps summary; `GET /v1/executions` (filter by status, workflow, date).
- `POST /v1/executions/{id}/cancel` → `cancelled` (cooperative; sets flag checked between steps).
- `POST /v1/executions/{id}/resume` `{node_id, decision, state_patch?}` → resolves a `human_gate`,
  re-enqueues. Validates execution is `waiting_human`.

**Testing**:
- `Integration: run → poll GET execution → status transitions pending→running→completed`.
- `Integration: cancel running execution → status cancelled, no further steps`.
- `Integration: resume a waiting_human execution with approval → continues; resume non-waiting → 409`.

#### 4.2 — Webhook ingestion with signature verification

**What**: Inbound webhook endpoint that triggers webhook-type workflows, with HMAC verification.

**Design**:
- `POST /v1/webhooks/{workflow_slug}`; per-workflow signing secret. Verify
  `X-AOP-Signature: sha256=<hmac(secret, raw_body)>` (constant-time compare). Map payload → workflow input.
- Wrap inbound payload in a CloudEvents envelope (`ce_type="webhook.received"`) for the audit/event log.
- Rate-limit per workflow via Redis token bucket.

**Testing**:
- `Integration: valid signature → 202, execution enqueued`.
- `Integration: invalid/missing signature → 401, nothing enqueued`.
- `Integration: replayed body within window with same delivery id → deduped (idempotent)`.

#### 4.3 — Cron scheduler

**What**: Reliable cron trigger for scheduled workflows.

**Design**:
- A single leader-elected scheduler (Redis lock) ticks every 30s, evaluates `cron_expression`
  (croniter) for active cron workflows, and enqueues runs with `idempotency_key = f"{workflow_id}:{fire_time}"`
  to guarantee exactly-one enqueue per scheduled instant even with multiple schedulers.

**Testing**:
- `Unit: croniter next-fire for "*/5 * * * *" → correct timestamps`.
- `Integration: due cron workflow → exactly one execution enqueued per fire time (dedup verified)`.
- `Integration: two schedulers, one leader → no duplicate enqueues`.

#### 4.4 — Real-time trace streaming

**What**: WebSocket + SSE so clients see steps and tokens as they happen.

**Design**:
- Worker publishes step/LLM events to Redis pub/sub channel `aop:exec:{id}`.
- `WS /v1/executions/{id}/stream` relays events; `GET /v1/executions/{id}/stream` (SSE) for agent
  token streaming. Auth via JWT/api-key on connect; org-scoped.

**Testing**:
- `Integration: subscribe WS → receive step.started/completed events in order during a run`.
- `Integration: unauthorised subscriber → connection rejected 4401`.

### Definition of Done
Workflows trigger via API, signed webhook, and cron; clients stream live progress; new routes in
the OpenAPI spec.

---

## Phase 5: Observability, Cost Attribution & Governance

### Purpose
Deliver the unified-observability and built-in-governance differentiators: OTel GenAI spans, cost
roll-ups, audit trails, policy enforcement, and PII redaction. Can parallel Phase 4.

### Tasks

#### 5.1 — OpenTelemetry GenAI tracing

**What**: Emit and persist OTel spans following GenAI semantic conventions.

**Design**:
- `observability/otel.py` configures a tracer + OTLP exporter (endpoint from settings).
- Span hierarchy mirrors execution→step→llm/tool. Agent spans use operation `invoke_agent` and
  attributes `gen_ai.agent.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`,
  `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason`.
- Spans persisted to `traces` (partitioned) and exported via OTLP when an endpoint is set.
- `GET /v1/executions/{id}/traces` returns the span tree.

**Testing**:
- `Unit: agent span carries required gen_ai.* attributes`.
- `Integration: completed execution → traces rows form a valid parent/child tree`.
- `Integration (mock OTLP collector): spans exported with correct operation names`.

#### 5.2 — Cost roll-up & dashboards API

**What**: Aggregate `llm_calls`/`tool_calls` into `cost_records` for per-agent/workflow/model reporting.

**Design**:
- `analytics/cost_rollup.py`: periodic job (cron-driven) upserts daily + monthly `cost_records`
  grouped by `(org, agent, workflow, model, period)`.
- `GET /v1/costs?group_by=agent|workflow|model&period=daily|monthly&from&to` → aggregated rows;
  `GET /v1/costs/summary` → org totals, top agents, fallback rate.

**Testing**:
- `Integration: 3 executions across 2 agents → cost_records aggregates match summed llm_calls`.
- `Integration: GET /v1/costs?group_by=model → correct per-model totals`.
- `Unit: rollup is idempotent — running twice for same period yields same totals (upsert)`.

#### 5.3 — Governance policies & enforcement

**What**: Data-driven policies evaluated at runtime, with violations recorded and actions enforced.

**Design**:
- `governance/policies.py` evaluates `governance_policies` against execution/step context. Types
  (per Suggestion 1): `pii_redaction, token_budget, model_allowlist, tool_allowlist,
  prompt_injection_guard, excessive_agency, output_validation, rate_limit, cost_limit,
  human_approval_required, data_residency`.
- Hooks: pre-LLM-call (model_allowlist, token_budget, cost_limit), pre-tool-call (tool_allowlist),
  post-output (output_validation, pii_redaction). Action ∈ `logged|warned|blocked|redacted`; blocked
  raises and fails the step; violations written to `policy_violations` + a `policy.violated` event.
- Maps to OWASP Agentic Top 10 (`excessive_agency`, `rogue_agent` via tool_allowlist + human gate).

**Testing**:
- `Unit: token_budget exceeded → violation severity=error, action=blocked`.
- `Unit: model_allowlist deny → call blocked before provider hit`.
- `Integration: workflow hitting a blocked policy → execution failed with policy_violation row`.

#### 5.4 — PII redaction & audit log

**What**: Detect/redact PII in inputs/outputs and write an immutable audit trail.

**Design**:
- `governance/redaction.py`: regex + (optional) NER detectors for `email, phone, ssn, credit_card`;
  redaction replaces with `[REDACTED:<type>]`. Applied to logged trace payloads and tool outputs
  when `pii_redaction` policy active.
- `governance/audit.py`: every state-changing API action + agent/system action writes `audit_log`
  (`actor_type`, `action`, `entity_type/id`, `changes_json`, `ip_address`) — supports ISO 42001 / EU AI Act.
- `GET /v1/audit` (admin) with filters; append-only (no update/delete endpoints).

**Testing**:
- `Unit: redact text with email+ssn → both replaced, other text intact`.
- `Integration: create agent → audit_log row with action=agent.created and diff`.
- `Integration: audit_log has no update/delete route (immutability)`.

### Definition of Done
Executions emit OTel spans; cost dashboards return correct aggregates; policies enforce/redact and
record violations; audit trail captures all mutations.

---

## Phase 6: Hybrid No-Code Builder & Dashboard UI

### Purpose
Address the visual-vs-code market split with a Next.js app: drag-and-drop graph builder (React Flow)
plus dashboards for executions, traces, and cost. Requires Phases 3–5 APIs.

### Tasks

#### 6.1 — UI foundation, auth, and API client

**What**: Next.js App Router app with auth, generated typed API client, and layout.

**Design**:
- Generate a TS client from the FastAPI OpenAPI spec (`openapi-typescript` + a thin fetch wrapper
  injecting the JWT). Login page → `POST /v1/auth/token`; token in httpOnly cookie. shadcn/ui + Tailwind shell.

**Testing**:
- `E2E (Playwright): login with valid creds → redirected to dashboard; invalid → error shown`.
- `Unit (Vitest): API client attaches Authorization header`.

#### 6.2 — Visual workflow builder

**What**: React Flow canvas to compose agent/tool/control nodes into a `graph_json`.

**Design**:
- Node palette (agent, tool, human_gate, conditional, parallel_fork/join). Canvas edges → `edges`.
- Client-side validation mirrors `domain/graph.py` rules; "Save" serialises to `graph_json` and
  `POST/PATCH /v1/workflows`. Code escape hatch: a raw-JSON editor synced with the canvas (hybrid).

**Testing**:
- `E2E: drag two agents, connect, save → workflow created with correct graph_json`.
- `E2E: invalid graph (orphan node) → save blocked with inline error`.

#### 6.3 — Execution, trace, and cost dashboards

**What**: Views for execution list/detail (live), trace tree, and cost charts.

**Design**:
- Executions list (filter/status); detail page subscribes to `WS /executions/{id}/stream` for live
  steps and SSE token streaming. Trace tree from `/traces`. Cost charts from `/v1/costs`.
  Time-travel panel renders checkpoints (Phase 3) with a "replay from here" action.

**Testing**:
- `E2E: run a workflow from UI → steps appear live; final output rendered`.
- `E2E: cost page renders per-agent totals matching API`.

### Definition of Done
A user builds, runs, and observes a workflow entirely in the UI; Playwright E2E suite green; UI built
in the Docker image.

---

## Phase 7: MCP & A2A Interoperability

### Purpose
Native standards-based interoperability: consume external tools via MCP, and expose/consume agents
across organisations via A2A. The cross-org delegation moat. Requires Phases 2–3.

### Tasks

#### 7.1 — MCP client integration

**What**: Connect agents to MCP servers and surface their tools in the registry.

**Design**:
- `tools/mcp_client.py` using the `mcp` SDK. From `agents.mcp_config_json` (servers, transport
  sse/streamable-http, auth), discover tools/resources/prompts and register them as `tool_type='mcp'`
  with their JSON Schema. At execution, MCP tools are invoked through the same `invoker.py` path.
- Pin to MCP spec `2025-11-25`; handle auth token refs (vault-style indirection).

**Testing**:
- `Integration (mock MCP server): connect → tools discovered and registered with schemas`.
- `Integration: agent uses an MCP tool in a workflow → tool_call recorded, result fed back`.
- `Integration: MCP server unauthorised → tool marked unavailable, clear error`.

#### 7.2 — A2A agent cards & cross-org delegation

**What**: Publish Agent Cards and call remote A2A agents as workflow nodes.

**Design**:
- `a2a/card.py` generates a signed Agent Card from an agent definition (`name, description, url,
  version, capabilities, authentication`). `GET /.well-known/agent.json` per A2A.
- `a2a/server.py` implements the A2A task lifecycle (`message/send`, task states) so external agents
  can delegate to ours. A `tool_type` extension or dedicated `a2a` node lets our workflows call remote
  agents with OAuth2 client-credentials + JWT identity (OIDC-A-style delegation claims).

**Testing**:
- `Integration: GET /.well-known/agent.json → valid signed Agent Card`.
- `Integration (mock A2A peer): workflow node delegates a task → remote result merged into state`.
- `Unit: Agent Card signature verifies with published key`.

### Definition of Done
Agents consume MCP tools and delegate to / serve A2A peers, with signed cards and authenticated cross-
org calls. Spec versions pinned.

---

## Phase 8: AI-Native Intelligence — Fallback, Optimisation, Anomaly Detection

### Purpose
Deliver the AI-native advantages from the research: intelligent multi-provider fallback, execution-
trace-driven prompt/tool optimisation suggestions, anomaly/drift detection, and governance-policy
suggestions. Requires Phases 3 and 5 (needs historical traces + cost data).

### Tasks

#### 8.1 — Intelligent fallback chains & token-aware scheduling

**What**: Automatic provider failover and rate-limit-aware scheduling at the LLM call layer.

**Design**:
- Extend `llm/router.py`: on rate-limit/5xx/timeout, walk `agent.fallback_models` in order, marking
  `is_fallback=true`; record which model ultimately answered. Token-aware queueing: per-provider
  Redis token-bucket so concurrent executions respect provider rate limits; prefer the cheapest model
  meeting the agent's quality tier when budget pressure (cost_limit policy) is high.

**Testing**:
- `Unit (mock): primary 429 then fallback 200 → result.is_fallback true, fallback model recorded`.
- `Unit: all providers fail → step error after exhausting chain`.
- `Integration: rate-bucket exhausted → calls queued, not dropped`.

#### 8.2 — Trace-driven optimisation & anomaly detection

**What**: Analyse historical executions to suggest prompt/tool/model improvements and flag anomalies.

**Design**:
- `analytics/optimizer.py`: aggregates failure modes per agent (stop_reason patterns, tool errors,
  retries) and produces suggestions (prompt clarifications, tool reordering, model substitution) via
  an LLM "meta-agent" over redacted traces. Output: `GET /v1/agents/{slug}/suggestions`.
- `analytics/anomaly.py`: statistical detectors on cost/latency/token distributions (z-score and
  EWMA) plus heuristic checks for hallucination/drift/security (OWASP Agentic: rogue agent,
  cascading failure). Emits `anomaly.detected` events + optional alerts.

**Testing**:
- `Integration: seed traces with a recurring tool failure → suggestion references that tool`.
- `Unit: latency 10x baseline → anomaly flagged with score`.
- `Integration: anomaly emits event and (mock) alert`.

#### 8.3 — Governance policy suggestion

**What**: Recommend governance policies from observed behaviour.

**Design**:
- Analyse `policy_violations` + tool/cost patterns to propose new `governance_policies` (e.g. tighten
  token budget, add tool_allowlist, require human_approval above a cost threshold). `GET /v1/governance/suggestions`;
  admin can accept → creates the policy + audit entry.

**Testing**:
- `Integration: repeated cost-limit near-misses → suggests lower token_budget policy`.
- `Integration: accepting a suggestion → governance_policies row + audit_log entry`.

### Definition of Done
Fallback chains engage automatically and are recorded; optimisation/anomaly/governance suggestions are
generated from real history and surfaced via API/UI.

---

## Phase 9: Hardening, SDKs & Release Packaging

### Purpose
Production readiness: client SDKs, security hardening against OWASP LLM/Agentic Top 10, performance,
docs, and a one-command self-hosted release.

### Tasks

#### 9.1 — Python & TypeScript SDKs

**What**: Thin, ergonomic clients generated from the OpenAPI spec plus hand-written helpers.

**Design**:
- `sdk/python` and `sdk/typescript`: auth, `define_agent/tool/workflow`, `run`, `stream`, `wait`.
  TS SDK emphasises type safety (Trigger.dev-style DX). Published examples in `examples/`.

**Testing**:
- `Integration: Python SDK runs a workflow against a test server → completed`.
- `Integration: TS SDK streams steps → events received`.

#### 9.2 — Security hardening (OWASP LLM & Agentic Top 10)

**What**: Address prompt injection, insecure tool execution, excessive agency, secret handling.

**Design**:
- Prompt-injection guards on tool outputs before re-injection; tool egress allowlists; sandboxed
  expression evaluation (already `simpleeval`); secrets via env/vault refs (never logged — redaction
  covers logs); per-tool network policy. Document mitigations mapped to each OWASP item.

**Testing**:
- `Integration: tool output containing an injection string → flagged/neutralised per policy`.
- `Unit: secrets never appear in trace/audit payloads (redaction asserted)`.
- `Security: dependency scan + bandit/ruff security rules in CI`.

#### 9.3 — Performance, deployment & docs

**What**: Load-test the engine, finalise Docker Compose/Helm, and write operator + API docs.

**Design**:
- Load test: N concurrent executions, measure throughput/latency; ensure trace/event partitions and
  indexes hold up. Provide `docker-compose.yml` (single-node) and a Helm chart (cloud). Auto-generated
  API docs from OpenAPI; quickstart in README.

**Testing**:
- `Load: 100 concurrent executions → p99 step latency within target, no lost executions`.
- `Integration: docker-compose up → healthz green, sample workflow runs end-to-end`.

### Definition of Done
SDKs published; OWASP mitigations documented and tested; one-command self-hosted deploy verified;
docs complete.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (db, auth, RBAC)              ─── required by everything
    │
Phase 2: Agent/Tool Registry + LLM Runtime        ─── requires Phase 1
    │
Phase 3: Durable Workflow Engine                  ─── requires Phase 2   ◀ core value
    ├── Phase 4: Triggers / Webhooks / Cron / Streaming ─── can parallel with Phase 5
    └── Phase 5: Observability / Cost / Governance      ─── can parallel with Phase 4
            │
Phase 6: No-Code Builder & Dashboard UI           ─── requires Phases 3,4,5
Phase 7: MCP & A2A Interoperability               ─── requires Phases 2,3 (parallel with 6)
Phase 8: AI-Native Intelligence                   ─── requires Phases 3,5 (parallel with 6,7)
    │
Phase 9: Hardening, SDKs & Release                ─── requires all prior phases
```

**Parallelism opportunities**
- Phases 4 and 5 can be built concurrently once Phase 3 lands.
- Phases 6, 7, and 8 can proceed in parallel after their dependencies (3/4/5) are met.

**Scope**: large (9 phases, full-stack multi-tenant platform with durable engine + UI + interop).

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; new behaviour has both happy-path and edge-case tests.
3. `ruff check` and `ruff format --check` pass; `mypy --strict` passes (backend); ESLint/Vitest pass (UI).
4. New Alembic migrations are created and `alembic upgrade head` / `downgrade` succeed cleanly.
5. New API endpoints appear in the auto-generated OpenAPI 3.1 spec with request/response schemas.
6. The feature works end-to-end in `docker-compose up` (API + worker + Postgres + Redis [+ UI]).
7. New config options are documented in `.env.example` and the README.
8. Relevant standards alignment is preserved (OTel GenAI attributes, CloudEvents envelope, JSON
   Schema 2020-12 tool defs, MCP/A2A spec versions pinned).
9. Security-sensitive changes are checked against the OWASP LLM/Agentic Top 10 and secrets are never
   logged (redaction asserted in tests).
```
