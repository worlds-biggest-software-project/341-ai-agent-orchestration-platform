# Standards & API Reference

> Project: AI Agent Orchestration Platform · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 42001:2023 — AI Management Systems**
- URL: https://www.iso.org/standard/42001
- The world's first international management system standard specifically for AI. Specifies requirements for establishing, implementing, maintaining, and continually improving an Artificial Intelligence Management System (AIMS) within organisations. Directly relevant to any agent orchestration platform seeking enterprise adoption in regulated industries; as of 2026 it is the de facto gold standard for AI governance certification, with the Colorado AI Act (effective June 2026) treating compliance as an affirmative defence.

**ISO/IEC 23053 — Framework for AI Systems Using Machine Learning**
- URL: https://www.iso.org/standard/74438.html
- Establishes a framework for describing AI systems built on ML pipelines, including lifecycle phases, component roles, and design considerations. Relevant to the internal architecture of agent execution engines and how agents, tools, and models are defined and composed.

**ISO/IEC 27001 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The foundational security management standard. Required by enterprise procurement teams evaluating orchestration platforms; particularly relevant for platforms handling sensitive payloads, secrets, or personally identifiable information across agent pipelines.

---

### W3C & IETF Standards

**W3C PROV-DM — Provenance Data Model**
- URL: https://www.w3.org/TR/prov-dm/
- Defines a data model for expressing provenance of data and processes in terms of entities, activities, and agents. Directly applicable to agent workflow audit trails: each agent execution can be modelled as a PROV Activity, with inputs and outputs as Entities attributed to agent Actors. Relevant for compliance, reproducibility, and debugging of multi-step agent pipelines.

**W3C Activity Streams 2.0**
- URL: https://www.w3.org/TR/activitystreams-core/
- JSON-based vocabulary for describing social and computational activities, actors, and objects. Provides a semantic vocabulary that overlaps with agent event streams (agent acted on object X at time T); useful reference when designing the event schema for agent execution logs.

**IETF RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The foundational authorization protocol. Any orchestration platform exposing APIs or integrating third-party tools must implement OAuth 2.0 flows for delegated access. Particularly relevant for cross-organisation agent delegation scenarios.

**IETF RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Standard for compact, URL-safe claims representation. Used to carry agent identity, permissions, and execution context tokens across service boundaries in multi-agent architectures.

**IETF RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header for expressing typed relationships between resources. Relevant for hypermedia-driven agent APIs where tool discovery follows linked-resource patterns.

**OpenID Connect Core 1.0 / OIDC-A (emerging)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. OIDC-A 1.0 (an emerging proposal as of 2025) extends the standard specifically for LLM-based agent identity, defining how an agent entity can be authenticated, delegated to, and authorised to act on behalf of a human principal.

---

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1 / 3.2**
- URL: https://www.openapis.org/ · Latest: https://handrews.github.io/OpenAPI-Specification/oas/latest.html
- The de-facto standard for describing REST APIs and agent tool schemas. OAS 3.1 achieved full JSON Schema 2020-12 alignment. As of 2026, OAS is the universal language for declaring tools available to agents; MCP tool descriptors, Amazon Bedrock action groups, and Google ADK tools all accept or emit OAS-compatible schemas. A well-designed orchestration platform should use OAS 3.1/3.2 as its canonical tool definition format.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- The vocabulary for validating and describing JSON data structures. Used everywhere agents and tools exchange structured data — function call inputs/outputs, state object schemas, workflow configuration. Fully adopted by OAS 3.1.

**AsyncAPI Specification 3.1**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
- The event-driven counterpart to OpenAPI: describes channels, messages, and bindings for async/message-driven APIs across protocols (Kafka, WebSocket, MQTT, AMQP). Directly applicable to agent orchestration platforms that use event-driven or pub/sub patterns for inter-agent communication and workflow triggers.

**CloudEvents (CNCF Graduated)**
- URL: https://cloudevents.io/ · Spec: https://github.com/cloudevents/spec
- Vendor-neutral specification for describing event data in a common format. A CNCF graduated project (January 2024). Provides a lightweight, interoperable envelope for agent lifecycle events (agent.started, agent.completed, agent.failed), enabling events to flow across observability platforms, trigger systems, and audit sinks without schema coupling.

**Protocol Buffers (proto3) / gRPC**
- URL: https://protobuf.dev/ · https://grpc.io/
- Language-neutral binary serialisation and RPC framework. Relevant for high-throughput inter-agent messaging and for SDK transport layers where JSON overhead is undesirable. The A2A protocol added gRPC transport support in its 2026 update.

---

### Security & Authentication Standards

**OWASP Top 10 for LLM Applications 2025**
- URL: https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/
- The canonical checklist of LLM-specific security vulnerabilities including prompt injection, insecure output handling, supply chain vulnerabilities, excessive agency, and system prompt leakage. Any agent orchestration platform must address these risks in its security design, particularly insecure tool execution and prompt injection via tool outputs.

**OWASP Top 10 for Agentic Applications 2026**
- URL: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
- An extension of the LLM Top 10 specifically for autonomous agent architectures. Covers cascading failures (ASI08), human-agent trust exploitation (ASI09), and rogue agent behaviour (ASI10). A required reference for any platform that orchestrates agents with real-world side effects.

**NIST AI 600-1 — Generative AI Profile**
- URL: https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- NIST's Generative AI Risk Profile, mapping GenAI-specific risks (confabulation, data privacy, dangerous recommendations) onto the AI RMF 1.0 four core functions (GOVERN, MAP, MEASURE, MANAGE). The forthcoming NIST AI Agent Interoperability Profile (expected Q4 2026) will extend this framework specifically to agentic systems.

**NIST AI RMF 1.0**
- URL: https://www.nist.gov/itl/ai-risk-management-framework
- The foundational US federal AI governance framework. Widely adopted as the risk management baseline for enterprise AI platforms; organised around four functions: Govern, Map, Measure, Manage. Relevant to the governance and compliance layer of any orchestration platform.

**EU AI Act — Agentic AI Compliance**
- URL: https://artificialintelligenceact.eu/
- The EU's comprehensive AI regulation, with August 2, 2026 as the enforcement date for high-risk AI systems. Autonomous agent platforms used in high-risk domains (employment decisions, credit, education, law enforcement) must meet obligations around human oversight, technical documentation, auditability, and EU database registration. Penalties reach 7% of global annual revenue.

---

### Agent Interoperability Protocols

**Model Context Protocol (MCP) — Specification 2025-11-25**
- URL: https://modelcontextprotocol.io/specification/2025-11-25 · GitHub: https://github.com/modelcontextprotocol/modelcontextprotocol
- Anthropic-originated open protocol, now hosted under the Linux Foundation, for standardising tool and context hand-offs between LLMs and external systems. MCP servers expose tools, resources, and prompts; MCP clients (LLM runtimes, agents) consume them. Rapidly becoming the universal tool integration layer. The 2026 roadmap focuses on transport scalability, agent-to-agent communication, and governance maturation. MCP Apps (SEP-1865) standardises interactive UI delivery from MCP servers.

**Agent2Agent (A2A) Protocol**
- URL: https://a2a-protocol.org/latest/ · Spec: https://a2a-protocol.org/latest/specification/ · GitHub: https://github.com/a2aproject/A2A
- Google-originated open standard (donated to Linux Foundation) for agent-to-agent discovery and delegation across organisational and framework boundaries. Agents advertise capabilities via an "Agent Card" JSON document; the protocol defines task lifecycle, message passing, and authentication for cross-agent workflows. 2026 updates added gRPC transport and signed Agent Cards. Native support exists in Google Agent Development Kit (ADK). Critical standard for any platform targeting cross-org or multi-vendor agent orchestration.

**OpenTelemetry (OTel) — GenAI Semantic Conventions**
- URL: https://opentelemetry.io/docs/specs/semconv/gen-ai/ · Agent spans: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/
- CNCF-governed open observability standard. The GenAI semantic conventions (experimental as of early 2026, stable track planned) define attribute names and span structures for tracing LLM calls, agent invocations, and tool executions. The `invoke_agent` operation name and `gen_ai.agent.*` attributes are the emerging standard for agent observability. Any orchestration platform emitting OTel spans is compatible with Grafana, Datadog, Honeycomb, and other observability backends without custom integrations.

---

## Similar Products — Developer Documentation & APIs

### LangGraph / LangChain

- **Description:** Graph-based stateful workflow engine for building multi-agent pipelines with cyclic graph topology, persistent checkpointing, and time-travel debugging.
- **API Documentation:** https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref.html
- **Python SDK Reference:** https://reference.langchain.com/python/langgraph/overview
- **JavaScript SDK Reference:** https://langchain-ai.github.io/langgraphjs/reference/
- **Developer Guide:** https://docs.langchain.com/oss/python/langgraph/overview
- **Standards:** REST/JSON; LangSmith observability API; OpenAPI-compatible tool schemas
- **Authentication:** LangSmith API Key for hosted platform; self-hosted deployments use configurable auth

---

### Temporal

- **Description:** Durable execution engine with workflow-as-code semantics; bulletproof state persistence, retries, and audit trails for long-running workflows.
- **API Documentation:** https://docs.temporal.io/
- **SDKs:** Python (https://docs.temporal.io/develop/python), TypeScript (https://docs.temporal.io/develop/typescript), Java (https://docs.temporal.io/develop/java), Go (https://docs.temporal.io/develop/go), .NET (https://docs.temporal.io/develop/dotnet)
- **GitHub (Python SDK):** https://github.com/temporalio/sdk-python
- **Developer Guide:** https://docs.temporal.io/workflows
- **Standards:** gRPC-based internal transport; REST API for management operations; JSON for configuration
- **Authentication:** mTLS for cluster access; Temporal Cloud uses namespace-scoped API keys

---

### Inngest AgentKit

- **Description:** Serverless-first durable workflow engine with purpose-built LLM orchestration primitives; AgentKit provides multi-agent network support with deterministic routing via MCP.
- **API Documentation:** https://agentkit.inngest.com/
- **Developer Guide:** https://agentkit.inngest.com/getting-started/quick-start
- **GitHub:** https://github.com/inngest/agent-kit
- **Main Docs (step.ai features):** https://www.inngest.com/docs/features/inngest-functions/steps-workflows/step-ai-orchestration
- **Standards:** REST/JSON; MCP (for tool integration); OpenTelemetry for tracing
- **Authentication:** Inngest API Key; supports webhook signing keys for event ingestion

---

### Trigger.dev

- **Description:** TypeScript-native open-source background jobs and durable workflows with Checkpoint-Resume semantics, LLM streaming support, and zero timeout constraints.
- **API Documentation:** https://trigger.dev/docs/introduction
- **LLM-indexed docs:** https://trigger.dev/docs/llms.txt
- **GitHub:** https://github.com/triggerdotdev/trigger.dev
- **AI Agents product page:** https://trigger.dev/product/ai-agents
- **Standards:** REST/JSON; Apache 2.0 (open source); TypeScript-first SDK
- **Authentication:** API keys per project; environment-scoped tokens

---

### CrewAI

- **Description:** Role-based multi-agent Python framework; agents defined by role, goal, and backstory collaborate on shared tasks via sequential, hierarchical, or consensual execution models.
- **API Documentation:** https://docs.crewai.com/
- **GitHub:** https://github.com/crewaiinc/crewai
- **Standards:** LiteLLM-compatible (supports OpenAI, Anthropic, Azure, Ollama); Pydantic for structured outputs; MIT License
- **Authentication:** LLM provider API keys; no native auth layer (delegation to deployment environment)

---

### Microsoft Agent Framework (MAF)

- **Description:** Enterprise-ready successor to Semantic Kernel and AutoGen; graph-based workflow orchestration with streaming, checkpointing, human-in-the-loop, and time-travel for Python and .NET.
- **API Documentation:** https://learn.microsoft.com/en-us/agent-framework/overview/
- **Migration Guide:** https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-semantic-kernel/
- **GitHub:** https://github.com/microsoft/agent-framework
- **Semantic Kernel Docs:** https://learn.microsoft.com/en-us/semantic-kernel/
- **Standards:** OpenAPI for tool definitions; Azure OpenAI-compatible; Microsoft.Extensions.AI foundation; .NET and Python at parity
- **Authentication:** Azure Entra ID (formerly Azure AD); API Key for model providers

---

### n8n

- **Description:** Visual node-based workflow automation platform with 400+ integrations and native LangChain AI nodes; supports hybrid code/no-code workflow definition.
- **API Documentation:** https://docs.n8n.io/api/
- **Webhook Docs:** https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- **Developer Guide:** https://docs.n8n.io/
- **Standards:** REST/JSON; OpenAPI for API operations; Webhook HTTP; fair-code licence (restrictive for commercial use)
- **Authentication:** API Key (header-based); OAuth 2.0 for third-party integrations; webhook signing

---

### Kore.ai Agent Platform

- **Description:** Enterprise multi-agent orchestration platform with centralised governance, compliance, audit logging, cost tracking, and anomaly detection for regulated industries.
- **API Documentation:** https://docs.kore.ai/agent-platform/apis
- **SDK Architecture:** https://docs.kore.ai/agent-platform/sdk/design-decisions/architecture/
- **Automation APIs:** https://docs.kore.ai/xo/apis/automation/api-list/
- **Web SDK:** https://github.com/Koredotcom/web-kore-sdk
- **Standards:** REST/JSON; OAuth 2.0 for platform auth; OpenAPI-compatible tool definitions
- **Authentication:** API Key in Authorization header; OAuth 2.0 scopes for fine-grained access

---

### OpenAI Responses API

- **Description:** OpenAI's most advanced stateful LLM interface, replacing the deprecated Assistants API (sunset August 26, 2026); supports tool use, MCP, computer use, and deep research natively.
- **API Documentation:** https://developers.openai.com/api/docs
- **Responses API Reference:** https://developers.openai.com/api/reference/responses/overview
- **Assistants Migration Guide:** https://developers.openai.com/api/docs/guides/migrate-to-responses
- **Standards:** REST/JSON; OpenAPI 3.x for request/response schemas; MCP native support
- **Authentication:** Bearer token (API Key); organisation and project scoping

---

### Anthropic Claude API (Tool Use / Agent SDK)

- **Description:** Anthropic's API for Claude models with first-class tool use, multi-step agent execution, MCP client support, and Agent SDK for managed agent loops.
- **API Documentation:** https://platform.claude.com/docs/en/home
- **Tool Use Overview:** https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
- **Agent SDK Overview:** https://code.claude.com/docs/en/agent-sdk/overview
- **GitHub (Python SDK):** https://github.com/anthropics/anthropic-sdk-python
- **Standards:** REST/JSON; OpenAPI-compatible tool schemas; MCP native; SDKs for Python, TypeScript, Java, Go, Ruby, C#, PHP
- **Authentication:** API Key (x-api-key header); organisation-level key management

---

## Notes

**Emerging and evolving areas:**

- The MCP and A2A protocols are both rapidly evolving (2025–2026). Implementers should pin to specific spec versions and monitor the respective Linux Foundation working groups for breaking changes.
- OIDC-A (OpenID Connect for Agents) remains a community proposal as of May 2026 and has not yet been formally standardised; monitor the OpenID Foundation for ratification.
- NIST's AI Agent Interoperability Profile is planned for Q4 2026 and will be the authoritative US government guidance on agent-to-agent standards compliance.
- The OTel GenAI semantic conventions are still marked experimental; production deployments should treat attribute names as potentially unstable until the 1.0 stable release.
- The OpenAI Assistants API sunsets August 26, 2026; any platform that integrated against it must migrate to the Responses API.
- EU AI Act high-risk enforcement begins August 2, 2026; platforms targeting EU enterprise customers in regulated verticals must complete conformity assessments before that date.
