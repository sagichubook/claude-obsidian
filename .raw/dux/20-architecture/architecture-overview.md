---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-1, D-2, D-5, D-34, H5]
---

# Architecture Overview (TRD)

**Purpose:** the system context, deployment topology, monorepo layout, and provider ports.

**Parents:** all BRs · **Canon:** ideal-state v2.2, with ADR R2 revisions applied.

**Authority: the ADRs win.** They supersede any legacy TRD or Blueprint row. If a diagram disagrees with an ADR, the ADR is correct and the diagram is stale.

## 1. System context (C1)

**System-context table rewritten 2026-07-19 (D-33, narrowed by D-34) for the self-hosted-Kubernetes-on-EKS stack — see [ADR-006 R4](adr-index.md#adr-006-r4--deployment-topology).**

External systems:

| System | Role |
|--------|------|
| **CloudNativePG** | application data, workflow state, pgvector (self-hosted Postgres operator on K8s) |
| **NATS + JetStream** | event bus (kill-switch pub/sub, continuous-assessment triggers), durable async queues |
| **Valkey** | cache, rate limits, session state, LLM response cache |
| **MinIO** | object storage, self-hosted S3-compatible, WORM/Object Locking for the audit anchor |
| **Cloudflare** | DNS, CDN (fronting MinIO-served static assets), edge WAF — no longer the deployment target |
| **LLM providers** | OpenAI (GPT tier, direct API); Claude via the **direct Bedrock SDK + NestJS fallback chain** — Bedrock → direct Anthropic → local vLLM (ADR-017 R3); Azure OpenAI EU at trigger (flagged, see [decisions-log D-34](../00-meta/decisions-log.md)) |
| **Customer AWS APIs** | asset discovery only (ADR-004) — unrelated to Dux's own hosting |
| **NVD / CISA KEV / EPSS** | CVE feeds |
| **Langfuse (self-hosted)** | LLM tracing |
| **MCP Security Gateway** | CaMeL-plane tool governance (see [architecture-diagrams §4](architecture-diagrams.md#4-camel-dual-llm-guardrail) and `40-ai-safety/camel-plane.md`) |
| **Self-hosted Temporal** | durable execution, on Kubernetes |
| **Self-hosted Firecracker** | microVM sandbox, on Kubernetes |

**Trust boundaries:**

- Users → Cloudflare edge (TLS, DDoS, coarse limits) → MinIO-served SPA / `api.dux.io` → NestJS API. Auth is JWT plus session cookies; per-tenant limits are applied **post-auth** by `@nestjs/throttler` + Valkey.
- Assessment agents → Temporal workflows. Workflow IDs are tenant-scoped; state lives in CloudNativePG.
- API ↔ NATS, over tenant-scoped kill-switch channels.
- Platform → customer AWS accounts, via cross-account IAM, per tenant (asset discovery only — Dux's own platform no longer runs on AWS).
- Connectors → NVD/KEV/EPSS: **read-only, and no tenant credentials on the wire.**
- Platform → LLM: API key (OpenAI, direct Anthropic fallback via Vault) or IAM/SigV4 (Bedrock primary), with tenant-scoped Langfuse metadata.
- Optional resident agents (Gate 5) → platform: HMAC or mTLS heartbeat, ≤4 KB, once per minute.

## 2. Deployment topology

**Kubernetes from Gate 1** (ADR-006 R4, 2026-07-19, D-33 narrowed by D-34) — managed control plane on **Amazon EKS**, with CloudNativePG, NATS+JetStream, Valkey, and MinIO run in-cluster rather than as external managed tiers. This is portable-hosting from day one: the same manifests run on any CSP or on-prem, so SOC 2/FedRAMP-conscious finance/healthcare buyers can audit them and, if required, redeploy into their own boundary — a property AWS-only ECS Fargate could not offer. EKS additionally restores FedRAMP-authorized-CSP/GovCloud availability that the prior DigitalOcean/Linode LKE target lacked.

Blast-radius isolation comes from separate K8s Deployments:

| K8s Deployment | Responsibility |
|-----------------|----------------|
| `dux-api` | NestJS API, SSE termination, auth, governance kernel, MCP gateway (in-process or sidecar) |
| `dux-connector-sync` | NVD/KEV/EPSS/AWS and vendor connector ingest — **isolated from the API** |
| `dux-sandbox` | investigation-script execution broker → self-hosted Firecracker microVMs — **isolated** |

**Frontend:** React + Vite SPA (TanStack Router/Query) — a static build served from MinIO behind Cloudflare CDN, talking only to `api.dux.io`.

**Secrets:** **HashiCorp Vault** (self-hosted on K8s), replacing AWS SSM Parameter Store (D-5 R2). **Temporal payloads carry secret *references*, never values** — unchanged.

**Local dev:** `docker compose up` — Postgres, PgBouncer, Valkey, MinIO, Vault, and Unleash, at parity. (Unchanged — local dev already ran self-hosted equivalents of everything now also true in production.)

**Network topology (resolves part of [OI-31](../00-meta/open-items.md) — Deployment Guide; rewritten 2026-07-19, D-33, narrowed by D-34).** A single EKS cluster per environment (dev/staging/prod), 3-AZ node pools with managed-node-group autoscaling (CPU >70%/2min or memory >80%/2min scales the node group; agent queue depth >50 scales Temporal workers). Node pools map to the three Deployment roles (table above), with K8s `NetworkPolicy` enforcing the same isolation ECS security groups previously provided. An **nginx Ingress Controller** fronts `dux-api`, terminating TLS and applying rate limiting via Valkey. CloudNativePG, NATS, Valkey, and MinIO run as in-cluster StatefulSets/operators — no external managed-tier network hop. Cross-account customer asset-discovery IAM (ADR-004) remains a signed AWS API call against the *customer's* AWS account, and is a separate concern from EKS being Dux's own platform-hosting target.

**WAF.** AWS WAF is retired with ECS Fargate. **Cloudflare's edge WAF** (DDoS, bot management, managed rule sets) becomes the sole WAF layer, sitting in front of `api.dux.io` and the MinIO-served SPA. **Falco** (in-cluster runtime security) is the compensating control for what AWS WAF's network-layer backstop used to catch closer to the workload — anomalous syscalls, sandbox escape attempts — layered with the unchanged `@nestjs/throttler` + Valkey application-layer limits (§1).

**Secrets-rotation cadence.**

| Secret class | Rotation | Mechanism |
|--------------|----------|-----------|
| Database credentials (CloudNativePG) | 90 days | Vault, self-service rotation UI ([runbooks §6](../60-operations/runbooks.md)) |
| Self-hosted Temporal mTLS client certs | 90 days | cert-manager or Vault (D-16 R2) |
| OAuth refresh tokens (vendor connectors) | per-vendor token lifetime, refreshed on use | Vault transit (ADR-011 R2, unchanged) |
| SSO/SCIM tokens | 90 days | Vault, emits `sso.scim.token.rotated` audit record ([runbooks §4](../60-operations/runbooks.md)) |
| Audit hash-chain key (`chain_key`) | quarterly | Vault, `audit/chain-key` ([data-model §2](data-model.md), unchanged) |
| LLM provider API keys (OpenAI, direct Anthropic fallback) | 180 days, or immediately on suspected exposure | Vault, 30/7/1-day expiry notification sequence ([runbooks §6](../60-operations/runbooks.md)) |
| Cloudflare API token (DNS/CDN rollback) | 90 days | Vault ([dr-bcp](../60-operations/dr-bcp.md)) |

Bedrock (ADR-017 R3, primary leg only) has no entry — it authenticates via the workload's IAM-bound credential native to the EKS node pool's service-account role, not a stored Vault credential.

## 3. Container architecture (C2)

Web (React + Vite on MinIO + Cloudflare CDN) → `dux-api` (K8s) → MCP Gateway → CloudNativePG / NATS / Valkey / MinIO. Connector-sync and sandbox run as separate K8s Deployments. The optional physical-resident agent (Gate 5) heartbeats to `dux-api`.

**Workflow process groups** are logically separated:

| Group | Metric | Guard |
|-------|--------|-------|
| Connector sync | `nvd_sync_queue_depth` | max 5 concurrent NVD activities cluster-wide, on a dedicated `connector-*` queue prefix — **so NVD 429 backoff can never starve assessment capacity** |
| Assessment | `workflow_actions_per_assessment` p95 | warn above 100, halt at 200 |

**Autoscale-on-queue-depth policy (resolves [OI-30](../00-meta/open-items.md)/DA-18; rewritten 2026-07-19, D-33 for K8s, narrowed by D-34 for EKS).** Each Deployment in §2 scales independently via a Kubernetes `HorizontalPodAutoscaler` (HPA), min 2 / max 10 replicas per service:

| Service | Scaling metric | Target |
|---------|----------------|--------|
| `dux-api` | nginx Ingress `RequestCountPerTarget` (Prometheus adapter) | 1,000 req/min/pod |
| `dux-connector-sync` | `nvd_sync_queue_depth` (custom Prometheus metric, published every 60 s) | scale out above 200 queued items; scale in below 50 |
| `dux-sandbox` | concurrent microVM count (`dux_cost_sandbox_seconds_per_tenant` derived gauge) | scale out above 80% of `5 concurrent microVMs × active tenants` |

Scale-out cooldown 60 s; scale-in cooldown 300 s, to avoid flapping on the same NVD-429-backoff bursts GOV-005/§3's connector-sync isolation already guards against.

## 4. Monorepo structure

```
dux/
├── packages/
│   ├── core/          # workflows (Temporal), CaMeL-plane, MCP tools, Saga coordinator
│   │   ├── ports/     # port interfaces (DIP)
│   │   ├── assessment/# PrerequisiteExtractor, ReasoningLoop, TraceRecorder, AssessmentActivity
│   │   ├── governance/# GOV-001–013 kernel
│   │   └── world-model/
│   ├── api/           # NestJS backend, auth, tenants, webhooks, SSE+POST realtime
│   │   └── projections/ # ExposureProjection, ProtectionProjection, ActionCardProjection
│   ├── web/           # React + Vite dashboard (TanStack Router/Query), exposure/trace viewer
│   ├── database/      # Drizzle schema + migrations, RLS policies, seed data
│   ├── connectors/    # NVD/KEV/AWS + vendor connectors (vendor-contract.ts)
│   ├── actions/       # vendor action catalog, policy gate, vendor adapters (ADR-012)
│   ├── observability/ # OTel, CostMetricsService, InstrumentedLLMClient, audit logging
│   ├── python-eval/   # DeepEval, Evidently, calibration
│   ├── notifications/ # notification queues, email/Slack/PDF templates (ADR-005)
│   ├── mcp/           # MCP gateway + tools
│   ├── llm/           # router, models.json, proxy-adapter
│   ├── security/      # script-rules (AST scanner), aibom/manifest.json (ADR-009)
│   ├── agents/        # agent-registry SSoT: {type}/ manifests, CODEOWNERS-gated (ADR-009)
│   └── adapters/      # ONLY place vendor SDKs may be imported
├── infra/             # Pulumi (TypeScript) app; Kubernetes/EKS single target (ADR-006 R4); NO vps scripts
├── tests/             # integration, e2e (Playwright), golden (250 CVEs), fuzz
└── turbo.json
```

**Dependency rules**, enforced by turbo and ESLint:

- `core/` → database, connectors, observability.
- `api/` → core, database, notifications.
- `web/` → api **types only**.
- **No circular dependencies.**
- **Only `packages/adapters/*` may import a vendor SDK** (`import/no-restricted-paths`).

## 5. Provider ports

Each port exists to guarantee a week-scale exit (ADR-013).

| Port | Gate-1 default | Swap targets |
|------|----------------|--------------|
| `AuthPort` | Better Auth (`BetterAuthAdapter`) | Supabase Auth ↔ WorkOS (enterprise SAML; LOCK-01 unbuilt) |
| `WorkflowPort` | **Self-hosted Temporal on K8s** (`TemporalWorkflowAdapter`) | Restate ↔ Hatchet ↔ DBOS (future cost spike) |
| `RealtimePort` | SSE + POST + **NATS core pub/sub** | Ably ↔ Centrifugo |
| `StoragePort` | **MinIO** (self-hosted, S3-compatible) | S3, R2 |
| `VectorPort` | pgvector in **CloudNativePG** | a dedicated vector store, at ~100M vectors |
| `GraphPort` | **Apache AGE** (Postgres extension, same CloudNativePG instance) — per-edge provenance + integrity hashing (ADR-020 R2) | Neo4j, at scale |
| `LLMProviderPort` | **Direct Bedrock SDK** (`@aws-sdk/client-bedrock-runtime`) behind `LLMProviderPort`; NestJS `LLMFallbackService` orchestrates Bedrock → direct Anthropic → local vLLM (ADR-010 R5) | Bifrost, evaluated at Gate 2 only if multi-provider routing complexity outgrows NestJS fallback logic |
| `ModelRouterPort` | in-process cost-aware router | — |
| `SandboxPort` | **`SelfHostedFirecrackerAdapter` (Gate-1 default, on K8s)** | `firecracker-containerd`/Kata as the interim K8s-integration bridge ↔ SmolVM-class sub-200 ms cold-start vendors. `NoOpSandboxAdapter` is the emergency kill path. E2B/`ManagedMicroVmAdapter` retired (ADR-015 R4) |
| `WorldModelQueryPort` | `PostgresWorldModelAdapter` (CloudNativePG) — agentic RAG loop over graph + vector + threat-intel (ADR-020 R2) | `HybridGraphWorldModelAdapter` (Neo4j trigger) |
| `VendorConnector` | AWS + ≥3 live at Gate 1: CrowdStrike, Wiz, and ServiceNow **or** Entra ID (ADR-011 R2) | Intune / Qualys (W2); long tail (W3) |
| `VendorActionPort` / `ActionPolicyPort` | unattended by default at Gate 1; HITL on anomaly escalation only | closed-loop validation at Gate 3 (US-019) |
| `NotificationPort` | **`NatsJetStreamNotificationAdapter`** (ADR-005 R2) | a dedicated queue service, if JetStream queue depth becomes the bottleneck |

`InstrumentedLLMClient` is a **Decorator** over `LLMProviderPort` — cost metering, cache, fallback, OTel.

`TenantContext` is a value object `{tenantId, userId?, connectorIds[], rlsSession}`, passed via `AsyncLocalStorage`.

**[OI-08](../00-meta/open-items.md) resolved 2026-07-16.** The port row above was pointing at a stale `DbosNotificationAdapter` default left over from before DBOS was demoted (ADR-007 R2) — but ADR-005 already specifies the actual notification engine as Neon-backed queues (`email_queue`/SES, `slack_queue`, `pdf_queue`/Gotenberg, `webhook_queue`), not BullMQ and never DBOS. The port table simply hadn't been updated to cite it. `NatsJetStreamNotificationAdapter` is that existing ADR-005 R2 mechanism (NATS JetStream durable queues — Neon-backed prior to 2026-07-19, D-33), named as the `NotificationPort` implementation — no new subsystem.

## 6. Technology stack

Pins dated 2026-07-19 (D-33 stack replacement, narrowed and extended by D-34).

| Layer | Technology |
|-------|-----------|
| API | NestJS, TypeScript |
| Durable execution | **Self-hosted Temporal on K8s** behind `WorkflowPort` |
| Database | CloudNativePG (self-hosted operator) + PgBouncer + Drizzle, with RLS FORCE |
| Graph | **Apache AGE** (Postgres extension, same CloudNativePG instance) — attack-path/asset-vulnerability-control relationships, per-edge provenance + integrity hashing (ADR-020 R2) |
| Vector | pgvector + pgvectorscale, same CloudNativePG instance |
| Retrieval | **Agentic RAG, enabled** — Temporal workflow (plan → retrieve → reason → decide → synthesize) with constrained decoding via Bedrock Converse API tool-use (ADR-020 R2) |
| Cache | Valkey — rate limits, LLM response cache, session state, Temporal activity-result cache |
| Event bus | NATS core (kill-switch pub/sub, SSE fan-out signaling, continuous-assessment triggers) + NATS JetStream (durable queues) |
| Storage | MinIO (self-hosted, S3-compatible), WORM/Object Locking for the audit anchor |
| Rate limiting | Cloudflare edge + `@nestjs/throttler` + Valkey |
| Frontend | React + Vite SPA (TanStack Router + TanStack Query), static build on MinIO behind Cloudflare CDN |
| Auth | Better Auth, via `AuthPort` |
| Asset discovery | AWS SDK v3 (customer AWS accounts — unrelated to Dux's own hosting) |
| CVE feeds | NVD API v2.0, CISA KEV JSON, EPSS (FIRST.org) |
| Notifications | notification queues on NATS JetStream + SES + Slack |
| LLM routing | **Direct Bedrock SDK behind `LLMProviderPort`, NestJS-level provider fallback/retry** (ADR-008 R2 / ADR-010 R5); Bifrost evaluated at Gate 2 only if routing complexity outgrows NestJS fallback logic |
| S-LLM (Gate 2 triage) | **Bedrock Converse API, cheapest available model** (e.g. `amazon.titan-text-lite-v2`) for classification/triage/dedup — no self-hosted inference (ADR-021, retires the vLLM + Phi-4 14B path) |
| Agent orchestration (assessment loop) | **Temporal TypeScript workflow calling AWS Bedrock Converse API directly** — no agent framework in the loop (ADR-021); see §7a |
| Observability | OTel GenAI → self-hosted Langfuse + **Grafana LGTM** (Loki/Tempo/Prometheus/Grafana, self-hosted) |
| Runtime security | Falco (in-cluster anomaly detection) |
| Container scanning | Trivy in CI/CD |
| Feature flags | Unleash — server-side, **fail-safe to false above 500 ms** (self-hosted, unchanged) |
| Secrets | HashiCorp Vault (self-hosted on K8s) |
| Sandbox | Self-hosted Firecracker on K8s, via `SandboxPort` (Gate-1 default, ADR-015 R4) |
| Deploy | Kubernetes (**Amazon EKS**, ADR-006 R4), provisioned via **Pulumi** (TypeScript) |
| Claude inference | Multi-provider via direct Bedrock SDK + NestJS fallback — Bedrock → direct Anthropic → local vLLM (ADR-017 R3) |
| Eval | python-eval (DeepEval), golden set of 250 CVEs |

**IaC tool: Pulumi, not AWS CDK (2026-07-19, D-33, supersedes the 2026-07-16 OI-17 CDK decision).** CDK is AWS-only and cannot provision a portable Kubernetes target. Pulumi keeps the OI-17 one-language rationale (TypeScript across app and infra) while adding the multi-cloud/on-prem portability the new stack requires — the only real alternative, Terraform (HCL, a second language), was rejected on the same "no existing footprint" grounds OI-17 originally used against it. `infra/` (§4) is the Pulumi app; stacks map one-to-one to the K8s Deployments in §2.

## 6a. Agent Execution Model (Temporal + Bedrock Direct)

**No agent framework sits inside the reasoning loop (ADR-021).** `ExploitabilityAssessmentWorkflow` is a Temporal TypeScript workflow that calls the Bedrock Converse API directly, as ordinary Temporal activities:

- A bounded `while` loop, max 10 turns.
- **S-LLM activity:** `modelId: anthropic.claude-haiku-4-5`, `outputConfig.textFormat` with a JSON schema — structured CVE fact extraction only, no `toolConfig`, and raw CVE text never reaches the P-LLM.
- **P-LLM activity:** `modelId: anthropic.claude-sonnet-4-6`, `toolConfig` enabled with the discovered MCP tool schemas — receives structured facts plus asset context, never raw CVE text.
- **Tool-execution activity:** each tool call is its own retryable Temporal activity (`@modelcontextprotocol/sdk` client, direct — no framework mediates the call).
- **Budget-check activity:** accumulates token cost per turn; hard ceiling **$0.75/assessment** (same SLO as ADR-008 R2), `BUDGET_EXCEEDED` aborts the workflow.
- **SSE-emitter activity:** streams `ConverseStream` deltas onto NATS for dashboard real-time updates.
- **HITL:** a Temporal signal pauses the workflow at `VendorActionGate` or above the high-exploitability threshold, same signal mechanism as every other Gate-3 human-approval path in this corpus (ADR-007 R3, ADR-020 R2).

**Message history lives in Postgres, not Temporal workflow state.** Bedrock Converse resends the full `messages` array every turn; ten turns of tool results can exceed Temporal's ~50 KB default state limit. The `assessment_messages` table (`id`, `assessment_id`, `turn`, `role`, `content` jsonb, `tool_use_id`, `created_at`) is the source of truth; workflow state carries only `history_id`, `turn_count`, `spent_usd`, and `status`. Detail: [workflows §3a](workflows.md).

## 7. Lives-inside architecture (logical residency)

**Dux runs in Dux Cloud.** Customer data is reached through **read-only APIs and OAuth**, not in-VPC compute.

**"Lives inside your environment" means deep, continuous *logical* visibility — not physical residency.** Sales copy must not imply otherwise.

The Unified Integration Layer:

```
Credential Manager (IAM STS + external ID, OAuth 2.0, scoped API keys, SAML/OIDC)
  → Evidence Collector (unified polling)
    → World Model graph
      → Exploitability Engine (LLM reasoning + rule engine + sandbox)
```

Physical residency — the `dux-resident-agent` DaemonSet — is **Gate 5 only**. The gate-by-gate customer-facing message table lives in [gtm-guardrails](../80-gtm/gtm-guardrails.md).
