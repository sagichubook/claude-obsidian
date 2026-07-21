---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-4, D-9, D-10, D-17, D-34, H4, H5]
---

# Kill Switch & Human-in-the-Loop

**Purpose:** the halt authority for every agent surface, and the contract for the human-review path. **Parents:** BR-003 · pre-seed security exit requirement.

**Nomenclature:** KS-L1–KS-L4 are kill-switch levels. They are distinct from CaMeL layers (CDA-L1–L8), context tiers (CTX-T1–3), and Series-B support tiers.

## 1. Kill-switch levels

| Level | Scope | Trigger authority | Use case |
|-------|-------|-------------------|----------|
| KS-L1 Session | a single assessment workflow | session owner, system | runaway single assessment |
| KS-L2 Tenant assessment | all assessment sessions plus the queue, frozen for one tenant (dashboard unaffected) | tenant admin, platform admin | compromised agent; tenant assessment incident |
| KS-L3 Tenant platform | all tenant-facing agent features; dashboard goes read-only | tenant admin, platform admin | tenant-wide security or billing incident |
| KS-L4 Global | the entire platform agent fleet | platform admin + on-call (two-person) | critical vulnerability, LLM outage, cross-tenant leak |

## 2. Propagation SLI

| Level | Path | Target | Degradation |
|-------|------|--------|-------------|
| L1 | Unleash flag + session terminate | ≤30 s (`unleash_evaluation_duration_ms` p99) | manual `admin:agent-halt` |
| L2 | `KillSwitchRelay` → NATS core pub/sub → workers poll `kill_switch_flags` | p99 <5 s (**KS-001**) | CloudNativePG `LISTEN/NOTIFY` (KS-007, <10 s) |
| L3 | as L2, plus dashboard read-only middleware | p99 <5 s | page `@platform-oncall` |
| L4 | as L2, plus global API assessment halt | p99 <5 s | two-person approval required |

**Two SLOs, never conflated:**

- **Propagation** (KS-001) covers KS-L2–L4 relay only, at p99 <5 s. KS-L1 is a separate Unleash path at ≤30 s.
- **Execution interruption** is p99 <30 s. Long activities check the flag on heartbeat ticks (every 10–30 s), so worst-case in-flight bleed after an L3 or L4 is one heartbeat interval — roughly 30 s, **not** `startToCloseTimeout`.

**Unleash Edge dependency check (resolves part of SR-17).** KS-L1 evaluates flags via the standard self-hosted Unleash SDK/server ([architecture-overview §6](../20-architecture/architecture-overview.md)), **not** Unleash Edge (the proxy/cache layer, EOL 2026-12-31). The kill switch has no dependency on Edge and is unaffected by its retirement. If Edge is adopted later for read-scaling, re-verify this before KS-L1 comes to depend on it.

## 3. Architecture

`KillSwitchRelay.checkBeforeOperation(tenantId, sessionId)` runs before every LLM call and every MCP tool invocation, checking the L1–L4 keys in parallel.

| Component | Detail |
|-----------|--------|
| NATS key/subject | `kill.{level}.{target_id}` — L4 is `kill.L4.global` |
| Pub/sub channel | `kill.switch.channel` (NATS core, self-hosted on K8s) |
| Fallback | CloudNativePG `kill_switch_flags` mirror, polled every 2 s when NATS is unavailable |
| Worker circuit breaker (fail-closed) | 2 consecutive missed NATS polls → mark the session `killed` in CloudNativePG, cancel in-flight steps, **then** SIGTERM — never SIGTERM before cancellation |

KS-007 timeline: Week-2 spike → Gate-1 release gate (`pnpm ops:test-kill-switch --fallback pg-notify --level L3 --iterations 50`) → mandatory cross-cutting at Gate 2.

## 4. Triggers

**Manual.** Admin UI (reason ≥10 characters), `POST /v1/admin/kill-switch`, or CLI: `pnpm admin:kill-switch --level L3 --tenant <uuid> --reason "…"`.

**Automatic.**

| Condition | Level |
|-----------|-------|
| 5 MCP denials in a session | L1 |
| >10 K tokens/min | L1 |
| >60 tool calls/min | L1 |
| Stripe failure >3 days / >7 days | L2 / L3 |
| >50% tool errors in 5 min, per tenant | L2 |
| Cross-tenant leak | L4 (two-person, `#security`) |
| HITL T4 escalation timeout, 30 min | L2 |

Every automatic trigger writes a `trigger=automatic` audit record and alerts `#security`. **An automatic trigger cannot self-clear** — clearing requires a platform admin plus false-positive evidence.

## 5. Behavior and authority

| Level | Behavior | Authority to trigger |
|-------|----------|---------------------|
| L1 | terminates the target session (cancel, no retry); new sessions still allowed | session owner or system |
| L2 | freezes the tenant's assessment sessions and queue; dashboard otherwise normal | tenant or platform admin |
| L3 | terminates all agent sessions; dashboard read-only | tenant or platform admin (MFA for platform admin) |
| L4 | terminates everything and blocks all | platform admin + a second admin's `#security` acknowledgement, persisted to `kill_switch_approvals` |

Activation is idempotent — a repeat call returns 200 with `{status: already_active, kill_id, activated_at, propagation_ms}`. Deactivation requires equal-or-higher authority; L3 and L4 additionally require a post-incident ticket, and L4 carries a 15-minute minimum cooldown. Target false-positive rate: **<1% of sessions**.

## 6. Business impact (PM sign-off)

MRR-at-risk is computed per level as `monthly_MRR × fraction × hours / 730`. Status-page, banner, and email copy live in [customer-lifecycle](../60-operations/customer-lifecycle.md).

| Role | Responsibility |
|------|----------------|
| Founder or PM | Phase-1 approver; async Slack, ≤15 min SLA |
| `@product-oncall` | approver post-Gate-2 |
| Founder + PM | both required before any external communication on an L4 |
| AI Safety Lead (`@ai-safety-oncall`) | mandatory halt authority within 60 s of a T3 escalation or an L4 suspicion |

## 7. HITL contract

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

Every `rollbackProcedure` URL above resolves to one of the five entries in the [mitigation-write-path rollback catalog](../10-product/features/mitigation-write-path.md) (R-01…R-05) — the audit-side rollback record backs every action; for the two mandatory-HITL actions it also renders in the pre-approval impact preview, not only post-hoc.

### 7a. Investigation-confidence HITL gate (D-34)

**A different axis from §7 above — this gates the Agentic RAG investigation output itself (a confidence score on the synthesized exploitability finding), not a vendor write action.** The two gates can compose: a low-confidence investigation can still route to an unattended-by-default write once it clears this gate, subject to §7's own rules.

[ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval) specifies the mechanism: a confidence band of 0.85–0.95 on the `agenticRAGWorkflow` synthesis step routes to a signal-based human-approval gate, 2-hour timeout, escalating to on-call — the same `WorkflowPort` HITL-signal primitive as every gate in this file, not a new transport, and **the same API surface as §7: `GET /chat/sessions/{id}/stream` (`hitl_request`) / `POST /chat/sessions/{id}/hitl-response`** ([application-api.md §2](../30-api/application-api.md#2-key-dtos), D-56) — not a dedicated endpoint. The v4.0 source doc frames the same escalation ladder in DUX-score-band terms; both describe the identical mechanism from different vantage points (workflow confidence vs. output score):

| Confidence / DUX-score band | Required approval | Timeout action |
|---|---|---|
| High confidence (≥0.95) / DUX score 9.0–10.0 | Synthesis proceeds unattended; if a downstream write is triggered, CISO or Security Lead per that action's own tier | — |
| Gate band (0.85–0.95) / DUX score 7.0–8.9 with confidence <0.95 | Manager-level approval on the investigation itself, via the signal-based gate above | escalate to on-call after 2 h (ADR-020 R2) |
| Below gate band (<0.85) | Loop continues (max 5 iterations) rather than escalating — insufficient confidence to present for approval at all | — |
| Bulk remediation >100 assets (downstream write, not the investigation gate) | Change Advisory Board, per §7's action-tier rules | decompose to smaller batches, not a timeout |
| First autonomous mitigation in a tenant (downstream write) | Security Team, per §7 | hold for validation |

**Network segmentation and IAM-policy-modification writes** (Network Security Team / Identity Team Lead, hold indefinitely) are not new investigation-gate categories — they route through §7's existing tier/action framework once a write is proposed; this table only extends the *investigation-confidence* axis that precedes any write decision.

## 8. Testing

| Test | Assertion | Cadence |
|------|-----------|---------|
| KS-001 | L2–L4 propagation <5 s p99 | every agent-code PR |
| KS-004 | L4 <10 s p99 | monthly staging smoke, release-blocking |
| KS-005 | Valkey degradation → <10 s DB polling | quarterly |
| L3 staging drill | zero new tool calls within 5 s | quarterly |
| L4 chaos | all workers halt | annually |
| `pnpm test:kill-switch-idempotency` | no step resumes after L2/L3/L4; in-flight tool calls return `GOVERNANCE_BLOCKED` | every agent-code PR |

NIST AI RMF GOVERN: the policy owner is Founder + Security, reviewed quarterly.
