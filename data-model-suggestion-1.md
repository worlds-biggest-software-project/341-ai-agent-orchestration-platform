# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AI Agent Orchestration Platform · Created: 2026-05-25

## Philosophy

This model assigns a dedicated table to every first-class concept in the agent orchestration domain — organisations, agents, tools, workflows, executions, steps, LLM calls, traces, governance policies, and cost records each live in their own table with strict foreign-key relationships. The design separates the definition layer (agents, tools, workflows) from the execution layer (runs, steps, LLM calls, traces) and the governance layer (policies, audit events, redaction rules), enabling rich cross-dimensional queries: cost per agent per model, execution latency by tool, token consumption by workflow version, and policy violation rates by team.

The schema aligns with industry standards wherever possible: traces follow OpenTelemetry GenAI semantic conventions, events use CloudEvents envelopes, tool definitions reference OpenAPI/JSON Schema, and agent interoperability surfaces align with MCP and A2A protocol structures. The governance layer maps to ISO/IEC 42001 AIMS requirements and NIST AI RMF functions.

**Best for:** Platform teams running multi-tenant agent infrastructure at scale, regulated enterprises needing queryable compliance surfaces (EU AI Act, ISO 42001), and organisations that need per-agent cost attribution with chargeback to business units.

**Trade-offs:**
- (+) Full referential integrity — no orphaned executions, orphaned steps, or untracked LLM calls
- (+) Every compliance surface (audit, redaction, policy violations) has its own queryable table
- (+) Standard SQL joins for any KPI (cost per agent, latency by tool, tokens by model)
- (+) Natural fit for OpenTelemetry span export and CloudEvents ingestion
- (-) 20 tables is significant migration surface
- (-) High-volume execution traces require careful partitioning strategy
- (-) Step-level granularity generates many rows per execution

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry GenAI Semantic Conventions | `traces` and `spans` tables use OTel attribute names (`gen_ai.agent.name`, `gen_ai.request.model`) |
| CloudEvents 1.0.2 | `events` table columns: `ce_source`, `ce_type`, `ce_specversion`, `ce_time` |
| OpenAPI 3.1 / JSON Schema 2020-12 | `tools.schema_json` stores tool input/output schemas in OpenAPI format |
| MCP (Model Context Protocol) | `tools.mcp_server_url`, `agents.mcp_config_json` for MCP tool integration |
| A2A (Agent2Agent Protocol) | `agents.a2a_card_json` stores Agent Card for cross-org discovery |
| AsyncAPI 3.1 | Informs event channel design; not stored directly |
| W3C PROV-DM | Execution traces map to PROV Activities, agent calls to PROV Agents |
| ISO/IEC 42001 | `governance_policies` and `audit_log` support AIMS requirements |
| NIST AI RMF 1.0 | Governance layer maps to Govern/Map/Measure/Manage functions |
| OWASP LLM Top 10 / Agentic Top 10 | `governance_policies.policy_type` includes prompt_injection, excessive_agency, rogue_agent |
| EU AI Act | `agents.risk_classification`, `audit_log` for conformity assessment |

---

## Organisation & Teams

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free','pro','enterprise')),
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"default_model":"claude-sonnet-4-6","token_budget_monthly":10000000,
    --   "pii_redaction_enabled":true,"eu_ai_act_classification":"limited_risk"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner','admin','developer','viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

CREATE INDEX idx_users_org ON users(org_id) WHERE is_active;

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL UNIQUE,
    key_prefix      CHAR(8) NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{execute,read}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_apikeys_org ON api_keys(org_id) WHERE is_active;
```

## Agent & Tool Registry

```sql
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    description     TEXT,
    system_prompt   TEXT,
    model           TEXT NOT NULL,
    model_provider  TEXT NOT NULL CHECK (model_provider IN ('anthropic','openai','google',
                        'azure_openai','bedrock','ollama','custom')),
    fallback_models TEXT[] DEFAULT '{}',
    temperature     NUMERIC(3,2) DEFAULT 0.7,
    max_tokens      INTEGER,
    token_budget    INTEGER,
    tool_ids        UUID[] DEFAULT '{}',
    role            TEXT,
    goal            TEXT,
    backstory       TEXT,
    mcp_config_json JSONB,
    -- Example: {"servers":[{"url":"https://mcp.example.com","name":"data-tools",
    --   "transport":"sse","auth":{"type":"bearer","token_ref":"vault://mcp/data-tools"}}]}
    a2a_card_json   JSONB,
    -- Example: {"name":"research-agent","description":"Searches and summarises documents",
    --   "url":"https://agents.example.com/research","version":"1.0",
    --   "capabilities":["search","summarise"],"authentication":{"schemes":["bearer"]}}
    risk_classification TEXT CHECK (risk_classification IN ('minimal','limited','high','unacceptable')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug, version)
);

CREATE INDEX idx_agents_org ON agents(org_id) WHERE is_active;

CREATE TABLE tools (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    description     TEXT NOT NULL,
    tool_type       TEXT NOT NULL CHECK (tool_type IN ('function','api','mcp','code_sandbox',
                                                       'browser','file_system','database','custom')),
    schema_json     JSONB NOT NULL,
    -- OpenAPI-compatible JSON Schema for input parameters and output
    -- Example: {"type":"object","properties":{"query":{"type":"string","description":"Search query"}},
    --   "required":["query"],"returns":{"type":"array","items":{"type":"object",
    --     "properties":{"title":{"type":"string"},"url":{"type":"string"},"snippet":{"type":"string"}}}}}
    endpoint_url    TEXT,
    mcp_server_url  TEXT,
    auth_config_json JSONB,
    timeout_ms      INTEGER DEFAULT 30000,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug, version)
);

CREATE INDEX idx_tools_org ON tools(org_id) WHERE is_active;
```

## Workflow Definitions

```sql
CREATE TABLE workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    description     TEXT,
    trigger_type    TEXT NOT NULL CHECK (trigger_type IN ('api','webhook','event','cron','manual')),
    cron_expression TEXT,
    graph_json      JSONB NOT NULL,
    -- Example: {"nodes":[{"id":"n1","type":"agent","agent_id":"uuid","name":"researcher"},
    --   {"id":"n2","type":"agent","agent_id":"uuid","name":"writer"},
    --   {"id":"n3","type":"human_gate","prompt":"Approve draft?"}],
    --  "edges":[{"from":"n1","to":"n2","condition":null},
    --   {"from":"n2","to":"n3","condition":null}],
    --  "entry_node":"n1"}
    input_schema_json JSONB,
    output_schema_json JSONB,
    timeout_ms      BIGINT DEFAULT 3600000,
    retry_policy_json JSONB,
    -- Example: {"max_retries":3,"backoff":"exponential","initial_delay_ms":1000,"max_delay_ms":60000}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    git_ref         TEXT,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug, version)
);

CREATE INDEX idx_workflows_org ON workflows(org_id) WHERE is_active;
CREATE INDEX idx_workflows_trigger ON workflows(org_id, trigger_type);
```

## Execution Runtime

```sql
CREATE TABLE executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    workflow_id     UUID NOT NULL REFERENCES workflows(id),
    workflow_version INTEGER NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','running','waiting_human','paused',
                                      'completed','failed','cancelled','timed_out')),
    trigger_type    TEXT NOT NULL,
    trigger_data    JSONB,
    input           JSONB,
    output          JSONB,
    error           JSONB,
    current_node    TEXT,
    state_json      JSONB NOT NULL DEFAULT '{}',
    idempotency_key TEXT UNIQUE,
    total_tokens    BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exec_org ON executions(org_id, status);
CREATE INDEX idx_exec_workflow ON executions(workflow_id, created_at DESC);
CREATE INDEX idx_exec_created ON executions(org_id, created_at DESC);

CREATE TABLE steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL REFERENCES executions(id) ON DELETE CASCADE,
    node_id         TEXT NOT NULL,
    agent_id        UUID REFERENCES agents(id),
    step_type       TEXT NOT NULL CHECK (step_type IN ('agent_call','tool_call','llm_call',
                        'human_gate','conditional','parallel_fork','parallel_join','subworkflow')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','running','waiting_human','completed',
                                      'failed','skipped','retrying')),
    input           JSONB,
    output          JSONB,
    error           JSONB,
    attempt         SMALLINT NOT NULL DEFAULT 1,
    tokens_input    INTEGER NOT NULL DEFAULT 0,
    tokens_output   INTEGER NOT NULL DEFAULT 0,
    cost_cents      BIGINT NOT NULL DEFAULT 0,
    duration_ms     INTEGER,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_exec ON steps(execution_id, created_at);
CREATE INDEX idx_steps_agent ON steps(agent_id);

CREATE TABLE llm_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id         UUID NOT NULL REFERENCES steps(id) ON DELETE CASCADE,
    execution_id    UUID NOT NULL REFERENCES executions(id),
    org_id          UUID NOT NULL,
    agent_id        UUID REFERENCES agents(id),
    model           TEXT NOT NULL,
    model_provider  TEXT NOT NULL,
    is_fallback     BOOLEAN NOT NULL DEFAULT false,
    tokens_input    INTEGER NOT NULL DEFAULT 0,
    tokens_output   INTEGER NOT NULL DEFAULT 0,
    tokens_cache_read INTEGER NOT NULL DEFAULT 0,
    cost_cents      BIGINT NOT NULL DEFAULT 0,
    temperature     NUMERIC(3,2),
    max_tokens      INTEGER,
    stop_reason     TEXT CHECK (stop_reason IN ('end_turn','max_tokens','tool_use','stop_sequence','error')),
    latency_ms      INTEGER,
    streaming       BOOLEAN NOT NULL DEFAULT false,
    request_hash    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_llm_exec ON llm_calls(execution_id);
CREATE INDEX idx_llm_org ON llm_calls(org_id, created_at DESC);
CREATE INDEX idx_llm_model ON llm_calls(org_id, model, created_at DESC);

CREATE TABLE tool_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id         UUID NOT NULL REFERENCES steps(id) ON DELETE CASCADE,
    execution_id    UUID NOT NULL REFERENCES executions(id),
    tool_id         UUID NOT NULL REFERENCES tools(id),
    input           JSONB NOT NULL,
    output          JSONB,
    error           JSONB,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','running','completed','failed','timed_out')),
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_toolcall_exec ON tool_calls(execution_id);
CREATE INDEX idx_toolcall_tool ON tool_calls(tool_id, created_at DESC);

CREATE TABLE checkpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL REFERENCES executions(id) ON DELETE CASCADE,
    step_id         UUID REFERENCES steps(id),
    state_json      JSONB NOT NULL,
    version         INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ckpt_exec ON checkpoints(execution_id, version DESC);
```

## Observability

```sql
CREATE TABLE traces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    execution_id    UUID NOT NULL REFERENCES executions(id),
    trace_id        TEXT NOT NULL,
    parent_span_id  TEXT,
    span_id         TEXT NOT NULL,
    operation_name  TEXT NOT NULL,
    span_kind       TEXT NOT NULL CHECK (span_kind IN ('client','server','internal','producer','consumer')),
    status_code     TEXT CHECK (status_code IN ('unset','ok','error')),
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- OTel GenAI attributes: gen_ai.agent.name, gen_ai.request.model,
    --   gen_ai.usage.input_tokens, gen_ai.usage.output_tokens, gen_ai.response.finish_reason
    events          JSONB NOT NULL DEFAULT '[]',
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_traces_exec ON traces(execution_id);
CREATE INDEX idx_traces_org ON traces(org_id, created_at DESC);
CREATE INDEX idx_traces_trace ON traces(trace_id);
CREATE INDEX idx_traces_attributes ON traces USING GIN (attributes);

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    subject         TEXT,
    data            JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_org ON events(org_id, ce_type, ce_time DESC);
CREATE INDEX idx_events_subject ON events(subject, ce_time DESC);
```

## Governance & Cost

```sql
CREATE TABLE governance_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    policy_type     TEXT NOT NULL CHECK (policy_type IN ('pii_redaction','token_budget',
                        'model_allowlist','tool_allowlist','prompt_injection_guard',
                        'excessive_agency','output_validation','rate_limit','cost_limit',
                        'human_approval_required','data_residency')),
    config_json     JSONB NOT NULL,
    -- Example (pii_redaction): {"patterns":["email","phone","ssn","credit_card"],"action":"redact"}
    -- Example (token_budget): {"max_tokens_per_execution":100000,"max_cost_cents_per_execution":500}
    -- Example (model_allowlist): {"allowed_models":["claude-sonnet-4-6","gpt-4o"],"deny_action":"block"}
    applies_to      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"agents":["uuid1","uuid2"],"workflows":["uuid3"]} or {} for all
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policies_org ON governance_policies(org_id) WHERE is_active;

CREATE TABLE policy_violations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    policy_id       UUID NOT NULL REFERENCES governance_policies(id),
    execution_id    UUID REFERENCES executions(id),
    step_id         UUID REFERENCES steps(id),
    violation_type  TEXT NOT NULL,
    severity        TEXT NOT NULL CHECK (severity IN ('info','warning','error','critical')),
    details         JSONB NOT NULL,
    action_taken    TEXT NOT NULL CHECK (action_taken IN ('logged','warned','blocked','redacted')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_violations_org ON policy_violations(org_id, created_at DESC);
CREATE INDEX idx_violations_exec ON policy_violations(execution_id);

CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','monthly')),
    agent_id        UUID REFERENCES agents(id),
    workflow_id     UUID REFERENCES workflows(id),
    model           TEXT,
    model_provider  TEXT,
    executions_count INTEGER NOT NULL DEFAULT 0,
    tokens_input    BIGINT NOT NULL DEFAULT 0,
    tokens_output   BIGINT NOT NULL DEFAULT 0,
    tokens_cache    BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    tool_calls_count INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  INTEGER,
    p99_latency_ms  INTEGER,
    error_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, period, period_type, agent_id, workflow_id, model)
);

CREATE INDEX idx_cost_org ON cost_records(org_id, period_type, period DESC);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user','agent','system','api_key')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes_json    JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Teams | 3 | organisations, users, api_keys |
| Agent & Tool Registry | 2 | agents, tools |
| Workflow Definitions | 1 | workflows |
| Execution Runtime | 5 | executions, steps, llm_calls, tool_calls, checkpoints |
| Observability | 2 | traces (partitioned), events (partitioned) |
| Governance & Cost | 4 | governance_policies, policy_violations, cost_records, audit_log |
| **Total** | **17** | |

---

## Key Design Decisions

1. **Agents and tools as versioned registries** — `agents` and `tools` use `(org_id, slug, version)` uniqueness, enabling version rollback and A/B testing of agent configurations. Executions reference the specific version that ran, not the latest.

2. **OpenAPI/JSON Schema for tool definitions** — `tools.schema_json` stores the tool's input/output schema in OpenAPI-compatible format. This is the universal language for agent tool descriptions across MCP, Bedrock, and ADK ecosystems.

3. **MCP and A2A as first-class fields** — `agents.mcp_config_json` defines which MCP servers an agent connects to. `agents.a2a_card_json` stores the A2A Agent Card for cross-org discovery. Both are JSONB to accommodate evolving protocol specs.

4. **Graph-based workflow definitions** — `workflows.graph_json` stores the DAG/DCG as nodes and edges with conditional routing. This supports sequential, hierarchical, and parallel agent topologies without a separate graph storage layer.

5. **Four-level execution hierarchy** — execution → steps → llm_calls/tool_calls provides three levels of granularity. Executions are the top-level workflow run. Steps map to individual graph nodes. LLM and tool calls are the atomic operations within a step. This mirrors the OTel span hierarchy.

6. **LLM call cost tracking with fallback chain** — `llm_calls` records every model invocation with token counts (input, output, cache), cost, latency, and whether it was a fallback. This enables per-model cost attribution and fallback chain effectiveness analysis.

7. **OpenTelemetry-native traces** — the `traces` table uses OTel attribute names (`gen_ai.agent.name`, `gen_ai.request.model`) and span structure. Traces can be exported to any OTel-compatible backend (Grafana, Datadog, Honeycomb) without field mapping.

8. **CloudEvents for event bus** — the `events` table uses the CloudEvents envelope, enabling integration with external event systems (Kafka, webhooks) and providing a standardised format for workflow triggers and inter-agent messages.

9. **Governance policies as data-driven rules** — `governance_policies` defines rules (PII redaction patterns, token budgets, model allowlists, prompt injection guards) as JSONB configuration. The execution engine evaluates policies at runtime; violations are recorded in `policy_violations` with the action taken.

10. **EU AI Act risk classification** — `agents.risk_classification` captures the EU AI Act risk level (minimal, limited, high, unacceptable). Combined with `audit_log` retention, this supports the conformity assessment documentation required for high-risk AI systems.

11. **Cost records for chargeback** — `cost_records` aggregates token consumption and cost by agent, workflow, model, and period. This enables per-team cost attribution and monthly chargeback reporting without querying the high-volume `llm_calls` table.

12. **Partitioned high-volume tables** — `traces`, `events`, and `audit_log` are partitioned by time range. These are write-heavy, append-only tables that grow with execution volume; partitioning keeps recent queries fast.
