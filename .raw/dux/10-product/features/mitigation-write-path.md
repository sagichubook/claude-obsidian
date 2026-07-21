---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-4, D-10, D-15, D-17, H4, H5]
---

# Feature — Mitigation & Remediation Write Path (US-004, US-016, US-018, US-019)

**Purpose:** the surfaces that take action in the customer's environment — fast actions, action cards, remediation tickets, and closed-loop validation.

**Nav:** Fast Actions + in-context CTAs · **Epic:** EP-06 · **BRs:** BR-002, BR-003.

**The write path ships at Gate 1 with earned per-action-class autonomy (D-17).** `network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation` execute unattended, human review reserved for anomaly escalation. `endpoint.isolate` and `patch.deploy_special_devices` require mandatory HITL on every call until each earns unattended execution via a field-proven Gate-3 safety record. Closed-loop validation (US-019) is **Gate 3**. Governed by [ADR-012 R3](../../20-architecture/adr-index.md) and the [governance kernel](../../40-ai-safety/governance-kernel.md); **every write flows through `VendorActionGate`**.

## US-016 Fast Actions

**Gate 1.** Unattended by default for `network.blocklist_add`/`policy.deploy_device_config`/`ticket.create_remediation`; mandatory HITL for `endpoint.isolate`/`patch.deploy_special_devices`.

**Job.** One-click lightweight mitigations on approved findings.

**Delivery.** `POST /fast-actions` → `QuickMitigationWorkflow`. The three earned-autonomy actions execute immediately, audit-logged and kill-switch-covered. `endpoint.isolate`/`patch.deploy_special_devices` raise a live HITL request and wait. Action cards — copy a SIEM query, ticket text, numbered steps — remain available as a manual fallback.

**Read contract.** The nav list is a filtered view of the existing US-004 mechanism: `?projection=action_cards&eligible_for=fast_action`, not a dedicated endpoint. A row carries `canonical_action_id`, the target `cve_id`/`finding_id`, `blast_radius`, `hitl_tier`, and `status` (`pending` | `executed` | `blocked`) — the same fields `VendorActionExecution` and the HITL tier table already define below, not new entities.

**Nav state.** The icon is visible and enabled at Gate 1. The automated route is the default for the three earned-autonomy actions.

**Safety.** The governance kernel and the kill switch gate execution (KS-L2 blocks enqueue). HITL fires **only** on an anomaly for the three earned-autonomy actions: a confidence abstention, a sandbox failure, or a T4 outlier. `endpoint.isolate`/`patch.deploy_special_devices` always wait on HITL.

## US-004 Action Cards

**Gate 1.** Unattended by default for `network.blocklist_add`/`policy.deploy_device_config`; mandatory HITL for `endpoint.isolate`/`patch.deploy_special_devices`. This file is the canonical spec for US-004; the journey summary lives in [security-stepper Step 4](security-stepper.md).

**Job.** Surface and execute lightweight mitigations — blocklist at Gate 1, Intune policy steps once the W2 connector lands — faster than a full patch, with a residual-risk count.

**Orchestration.** The mitigation-recommendation activity, or `QuickMitigationWorkflow`. MCP write tools execute through the governance kernel. HITL tiers T2/T3 classify each action for audit and escalation on the three earned-autonomy actions; T3 is a mandatory pre-approval gate on `endpoint.isolate`. Agent: AI #4.

**API.** `?projection=action_cards`. `POST /mitigations` executes through `VendorActionGate` — unattended at Gate 1 for the three earned-autonomy actions, mandatory HITL for `endpoint.isolate`/`patch.deploy_special_devices`. Webhooks: `mitigation.executed`, `mitigation.blocked`.

**Safety.** Write tools require governance-kernel gates plus the kill switch. HITL escalates only on an anomaly for the three earned-autonomy actions; it is a mandatory gate, not an escalation, for `endpoint.isolate`/`patch.deploy_special_devices`. KS-L2 blocks new proposals.

**Canonical mitigation kinds** ([catalog §6](../catalogs.md)):

| Action | Tier | Posture |
|--------|------|---------|
| `endpoint.isolate` | T3 | mandatory HITL (D-17) |
| `network.blocklist_add` | T2 | unattended by default |
| `policy.deploy_device_config` | T2 | unattended by default (once Intune connector ships) |
| `patch.deploy_special_devices` | T3 | mandatory HITL (no API rollback + D-17) |

## US-018 Remediation Ticket Panel

**Gate 1** for create and route, unattended by default.

**Job.** A security engineer sees a ServiceNow/ITSM ticket created from Exposure or Chat, with an assignee and an SLA.

**Orchestration.** `RemediationWorkflow` creates and routes tickets automatically — T1, the lowest blast radius. Status updates arrive by webhook. Canonical action: `ticket.create_remediation`. **Unattended closed-loop auto-close remains Gate 3** (US-019).

**API.** Gate-1 create and route endpoints. Webhooks: `remediation.ticket_created`, and `ticket.created` / `updated` / `resolved` / `reopened`. Data: `REMEDIATION_TICKET`.

**Safety.** A ticket write failure enters a retry saga and is audited. External writes use step-effect idempotency — a `mutation_key` plus resume-time reconciliation. KS-L2 prevents new ticket creation.

## US-019 Mitigation Validation Panel

**Gate 3** · status: draft.

**Job.** Confirm that post-mitigation exposure actually dropped, by re-assessing — the closed loop.

**Orchestration.** `ClosedLoopValidationWorkflow` (FR-012) triggers the re-assessment. A pass/fail badge lands on the US-004 card, and the residual count updates US-011. Flag: `closed_loop_validation`.

**Safety.** A failed validation escalates to HITL. **A finding is never auto-closed on a timeout alone.**

## Write-path mechanics (every write action)

1. **Governance kernel chain:** `IntentGate → BudgetGate → EffectGate → VendorActionGate → HITLGate`. `HITLGate` defaults to `continue` — execute and audit, returning `escalate` only on an anomaly — for `network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`. For `endpoint.isolate`/`patch.deploy_special_devices` it defaults to `escalate` on every call (D-17).
2. **`VendorActionGate`** maps `canonical_action_id → native_action_name` and persists both in `VendorActionExecution` audit records. **Connectors must not call vendor mutation APIs.**
3. **HITL tiers (T1–T3)** classify every action for audit and escalation. The SSE `hitl_request` / POST `hitl_response` contract fires only on escalation for the three earned-autonomy actions, and on every call for `endpoint.isolate`/`patch.deploy_special_devices` — the **`rollbackProcedure` URL is required in the payload regardless of HITL posture**.
4. **Post-action refresh:** persist the execution → targeted connector delta sync (`sync_reason=post_mitigation`) → hand off to `ClosedLoopValidationWorkflow` (Gate 3), so downstream screens read fresh World Model state.

## Marketing reconciliation

"Lightweight mitigations" and "rapid remediation" are claim-safe at Gate 1 without a caveat, because the write path executes at Gate 1 — see [gtm-guardrails](../../80-gtm/gtm-guardrails.md).

Gate 3 refers specifically to closed-loop validation (US-019). A "self-healing" or "fully automated remediation" claim therefore still needs that qualifier: at Gate 1 Dux acts; **confirming the action worked is Gate 3.**

## Rollback catalog

Every `rollbackProcedure` URL on a write action's audit/HITL payload resolves to one of the five entries below. `VendorActionGate` (GOV-014, [governance-kernel §4](../../40-ai-safety/governance-kernel.md)) will not authorize unattended execution of an action whose entry is missing.

| ID | Action | Compensating procedure | Trigger |
|----|--------|--------------------------|---------|
| R-01 | `endpoint.isolate` | `endpoint.restore_network` through the same EDR adapter — restores the pre-isolation network policy, keyed by the isolation's `mutation_key` for idempotent reversal | HITL rejection, a T3 escalation resolved "false positive", or a customer-initiated rollback from Tenant Settings |
| R-02 | `network.blocklist_add` | `network.blocklist_remove` — reverts the specific rule by the vendor-native rule ID persisted in the audit record; never a broader "flush blocklist" | same |
| R-03 | `policy.deploy_device_config` | `policy.restore_previous_config` — redeploys the device's prior config snapshot, captured immediately before `policy.deploy_device_config` ran | same |
| R-04 | `patch.deploy_special_devices` | `patch.rollback_to_prior_version` where the vendor patch-management API supports it. **Firmware-only devices have no API-level rollback** — the procedure there is a manual runbook, and `GOV-TOOL-04` holds that case to a mandatory HITL gate rather than letting it execute unattended without an undo path | same, or a post-patch device-health check failure |
| R-05 | `ticket.create_remediation` | `ticket.cancel` — closes the ticket with reason `superseded_by_rollback`; no environment state to revert | HITL rejection or duplicate-ticket detection |

## Gate-3 remediation orchestration

Evaluation framework (candidate, criteria C1–C5, and fallback order) defined in [ADR-012 §Gate-3 remediation orchestration](../../20-architecture/adr-index.md#adr-012-r3--vendor-action-framework) — resolves OI-29. The spike against C1–C5 is Gate-3-scoped backlog work under EP-06, not yet run.
