---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-2, D-13, D-16, D-17, D-33, D-34]
---

# Architecture Diagrams — End to End

**Purpose:** a single rendered diagram set for the stack and the agent-specific layers (harness, evals, guardrails), companion to the prose specs in this directory and in `40-ai-safety/`.

**Authority:** where a diagram and an ADR disagree, the ADR wins — see [adr-index.md](adr-index.md).

## 1. System context

**Rewritten 2026-07-19 (D-33, narrowed by D-34) for the self-hosted-Kubernetes-on-EKS stack.** Users reach Dux only through Cloudflare edge and `api.dux.io`. Connectors reach vendor and threat-intel systems read-only; no tenant credentials cross the wire to intel sources. The platform reaches customer cloud accounts via per-tenant cross-account IAM, and reaches LLM providers by direct API key (OpenAI), IAM/SigV4 (Bedrock primary), or a direct Anthropic API key (fallback leg).

```mermaid
flowchart LR
    User[Security engineer / CISO / SRE]
    Edge[Cloudflare edge - DNS/CDN/WAF]
    API[api.dux.io - NestJS API, on K8s]
    Temporal[Self-hosted Temporal workflows]
    PG[(CloudNativePG - RLS FORCE)]
    NATS[(NATS core + JetStream - pub/sub, queues)]
    Valkey[(Valkey - cache)]
    MinIO[(MinIO - object storage)]
    Intel[NVD / KEV / EPSS / GitHub / MSRC - read only]
    Vendor[Customer cloud - AWS via cross-account IAM]
    OpenAI[OpenAI - direct API]
    Bedrock[AWS Bedrock - Claude primary, IAM/SigV4]
    Anthropic[Anthropic direct API - Claude fallback]

    User --> Edge --> API
    API --> Temporal
    API --> PG
    API --> NATS
    API --> Valkey
    API --> MinIO
    Temporal --> Intel
    Temporal --> Vendor
    Temporal --> OpenAI
    Temporal --> Bedrock
    Temporal --> Anthropic
```

## 2. Container / service topology

Three Kubernetes Deployments carry the workload from Gate 1: the API, connector sync, and sandbox broker (the LiteLLM proxy is retired — Bedrock calls go direct from `dux-api` via `LLMProviderPort`, ADR-010 R5). Each is isolated from the others; only `dux-connector-sync` and `dux-sandbox` are permitted to reach outside the platform boundary for ingest and script execution respectively.

```mermaid
flowchart TB
    subgraph K8s[Kubernetes - Amazon EKS]
        APIsvc[dux-api - NestJS, SSE, auth, governance kernel, MCP gateway, LLMProviderPort direct-Bedrock-SDK]
        Conn[dux-connector-sync - NVD/KEV/EPSS/AWS/vendor ingest]
        Sandbox[dux-sandbox - investigation script execution broker]
    end
    Web[React + Vite SPA - MinIO + Cloudflare CDN]
    SelfTemporal[Self-hosted Temporal]
    PGDB[(CloudNativePG + PgBouncer + pgvector + Apache AGE)]
    NATSC[(NATS core + JetStream)]
    ValkeyC[(Valkey)]
    MinIOStore[(MinIO)]
    Firecracker[Self-hosted Firecracker microVM]
    MCPGW[MCP Gateway]
    Bedrock[AWS Bedrock - Claude primary]

    Web --> APIsvc
    APIsvc --> MCPGW
    APIsvc --> SelfTemporal
    APIsvc --> PGDB
    APIsvc --> NATSC
    APIsvc --> ValkeyC
    APIsvc --> MinIOStore
    APIsvc --> Bedrock
    Conn --> PGDB
    Sandbox --> Firecracker
    SelfTemporal --> Conn
    SelfTemporal --> Sandbox
```

## 3. Agent orchestration / harness

Each assessment runs as one Temporal child workflow per tenant, on a tenant-scoped task queue (`assessment-{tenant_id}`). No agent framework mediates the loop (ADR-021) — the workflow calls the Bedrock Converse API directly, as ordinary Temporal activities, with message history read/written to Postgres and turn deltas fanned out over NATS. `TraceRecorder` persists the run without making any LLM calls itself.

```mermaid
flowchart TD
    WF["ExploitabilityAssessmentWorkflow - Temporal, per-tenant queue"]
    SLLM["S-LLM Extract activity - Bedrock Converse, outputConfig.textFormat, no tools"]
    PLLM["P-LLM Reason activity - Bedrock Converse, toolConfig enabled"]
    Tool["Tool Executor activity - MCP SDK direct, per tool call"]
    Budget["Budget Check activity - accumulate tokens, $0.75 ceiling"]
    SSE["SSE Emitter activity - NATS JetStream -> dashboard"]
    PG[(Postgres - assessment_messages)]
    HITL["HITL Gate - Temporal signal"]
    Trace["TraceRecorder - persistence only"]

    WF --> SLLM --> PLLM
    PLLM -- tool_use --> Tool --> PLLM
    PLLM --> Budget
    PLLM --> SSE
    PLLM -- mutation required --> HITL --> PLLM
    WF --> PG
    PLLM --> PG
    PLLM --> Trace
```

State machine per activity: `IDLE -> REASONING -> TOOL_CALLING -> EVALUATING -> {COMPLETE | BLOCKED | FAILED}`.

## 4. CaMeL dual-LLM guardrail

The Suspicious LLM (S-LLM) is the only component that reads untrusted content (CVE text, tool output); it never executes tools and its output is schema-constrained. The Privileged LLM (P-LLM) executes tools and reasons over the assessment, but never sees raw untrusted text. A tool schema with an unconstrained free-text field defeats the boundary, so P-LLM tool schemas are structured JSON only.

```mermaid
flowchart LR
    CVE[Untrusted CVE text / tool output]
    SLLM["S-LLM - extracts structured prerequisites, JSON schema"]
    Valid{Schema valid?}
    PLLM["P-LLM - reasons, calls tools, never sees raw text"]
    Blocked["GOVERNANCE_BLOCKED / L1 kill switch"]

    CVE --> SLLM --> Valid
    Valid -- yes --> PLLM
    Valid -- no, after retries --> Blocked
```

## 5. Governance kernel chain (guardrails)

Every LLM call and MCP tool invocation is checked by `KillSwitchRelay` first; an active kill switch short-circuits the whole chain with a 503. Otherwise, five gates run in sequence, ending with `VendorActionGate` — the only legal path to a vendor mutation API — and then `HITLGate`, whose default outcome depends on the specific action's confidence floor.

```mermaid
flowchart LR
    KS{KillSwitchRelay active?}
    Intent[IntentGate]
    Budget[BudgetGate]
    Effect[EffectGate]
    Vendor["VendorActionGate - GOV-TOOL-01..05 confidence floor + rollback on file"]
    HITL[HITLGate]
    Exec[Vendor action executes + audit]
    Block[Blocked / 503]

    KS -- yes --> Block
    KS -- no --> Intent --> Budget --> Effect --> Vendor --> HITL --> Exec
```

`VendorActionGate` outcome by tool: `network.blocklist_add` and `policy.deploy_device_config` need confidence >= 0.75 or escalate to HITL; `ticket.create_remediation` always executes unattended; `endpoint.isolate` and `patch.deploy_special_devices` require a live HITL response on every call, with no confidence floor that bypasses it.

## 6. MCP gateway security layers

Six defense layers sit between the reasoning loop and every tool call, whether a read-only research tool or a vendor write action.

```mermaid
flowchart TB
    Loop2[ReasoningLoop]
    L1[L1 - Auth / agent identity]
    L2[L2 - Schema integrity, hash-pinned tool manifests]
    L3[L3 - I/O sanitization]
    L4[L4 - Network egress allowlist]
    L5[L5 - Observability]
    L6[L6 - Multi-server isolation]
    Tools[MCP tool servers - read-only research + vendor write]

    Loop2 --> L1 --> L2 --> L3 --> L4 --> L5 --> L6 --> Tools
```

## 7. Vendor write-path sequence

Fast actions and the mitigation write path share the same gate. Three of the five canonical actions execute unattended by default and only escalate to HITL on anomaly (confidence-abstention band, sandbox timeout/OOM, T4 outlier); two fleet-impacting actions require a live human response on every call until each earns unattended execution via a field-proven safety record.

```mermaid
sequenceDiagram
    participant Client
    participant API as POST /fast-actions
    participant Gate as VendorActionGate
    participant Human as HITL reviewer
    participant Adapter as VendorActionAdapter
    participant Vendor as Vendor native API
    participant Audit as Audit log

    Client->>API: request action
    API->>Gate: canonical_action_id, parameters
    alt unattended action (3 of 5) - no anomaly
        Gate->>Adapter: execute
        Adapter->>Vendor: native mutation call
        Vendor-->>Adapter: result
        Adapter-->>Audit: ActionResult + audit
    else anomaly on unattended action, or mandatory-HITL action (2 of 5)
        Gate->>Human: hitl_request (tier, blast radius, rollback URL)
        Human-->>Gate: hitl_response
        Gate->>Adapter: execute (if approved)
        Adapter->>Vendor: native mutation call
        Vendor-->>Adapter: result
        Adapter-->>Audit: ActionResult + audit
    end
```

## 8. Evals & confidence pipeline

Golden-set regression is a merge-blocking CI gate. The exploitability verdict itself is scored by a three-signal confidence ensemble, calibrated with Platt scaling, and mapped to abstention bands that decide routing, including whether a case escalates into the HITL path shown in diagram 5.

```mermaid
flowchart LR
    Golden["Golden-set regression - CI merge-blocking"]
    AgentDojo["AgentDojo benchmark - defended vs undefended task completion"]
    Logprobs["Logprobs signal - weight 0.40"]
    Entropy["Semantic entropy signal - weight 0.35"]
    Verbalized["Verbalized confidence signal - weight 0.25"]
    Ensemble[Confidence ensemble]
    Platt["Platt scaling - ECE gate <= 0.15"]
    Abstain["Abstention bands - exploitable / likely / unlikely / not_exploitable / insufficient_data"]

    Logprobs --> Ensemble
    Entropy --> Ensemble
    Verbalized --> Ensemble
    Ensemble --> Platt --> Abstain
    Golden -.CI gate.-> Ensemble
    AgentDojo -.CI gate.-> Ensemble
```

## 9. Sandbox execution

Investigation scripts written by the agent are statically scanned before they ever run, then executed in a fresh microVM that is discarded after one invocation, with default-drop network egress.

```mermaid
flowchart LR
    Script[LLM-generated investigation script]
    AST["ScriptSecurityScanner - AST pre-scan, blocks exec/eval/subprocess/os.system"]
    Reject[Rejected]
    VM["Self-hosted Firecracker microVM - fresh per invocation, 512MB/1CPU/60s"]
    Egress["Egress - default DROP, NVD/GitHub allowlist only"]

    Script --> AST
    AST -- blocked pattern --> Reject
    AST -- clean --> VM --> Egress
```

## 10. Multi-tenant isolation overlay

Isolation is enforced at every layer the request touches: Postgres RLS with FORCE, composite `(tenant_id, id)` lookups, tenant-scoped Temporal task queues, tenant-scoped kill-switch pub/sub channels, and HMAC-hashed tenant IDs in all telemetry so raw tenant IDs never appear in logs or traces.

```mermaid
flowchart TB
    Req[Request with JWT tenant_id claim]
    Tx["Transaction - SET LOCAL app.tenant_id"]
    RLS["CloudNativePG RLS FORCE - 4 policies on every tenant table"]
    Queue["Self-hosted Temporal task queue - assessment-{tenant_id}"]
    KillCh["Kill-switch channel - tenant-scoped NATS core pub/sub"]
    Telemetry["OTel spans - dux.tenant_id_hash, HMAC-SHA256[:8]"]

    Req --> Tx --> RLS
    Req --> Queue
    Req --> KillCh
    Req --> Telemetry
```

## 11. Observability closing the loop

Every LLM and MCP call is wrapped by an instrumented client so no call bypasses tracing. Spans follow the OTel GenAI semantic convention into self-hosted Langfuse and self-hosted Grafana LGTM (Loki/Tempo/Prometheus), and burn-rate alerts watch the resulting metrics.

```mermaid
flowchart LR
    Call[LLM call / MCP tool call]
    Client["InstrumentedLLMClient / gateway"]
    OTel["OTel GenAI spans"]
    Langfuse[Self-hosted Langfuse]
    Grafana[Self-hosted Grafana LGTM]
    Burn["SLO burn-rate alerts - availability, cost cap, workflow actions, cache hit rate"]

    Call --> Client --> OTel
    OTel --> Langfuse
    OTel --> Grafana
    Grafana --> Burn
```

## Sources

[architecture-overview.md](architecture-overview.md) · [adr-index.md](adr-index.md) · [workflows.md](workflows.md) · [multi-tenancy.md](multi-tenancy.md) · [data-model.md](data-model.md) · [governance-kernel.md](../40-ai-safety/governance-kernel.md) · [kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md) · [camel-plane.md](../40-ai-safety/camel-plane.md) · [mcp-security.md](../40-ai-safety/mcp-security.md) · [confidence-calibration.md](../40-ai-safety/confidence-calibration.md) · [sandbox-execution.md](../40-ai-safety/sandbox-execution.md) · [safety-overview.md](../40-ai-safety/safety-overview.md) · [mitigation-write-path.md](../10-product/features/mitigation-write-path.md) · [catalogs.md](../10-product/catalogs.md) · [observability-slo.md](../60-operations/observability-slo.md)
