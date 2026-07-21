---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-3, D-4, D-14, D-17, H5]
---

# Governance Kernel (GOV-001–013)

**Purpose:** the synchronous, fail-closed gates evaluated before every privileged agent action. **Parents:** BR-003.

There is no bypass path in Phase 1. Gates run as a Chain of Responsibility, each handler returning `continue`, `block`, or `escalate`:

```
IntentGate → BudgetGate → EffectGate → VendorActionGate → HITLGate
```

`HITLGate`'s default outcome is set per action by its `GOV-TOOL-*` row (§4), not uniformly. For the three earned-autonomy actions (`ticket.create_remediation`, `network.blocklist_add`, `policy.deploy_device_config`) it defaults to `continue` — it logs the tier, writes the audit record, and executes, returning `escalate` only on an anomaly: a confidence-abstention band, a sandbox `TIMEOUT` or `OOM`, or a T4 outlier. For the two fleet-impacting actions still earning autonomy (`endpoint.isolate`, `patch.deploy_special_devices`) it defaults to `escalate` — every call waits on the tier's approve/deny surface regardless of confidence, until that action class is promoted (D-17). See [kill-switch-hitl](kill-switch-hitl.md).

## 1. Gates

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
| GOV-014 | VendorActionGate | the canonical write action is checked against its `GOV-TOOL-*` row (§4) — the action's evidence-confidence floor is met, and a rollback procedure is on file | below the confidence floor: `VENDOR_ACTION_BLOCKED` → HITL escalation at the tool's tier |

**Kill-switch short-circuit.** The kernel consults `KillSwitchRelay` *before* evaluating any gate. An active L3-or-higher switch returns 503 without running the chain, and emits a `governance.kill_switch_short_circuit` audit record carrying `tenant_id`, `session_id`, `level`, and the gates skipped.

## 2. Cost threshold evaluation order

First match wins. Thresholds are derived from the R2 envelope (D-3):

1. Per-assessment soft circuit breaker at **$0.675 / assessment** (ADR-008).
2. Hourly **CostCap $25/hour** (GOV-007).
3. Workflow circuit breaker at **2× baseline** (GOV-005).

Token-runaway runbooks cite this order, not CostCap alone. `CostCapGate` (`packages/core/governance/`) queries hourly spend per tenant; on breach it calls `KillSwitchRelay.activate(L2, tenantId, 'budget_exceeded')`.

## 3. Latency budget and metrics

| Period | DLP engine | Budget |
|--------|-----------|--------|
| Weeks 1–10 | regex | sequential p99 <75 ms total; DLP <5 ms; every other gate <15 ms |
| Week 11+ | Presidio NER | DLP <60 ms p99; parallelize DLP with Intent where safe |

Prometheus exports `governance_gate_denials_total{gate,reason}` and `governance_gate_duration_seconds`.

## 4. Verification

`pnpm test:governance-kernel` exercises the GOV-001–013 failure modes. It is merge-blocking before Gate 1.

Each MCP tool is also mapped to a tool-use risk matrix (NIST Agentic MAP): consequence scope, reversibility, required HITL tier, and ASI02/ASI04 linkage — the `GOV-TOOL-*` rows below.

| ID | Canonical action | Consequence scope | Reversibility | Unattended confidence floor | Below floor | ASI linkage |
|----|-------------------|--------------------|----------------|------------------------------|--------------|--------------|
| GOV-TOOL-01 | `endpoint.isolate` | single asset (T3) | reversible — [rollback R-01](../10-product/features/mitigation-write-path.md) | none — mandatory HITL T3 regardless of confidence, until this action class is promoted (D-17) | HITL T3 (mandatory) | ASI02, ASI04 |
| GOV-TOOL-02 | `network.blocklist_add` | subnet (T2) | reversible — R-02 | ≥0.75 | HITL T2 | ASI02, ASI04 |
| GOV-TOOL-03 | `policy.deploy_device_config` | subnet (T2) | reversible — R-03 | ≥0.75 | HITL T2 | ASI02, ASI04 |
| GOV-TOOL-04 | `patch.deploy_special_devices` | single asset (T3) | conditional — R-04; firmware-only devices have no API-level rollback | none — mandatory HITL T3 regardless of confidence | HITL T3 (mandatory) | ASI02, ASI04, ASI09 |
| GOV-TOOL-05 | `ticket.create_remediation` | tenant metadata only (T1) | reversible — R-05 | ≥0.60 | n/a — T1 always executes unattended | ASI02 |

**`VendorActionGate` (GOV-014)** evaluates a write call against its `GOV-TOOL-*` row before the vendor adapter runs: below the confidence floor, on `GOV-TOOL-04` with no API-level rollback available, or on any mandatory-HITL row (`GOV-TOOL-01`, `GOV-TOOL-04`), it returns `escalate` at the row's HITL tier instead of `continue`. The confidence floor is the assessment's calibrated exploitability confidence (see [confidence-calibration](confidence-calibration.md)), not a raw model logit.

**Promotion to unattended execution (D-17).** `GOV-TOOL-01` (`endpoint.isolate`) earns the same earned-autonomy path already defined for `GOV-TOOL-04`-class actions: it moves off mandatory HITL only once the Gate-3 `ClosedLoopValidationWorkflow` has a field-proven safety record for that action class (a minimum observed sample of HITL-approved executions with zero unrecovered false-positive isolations). Until that record exists, `endpoint.isolate` and `patch.deploy_special_devices` are the only two canonical write actions that do not execute unattended at Gate 1.

**Write-action safety invariant.** No future `GOV-TOOL-*` row may allowlist a write action that disables or weakens MFA, logging, encryption, or audit — on the target system, on Dux's own control plane, or on the audit trail of the action itself. This is a standing constraint on the `GOV-TOOL-*` allowlist, not a per-action judgment call: `VendorActionGate` (GOV-014) is the same gate that already blocks unattended execution of any action whose rollback entry is missing (§5), and it blocks on this invariant the same way — a proposed action that trips it never reaches `continue`, regardless of confidence or blast tier. `pnpm test:governance-kernel` (§4) is merge-blocking on this invariant exactly as it is on GOV-001–014's other failure modes: a new `GOV-TOOL-*` row cannot land without a passing case proving it does not touch MFA, logging, encryption, or audit posture.

## 5. Rollback dependency

Every `GOV-TOOL-*` row names a rollback ID (R-01…R-05); the compensating procedure for each lives in [mitigation-write-path §Rollback catalog](../10-product/features/mitigation-write-path.md) — `VendorActionGate` will not authorize unattended execution of an action whose rollback entry is missing.
