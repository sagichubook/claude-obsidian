---
owner: Founder
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: []
---

# Quick Reference Card

A pure compilation of facts already documented elsewhere in this corpus — no new claims, no new numbers. If a figure here ever disagrees with its source doc, the source doc wins; fix this page to match.

## Write-action autonomy (5 canonical actions)

| Action | HITL posture (D-17) | Rollback |
|---|---|---|
| `endpoint.isolate` | mandatory HITL — every call | R-01 |
| `network.blocklist_add` | unattended by default, anomaly-only escalation | R-02 |
| `policy.deploy_device_config` | unattended by default, anomaly-only escalation | R-03 |
| `patch.deploy_special_devices` | mandatory HITL — every call | R-04 |
| `ticket.create_remediation` | unattended by default, anomaly-only escalation | R-05 |

See [mitigation-write-path](../10-product/features/mitigation-write-path.md), [governance-kernel §Intro](../40-ai-safety/governance-kernel.md), [decisions-log D-17](decisions-log.md).

## Kill-switch levels

| Level | Scope | Propagation target |
|---|---|---|
| KS-L1 Session | single assessment workflow | ≤30 s |
| KS-L2 Tenant assessment | all sessions + queue, one tenant | p99 <5 s (KS-001) |
| KS-L3 Tenant platform | all tenant agent features, dashboard read-only | p99 <5 s |
| KS-L4 Global | entire platform agent fleet | p99 <5 s, two-person approval |

Execution-interruption SLO: p99 <30 s (one heartbeat interval). See [kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md).

## Governance gates (GOV-001–014)

Chain: `IntentGate → BudgetGate → EffectGate → VendorActionGate → HITLGate`

| Gate | Threshold |
|---|---|
| GOV-003 ActionBudget | warn >100, halt ≥200 weighted actions/assessment (LLM=1, MCP-read=2, MCP-write=5) |
| GOV-007 CostCap | $25/hour per-tenant hard cap |
| GOV-010 Loop | max 50 P-LLM iterations/assessment, max 10 MCP calls/iteration |
| GOV-014 VendorActionGate | write action blocked unless confidence floor met AND a rollback procedure is on file |

Cost-threshold evaluation order (first match wins): $0.675/assessment soft breaker → $25/hour CostCap → 2× rolling-baseline circuit breaker. See [governance-kernel](../40-ai-safety/governance-kernel.md).

## Confidence bands

| Band | Label |
|---|---|
| [0.85, 1.00] | `exploitable` |
| [0.70, 0.85) | `likely` |
| [0.40, 0.70) | `unlikely` |
| [0.00, 0.40) | `not_exploitable` |
| abstain | `insufficient_data` |

Scoring mechanism: 3-signal ensemble (logprob 0.40 / semantic entropy 0.35 / verbalized confidence 0.25) → Platt scaling. This is confidence-in-exploitability, not a composite CVSS×EPSS×criticality×exposure risk score. See [taxonomy §Confidence scoring methodology](../10-product/taxonomy.md), [confidence-calibration](../40-ai-safety/confidence-calibration.md).

## Compliance control-ID families in use

| Framework | Where mapped |
|---|---|
| SOC 2 (AICPA TSC: CC6/7/8, A1, Confidentiality, PI, Privacy) | [compliance-program §2](../70-governance/compliance-program.md) |
| ISO/IEC 27001:2022 (internal IDs + Annex A numbers) | [compliance-program §3](../70-governance/compliance-program.md) |
| ISO 42001 / EU AI Act Art. 9 | [compliance-program §3](../70-governance/compliance-program.md) |
| NIST AI RMF + NIST CSF 2.0 | [compliance-program §8](../70-governance/compliance-program.md) |
| CIS Controls v8 | [compliance-program §8](../70-governance/compliance-program.md) |
| OWASP Agentic (ASI01–10) / LLM (LLM01–10) / MCP Top 10 / NHI Top 10 | [owasp-assessments](../40-ai-safety/owasp-assessments.md), [compliance-program §5](../70-governance/compliance-program.md) |

## Gate-1 exit criteria (selected)

- Zero cross-tenant fuzz reads (`pnpm test:fuzz-tenant-id` merge-blocking).
- Zero Critical findings in the adversarial suite.
- RLS `FORCE` on every tenant-scoped table, CI-verified (`check-rls.sh`).

See [product-overview](../10-product/product-overview.md).
