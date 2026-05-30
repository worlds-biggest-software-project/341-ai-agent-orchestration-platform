# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: AI Agent Orchestration Platform · Created: 2026-05-25

## Philosophy

This model treats every state change in the agent orchestration platform as an immutable event — a workflow execution starting, an agent being invoked, an LLM call completing with token counts, a tool returning results, a governance policy being violated, a human gate being approved. The `event_store` is the single source of truth; all queryable state is materialised into read-model tables via projections (CQRS pattern).

Agent orchestration is a domain uniquely suited to event sourcing. Every step in a multi-agent workflow is already an event — the execution engine processes events to advance workflow state. The audit trail required by ISO 42001, EU AI Act, and NIST AI RMF is not a separate logging layer; it IS the data model. Time-travel debugging is native: replaying events to a specific checkpoint reconstructs the exact execution state at that point. New analytics (cost optimisation, prompt refinement, anomaly detection) can be derived from historical events without schema migration.

Events follow the CloudEvents specification and align with OpenTelemetry GenAI semantic conventions. The event stream is the natural integration surface: inbound webhook events trigger workflows, LLM provider responses are events, and outbound events can be published to OTel collectors, audit sinks, and billing systems.

**Best for:** Production-grade orchestration platforms serving regulated enterprises, platforms that need tamper-proof audit trails for compliance certification, and teams that want to derive new AI-improvement analytics (prompt optimisation, fallback effectiveness, cost forecasting) from historical execution data.

**Trade-offs:**
- (+) Complete, immutable audit trail — ISO 42001 and EU AI Act compliance by construction
- (+) Time-travel debugging is native event replay, not a separate feature
- (+) New read models derived from existing events as analytics needs evolve
- (+) Natural alignment with OTel span emission and CloudEvents integration
- (+) Execution cost attribution is event-level precise, not estimated
- (-) Eventual consistency — read models lag behind event writes
- (-) Event schema evolution requires careful versioning (upcasting)
- (-) Higher storage volume than state-only models
- (-) Debugging requires event replay rather than inspecting current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0.2 | `event_store` columns: `ce_source`, `ce_type`, `ce_specversion`, `ce_time` |
| OpenTelemetry GenAI | Event payloads use OTel attribute names; spans materialised from events |
| OpenAPI 3.1 / JSON Schema | Tool schemas in `tool_registered` event payloads |
| MCP | MCP configuration in `agent_registered` event payloads |
| A2A | Agent Card in `agent_registered` event payloads |
| AsyncAPI 3.1 | Event channel design follows AsyncAPI patterns |
| W3C PROV-DM | Events map to PROV Activities (executions), Entities (inputs/outputs), Agents (actors) |
| ISO/IEC 42001 | Event store IS the AIMS audit trail |
| NIST AI RMF 1.0 | Governance events map to Govern/Measure/Manage functions |
| OWASP LLM/Agentic Top 10 | Policy violation events cover prompt injection, excessive agency, rogue agent |
| EU AI Act | Risk classification in agent events; immutable audit for conformity assessment |

---

## Infrastructure Tables

### event_store

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'execution','agent','tool','workflow',
                        'governance','org','billing')),
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL DEFAULT '/agent-orchestration',
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),

    org_id          UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user','agent','system','api_key')),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_es_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_es_org ON event_store(org_id, created_at DESC);
CREATE INDEX idx_es_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_es_actor ON event_store(actor_id, created_at DESC);
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Catalogue

### Execution Stream (`stream_type = 'execution'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `execution_started` | workflow_id, workflow_version, trigger_type, trigger_data, input, idempotency_key | Workflow run begins |
| `step_started` | node_id, agent_id, step_type, input | Graph node entered |
| `llm_call_started` | step_id, model, provider, temperature, max_tokens | LLM invocation begins |
| `llm_call_completed` | step_id, model, provider, tokens_input, tokens_output, tokens_cache_read, cost_cents, latency_ms, stop_reason, is_fallback | LLM response received |
| `llm_call_failed` | step_id, model, provider, error, latency_ms | LLM error |
| `llm_fallback_triggered` | step_id, failed_model, fallback_model, reason | Fallback chain activated |
| `tool_call_started` | step_id, tool_id, input | Tool invocation begins |
| `tool_call_completed` | step_id, tool_id, output, latency_ms | Tool response |
| `tool_call_failed` | step_id, tool_id, error, latency_ms | Tool error |
| `step_completed` | node_id, output, tokens_total, cost_cents, duration_ms | Graph node done |
| `step_failed` | node_id, error, attempt | Step failure |
| `step_retrying` | node_id, attempt, delay_ms | Retry scheduled |
| `human_gate_reached` | node_id, prompt, context | Waiting for human |
| `human_gate_approved` | node_id, approved_by, modifications | Human approved |
| `human_gate_rejected` | node_id, rejected_by, reason | Human rejected |
| `checkpoint_created` | version, state_snapshot | State checkpoint |
| `execution_completed` | output, total_tokens, total_cost_cents, duration_ms | Success |
| `execution_failed` | error, total_tokens, total_cost_cents | Failure |
| `execution_cancelled` | cancelled_by, reason | Manual cancellation |
| `execution_timed_out` | timeout_ms | Timeout |

### Agent Stream (`stream_type = 'agent'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `agent_registered` | name, slug, version, model, provider, system_prompt, tools[], mcp_config, a2a_card, risk_classification | Agent created/updated |
| `agent_version_created` | slug, version, changes | New version |
| `agent_deactivated` | slug, reason | Agent disabled |
| `agent_model_changed` | slug, old_model, new_model, reason | Model swap |
| `agent_prompt_updated` | slug, version, prompt_hash | Prompt change |

### Tool Stream (`stream_type = 'tool'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `tool_registered` | name, slug, version, tool_type, schema_json, endpoint_url, mcp_server_url | Tool created |
| `tool_version_created` | slug, version, schema_changes | Schema update |
| `tool_deactivated` | slug, reason | Tool disabled |

### Workflow Stream (`stream_type = 'workflow'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `workflow_created` | name, slug, version, trigger_type, graph_json, input_schema, output_schema | Workflow defined |
| `workflow_version_created` | slug, version, graph_changes | Version bump |
| `workflow_activated` | slug, version | Set as active |
| `workflow_deactivated` | slug, reason | Disabled |

### Governance Stream (`stream_type = 'governance'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `policy_created` | name, policy_type, config, applies_to | New policy |
| `policy_updated` | policy_id, old_config, new_config | Policy change |
| `policy_violated` | policy_id, execution_id, step_id, violation_type, severity, details, action_taken | Violation detected |
| `pii_redacted` | execution_id, step_id, field_path, pattern_matched | PII redaction applied |
| `token_budget_warning` | execution_id, tokens_used, budget, pct | Approaching limit |
| `token_budget_exceeded` | execution_id, tokens_used, budget | Budget blown |
| `prompt_injection_detected` | execution_id, step_id, confidence, input_hash | Injection attempt |
| `excessive_agency_flagged` | execution_id, step_id, tool_calls_count, threshold | Too many tool calls |
| `anomaly_detected` | agent_id, anomaly_type, baseline, observed, confidence | Drift/hallucination |

### Org Stream (`stream_type = 'org'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `org_created` | name, slug, plan | Organisation setup |
| `user_added` | user_id, email, role | Team member added |
| `user_role_changed` | user_id, old_role, new_role | Permission change |
| `api_key_created` | key_id, name, scopes | Key issued |
| `api_key_revoked` | key_id, reason | Key revoked |
| `settings_updated` | changed_fields | Org config change |

### Billing Stream (`stream_type = 'billing'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `cost_incurred` | execution_id, agent_id, model, provider, tokens_input, tokens_output, cost_cents | Per-call cost |
| `daily_usage_aggregated` | date, total_tokens, total_cost_cents, executions_count, breakdown | Daily rollup |
| `budget_alert` | budget_type, used, limit, pct | Budget threshold hit |
| `invoice_generated` | period, total_cost_cents, line_items | Monthly invoice |

---

## Read Model Tables

### rm_executions

```sql
CREATE TABLE rm_executions (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    workflow_id     UUID NOT NULL,
    workflow_name   TEXT,
    workflow_version INTEGER NOT NULL,
    status          TEXT NOT NULL,
    trigger_type    TEXT,
    input           JSONB,
    output          JSONB,
    error           JSONB,
    current_node    TEXT,
    steps           JSONB NOT NULL DEFAULT '[]',
    total_tokens    BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    step_count      INTEGER NOT NULL DEFAULT 0,
    llm_call_count  INTEGER NOT NULL DEFAULT 0,
    tool_call_count INTEGER NOT NULL DEFAULT 0,
    violation_count INTEGER NOT NULL DEFAULT 0,
    models_used     TEXT[],
    agents_used     TEXT[],
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_exec_org ON rm_executions(org_id, status);
CREATE INDEX idx_rm_exec_workflow ON rm_executions(workflow_id, created_at DESC);
CREATE INDEX idx_rm_exec_models ON rm_executions USING GIN (models_used);
CREATE INDEX idx_rm_exec_agents ON rm_executions USING GIN (agents_used);
```

### rm_agent_performance

```sql
CREATE TABLE rm_agent_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    agent_slug      TEXT NOT NULL,
    agent_version   INTEGER NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','weekly','monthly')),

    invocations     INTEGER NOT NULL DEFAULT 0,
    successes       INTEGER NOT NULL DEFAULT 0,
    failures        INTEGER NOT NULL DEFAULT 0,
    success_rate_pct NUMERIC(5,2) NOT NULL DEFAULT 0,

    tokens_input    BIGINT NOT NULL DEFAULT 0,
    tokens_output   BIGINT NOT NULL DEFAULT 0,
    tokens_cache    BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    avg_cost_cents  BIGINT NOT NULL DEFAULT 0,

    avg_latency_ms  INTEGER,
    p50_latency_ms  INTEGER,
    p99_latency_ms  INTEGER,

    tool_calls      INTEGER NOT NULL DEFAULT 0,
    fallback_count  INTEGER NOT NULL DEFAULT 0,
    fallback_rate_pct NUMERIC(5,2) NOT NULL DEFAULT 0,

    top_tools       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"tool":"web_search","calls":42,"avg_latency":230},
    --   {"tool":"code_sandbox","calls":15,"avg_latency":1200}]

    model_breakdown JSONB NOT NULL DEFAULT '{}',
    -- Example: {"claude-sonnet-4-6":{"calls":50,"tokens":120000,"cost_cents":360},
    --   "gpt-4o":{"calls":5,"tokens":15000,"cost_cents":45,"all_fallback":true}}

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, agent_slug, agent_version, period, period_type)
);

CREATE INDEX idx_rm_agent_perf ON rm_agent_performance(org_id, period_type, agent_slug, period DESC);
```

### rm_cost_dashboard

```sql
CREATE TABLE rm_cost_dashboard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','weekly','monthly')),

    total_cost_cents     BIGINT NOT NULL DEFAULT 0,
    total_tokens         BIGINT NOT NULL DEFAULT 0,
    total_executions     INTEGER NOT NULL DEFAULT 0,
    total_llm_calls      INTEGER NOT NULL DEFAULT 0,
    total_tool_calls     INTEGER NOT NULL DEFAULT 0,

    cost_by_model   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"claude-sonnet-4-6":{"cost":36000,"tokens":1200000,"calls":500},
    --   "gpt-4o":{"cost":4500,"tokens":150000,"calls":50}}

    cost_by_agent   JSONB NOT NULL DEFAULT '{}',
    cost_by_workflow JSONB NOT NULL DEFAULT '{}',
    cost_by_tool    JSONB NOT NULL DEFAULT '{}',

    budget_used_pct NUMERIC(5,2),
    projected_monthly_cost_cents BIGINT,

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, period, period_type)
);

CREATE INDEX idx_rm_cost ON rm_cost_dashboard(org_id, period_type, period DESC);
```

### rm_governance

```sql
CREATE TABLE rm_governance (
    org_id          UUID PRIMARY KEY,

    -- Policy violations
    violations_24h  INTEGER NOT NULL DEFAULT 0,
    violations_7d   INTEGER NOT NULL DEFAULT 0,
    violations_30d  INTEGER NOT NULL DEFAULT 0,
    critical_violations_30d INTEGER NOT NULL DEFAULT 0,

    violations_by_type JSONB NOT NULL DEFAULT '{}',
    -- Example: {"pii_redaction":{"count":12,"action":"redacted"},
    --   "token_budget":{"count":3,"action":"warned"},
    --   "prompt_injection":{"count":1,"action":"blocked"}}

    -- Agent health
    agents_with_high_fallback INTEGER NOT NULL DEFAULT 0,
    agents_with_anomalies INTEGER NOT NULL DEFAULT 0,
    agents_high_risk INTEGER NOT NULL DEFAULT 0,

    -- Compliance
    active_policies INTEGER NOT NULL DEFAULT 0,
    pii_redaction_enabled BOOLEAN NOT NULL DEFAULT false,
    eu_ai_act_classification TEXT,
    last_audit_export TIMESTAMPTZ,

    -- Security
    api_keys_active INTEGER NOT NULL DEFAULT 0,
    api_keys_expired INTEGER NOT NULL DEFAULT 0,
    users_without_mfa INTEGER NOT NULL DEFAULT 0,

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### rm_org_dashboard

```sql
CREATE TABLE rm_org_dashboard (
    org_id          UUID PRIMARY KEY,
    org_name        TEXT NOT NULL,

    -- Overview
    active_agents   INTEGER NOT NULL DEFAULT 0,
    active_tools    INTEGER NOT NULL DEFAULT 0,
    active_workflows INTEGER NOT NULL DEFAULT 0,
    active_users    INTEGER NOT NULL DEFAULT 0,

    -- Today
    executions_today INTEGER NOT NULL DEFAULT 0,
    running_now     INTEGER NOT NULL DEFAULT 0,
    waiting_human   INTEGER NOT NULL DEFAULT 0,
    failed_today    INTEGER NOT NULL DEFAULT 0,
    cost_today_cents BIGINT NOT NULL DEFAULT 0,

    -- Trends
    cost_trend_7d   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"date":"2026-05-19","cost":4200},{"date":"2026-05-20","cost":3800}]

    exec_trend_7d   JSONB NOT NULL DEFAULT '[]',
    error_rate_7d   NUMERIC(5,2),

    -- Alerts
    alerts          JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type":"budget_warning","message":"75% of monthly token budget used","severity":"warning"},
    --   {"type":"agent_anomaly","agent":"researcher","message":"Fallback rate 40% (baseline 5%)","severity":"high"}]

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Event Replay Queries

### Time-travel: reconstruct execution state at a checkpoint

```sql
SELECT payload
FROM event_store
WHERE stream_type = 'execution' AND stream_id = $1
  AND created_at <= (
      SELECT ce_time FROM event_store
      WHERE stream_type = 'execution' AND stream_id = $1 AND event_type = 'checkpoint_created'
        AND (payload->>'version')::integer = $2
      LIMIT 1
  )
ORDER BY version ASC;
```

### Model cost comparison (which models are most cost-effective?)

```sql
SELECT payload->>'model' AS model, payload->>'provider' AS provider,
       COUNT(*) AS calls,
       SUM((payload->>'tokens_input')::integer + (payload->>'tokens_output')::integer) AS total_tokens,
       SUM((payload->>'cost_cents')::bigint) AS total_cost_cents,
       AVG((payload->>'latency_ms')::integer) AS avg_latency_ms,
       COUNT(*) FILTER (WHERE (payload->>'is_fallback')::boolean) AS fallback_calls
FROM event_store
WHERE stream_type = 'execution' AND event_type = 'llm_call_completed'
  AND org_id = $1
  AND ce_time >= now() - interval '30 days'
GROUP BY payload->>'model', payload->>'provider'
ORDER BY total_cost_cents DESC;
```

### Fallback chain effectiveness

```sql
WITH fallbacks AS (
    SELECT payload->>'execution_id' AS exec_id,
           payload->>'failed_model' AS failed_model,
           payload->>'fallback_model' AS fallback_model,
           payload->>'reason' AS reason,
           ce_time
    FROM event_store
    WHERE stream_type = 'execution' AND event_type = 'llm_fallback_triggered'
      AND org_id = $1 AND ce_time >= now() - interval '7 days'
)
SELECT failed_model, fallback_model, reason,
       COUNT(*) AS fallback_count
FROM fallbacks
GROUP BY failed_model, fallback_model, reason
ORDER BY fallback_count DESC;
```

### Governance audit — all policy violations in period

```sql
SELECT ce_time, payload->>'policy_type' AS policy_type,
       payload->>'severity' AS severity,
       payload->>'execution_id' AS execution_id,
       payload->>'action_taken' AS action_taken,
       payload->'details' AS details
FROM event_store
WHERE stream_type = 'governance'
  AND event_type = 'policy_violated'
  AND org_id = $1
  AND ce_time BETWEEN $2 AND $3
ORDER BY ce_time DESC;
```

### Agent prompt optimisation signal (failure patterns by agent)

```sql
WITH agent_failures AS (
    SELECT payload->>'node_id' AS agent_node,
           payload->'error' AS error,
           ce_time
    FROM event_store
    WHERE stream_type = 'execution' AND event_type = 'step_failed'
      AND org_id = $1 AND ce_time >= now() - interval '14 days'
)
SELECT agent_node,
       COUNT(*) AS failure_count,
       jsonb_agg(DISTINCT error->>'type') AS error_types,
       MIN(ce_time) AS first_failure,
       MAX(ce_time) AS last_failure
FROM agent_failures
GROUP BY agent_node
HAVING COUNT(*) >= 3
ORDER BY failure_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_executions, rm_agent_performance, rm_cost_dashboard, rm_governance, rm_org_dashboard |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Seven stream types cover the full platform** — execution, agent, tool, workflow, governance, org, and billing. The execution stream is highest volume (every LLM call, tool call, and step transition). The governance stream is the compliance surface. The billing stream tracks cost events independently for financial reconciliation.

2. **Execution events at LLM-call granularity** — `llm_call_started`, `llm_call_completed`, and `llm_call_failed` events capture individual model invocations with token counts, cost, latency, and fallback status. This is the foundation for cost attribution, model comparison, and prompt optimisation signals.

3. **Fallback chain as event sequence** — `llm_call_failed` → `llm_fallback_triggered` → `llm_call_completed` events document the exact fallback path. The read model aggregates fallback frequency and effectiveness per model pair.

4. **Governance stream as independent audit surface** — policy violations, PII redactions, prompt injection detections, and anomalies live in their own stream. Compliance auditors can query this stream without accessing execution payloads — essential for ISO 42001 and EU AI Act conformity assessment.

5. **Time-travel debugging by construction** — `checkpoint_created` events capture execution state snapshots. Replaying events up to any checkpoint reconstructs the exact state at that point. No separate time-travel feature needed.

6. **Billing events enable precise cost attribution** — every `cost_incurred` event links an execution, agent, model, and token count. The `rm_cost_dashboard` aggregates these into per-model, per-agent, per-workflow cost breakdowns for chargeback.

7. **Agent performance read model** — `rm_agent_performance` tracks per-agent success rates, latency distributions, tool usage patterns, model breakdowns, and fallback rates. This enables the "agent specialisation discovery" and "prompt refinement" analytics that no analysed incumbent offers.

8. **CloudEvents envelope on every event** — `ce_source`, `ce_type`, `ce_specversion`, `ce_time` enable event routing to external systems (OTel collectors, Kafka topics, webhook subscribers) without format translation.

9. **OTel spans materialised from events** — rather than a separate trace store, OTel-compatible spans are materialised from execution events into the `rm_executions` read model. Spans can also be emitted via OTLP at event-write time for real-time observability in Grafana/Datadog.

10. **W3C PROV-DM alignment** — execution events map naturally to PROV-DM: executions are Activities, inputs/outputs are Entities, agents and users are PROV Agents. This enables provenance export for reproducibility and compliance documentation.
