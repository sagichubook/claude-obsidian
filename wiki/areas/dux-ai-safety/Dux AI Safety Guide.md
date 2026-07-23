---
type: area
title: "Dux AI Safety Guide"
address: c-000003
topic: "dux/ai-safety"
created: 2026-07-22
updated: 2026-07-23
tags: [area, dux, dux/ai-safety]
status: mature
sources: [".raw/dux/40-ai-safety/safety-overview.md", ".raw/dux/40-ai-safety/governance-kernel.md", ".raw/dux/40-ai-safety/kill-switch-hitl.md", ".raw/dux/40-ai-safety/camel-plane.md", ".raw/dux/40-ai-safety/sandbox-execution.md", ".raw/dux/40-ai-safety/mcp-security.md"]
related: ["[[Dux]]", "[[Dux AI Safety Operations Reference]]", "[[Dux Product Guide]]", "[[Dux Feature Reference]]", "[[Dux Architecture Guide]]"]
---

# Dux AI Safety Guide

### An agent with your CrowdStrike credentials is either your best analyst or your worst incident — this is how Dux makes sure it's the former

Navigation: [[Dux]] | [[Dux AI Safety Operations Reference]] | [[Dux Architecture Guide]]

Picture the agent's actual working day. It reads a customer's asset inventory, network topology, identity graph, and every control already deployed. It ingests a stream of CVE text, exploit code, and threat intel scraped from the open internet — content nobody vetted before it landed in the agent's context window. And then, with nothing more than its own judgment and a set of API credentials, it reaches out and *acts*: isolating an endpoint, pushing a firewall rule, filing a ticket against a real person's queue. Read everything, trust nothing, and be able to touch production. Security researchers have a name for that combination: the **lethal trifecta**, and Dux agents hold all three legs of it by design, because that's what makes them useful.

1. **They read private customer data** — assets, runtime, identity, network, existing controls.
2. **They ingest untrusted content** — CVE text, exploit code, threat intel.
3. **They can act externally** — mitigation deployment, vendor API calls.

Hold all three at once and you've built a lateral-movement vector standing inside the customer's own environment: prompt-inject the agent through a poisoned CVE writeup, and the attacker inherits every credential and network path the agent holds. This is the 2026 supervisor risk Red Hat has been warning enterprises about — an agent with access to everything is a single point of compromise. The rest of this document is the answer to that risk, leg by leg.

| Leg | Control |
|-----|---------|
| (1) private data | RLS FORCE + tenant isolation contain what any single agent can reach |
| (2) untrusted content | CaMeL neutralizes the injection vector; role-typed tool allowlists bound the blast radius |
| (3) external action | Governance-kernel gates + kill switch + hash-chained audit on every write; human review escalates on anomaly |

**The CISO-facing statement of this posture:** Dux takes governed, audited, kill-switch-backed action in your environment at machine speed. Human review is available on request per asset class, and automatic on anomalies — it is not required on every write.

That last sentence is a deliberate design choice, not a hedge. At Gate 1, executed investigation code ships unattended, and three of five vendor write actions — `network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation` — ship **unattended by default**. Each still inherits the full spine below: tenant isolation, governance gates, kill switch, and hash-chained audit. Human review is an anomaly-escalation path for those three, not a gate on every write. The two fleet-impacting actions — `endpoint.isolate`, `patch.deploy_special_devices` — are the deliberate exception: mandatory HITL on every call, until each earns unattended execution via a field-proven safety record (D-17).

## Dux itself is the target — a credential honeypot by construction

Before getting into the controls, it's worth sitting with the uncomfortable architectural fact they exist to answer. Every marketed write — isolate endpoint, deploy policy, create ticket — executes against the **customer's** CrowdStrike, Intune, or ServiceNow, using **the customer's credentials, held by Dux**. That means a platform compromise doesn't just leak Dux's own data; it hands an attacker write access to every customer's EDR, cloud, and ITSM in one shot. This is the highest-value target the architecture creates, and any serious enterprise security reviewer will ask about it directly.

The answer is three compounding controls, aimed equally at agent misjudgment and at platform-compromise replay:

- **Least-privilege scoped action credentials, per connector.** Write scope is limited to the 5 canonical actions; never broad admin.
- **Per-action credential minting** — short-lived, not long-lived stored keys, wherever the vendor supports it. Where it does not: AES-256 at rest with SSM/Vault transit (ADR-011).
- **Bounded blast radius.** Worst case per tenant is the canonical action set on connected assets, bounded by governance-kernel budget and effect gates, the kill switch, and hash-chained audit. Replay of an approved action is countered by `mutation_key` idempotency plus audit anchoring.

## The six-control safety spine

Whatever else changes as the product evolves, every agent surface — investigation, chat, physical-residency — inherits all six of these, without exception:

| Control | Guarantee |
|---------|-----------|
| RLS FORCE tenant isolation | no cross-tenant read or write is reachable, including by a compromised agent |
| CaMeL dual-LLM boundary | untrusted content never reaches a tool-calling context |
| MCP gateway | deny-by-default tool access |
| Kill switch KS-L1–L4 | halt propagates in <5 s, tenant-scoped |
| Hash-chained audit | every action is recorded and tamper-evident |
| AIBOM CI | supply chain is pinned and drift-blocked |

### Defense in depth, layer by layer (L1–L8)

Zoom in on any single request and it passes through eight stacked layers before it can touch a customer's infrastructure — no single layer is trusted to hold alone:

| Layer | Control | Addresses |
|-------|---------|-----------|
| L1 Input containment | CaMeL dual-LLM | ASI01, ASI06 |
| L1b Output classifier | inline classifier on S-LLM output, pre-schema (Month 2) | ASI01 |
| L2 Structured output | constrained decoding (FSM / JSON schema) | ASI01, ASI05 |
| L3 Tool contracts | MCP hash pinning + schema validation | ASI02, ASI04 |
| L4 Identity | JWT with SPIFFE-format claims (SPIRE target Month 3) | ASI03 |
| L4b Inter-agent auth | signed service JWT per hop | ASI07 |
| L5 Execution isolation | Self-hosted Firecracker microVM on K8s (Gate 1) + AST pre-scan; gVisor read-only as defense in depth | ASI05, ASI10 |
| L6 Governance gates | Intent + Budget + Effect + DLP kernel | ASI01–ASI10 |
| L7 Audit | HMAC-SHA256 hash chain + hourly MinIO Object Locking anchoring | ASI09 |
| L8 Kill switch | <5 s propagation, tenant-scoped | ASI10 |

### The STRIDE pass

Threat-modeling this from the classic six angles surfaces nothing exotic — the interesting part is that every row already has a shipping mitigation, not a roadmap item:

| Threat | Vector | Mitigation |
|--------|--------|-----------|
| Spoofing | agent-to-platform auth | JWT / SPIFFE workload identity |
| Tampering | data in transit | TLS 1.3; mTLS service-to-service (seed) |
| Goal hijack / injection | untrusted CVE and intel | CaMeL dual-LLM + Intent gate |
| Supply chain (ASI04) | MCP tool schema drift | hash pinning + CI drift block + rotation |
| Repudiation | audit logs | HMAC chain (Vault key) + MinIO Object Locking anchoring |
| Information disclosure | customer data | RLS; AES-256; field-level PII encryption |
| Denial of service | API / assessment flood | `@nestjs/throttler` + Valkey; circuit breakers |
| Elevation of privilege | agent tool escalation | least-privilege RBAC; MCP allowlist; governance gates |

## Where each control is specified

The spine above is an index, not the whole story — each control has a dedicated deep-dive, either as its own file or as a section below:

| Concern | Specification |
|---------|---------------|
| Dual-LLM boundary, prerequisite schema, CDA, retrieval | [camel-plane](camel-plane.md) |
| GOV-001–014 synchronous gates | [governance-kernel](governance-kernel.md) |
| KS-L1–L4 and the HITL contract | [kill-switch-hitl](kill-switch-hitl.md) |
| Agent identity, lifecycle, shadow AI | agent-identity |
| MCP policy PS-001–016, tool catalog | [mcp-security](mcp-security.md) |
| Self-hosted Firecracker microVM, AST scan, decision matrix | [sandbox-execution](sandbox-execution.md) |
| Confidence ensemble, Platt scaling, abstention | confidence-calibration |
| OWASP ASI01–10 / LLM01–10 / MCP crosswalk | owasp-assessments |
| The 12 canonical agentic incident runbooks | incident-runbooks |

## What "safe enough to ship" means at Gate 1

Every safety program eventually has to answer the question a board or a lead investor will ask: what's actually blocking launch? For Dux, the Gate-1 safety exit criteria are narrow and specific:

- OWASP LLM and Agentic assessments at **Partial or better**.
- **ASI01 and ASI02 Implemented.** ASI10 Partial or better — kill switch and cost cap Implemented; eBPF deferred to Series A Month 9.
- **LLM01, LLM06, LLM10 Implemented.**
- **LLM09 is the only Gate-1 blocker** — the EXP-CIT-001 citation test.

One test, one row, standing between the current state and Gate 1. Everything documented below exists to get the rest of the program to that same bar.

## The governance kernel: a synchronous chain nothing bypasses

If the six-control spine is the constitution, the governance kernel is the courtroom every privileged action has to pass through before it executes. It's a synchronous, fail-closed chain of gates evaluated before every privileged agent action, with no bypass path in Phase 1. Gates run as a Chain of Responsibility, each handler returning `continue`, `block`, or `escalate`:

```
IntentGate → BudgetGate → EffectGate → VendorActionGate → HITLGate
```

### The kill switch short-circuits everything upstream of it

Before any gate even runs, the kernel consults `KillSwitchRelay`. `KillSwitchRelay.checkBeforeOperation(tenantId, sessionId)` runs before every LLM call and every MCP tool invocation, checking the L1–L4 keys in parallel — so an active L3-or-higher switch returns 503 without running the chain at all, and emits a `governance.kill_switch_short_circuit` audit record carrying `tenant_id`, `session_id`, `level`, and the gates skipped. Nothing downstream gets a chance to argue.

### The full gate table, GOV-001 through GOV-014

Fourteen gates, each with a narrow job and a specific failure mode:

| ID | Gate | Checks | Failure |
|----|------|--------|---------|
| GOV-001 | Intent | action matches the `assessment_plan` schema (allowed tool sequence, target CVEs, max steps) | `GOVERNANCE_BLOCKED` → AI-AGENT P1 |
| GOV-002 | Budget | token and cost limits per tenant/session | `BUDGET_EXCEEDED` → AGENTIC-SAAS P0-C if >3× baseline |
| GOV-003 | ActionBudget | per assessment: warn >100, halt ≥200 weighted actions (LLM = 1, MCP read = 2, MCP write = 5); monotonic replay-safe counter | `ACTION_BUDGET_WARN` / `ACTION_BUDGET_EXCEEDED` |
| GOV-004 | WorkflowTenantBudget | daily cap — Starter 500 / Pro 5,000 / Enterprise floor 50,000. Changing it requires both tenant and platform admin | `WORKFLOW_TENANT_BUDGET_EXCEEDED` |
| GOV-005 | WorkflowCircuitBreaker | tenant exceeds 2× its rolling 7-day baseline actions/hour | L2 `workflow_cost_runaway` + banner within 15 min |
| GOV-006 | CostForecast | forecast before start; re-forecast every 25 actions, or when asset count grows by 50% | `COST_FORECAST_EXCEEDED` |
| GOV-007 | CostCap | hard per-tenant spend cap ($25/hour default), checked before every LLM call | L2 `budget_exceeded`; `GOVERNANCE_BLOCKED` |
| GOV-008 | Effect | tool side effects stay within tenant scope; blast tiers — single asset → T2, subnet → T3, tenant-wide → T4 | block + audit |
| GOV-009 | DLP | regex (<5 ms) in Weeks 1–10; Presidio NER from Week 11, after a 2-week shadow run (enforce only once FP <5%) | redact or block |
| GOV-010 | Loop | max 50 P-LLM iterations per assessment; max 10 MCP calls per P-LLM iteration; high-blast agents capped at `min(50, 10 × P-LLM iterations)` | `GOVERNANCE_BLOCKED`, optionally L1 |
| GOV-011 | PromptScreen | the trusted user prompt is screened before the P-LLM plan (CaMeL+) | `PROMPT_SCREEN_BLOCKED` |
| GOV-012 | OutputAudit | S-LLM / Q-LLM output scanned for instruction leakage before it reaches the P-LLM | `OUTPUT_AUDIT_BLOCKED` |
| GOV-013 | TieredRisk | a tool's blast-radius tier must match the session `risk_tier` | `TIERED_RISK_BLOCKED` → HITL per tier |
| GOV-014 | VendorActionGate | the canonical write action is checked against its `GOV-TOOL-*` row (below) — the action's evidence-confidence floor is met, and a rollback procedure is on file | below the confidence floor: `VENDOR_ACTION_BLOCKED` → HITL escalation at the tool's tier |

### How cost thresholds are evaluated

Runaway spend is one of the most common ways an agentic system fails in the wild, so Dux checks it three separate ways, in a fixed order — first match wins. The thresholds are derived from the R2 envelope (D-3):

1. Per-assessment soft circuit breaker at **$0.675 / assessment** (ADR-008).
2. Hourly **CostCap $25/hour** (GOV-007).
3. Workflow circuit breaker at **2× baseline** (GOV-005).

Token-runaway runbooks cite this order, not CostCap alone. `CostCapGate` (`packages/core/governance/`) queries hourly spend per tenant; on breach it calls `KillSwitchRelay.activate(L2, tenantId, 'budget_exceeded')`.

### Keeping the gate chain fast

None of this matters if it makes the agent unusably slow, so every gate carries a latency budget:

| Period | DLP engine | Budget |
|--------|-----------|--------|
| Weeks 1–10 | regex | sequential p99 <75 ms total; DLP <5 ms; every other gate <15 ms |
| Week 11+ | Presidio NER | DLP <60 ms p99; parallelize DLP with Intent where safe |

Prometheus exports `governance_gate_denials_total{gate,reason}` and `governance_gate_duration_seconds`, and `pnpm test:governance-kernel` exercises the GOV-001–014 failure modes as a merge-blocking check before Gate 1.

## VendorActionGate: the risk matrix that decides who executes unattended

GOV-014 is the gate that actually decides whether a write happens now or waits for a human, so it earns its own section. `VendorActionGate` authorizes every canonical write action against the `GOV-TOOL-*` risk matrix — consequence scope, reversibility, and an unattended-execution confidence floor per action — and falls back to a HITL tier below that floor. Each action's compensating rollback procedure is on file in the [mitigation-write-path rollback catalog](../10-product/features/mitigation-write-path.md). Connectors must not call vendor mutation APIs directly.

| ID | Canonical action | Consequence scope | Reversibility | Unattended confidence floor | Below floor | ASI linkage |
|----|-------------------|--------------------|----------------|------------------------------|--------------|--------------|
| GOV-TOOL-01 | `endpoint.isolate` | single asset (T3) | reversible — rollback R-01 | none — mandatory HITL T3 regardless of confidence, until this action class is promoted (D-17) | HITL T3 (mandatory) | ASI02, ASI04 |
| GOV-TOOL-02 | `network.blocklist_add` | subnet (T2) | reversible — R-02 | ≥0.75 | HITL T2 | ASI02, ASI04 |
| GOV-TOOL-03 | `policy.deploy_device_config` | subnet (T2) | reversible — R-03 | ≥0.75 | HITL T2 | ASI02, ASI04 |
| GOV-TOOL-04 | `patch.deploy_special_devices` | single asset (T3) | conditional — R-04; firmware-only devices have no API-level rollback | none — mandatory HITL T3 regardless of confidence | HITL T3 (mandatory) | ASI02, ASI04, ASI09 |
| GOV-TOOL-05 | `ticket.create_remediation` | tenant metadata only (T1) | reversible — R-05 | ≥0.60 | n/a — T1 always executes unattended | ASI02 |

`VendorActionGate` evaluates a write call against its `GOV-TOOL-*` row before the vendor adapter runs: below the confidence floor, on `GOV-TOOL-04` with no API-level rollback available, or on any mandatory-HITL row (`GOV-TOOL-01`, `GOV-TOOL-04`), it returns `escalate` at the row's HITL tier instead of `continue`. The confidence floor is the assessment's calibrated exploitability confidence (see confidence-calibration), not a raw model logit.

### Earning autonomy: the path off mandatory HITL (D-17)

`GOV-TOOL-01` (`endpoint.isolate`) earns the same earned-autonomy path already defined for `GOV-TOOL-04`-class actions: it moves off mandatory HITL only once the Gate-3 `ClosedLoopValidationWorkflow` has a field-proven safety record for that action class — a minimum observed sample of HITL-approved executions with zero unrecovered false-positive isolations. Until that record exists, `endpoint.isolate` and `patch.deploy_special_devices` are the only two canonical write actions that do not execute unattended at Gate 1.

### The one invariant no future action can override

No future `GOV-TOOL-*` row may allowlist a write action that disables or weakens MFA, logging, encryption, or audit — on the target system, on Dux's own control plane, or on the audit trail of the action itself. This is a standing constraint on the `GOV-TOOL-*` allowlist, not a per-action judgment call: `VendorActionGate` (GOV-014) is the same gate that already blocks unattended execution of any action whose rollback entry is missing, and it blocks on this invariant the same way — a proposed action that trips it never reaches `continue`, regardless of confidence or blast tier. `pnpm test:governance-kernel` is merge-blocking on this invariant exactly as it is on GOV-001–014's other failure modes: a new `GOV-TOOL-*` row cannot land without a passing case proving it does not touch MFA, logging, encryption, or audit posture.

Every `GOV-TOOL-*` row names a rollback ID (R-01…R-05); the compensating procedure for each lives in the mitigation-write-path rollback catalog — `VendorActionGate` will not authorize unattended execution of an action whose rollback entry is missing.

## Kill switch and human-in-the-loop: the halt authority

Every safety architecture eventually needs an emergency stop, and Dux's is layered rather than binary — you don't have to choose between "kill one bad session" and "shut down the platform."

### The four levels

| Level | Scope | Trigger authority | Use case |
|-------|-------|-------------------|----------|
| KS-L1 Session | a single assessment workflow | session owner, system | runaway single assessment |
| KS-L2 Tenant assessment | all assessment sessions plus the queue, frozen for one tenant (dashboard unaffected) | tenant admin, platform admin | compromised agent; tenant assessment incident |
| KS-L3 Tenant platform | all tenant-facing agent features; dashboard goes read-only | tenant admin, platform admin | tenant-wide security or billing incident |
| KS-L4 Global | the entire platform agent fleet | platform admin + on-call (two-person) | critical vulnerability, LLM outage, cross-tenant leak |

### How fast a halt actually propagates

A kill switch that takes minutes to take effect isn't much of a kill switch, so propagation is measured and tested per level:

| Level | Path | Target | Degradation |
|-------|------|--------|-------------|
| L1 | Unleash flag + session terminate | ≤30 s (`unleash_evaluation_duration_ms` p99) | manual `admin:agent-halt` |
| L2 | `KillSwitchRelay` → NATS core pub/sub → workers poll `kill_switch_flags` | p99 <5 s (**KS-001**) | CloudNativePG `LISTEN/NOTIFY` (KS-007, <10 s) |
| L3 | as L2, plus dashboard read-only middleware | p99 <5 s | page `@platform-oncall` |
| L4 | as L2, plus global API assessment halt | p99 <5 s | two-person approval required |

Two SLOs, never conflated: **propagation** (KS-001) covers KS-L2–L4 relay only, at p99 <5 s — KS-L1 is a separate Unleash path at ≤30 s. **Execution interruption** is p99 <30 s: long activities check the flag on heartbeat ticks (every 10–30 s), so worst-case in-flight bleed after an L3 or L4 is one heartbeat interval — roughly 30 s, **not** `startToCloseTimeout`.

**A dependency note worth stating explicitly:** KS-L1 evaluates flags via the standard self-hosted Unleash SDK/server ([architecture-overview §6](../20-architecture/architecture-overview.md)), **not** Unleash Edge — the proxy/cache layer that's EOL 2026-12-31. The kill switch has no dependency on Edge and is unaffected by its retirement. If Edge is adopted later for read-scaling, re-verify this before KS-L1 comes to depend on it.

### What's actually running under the hood

| Component | Detail |
|-----------|--------|
| NATS key/subject | `kill.{level}.{target_id}` — L4 is `kill.L4.global` |
| Pub/sub channel | `kill.switch.channel` (NATS core, self-hosted on K8s) |
| Fallback | CloudNativePG `kill_switch_flags` mirror, polled every 2 s when NATS is unavailable |
| Worker circuit breaker (fail-closed) | 2 consecutive missed NATS polls → mark the session `killed` in CloudNativePG, cancel in-flight steps, **then** SIGTERM — never SIGTERM before cancellation |

KS-007's timeline: a Week-2 spike test, then a Gate-1 release gate (`pnpm ops:test-kill-switch --fallback pg-notify --level L3 --iterations 50`), then mandatory cross-cutting testing at Gate 2.

### What actually pulls the trigger

Manual activation runs through the Admin UI (reason ≥10 characters), `POST /v1/admin/kill-switch`, or CLI: `pnpm admin:kill-switch --level L3 --tenant <uuid> --reason "…"`. But most halts should never need a human to notice first:

| Condition | Level |
|-----------|-------|
| 5 MCP denials in a session | L1 |
| >10 K tokens/min | L1 |
| >60 tool calls/min | L1 |
| Stripe failure >3 days / >7 days | L2 / L3 |
| >50% tool errors in 5 min, per tenant | L2 |
| Cross-tenant leak | L4 (two-person, `#security`) |
| HITL T4 escalation timeout, 30 min | L2 |

Every automatic trigger writes a `trigger=automatic` audit record and alerts `#security`. **An automatic trigger cannot self-clear** — clearing requires a platform admin plus false-positive evidence, a deliberate friction against a compromised or panicking operator waving the incident away too quickly.

### What each level actually does, and who's allowed to pull it

| Level | Behavior | Authority to trigger |
|-------|----------|---------------------|
| L1 | terminates the target session (cancel, no retry); new sessions still allowed | session owner or system |
| L2 | freezes the tenant's assessment sessions and queue; dashboard otherwise normal | tenant or platform admin |
| L3 | terminates all agent sessions; dashboard read-only | tenant or platform admin (MFA for platform admin) |
| L4 | terminates everything and blocks all | platform admin + a second admin's `#security` acknowledgement, persisted to `kill_switch_approvals` |

Activation is idempotent — a repeat call returns 200 with `{status: already_active, kill_id, activated_at, propagation_ms}` rather than erroring or double-firing. Deactivation requires equal-or-higher authority; L3 and L4 additionally require a post-incident ticket, and L4 carries a 15-minute minimum cooldown. Target false-positive rate: **<1% of sessions**.

### The business side of pulling the emergency brake

A kill switch has revenue consequences, so PM sign-off is built into the escalation path. MRR-at-risk is computed per level as `monthly_MRR × fraction × hours / 730`. Status-page, banner, and email copy live in customer-lifecycle.

| Role | Responsibility |
|------|----------------|
| Founder or PM | Phase-1 approver; async Slack, ≤15 min SLA |
| `@product-oncall` | approver post-Gate-2 |
| Founder + PM | both required before any external communication on an L4 |
| AI Safety Lead (`@ai-safety-oncall`) | mandatory halt authority within 60 s of a T3 escalation or an L4 suspicion |

### The HITL contract: what "human in the loop" actually means for five write actions

**Writes execute unattended by default, except the two fleet-impacting actions (D-17).** Three canonical write actions — `network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation` — execute at Gate 1 without waiting for a `hitl_response`; for these three, the tier and transport mechanics below are the **anomaly-escalation path**, not a default approval gate. The two actions with the largest single-action blast radius — `endpoint.isolate`, `patch.deploy_special_devices` — require a live `hitl_response` on **every** call, mandatory regardless of confidence (`GOV-TOOL-01`/`GOV-TOOL-04`), until each earns unattended execution via a field-proven Gate-3 safety record.

Tiers T1–T4 still classify every write, for audit and for escalation. The flow:

1. The governance kernel intercepts the write tool call and assigns a tier.
2. For `endpoint.isolate` / `patch.deploy_special_devices`, it raises a live HITL record immediately and **waits** for `hitl_response`. For the other three, it checks `pre_approved_scope[tenant]` and T2 embedding similarity.
3. On the three earned-autonomy actions, it **executes immediately**, writing the tier and the impact preview to the hash-chained audit trail.
4. On those three only, it raises a live HITL record **on an anomaly**: a confidence-calibration abstention band, a sandbox `TIMEOUT` or `OOM` (D-9), or a T4 extreme-blast-radius outlier.

| Transport | Message | Required fields |
|-----------|---------|-----------------|
| SSE (server → client) | `hitl_request` — mandatory pre-approval on `endpoint.isolate`/`patch.deploy_special_devices`; anomaly escalation only on the other three | `requestId`, `agentId`, `toolName`, `canonicalActionId`, `vendor`, `nativeActionName`, `parameters`, `timeoutAt` (ISO 8601), `tier` (T1–T4), `affectedResources[]`, `blastRadius` (low/medium/high), **`rollbackProcedure` (URL)** |
| POST (client → server) | `hitl_response` — same split | `requestId`, `decision` (approve/deny), `userId` |

A denial returns `GOVERNANCE_BLOCKED`. A T3 request times out at 15 min to `@ai-safety-oncall`; a T4 request times out at 30 min to an automatic L2. Round-trip SLA is p95 <15 s on the T3 synchronous path. Transport is HTTP/2, capped at 5 concurrent SSE streams per `user_id`, with `Last-Event-ID` replay from `chat_session_events` over a 1-hour window.

The approve/deny UI (`hitl_ui`, `chat_write_tools` — D-4) is a **Gate-1 blocker** for `endpoint.isolate` and `patch.deploy_special_devices` — those two actions cannot ship before the surface exists. For the other three it remains the anomaly-escalation surface only.

**Pre-execution impact preview (H4).** An approver must see impact before approving: `affectedResources[]`, `blastRadius`, and `rollbackProcedure` render as an impact preview — pre-approved, not post-hoc, on `endpoint.isolate`/`patch.deploy_special_devices`. On the three earned-autonomy actions the same fields are written to the audit record instead — post-hoc reviewable rather than pre-approved.

**Pre-approved scope (H5).** `pre_approved_scope[tenant]` applies to the three earned-autonomy actions, where it spares a human approver from alert fatigue on anomaly escalations. A tenant that wants to require human review for specific high-sensitivity asset classes on those three configures it here; `endpoint.isolate`/`patch.deploy_special_devices` are HITL-gated regardless of this setting.

Every `rollbackProcedure` URL above resolves to one of the five entries in the mitigation-write-path rollback catalog (R-01…R-05) — the audit-side rollback record backs every action; for the two mandatory-HITL actions it also renders in the pre-approval impact preview, not only post-hoc.

### A second, separate gate: confidence in the investigation itself (D-34)

This is a different axis from the write-action HITL above — it gates the Agentic RAG investigation output itself (a confidence score on the synthesized exploitability finding), not a vendor write action. The two gates can compose: a low-confidence investigation can still route to an unattended-by-default write once it clears this gate, subject to the write-action rules above.

ADR-020 R2 specifies the mechanism: a confidence band of 0.85–0.95 on the `agenticRAGWorkflow` synthesis step routes to a signal-based human-approval gate, 2-hour timeout, escalating to on-call — the same `WorkflowPort` HITL-signal primitive as every gate, not a new transport, and the same API surface as the write HITL: `GET /chat/sessions/{id}/stream` (`hitl_request`) / `POST /chat/sessions/{id}/hitl-response`.

| Confidence / DUX-score band | Required approval | Timeout action |
|---|---|---|
| High confidence (≥0.95) / DUX score 9.0–10.0 | Synthesis proceeds unattended; if a downstream write is triggered, CISO or Security Lead per that action's own tier | — |
| Gate band (0.85–0.95) / DUX score 7.0–8.9 with confidence <0.95 | Manager-level approval on the investigation itself, via the signal-based gate above | escalate to on-call after 2 h (ADR-020 R2) |
| Below gate band (<0.85) | Loop continues (max 5 iterations) rather than escalating — insufficient confidence to present for approval at all | — |
| Bulk remediation >100 assets (downstream write, not the investigation gate) | Change Advisory Board, per the write-action tier rules | decompose to smaller batches, not a timeout |
| First autonomous mitigation in a tenant (downstream write) | Security Team, per write-action tiers | hold for validation |

Network segmentation and IAM-policy-modification writes route through the existing tier/action framework once a write is proposed; this table only extends the investigation-confidence axis that precedes any write decision.

### Proving the kill switch actually works

A kill switch nobody has tested is a kill switch that fails when it matters most:

| Test | Assertion | Cadence |
|------|-----------|---------|
| KS-001 | L2–L4 propagation <5 s p99 | every agent-code PR |
| KS-004 | L4 <10 s p99 | monthly staging smoke, release-blocking |
| KS-005 | Valkey degradation → <10 s DB polling | quarterly |
| L3 staging drill | zero new tool calls within 5 s | quarterly |
| L4 chaos | all workers halt | annually |
| `pnpm test:kill-switch-idempotency` | no step resumes after L2/L3/L4; in-flight tool calls return `GOVERNANCE_BLOCKED` | every agent-code PR |

NIST AI RMF GOVERN: the policy owner is Founder + Security, reviewed quarterly.

## CaMeL: the dual-LLM boundary against a poisoned CVE writeup

Everything above assumes the agent's reasoning hasn't already been hijacked. CaMeL is the layer that keeps it that way — Dux's primary Phase-1 defense against prompt injection carried by untrusted content. It's based on CaMeL (Google DeepMind, arXiv:2503.18813), with CaMeL+ hardening (arXiv:2505.22852; Microsoft Zero Trust SFI 2026).

The core idea is a strict split of labor between two models that never fully trust each other:

| Component | Role | Capability |
|-----------|------|-----------|
| **S-LLM** (Suspicious) | processes untrusted content — CVE descriptions, exploits, threat intel | read-only; **never** executes tools; its output is treated as untrusted |
| **P-LLM** (Privileged) | processes sanitized tool results; drives the agent loop | executes tools; **never** sees raw untrusted content or unsanitized MCP output |

### How data actually flows through the split

```
CVE text → S-LLM (extract prerequisites, JSON schema) → structured output → P-LLM (reason + tool calls)
         ↘ validation fail (≤3 retries)
           → agent_audit_log.camel.schema_validation_failed
           → GOVERNANCE_BLOCKED or L1

MCP tool result → S-LLM sanitizer OR structured parser → P-LLM (never raw tool JSON)
```

This isn't a separate durable runtime — it's request/response middleware invoked synchronously from Temporal workflow activities. The path per tool dispatch is `Activity → Governance Kernel → CaMeL S-LLM/P-LLM split → MCP Gateway → Activity return`. Activity timeout is 30 s wall-clock, after which a partial result escalates to HITL T3.

**The critical invariant, stated plainly:** P-LLM tool schemas **must not** contain unconstrained `string` fields carrying S-LLM output. Everything crossing S-LLM → P-LLM passes through strictly typed, schema-validated structures with `additionalProperties: false`. This is the boundary; a single free-text field defeats it.

### CaMeL+ hardening

| Control | Implementation | Gate |
|---------|----------------|------|
| **Input screening** — scan trusted user prompts for injection before P-LLM planning | regex + lightweight classifier (`packages/core/camel/prompt-screen.ts`), <5 ms p99 | GOV-011 |
| **Output auditing** — detect instruction leakage or tool-call hints in S-LLM output | schema validation + leakage patterns; logs `camel.output_audit_failed` | GOV-012 |
| **Tiered-risk access** — map tool tiers to HITL: read MCP = T1, write = T3+, tenant-wide = T4 | extends the Effect gate (GOV-008) | GOV-013 |

### The S-LLM output boundary, in practice

Zod / JSON Schema with `additionalProperties: false`. Max 3 retries, then `GOVERNANCE_BLOCKED` — optionally L1 if injection is suspected.

The `description` field is a controlled vocabulary (enum) only. No free-text natural language ever enters P-LLM context. The full untrusted text is retained as `sources[].content_hash` (SHA-256) for audit; the P-LLM never sees raw prose.

```json
{ "type": "object", "additionalProperties": false,
  "required": ["prerequisites", "sources"],
  "properties": {
    "prerequisites": { "type": "array", "items": { "type": "object", "additionalProperties": false,
      "required": ["id","description","requires","affected_services"],
      "properties": {
        "id": { "type": "string" },
        "description": { "enum": ["network_access","authentication","local_access","user_interaction","privilege_escalation","none"] },
        "requires": { "enum": ["network_access","authentication","local_access","user_interaction"] },
        "affected_services": { "type": "array", "items": { "type": "string" } } } } },
    "sources": { "type": "array", "items": { "type": "object", "additionalProperties": false,
      "required": ["name","url","confidence"],
      "properties": { "name": { "enum": ["nvd","github","exploitdb","threat_intel"] },
        "url": { "type": "string", "format": "uri" },
        "confidence": { "type": "number", "minimum": 0, "maximum": 1 } } } } } }
```

**Citation logic (US-001).** Top 3 sources by `confidence` descending; ties broken by source priority — NVD > ExploitDB > GitHub > threat_intel.

**SSRF protections.** Allowlisted domains only (`nvd.nist.gov`, `github.com`, `exploit-db.com`, curated intel hosts). Private and link-local IPs are blocked on any server-side fetch.

### Five ways a Constrained Decoding Attack gets stopped

A CDA tries to smuggle instructions through the narrow, schema-constrained channel itself. Dux layers five separate mitigations against it:

1. **Schema minimization** — only essential fields; no large enum dictionaries.
2. **Output sanitization** — strip instruction-like patterns before P-LLM ingestion.
3. **Critic cross-check** — compare the P-LLM's `affected_services` against `getAssetsForCVE`. A mismatch downgrades confidence and escalates to HITL T3.
4. **Calibration independence** — the 3-signal ensemble (EPSS, KEV, asset exposure) is independent; the S-LLM's own `confidence` must not influence it.
5. **Connector poisoning** — connector-asserted controls are untrusted for negative verdicts. "A control blocks this exploit" requires independent AWS-API corroboration (native `query_controls`), never the connector cache alone. A sudden coverage jump — more than 20% of assets gaining a blocking control within 24 h — downgrades confidence, escalates to HITL T3, and writes a `connector_drift_anomaly` audit record.

Worth being precise about what #5 catches and what it doesn't: this detects velocity, not drift-from-baseline. The 20%-in-24h check catches a sudden jump in asserted coverage; it does not catch a slow, sub-threshold erosion away from an asset's established control baseline, and there is no per-asset baseline snapshot to erode away from today. Config-drift-vs-baseline detection is an explicit roadmap item, not a silently-covered case of this check.

### The one attack redundancy can't fully close

Not every risk gets a clean mitigation. **Branch Steering** is a data-flow attack that redundancy defenses cannot fully block, even under strict dual-LLM isolation (arXiv:2601.09923 — the same work that names architectural isolation the only known robust defense). This is an **accepted residual risk**; the critic cross-check above is a partial mitigation, not a fix.

### Benchmarking the defense, honestly

AgentDojo v1.2, pinned in the AIBOM. Regression suite: `pnpm test:camel-benchmark`.

Two metrics get quoted in this space, and they are easy to conflate — Dux is deliberate about never doing so:

| Metric | Value | Meaning |
|--------|-------|---------|
| CaMeL+ paper task completion | 67% | completion under a strict untrusted-data policy — the enterprise-hardening baseline |
| Dux CI target | ~77% defended vs ~84% undefended | defense-layer uplift, **not** a raw completion rate |

Any customer-facing copy citing a percentage must state which metric it means.

## The World Model: what the agent is allowed to know

Underneath the dual-LLM split sits a logical abstraction over tenant-scoped PostgreSQL — `ASSET`, `ASSET_RELATIONSHIP`, `CONTROL`, `FINDING`, `CVE`, `USER_PREFERENCE`, versioned through `world_model_versions`.

| Query | Purpose |
|-------|---------|
| `getAssetsForCVE` | assets affected by a CVE |
| `getControlsForAsset` | controls in force on an asset |
| `summarizeContext` | ≤2 K-token summary via `gpt-5.4-mini` |

Rate limit: 100 queries/min/tenant, then `BUDGET_EXCEEDED`. Chat read-replica lag stays under 5 s.

Large tenants get compressed context rather than a flood of raw rows, selected by asset count:

| Tier | Asset count | Context |
|------|-------------|---------|
| L1 | <100 | full detail, ≤32 K tokens |
| L2 | 100–1 K | subnet / VPC summaries, ≤16 K tokens |
| L3 | >1 K | risk profile only, ≤8 K tokens |

## Agentic RAG: why retrieval was off, and why it's on now

For most of Phase 1, Dux deliberately ran with **no vector RAG** — structured SQL retrieval only, with `pgvector`/`tenant_embeddings` provisioned but sitting inactive. That wasn't an oversight; it was a posture, with two explicit triggers that would justify flipping it on: a chat p95 SLO breach attributable to structured-query limits, or a threat-intel corpus exceeding structured-ingest capacity.

Neither trigger fired organically. Instead, `rag_enabled = true` shipped by direct decision on 2026-07-19 (D-34) — Agentic RAG, structured SQL retrieval extended with pgvector + Apache AGE graph retrieval and constrained decoding (ADR-020 R2), superseding the wait-for-a-trigger posture above.

### The enable-precondition scorecard

Flipping on retrieval isn't free — it's gated behind five preconditions. Four are confirmed; one remains open:

- **Tenant-scoped isolation — confirmed implemented.** `tenant_embeddings` declaratively partitioned by `tenant_id` with a local HNSW index per partition — ANN recall is tenant-scoped by construction, not by index-key ordering.
- **Hybrid vector + BM25 — confirmed implemented.** Agentic RAG runs hybrid vector + BM25 retrieval over `tenant_embeddings`, extended with the Apache AGE graph layer.
- **Poisoning controls — embedding-signing now specified.** Graph-edge poisoning has per-edge provenance + integrity hashing (ADR-020 R2), and connector rows carry `source_connector_id`/`integrity_hash` for transport integrity. `tenant_embeddings` rows get the identical pattern (D-55): each row carries an `integrity_hash` — SHA-256 over `(source_content, embedding_vector, tenant_id, source_connector_id)` — computed at write time and checked at retrieval time. A valid hash confirms the embedding hasn't been altered since write, but the hash alone cannot justify a false-negative verdict (CDA #5) — it's tamper-evidence, not an attestation that the source content itself was truthful.
- **LLM04/LLM08 reassessment — closed (D-55).**
- **ISO-012 adversarial-neighbor test execution — still open, tracked as OI-59.**

### Connector trust, stated once and applied everywhere

Connector-synced rows carry `source_connector_id` and `integrity_hash` — these establish *transport* integrity only. They may support positive evidence, but cannot on their own justify a false-negative verdict (CDA #5).

## Sandbox execution: where the agent's own code actually runs

CaMeL keeps untrusted content from hijacking the agent's reasoning. Sandboxing keeps the agent's own generated code from breaking out of its box — a genuinely different threat, since the code executing isn't attacker-controlled input, it's the agent's own output, and it still has to be treated as hostile.

Self-hosted Firecracker on Kubernetes ships at Gate 1 (D-33 — accelerated from R3's Gate-2/3-ICP-driven timeline). The managed-microVM era (E2B default, Modal fallback) is retired, not kept as a fallback.

### The decision matrix

| Case | Adapter | Behavior |
|------|---------|----------|
| **Gate 1** | **`SelfHostedFirecrackerAdapter`** via `SandboxPort` (`firecracker-containerd` or Kata as the K8s-integration bridge) | investigation-script execution behind the port; AST pre-scan mandatory; `execution_results` populated |
| Emergency kill path | `NoOpSandboxAdapter` | artifact-only; `execution_results = null`. Only during a kill-path event |
| Gate 2+ read-only containers | gVisor | defense in depth for non-script MCP tool containers **only** — never for LLM-generated script execution |
| Gate 5 physical residency | customer-side DaemonSet + `SandboxPort` remote adapter | optional physical-residency agents |

### Why a microVM, and not just a container

Shared-kernel containers are **not** a security boundary for AI-generated code — the named escape CVEs are **CVE-2024-21626** (runc, "Leaky Vessels") and CVE-2024-0132, and container sandboxes demonstrate escape under adversarial LLM code (SandboxEscapeBench, 2026). gVisor is therefore acceptable only as read-only defense in depth. Now that the deployment target is Kubernetes on EKS (ADR-006 R4) rather than a managed PaaS, Firecracker runs directly, self-hosted, in-boundary.

Self-hosting doesn't mean the boundary itself is CVE-free (SR-07, still applies self-hosted). Two 2026 Firecracker escape-class CVEs are on record: **CVE-2026-5747** (virtio-pci out-of-bounds, opt-in PCI path only; fixed 1.14.4/1.15.1) and **CVE-2026-1386** (jailer symlink handling). Firecracker patch cadence — not just the container-vs-microVM boundary decision above — is an active, ongoing control, not a one-time architectural choice, and self-hosting makes Dux directly responsible for patch currency rather than trusting a vendor's SLA. A newly disclosed VMM-class CVE is therefore a kill-path rehearsal trigger: on disclosure, the on-call AI Safety Lead confirms the in-cluster Firecracker version against the fix, and if patching lags, exercises the emergency `NoOpSandboxAdapter` kill path rather than waiting.

VM reuse is a data-leak vector, so every invocation gets a fresh ephemeral microVM, never reused across runs. This durable-workflow → ephemeral-sandbox shape aligns with the Temporal-community Sandbox Orchestration Harness reference (temporal.io blog, May 2026) — a Code-Exchange community pattern, not an official Temporal product. Dux deliberately tightens beyond that reference to fresh-microVM-per-invocation with no snapshot reuse (SR-08).

The self-hosted operational checklist supersedes the Week-2 vendor checklist (D-33): `firecracker-containerd`/Kata runtimeclass integration on the K8s node pool (EP-01-F02-T11); Firecracker version and patch SLA now an internal ops responsibility, not a vendor-evidenced claim; a subscription to Firecracker's own CVE/security-advisory feed; kill-path fallback verified. No subprocessor code-residency review is needed — investigation code never leaves Dux's own cluster.

### How a request finds its way to the right sandbox

```
LLM-generated investigation script?
  YES → SelfHostedFirecrackerAdapter via SandboxPort only
        (firecracker-containerd/Kata as the K8s-integration bridge)
  NO  → read-only MCP API call?
          YES → gVisor read-only container (defense in depth), or no sandbox
          NO  → optional physical-resident agent on customer infra (Gate 5)
                → customer-side sandbox + eBPF, Month 9
```

## Before a script ever reaches a microVM: the AST scan pipeline

`ScriptSecurityScanner` runs before every execution, so a hostile script never even gets the chance to run inside its sandbox:

1. The agent submits a script.
2. It is parsed — `@babel/parser` for TS/JS, tree-sitter for Python.
3. The AST is walked against `packages/security/script-rules/`.
4. **Pass** → enqueue the microVM. **Fail** → `SCRIPT_BLOCKED` plus an audit record, and no execution.

| Component | Specification |
|-----------|---------------|
| Blocked calls | `exec()`, `eval()`, `Function()`, `child_process`, `subprocess`, `os.system`, `spawn` |
| Blocked imports | `fs` (except a read-only allowlist), `net`, `http`, `https`, `dgram`, `child_process`, `cluster` |
| Network egress | default DROP; allowlist is NVD and GitHub only; no AWS metadata endpoints unless the tenant opts in, with IMDSv2 hop limit = 1 |
| Sandbox limits | read-only filesystem (tmpfs for output); 512 MB / 1 CPU; 60 s wall-clock |
| Audit | SHA-256 script hash, output, `scanner_version`, `ruleset_version` — written to `AUDIT_EVENT`. **Execution is blocked if `ruleset_version` ≠ the deployed scanner** |

| Failure code | Event | Cause |
|--------------|-------|-------|
| `SCRIPT_BLOCKED` | `script.security.blocked` | AST rule violation |
| `SCRIPT_SANDBOX_TIMEOUT` | `script.sandbox.timeout` | 60 s wall-clock exceeded |
| `SCRIPT_SANDBOX_OOM` | `script.sandbox.oom` | 512 MB exceeded |

`pnpm test:script-security` is the CI gate — merge-blocking on any new blocked-pattern bypass.

## The eBPF roadmap: what's compensating for it today

Syscall-level filtering is the deepest layer of execution isolation, and it isn't live yet — the roadmap is explicit about what stands in for it in the meantime:

| Phase | Timeline | Scope |
|-------|----------|-------|
| Gate 2 pilot | Seed Month 2+ | read-only syscall audit on investigation scripts |
| Series A Months 1–8 | interim | self-hosted Firecracker + MCP allowlist as compensating controls; quarterly board disclosure |
| **Series A Month 9** | mandatory | per-agent eBPF syscall filtering for transaction-touching agents (kill switch + purple-team sign-off) |
| Series B | hardening | policy hardening, escape detection, quarterly purple team |

## When a script fails partway through: what the verdict is allowed to say

A script that times out or blows its memory budget isn't a clean pass or a clean fail — it needs its own rules, so an assessment never silently reports success on missing execution:

| Failure | Retry | Verdict effect |
|---------|-------|----------------|
| `SCRIPT_BLOCKED` | none | the blocked script earns no exploitable-verdict credit |
| `SCRIPT_SANDBOX_TIMEOUT` | 1 retry | still failing, with a strong verdict (`exploitable` / `likely`) → HITL T3 |
| `SCRIPT_SANDBOX_OOM` | none | strong verdict → HITL T3 |

An assessment never silently completes as `exploitable` on missing execution.

**Per-tenant sandbox budget:** 300 sandbox-seconds per hour and 5 concurrent microVMs, enforced in the governance kernel (`BudgetGate`). A breach raises `budget_exceeded` → L2. The enforcement metric is `dux_cost_sandbox_seconds_per_tenant`.

## MCP security: the gateway between the agent and every external system

Every read and every write the agent makes to the outside world crosses the Model Context Protocol gateway, which is why it gets its own sixteen-policy framework (PS-001–016). MCP has documented protocol-level holes that neither NIST AI RMF nor ISO 42001 covers: absent capability attestation, bidirectional sampling without origin authentication, and implicit multi-server trust (arXiv:2601.17549). PS-001–016 answers the big three. Cross-walk the policy against the fuller MCP-38 threat taxonomy (arXiv:2603.18063, 38 enumerated threats) at seed.

### The Phase-1 tool catalog

Every tool is registered in the AIBOM (`security/aibom/manifest.json`) and the MCP gateway, with SHA-256 schema pins (PS-006). `tenant_id` is injected from the agent JWT and is **never** accepted as a tool parameter (MCP-005).

**Read-only tools** — the research surface, where the agent goes to gather evidence:

| Tool | `integration_id` | Rate (tenant / session) | Attack story |
|------|------------------|------------------------|--------------|
| `search_nvd(cve_id)` | `nvd` | 200 / 50 rpm | AIBOM-NS-001 — hijack via malicious CVE text |
| `search_github(cve_id)` | `github-research` | 30 / 15 rpm | AIBOM-GH-001 — SSRF via repo URL |
| `search_exploitdb(cve_id)` | `metasploit-index` | 30 / 15 rpm | AIBOM-EDB-001 — poisoned module metadata |
| `search_threat_intel(cve_id)` | `medium-rss` | 20 / 10 rpm | AIBOM-TI-001 — malicious blog redirect |
| `search_msrc(cve_id)` | `microsoft-msrc` | 30 / 15 rpm | AIBOM-MSRC-001 — spoofed advisory content |
| `query_assets(filters)` | `aws` | 100 / 50 rpm (max 50 rows/call; overflow → `summarizeContext`) | AIBOM-QA-001 — filter injection |
| `query_controls(asset_id)` | `aws` | 100 / 50 rpm | AIBOM-QC-001 — cross-asset enumeration |

The gateway enforces `min(tenant_limit, session_remaining)`. All tools are hash-pinned; the hash check runs <2 ms p99 from an in-memory cache, with a full `admin:mcp-schema-diff` on cache miss.

**Write tools** — where a mistake actually touches customer infrastructure, so each one carries its own attack story:

| PS ID | Tool | `integration_id` | Rate (tenant / session) | Attack story |
|-------|------|-------------------|--------------------------|--------------|
| PS-012 | `endpoint.isolate(asset_id, mutation_key)` | `crowdstrike` | 10 / 5 rpm | AIBOM-EDR-001 — spoofed high-confidence exploitability verdict engineered to force a false-positive isolation of a production host |
| PS-013 | `network.blocklist_add(rule)` | `crowdstrike` | 20 / 10 rpm | AIBOM-FW-001 — blocklist-rule injection to self-DoS a legitimate service or IP range |
| PS-014 | `policy.deploy_device_config(device_id, config)` | `intune` (Gate 3, W2) | 20 / 10 rpm | AIBOM-MDM-001 — poisoned config payload delivered through a compromised device-config template |
| PS-015 | `patch.deploy_special_devices(device_id, patch_ref)` | — (no connector pinned) | 10 / 5 rpm | AIBOM-PATCH-001 — crafted patch target forces a firmware downgrade with no API-level rollback |
| PS-016 | `ticket.create_remediation(finding_id, assignee)` | `servicenow` | 60 / 30 rpm | AIBOM-ITSM-001 — prompt-injected ticket content used for social engineering against the assignee |

### Six layers of defense around the gateway

| Layer | Focus | Policy IDs |
|-------|-------|-----------|
| L1 Auth & identity | per-agent short-lived JWT (`agent_id` + `tenant_id` per request) | PS-001, PS-002, PS-005, PS-011 |
| L2 Schema integrity | SHA-256 pins; ECDSA signing (hash-only interim); re-validate before invocation | PS-006, PS-009 (seed) |
| L3 I/O sanitization | JSON Schema; regex DLP <5 ms; SSRF allowlist | PS-003, PS-008 |
| L4 Network egress | gateway-only egress; Host validation; sandbox allowlist | PS-004 |
| L5 Observability | hashed I/O; SIEM Loki `mcp.audit`, 730-day retention | PS-007 |
| L6 Multi-server isolation | per-server security domain; no cross-server shadowing | PS-002, PS-006 |

### The policy statements, PS-001 through PS-016

**PS-001 Registration.** Explicit tenant-admin registration; no implicit discovery. DCR is disabled pre-seed. The gateway exposes `/.well-known/oauth-protected-resource` (RFC 9728). Agents must not invoke unregistered servers.

**PS-002 Tenant-scoped allowlists.** Per-agent allowlist; no cross-tenant credential sharing; changes require admin approval and are audited.

**PS-003 Tool invocation (L3).** Research tools are egress-DLP'd — no tenant asset identifiers in outbound queries. `query_assets` returns ≤50 rows. JSON Schema is strict (`additionalProperties: false`). Output is secret- and PII-scanned at <5 ms p99. The URL allowlist blocks RFC1918, link-local, and metadata addresses. Tool timeout is 30 s. The kill switch is checked before each invocation. Instruction-like output patterns are redacted and raise `mcp.tool_output_injection_suspect`, optionally L1.

**PS-004 Egress (L4).** Gateway-only, allowlisted URLs; no direct outbound path bypassing the gateway. TLS 1.3 preferred, 1.2 minimum. The egress proxy blocks private IPs — verified by `test:mcp-security --case MCP-SSR-001`.

**PS-005 Credentials (L1).** Per-agent session JWT. The token's `aud` / `resource` must match `server_id`; cross-server reuse is rejected with `mcp.token_audience_mismatch`. Credentials live in SSM or Vault via `SecretsPort` — never in a plaintext database column — scoped to tenant plus connection, rotated every 90 days, and never present in prompts, logs, or telemetry.

Credential isolation is row-scoped by design, not path-scoped: all tenants' connector credentials sit behind one shared Vault KEK rather than per-tenant Vault paths, a deliberate cost/ops tradeoff at this stage — tenant separation is enforced by the `tenant_id`-scoped row and `SecretsPort` access control, not by per-tenant key material.

**PS-006 Supply chain (L2).** ASI04 checklist before allowlisting. SHA-256 pin over canonical JSON (`name` + `description` + `input_schema`). `admin:mcp-tool-lint` blocks instruction-like descriptions. `admin:mcp-scan` (Invariant / Snyk mcp-scan) runs on every registration change to detect tool poisoning and rug-pulls, and is merge-blocking. The hash is re-validated before every invocation; on drift the tool is auto-disabled and re-consent is required. Pins rotate quarterly. Per-tool circuit breaker: 5 failures in 60 s.

**PS-007 Audit (L5).** Log `agent_id`, `tenant_id`, `tool_name`, `server_id`, `session_id`, `request_id`, timestamp, outcome, and `latency_ms`. Store `input_hash` and `output_hash`. Correlate through the `InstrumentedLLMClient` span. SIEM: Loki `mcp.audit`, 730-day retention. Alert above 50 denials/hour/tenant; L1 after 5 consecutive denials in a session.

**PS-008 PII (opt-in overlay).** PII in tool output requires tenant-admin opt-in (`tenant_settings.pii_opt_in_at`) before it may enter LLM context. Otherwise it is redacted per PS-003.

**PS-009 Message signing (seed).** ECDSA P-256 over request and response, with a 128-bit `nonce` and `iat`. Reject duplicates and anything older than 5 minutes. Signing is mutual; keys live in Vault with 90-day rotation. Pre-seed interim: PS-006 hash-only satisfies exit.

**PS-010 OS sandboxing (seed).** Pre-seed container hardening — read-only root filesystem, non-root user. Seccomp profiles land at Gate 2.

**PS-011 Remote OAuth (L1).** OAuth 2.1 with PKCE (S256). Token endpoint, redirect allowlist, and `resource` parameter (RFC 8707) are documented at registration.

**PS-012 `endpoint.isolate` (GOV-TOOL-01).** Mandatory HITL T3 on every call regardless of confidence (D-17) — no unattended path exists yet. `VendorActionGate` will not authorize execution without rollback R-01 (`endpoint.restore_network`, keyed by `mutation_key`) on file. Promotes to unattended execution only on a field-proven Gate-3 safety record for the action class.

**PS-013 `network.blocklist_add` (GOV-TOOL-02).** Unattended by default at Gate 1; HITL T2 fires only below the 0.75 confidence floor. Rollback R-02 (`network.blocklist_remove`) reverts the specific vendor-native rule ID only — never a broader flush.

**PS-014 `policy.deploy_device_config` (GOV-TOOL-03).** Unattended by default once the `intune` connector ships (Gate 3, W2); HITL T2 below the 0.75 confidence floor. Rollback R-03 (`policy.restore_previous_config`) redeploys the pre-deploy config snapshot.

**PS-015 `patch.deploy_special_devices` (GOV-TOOL-04).** Mandatory HITL T3 on every call — firmware-only devices have no API-level rollback, so `GOV-TOOL-04` holds this action to mandatory HITL rather than letting it execute without an undo path. Rollback R-04 (`patch.rollback_to_prior_version`) applies only where the vendor patch-management API supports it; otherwise the procedure is a manual runbook.

**PS-016 `ticket.create_remediation` (GOV-TOOL-05).** Unattended at Gate 1 — the lowest blast tier (T1), no confidence floor gate. Rollback R-05 (`ticket.cancel`, reason `superseded_by_rollback`) fires on HITL rejection or duplicate-ticket detection.

### The gateway's own circuit breaker

| Property | Value |
|----------|-------|
| Trip condition | >50% tool errors in 5 min, **or** p99 latency >2 s |
| Cooldown | 60 s (gateway returns 503) |
| Automatic close | 5 min of stable metrics |

Operator escalation: a per-tool 503 (PS-006) → retry after 60 s. An aggregate 503 → check upstream MCP health. Sustained failure → L2 kill switch.

### How all of this is enforced, not just documented

| Control | Mechanism |
|---------|-----------|
| CI on MCP PRs | `security-review` label + allowlist + risk classification |
| Hardcoded MCP URLs | blocked by ESLint |
| Secrets | Gitleaks |
| Integration tests | MCP-001–005 — unregistered server denied; schema drift blocked; PII redacted; egress allowlist enforced; cross-tenant JWT rejected |
| Compliance evidence | `security/owasp/agentic-assessment-*.json`; gateway audit logs retained 2 years |

### Server inventory

| Server | Credential |
|--------|-----------|
| `mcp-nvd-feed` | session JWT + NVD key in Vault |
| `mcp-aws-connector` | session JWT + IAM assume-role |

### What's out of scope entirely, for now

| Class | Phase-1 status |
|-------|----------------|
| Read-only | shipped — catalog above |
| Write | shipped — the five ADR-012 R3 canonical actions only; 3 unattended by default with HITL on anomaly escalation, 2 (`endpoint.isolate`, `patch.deploy_special_devices`) mandatory HITL on every call (D-17). Catalog: PS-012–016 |
| External | prohibited in Phase 1 |
| Code | prohibited in Phase 1 |
| Financial | prohibited in Phase 1 |

The OWASP MCP Top 10 crosswalk lives in owasp-assessments.

## Sources

- `.raw/dux/40-ai-safety/safety-overview.md`
- `.raw/dux/40-ai-safety/governance-kernel.md`
- `.raw/dux/40-ai-safety/kill-switch-hitl.md`
- `.raw/dux/40-ai-safety/camel-plane.md`
- `.raw/dux/40-ai-safety/sandbox-execution.md`
- `.raw/dux/40-ai-safety/mcp-security.md`
