---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-7, D-8, D-34]
---

# EP-03 — Exploitability Assessment Engine

**Objective:** BR-002 (core product) + BR-007 AI-BOM accuracy (KPIs: MTXV <15 min/CVE; golden-set ≥80% by Gate 1, <2% regression; cost ≤$0.75/assessment) · **Metrics:** golden set + DeepEval CI; TR-NFR-005 start p95 <2 s; cost gates D-3 · **Target:** Gate 1 (Week 12, [D-7 R1](../00-meta/decisions-log.md)); vertical slice Week 6 · **Priority:** P0 · **Constraints:** ADR-007 R3 (self-hosted Temporal, orchestrator-worker, child-workflow-per-tenant), ADR-008 R2 (CaMeL routing, domicile rules), ADR-015 R4 (self-hosted Firecracker microVM + AST pre-scan), governance-kernel gates before every tool dispatch.

## EP-03-F01 — Assessment workflow & CaMeL plane

**Parent:** EP-03 · **FRs:** FR-004 · **ACs:** [security-stepper US-001](../10-product/features/security-stepper.md); golden-set floors 65/75/80%; citation-first (EXP-CIT-001) · **Dependencies:** EP-01 (RLS), EP-02 (evidence), self-hosted Temporal namespace (EP-01-F02-T02f) · **API boundary:** `POST /research/queue` (idempotent), `ExploitabilityAssessmentWorkflow` · **SLA:** start p95 <2 s; actions/assessment p95 ≤80, halt ≥200.

### US-001 — Prerequisites Analysis (Gate 1) · 13 pts · persona: security engineer
**ACs:** prerequisite extraction w/ cited sources; `INSUFFICIENT_DATA` degraded path; factor cards from catalog. **DoD:** merge gates + golden set + DeepEval. **Spikes:** Week-4 DBOS chat spike (orchestration path), llguidance Week 4 (AI-77).

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-001-T01 | Self-hosted Temporal namespace setup: namespace-per-env, tenant task queues, cert-manager mTLS/data-converter config (D-2) | DevOps | Config | 16 | 1–2 | TS-2 |
| US-001-T02 | `ExploitabilityAssessmentWorkflow` skeleton (child-per-tenant, saga, continue-as-new, heartbeats) | Backend | Code | 32 | 2–4 | TS-1 |
| US-001-T03 | CaMeL split: S-LLM extractor (public CVE text) + P-LLM reasoner (customer context) + schema boundary | Backend | Code | 32 | 3–5 | PY-1 |
| US-001-T04 | `prerequisite-extractor` subagent + factor-card mapping + citation enforcement | Backend | Code | 20 | 4–6 | PY-1 |
| US-001-T05 | llguidance structured-output spike (fallback Zod+retry, AI-77) | Backend | Research | 8 | 4 | PY-2 |
| US-001-T06 | `POST /research/queue` + `AssessmentDeduplicationService` (idempotency) | Backend | Code | 12 | 4–5 | TS-1 |
| US-001-T07 | Golden set v1 (250 CVEs) + DeepEval harness + stratified merge gate | QA | Test | 32 | 3–6 | PY-2 |
| US-001-T08 | Prompt-injection regression corpus (custom + Promptfoo PR gate) | Security | Test | 16 | 5–6 | SEC |
| US-001-T09 | Golden-set environment fixtures (H1/D-8): (CVE × synthetic-environment) pairs with per-environment ground truth + adversarial control-present/absent, reachable/unreachable cases | QA | Test | 20 | 4–7 | PY-2 |
| US-001-T13 | `search_msrc` MCP tool: AIBOM registration, SHA-256 schema pin, rate limit, attack-story threat model (`mcp-security.md`) — closes a documentation/build gap found in the 2026-07-12 traceability audit: `catalogs.md`'s `microsoft-msrc` research source had no MCP tool | Backend | Code | 8 | 6 | PY-1 |
| EP-03-F01-T09 | **Rewritten (D-34):** `LLMProviderPort` direct Bedrock SDK integration (`@aws-sdk/client-bedrock-runtime`) + NestJS `LLMFallbackService` (Bedrock → direct Anthropic → local vLLM, weighted retry/reweighting) — per-tenant budget enforcement and **tenant-namespaced** semantic caching as NestJS interceptors on Valkey, not a proxy sidecar (costed spine, [ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer)) | Backend | Code | 28 | 3–5 | PY-2 |
| EP-03-F01-T10 | `InstrumentedLLMClient` decorator: per-tenant cost attribution (`LLM_USAGE_EVENT` — Gate-1 AC), fallback, OTel GenAI, sanitization + `no-direct-llm-sdk` CI (costed spine) | Backend | Code | 16 | 4–6 | PY-2 |
| EP-03-F01-T11 | Rule-based confidence bands + taxonomy projections (Platt → Gate 2; costed spine) | Backend | Code | 16 | 6–7 | PY-1 |
| US-001-T12 | Extend control-check catalog with WAF/IPS/segmentation control types (`taxonomy.md` control_type enum, beyond current EDR + cloud-SG) — competitor-scan CS-15, claims-priority HIGH (2026-07-11) | Backend | Code | 14 | 6–7 | PY-1 |
| US-001-T14 | Golden-set held-out validation of the 45%-cache cost assumption behind the <$0.75/workflow SLO — measured per-tier hit rate, not assumed (SR-01/stack-review §2; CI-07 A1/A2/D1 closure) | QA | Test | 10 | 11 | PY-2 |

## EP-03-F02 — Sandbox execution & assessment quality

**Parent:** EP-03 · **FRs:** FR-026, FR-017 · **ACs:** [assessment-trace spec](../10-product/features/assessment-trace.md); `execution_results` populated at Gate 1; AST pre-scan before every run · **Dependencies:** `firecracker-containerd`/Kata K8s integration (EP-01-F02-T11) · **API boundary:** `SandboxPort` → `SelfHostedFirecrackerAdapter`; `GET /assessments/{id}/trace` · **SLA:** hash-chained steps; drift 2σ alert (FR-017).

### US-017 — Assessment Trace + execution results (Gate 1, promoted) · 13 pts · persona: security engineer
**ACs:** step timeline hash-chained; code artifact read-only; `execution_results` non-null (null only via kill path). **DoD:** merge gates + `test:sandbox-execution-isolation` + `test:script-ast-scan-reject`.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-017-T01 | **(D-33)** Self-hosted Firecracker operational checklist: `firecracker-containerd`/Kata runtimeclass, patch-SLA ownership, CVE-advisory subscription, kill-path | Security | Research | 12 | 2 | SEC |
| US-017-T02 | `SandboxPort` + `SelfHostedFirecrackerAdapter` (self-hosted Firecracker on K8s) | Backend | Code | 24 | 3–5 | PY-2 |
| US-017-T03 | `ScriptSecurityScanner` AST pre-scan (reject before every execution) | Security | Code | 16 | 4–5 | PY-2 |
| US-017-T04 | Investigation code generation → execution → artifact storage (R2, integrity hash) | Backend | Code | 24 | 5–7 | PY-1 |
| US-017-T05 | Trace API + hash-chain verification + trace viewer backend (Week 10) | Backend | Code | 16 | 8–10 | TS-1 |
| US-017-T06 | `EvaluationAgent` — 10% stratified drift sampling vs 30-day baseline (FR-017, Week 10) | Data | Code | 14 | 9–10 | PY-2 |
| US-017-T07 | Sandbox isolation + AST-reject test suites | QA | Test | 12 | 6–7 | TS-3 |
| EP-03-F02-T08 | Sandbox tenant cap (300 s/hr + 5 VMs) + `NoOpSandboxAdapter` kill-path + partial-failure state machine (D-9; costed spine) | Backend | Code | 12 | 6–7 | PY-2 |
| EP-03-F02-T09 | Burst-tier sandbox quota model: reconcile the 300 s/hr + 5-concurrent-microVM budget (D-9) against the "zero-day in minutes" claim — defines a burst allowance still bounded by the D-9 caps (SR-11/CI-07 B5 closure) | Security | Research | 8 | 7 | SEC |

## EP-03-F03 — Outcome learning — **Deferred (post-Gate-3 candidate)**

**Parent:** EP-03 · **FR:** FR-027 · Story US-025 (draft) — behind an `outcome_learning` flag; no tasks until Gate-3+ decision and sufficient remediation-outcome data volume exists. Added 2026-07-11 per competitor-scan disposition CS-01 (accepted, deferred).

### US-025 — Outcome Learning (deferred, draft) · persona: security engineer / data
**Scope (draft):** track remediation outcome (success / failure / human_override) per `asset_vuln`; feed back into exploitability-score confidence and mitigation ranking. Confirmed distinct from US-009 Preference Learning (stated risk-appetite, not empirical outcome tracking) — see `00-meta/competitor-scan-2026-07-10-dux-plus.md` CS-01 for full analysis before that file was retired into this entry.

## EP-03-F04 — Predictive risk forecasting (E4) — **Spec authored, build unscheduled**

**Parent:** EP-03 · Promoted from aspirational to committed roadmap capability per the 2026-07-13 claims-alignment directive (CI-07; [vision-reference §3](../00-meta/vision-reference.md) E4). **BR-013 → EP-03 → US-028 → FR-030, Gate 2** — spec closed 2026-07-16 (D-24): [predictive-risk-forecasting](../10-product/features/predictive-risk-forecasting.md). The row below funded spec-authoring only; **no implementation task exists yet** — sizing the build (new `EPSS_SCORE_HISTORY` ingest, `AssetRiskTrendWorkflow`, `GET /assets/risk-trend`) is unscheduled, separate work for the Gate-2 planning pass.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| EP-03-F04-T01 | Author the BR/FR/US spec for predictive risk forecasting (E4) — **done**, see `predictive-risk-forecasting.md` | Product | Research | 8 | 11 | PM |

**EP-03 total: 426 h** (286 base + 72 costed spine + 20 H1 + 14 h US-001-T12/CS-15 + 8 h US-001-T13/2026-07-12 traceability audit + 10 h US-001-T14/CI-07 A1 + 8 h EP-03-F02-T09/CI-07 B5 + 8 h EP-03-F04-T01/CI-07 E4, 2026-07-16; costed spine corrected 76→72 h, 2026-07-19 — EP-03-F01-T09's D-34 rewrite (LiteLLM-gateway integration → direct-Bedrock-SDK integration) lowered its estimate 32→28 h without the footer cascading; task-row hours are ground truth, this total is recomputed to match them, see [decisions-log](../00-meta/decisions-log.md)); EP-03-F03 adds 0 h (deferred, no tasks)
