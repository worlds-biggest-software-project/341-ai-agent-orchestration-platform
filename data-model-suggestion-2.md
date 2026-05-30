# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: AI Agent Orchestration Platform · Created: 2026-05-25

## Philosophy

This model collapses the 17-table normalized schema into 8 core tables by embedding related data as JSONB documents within their parent entities. Organisations embed their user roster, API keys, agent registry, tool catalogue, and governance policies. Workflows embed their graph definition and agent/tool references. Executions embed the complete step-by-step trace including LLM calls, tool calls, checkpoints, and cost attribution. The principle: loading a single execution should return everything needed to debug, replay, or audit it in one query.

The relational anchors — `workflows`, `executions`, and `cost_records` — remain as standalone tables because they have independent lifecycles: workflows are versioned and deployed independently, executions are the primary unit of operational monitoring, and cost records drive billing and chargeback. The `audit_log` remains relational for compliance queryability.

**Best for:** Early-stage orchestration platforms prioritising development speed, single-query execution debugging, and schema flexibility as agent primitives evolve rapidly. Ideal for single-org or small-team deployments where cross-org governance is secondary to developer velocity.

**Trade-offs:**
- (+) 8 tables — minimal migration surface, fast to iterate as agent standards evolve
- (+) Single-row fetch loads a complete execution with all steps, LLM calls, tool calls, and traces
- (+) JSONB flexibility accommodates evolving MCP/A2A protocol specs without schema changes
- (+) Agent and tool registries embedded in org allow rapid iteration without migrations
- (-) Cross-execution analytics (cost per agent, latency by model) require JSONB extraction
- (-) Embedded step arrays grow unbounded for long-running workflows with many iterations
- (-) No foreign-key enforcement on embedded agent/tool references
- (-) Governance policy evaluation requires JSONB parsing at runtime

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry GenAI | OTel attributes embedded in `executions.trace_json` span objects |
| CloudEvents 1.0.2 | Trigger events in `executions.trigger_data` use CloudEvents envelope |
| OpenAPI 3.1 / JSON Schema | Tool schemas embedded in `organisations.tools_json[].schema` |
| MCP | MCP server config in `organisations.agents_json[].mcp_config` |
| A2A | Agent Cards in `organisations.agents_json[].a2a_card` |
| ISO/IEC 42001 | Governance policies in `organisations.policies_json[]` |
| OWASP LLM/Agentic Top 10 | Policy types in `organisations.policies_json[]` |

---

## Core Tables

### organisations

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free','pro','enterprise')),
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"default_model":"claude-sonnet-4-6","token_budget_monthly":10000000,
    --   "pii_redaction_enabled":true,"eu_ai_act_classification":"limited_risk"}

    users_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","email":"dev@example.com","full_name":"Jane Dev",
    --   "role":"developer","is_active":true,"last_login_at":"2026-05-25T10:00:00Z"}]

    api_keys_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"prod-key","key_prefix":"sk-prod-","key_hash":"sha256...",
    --   "scopes":["execute","read"],"expires_at":null,"is_active":true}]

    agents_json     JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"researcher","slug":"researcher","version":1,
    --   "model":"claude-sonnet-4-6","model_provider":"anthropic",
    --   "fallback_models":["gpt-4o"],"system_prompt":"You are a research assistant...",
    --   "temperature":0.7,"max_tokens":4096,"token_budget":50000,
    --   "tool_ids":["uuid1","uuid2"],"role":"Research Analyst","goal":"Find relevant information",
    --   "risk_classification":"limited",
    --   "mcp_config":{"servers":[{"url":"https://mcp.example.com","name":"data-tools"}]},
    --   "a2a_card":{"name":"researcher","capabilities":["search","summarise"]}}]

    tools_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"web_search","slug":"web-search","version":1,
    --   "tool_type":"api","description":"Search the web",
    --   "schema":{"type":"object","properties":{"query":{"type":"string"}},"required":["query"]},
    --   "endpoint_url":"https://api.search.example.com","timeout_ms":10000,
    --   "mcp_server_url":null}]

    policies_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"PII Redaction","policy_type":"pii_redaction",
    --   "config":{"patterns":["email","phone","ssn"],"action":"redact"},
    --   "applies_to":{},"is_active":true},
    --  {"id":"uuid","name":"Token Budget","policy_type":"token_budget",
    --   "config":{"max_tokens_per_execution":100000,"max_cost_cents":500},
    --   "applies_to":{},"is_active":true}]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_slug ON organisations(slug);
CREATE INDEX idx_orgs_agents ON organisations USING GIN (agents_json);
CREATE INDEX idx_orgs_tools ON organisations USING GIN (tools_json);
```

### workflows

Workflows remain relational — they are versioned, deployed independently, and referenced by many executions.

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
    -- Example: {"nodes":[{"id":"n1","type":"agent","agent_slug":"researcher"},
    --   {"id":"n2","type":"agent","agent_slug":"writer"},
    --   {"id":"n3","type":"human_gate","prompt":"Approve draft?"}],
    --  "edges":[{"from":"n1","to":"n2"},{"from":"n2","to":"n3"}],
    --  "entry_node":"n1"}

    input_schema_json JSONB,
    output_schema_json JSONB,
    timeout_ms      BIGINT DEFAULT 3600000,
    retry_policy_json JSONB,
    git_ref         TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug, version)
);

CREATE INDEX idx_workflows_org ON workflows(org_id) WHERE is_active;
```

### executions

The central document — embeds the complete execution trace: steps, LLM calls, tool calls, checkpoints, policy violations, and OTel spans.

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
    idempotency_key TEXT UNIQUE,

    steps_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","node_id":"n1","agent_slug":"researcher",
    --   "step_type":"agent_call","status":"completed","attempt":1,
    --   "input":{"query":"Latest AI agent standards"},
    --   "output":{"results":[...]},
    --   "llm_calls":[{"id":"uuid","model":"claude-sonnet-4-6","provider":"anthropic",
    --     "tokens_input":1200,"tokens_output":800,"tokens_cache_read":0,
    --     "cost_cents":3,"latency_ms":1450,"stop_reason":"end_turn","is_fallback":false}],
    --   "tool_calls":[{"id":"uuid","tool_slug":"web-search",
    --     "input":{"query":"MCP protocol 2026"},"output":{"results":[...]},
    --     "status":"completed","latency_ms":230}],
    --   "tokens_input":1200,"tokens_output":800,"cost_cents":3,
    --   "duration_ms":1680,"started_at":"...","completed_at":"..."}]

    checkpoints_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"version":1,"step_id":"uuid","state":{...},"created_at":"..."}]

    violations_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"policy_id":"uuid","policy_type":"token_budget","severity":"warning",
    --   "details":{"tokens_used":95000,"budget":100000},"action_taken":"warned","at":"..."}]

    trace_json      JSONB NOT NULL DEFAULT '[]',
    -- OTel spans for this execution
    -- Example: [{"trace_id":"abc123","span_id":"def456","operation":"invoke_agent",
    --   "span_kind":"internal","attributes":{"gen_ai.agent.name":"researcher",
    --     "gen_ai.request.model":"claude-sonnet-4-6"},
    --   "start_time":"...","end_time":"...","duration_ms":1680,"status":"ok"}]

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
CREATE INDEX idx_exec_steps ON executions USING GIN (steps_json);
```

### cost_records

Aggregated cost data remains relational for billing queries and chargeback.

```sql
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','monthly')),
    agent_slug      TEXT,
    workflow_slug   TEXT,
    model           TEXT,
    model_provider  TEXT,
    executions_count INTEGER NOT NULL DEFAULT 0,
    tokens_input    BIGINT NOT NULL DEFAULT 0,
    tokens_output   BIGINT NOT NULL DEFAULT 0,
    tokens_cache    BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    tool_calls_count INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  INTEGER,
    error_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, period, period_type, agent_slug, workflow_slug, model)
);

CREATE INDEX idx_cost_org ON cost_records(org_id, period_type, period DESC);
```

### audit_log

```sql
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

## Example Queries

### Load a complete execution with all debugging context

```sql
SELECT e.*, w.name AS workflow_name, w.graph_json
FROM executions e
JOIN workflows w ON w.id = e.workflow_id
WHERE e.id = $1;
```

### Cost per agent this month (JSONB extraction from executions)

```sql
SELECT step->>'agent_slug' AS agent,
       COUNT(DISTINCT e.id) AS executions,
       SUM((step->>'tokens_input')::bigint + (step->>'tokens_output')::bigint) AS total_tokens,
       SUM((step->>'cost_cents')::bigint) AS total_cost_cents
FROM executions e,
     jsonb_array_elements(e.steps_json) AS step
WHERE e.org_id = $1
  AND step->>'step_type' = 'agent_call'
  AND e.created_at >= date_trunc('month', now())
GROUP BY step->>'agent_slug'
ORDER BY total_cost_cents DESC;
```

### Model fallback frequency

```sql
SELECT llm->>'model' AS model, llm->>'model_provider' AS provider,
       COUNT(*) AS call_count,
       COUNT(*) FILTER (WHERE (llm->>'is_fallback')::boolean) AS fallback_count,
       AVG((llm->>'latency_ms')::integer) AS avg_latency_ms
FROM executions e,
     jsonb_array_elements(e.steps_json) AS step,
     jsonb_array_elements(step->'llm_calls') AS llm
WHERE e.org_id = $1
  AND e.created_at >= now() - interval '7 days'
GROUP BY llm->>'model', llm->>'model_provider'
ORDER BY call_count DESC;
```

### Policy violations by type

```sql
SELECT v->>'policy_type' AS policy_type,
       v->>'severity' AS severity,
       COUNT(*) AS violation_count
FROM executions e,
     jsonb_array_elements(e.violations_json) AS v
WHERE e.org_id = $1
  AND e.created_at >= now() - interval '30 days'
GROUP BY v->>'policy_type', v->>'severity'
ORDER BY violation_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation (with users, API keys, agents, tools, policies) | 1 | organisations |
| Workflows | 1 | workflows |
| Executions (with steps, LLM/tool calls, checkpoints, violations, traces) | 1 | executions |
| Cost Records | 1 | cost_records |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **5** | |

---

## Key Design Decisions

1. **Agents and tools embedded in organisation** — the agent/tool registry is org-scoped and changes relatively infrequently. When an execution runs, a snapshot of the agent/tool configuration is embedded in the step (via slug reference), decoupling execution history from future registry changes.

2. **Complete execution as single document** — steps, LLM calls, tool calls, checkpoints, policy violations, and OTel spans all embed inside the execution row. This mirrors the primary access pattern: a developer debugging an execution loads the entire trace in one query.

3. **Governance policies embedded in organisation** — policy rules are org-scoped configuration. The execution engine reads them from the org document at runtime. Violations are recorded inside the execution document, linking policy → violation → step.

4. **Workflows remain relational** — workflows are versioned, deployed independently, and referenced by many executions. The graph definition is JSONB within the workflow row, but the workflow entity itself needs relational linking for version history and deployment tracking.

5. **Cost records remain relational** — billing and chargeback queries need fast aggregation across time periods. A pre-aggregated relational table avoids expensive JSONB extraction across all executions for monthly billing.

6. **OTel spans embedded in execution** — rather than a separate traces table, OTel spans for each execution are embedded in `trace_json`. This keeps all debugging context in one place. For export to external OTel backends (Grafana, Datadog), spans are emitted via OTLP at write time rather than queried from the database.

7. **LLM calls nested inside steps** — each step's `llm_calls` array captures every model invocation including fallbacks. This enables step-level cost attribution and model comparison without a separate join.

8. **Checkpoints as embedded array** — time-travel debugging reads checkpoints from `checkpoints_json` within the execution. For long-running workflows with many checkpoints, only the latest N are kept in the document; older ones are archived.

9. **Agent Cards (A2A) and MCP config as JSONB** — both protocols are evolving rapidly. JSONB fields accommodate spec changes without schema migration. When protocols stabilise, dedicated columns can be extracted.

10. **GIN index on steps** — `steps_json` gets a GIN index to support containment queries (find executions that used a specific tool, find executions where a particular agent failed).
