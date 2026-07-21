---
owner: Founder
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-7, D-8, D-19, D-34, D-35, D-40, D-42]
---

# Execution Traceability Matrix — Phase 1 Portfolio

**Purpose:** the portfolio matrix (Epic → Feature → Story), and the hour rollup against capacity.

**Status values:** `Defined` (feature) · `To Do` (task) · `Deferred` (gate-fenced).

**All statuses cascade upward — an epic is Done only when every child task is Done.** Task detail lives in the per-epic backlog files.

> **The hour figures in this file are machine-checked.** `scripts/validate-playbooks.py` reconciles each epic's task rows against its `**EP-NN total**` line, and both against the rollup table below. **Change one without the other and the merge gate fails.**

## Portfolio matrix (Epic / Feature / Story level)

| Level | ID | Title | Parent | Children | Status | Owner | Tech-lead notes |
|-------|----|-------|--------|----------|--------|-------|-----------------|
| Epic | EP-01 | Multi-tenant platform & auth | BR-001 | F01–F02 | Defined | CTO | RLS + fuzz = Gate-1 hard blocker (Week 4) |
| Feature | EP-01-F01 | Tenant platform: auth, isolation, settings, lifecycle | EP-01 | US-014 (+2 enabler) | Defined | TS-1 | PgBouncer session mode mandatory for RLS |
| Story | US-014 | Tenant Settings | EP-01-F01 | T01–T08 | To Do | TS-1 | 13 pts; PgBouncer fuzz spike Week 4 |
| Feature | EP-01-F02 | Platform foundations & launch blockers | EP-01 | T01–T09 (enabler) | Defined | TS-2 | trust/status 200 blocks first NDA partner (range corrected 2026-07-12) |
| Epic | EP-02 | Environmental data ingestion | BR-004 | F01–F05 | Defined | CTO | ≥3 vendor connectors live = Gate-1 criterion |
| Feature | EP-02-F01 | Connector Hub & first-party ingest | EP-02 | US-013 (+1 enabler) | Defined | TS-1 | AWS P0 for Week-6 vertical slice |
| Story | US-013 | Connector Hub | EP-02-F01 | T01–T08 | To Do | TS-1 | 13 pts; sync as isolated ECS service |
| Feature | EP-02-F02 | Vendor framework & evidence screens | EP-02 | US-002, US-003 (+2 enabler) | Defined | TS-2 | vendor sandbox accounts needed Week 3 |
| Story | US-002 | Asset Context Evidence | EP-02-F02 | T01–T06 | To Do | TS-2 | 5 pts; BS-19 DTO spike first; +T05 criticality/env fields (CS-16); +T06 Splunk connector |
| Story | US-003 | Protection Breakdown | EP-02-F02 | T01–T05 | To Do | TS-1 | 8 pts; CrowdStrike + Wiz |
| Feature | EP-02-F03 | Ownership inference | EP-02 | US-007 | Defined | PY-1 | ServiceNow OR Entra — pick by partner stack |
| Story | US-007 | Ownership Inference | EP-02-F03 | T01–T06 | To Do | PY-1 | 8 pts; +T06 broader auto-tagging (CS-25, after US-002-T05) |
| Feature | EP-02-F04 | Physical residency | EP-02 | US-020 | **Deferred (Gate 5)** | — | needs signed on-prem contract |
| Feature | EP-02-F05 | Proactive tool discovery | EP-02 | US-027 | Defined | TS-1 | added 2026-07-11, CS-23 |
| Story | US-027 | Proactive Tool Discovery | EP-02-F05 | T01–T04 | To Do | TS-1 | 8 pts; Gate 2 candidate |
| Epic | EP-03 | Exploitability assessment engine | BR-002/007 | F01–F04 | Defined | CTO | core value path; golden set = regression gate |
| Feature | EP-03-F01 | Assessment workflow & CaMeL plane | EP-03 | US-001 | Defined | PY-1 | Temporal + CaMeL; Week-4 spike gates chat |
| Story | US-001 | Prerequisites Analysis | EP-03-F01 | T01–T09, T12–T14 | To Do | PY-1 | 13 pts; largest story; T10/T11 are `EP-03-F01` enabler tasks, not US-001's own (corrected range 2026-07-12); +T12 WAF/IPS/segmentation (CS-15); +T13 `search_msrc` MCP tool; +T14 golden-set cache-hit validation (CI-07 A1) |
| Feature | EP-03-F02 | Sandbox execution & quality | EP-03 | US-017 (+2 enabler) | Defined | PY-2 | Self-hosted Firecracker checklist Week 2 (D-33); +T09 burst-tier quota model (CI-07 B5) |
| Story | US-017 | Assessment Trace + execution results | EP-03-F02 | T01–T07 | To Do | PY-2 | 13 pts; `execution_results` = Gate-1 assertion |
| Feature | EP-03-F03 | Outcome learning | EP-03 | US-025 | **Deferred (post-Gate-3)** | — | added 2026-07-11, CS-01 |
| Feature | EP-03-F04 | Predictive risk forecasting (E4) | EP-03 | T01 | **Spec authored (D-24); build unscheduled** | PM | spec-authoring task (T01) done 2026-07-16; implementation sizing is separate, unscheduled Gate-2 planning work |
| Epic | EP-04 | Continuous re-assessment | BR-002 | F01 | Defined | CTO | GCIS G1 claim-safety blocker |
| Feature | EP-04-F01 | Continuous assessment engine | EP-04 | US-021 | Defined | TS-1 | most triggers must resolve w/o LLM call |
| Story | US-021 | Re-assessment Scheduler | EP-04-F01 | T01–T06 | To Do | TS-1 | 8 pts |
| Epic | EP-05 | Analyst surfaces & APIs | BR-002/006/008/010 | F01–F03 | Defined | PM | design-partner demo surface |
| Feature | EP-05-F01 | Dashboards & drill-down | EP-05 | US-010, US-011, US-012 (+2 enabler) | Defined | TS-3 | CTE benchmark Week 4 feeds NFR-004; +T07 H9 outcome instrumentation (CI-07 C1/C2) |
| Story | US-010 | Research Dashboard | EP-05-F01 | T01–T05 | To Do | TS-3 | 8 pts |
| Story | US-011 | Exposure Analysis | EP-05-F01 | T01–T05 | To Do | TS-3 | 13 pts; p95 <500 ms @1 K assets |
| Story | US-012 | Dashboard Home | EP-05-F01 | T01–T04 | To Do | TS-3 | 5 pts |
| Feature | EP-05-F02 | Chat guidance | EP-05 | US-008 | Defined | PY-1 | blocked until Week-4 spike passes |
| Story | US-008 | Chat Guidance | EP-05-F02 | T01–T08 | To Do | PY-1 | 13 pts; writes Week 12 only; +T08 remediation-option ranking |
| Feature | EP-05-F03 | Help & support | EP-05 | US-015 | **Deferred (Gate-1 capacity fallback, D-19)** | PM | P1, not Gate-1 blocking |
| Story | US-015 | Help & Support | EP-05-F03 | T01–T03 | Deferred | TS-3 | 2 pts |
| Epic | EP-06 | Mitigation & remediation write path | BR-002/003 | F01–F04 | Defined | CTO | D-4 as amended 2026-07-13: writes unattended by default; anomaly-escalation approve/deny surface by Week 12 |
| Feature | EP-06-F01 | Action cards & fast actions | EP-06 | US-004, US-016 | Defined | TS-1 | rollback catalog (E-4) is a ship prerequisite (required audit field) — authored, D-15, OI-03 closed |
| Story | US-004 | Action Cards | EP-06-F01 | T01–T07 | To Do | TS-1 | 8 pts; unattended, T2/T3 tiers; +T07 IAM/segmentation compensating controls (CS-20) |
| Story | US-016 | Fast Actions | EP-06-F01 | T01–T04 | To Do | TS-2 | 5 pts; `POST /fast-actions` unattended at Gate 1 |
| Feature | EP-06-F02 | Control refinements | EP-06 | US-005 | **Deferred (Gate-1 capacity fallback, D-19)** | TS-2 | needs Wiz/Qualys ingest live |
| Story | US-005 | Control Refinements | EP-06-F02 | T01–T03 | Deferred | TS-2 | 5 pts |
| Feature | EP-06-F03 | Remediation tickets | EP-06 | US-018 | Defined | TS-1 | T1 tier, unattended; ServiceNow dependency |
| Story | US-018 | Remediation Ticket Panel | EP-06-F03 | T01–T06 | To Do | TS-1 | 8 pts; +T06 severity-tiered routing via existing Slack/PagerDuty (CS-27) |
| Feature | EP-06-F04 | Closed-loop validation | EP-06 | US-019 | **Deferred (Gate 3)** | — | needs field-proven safety record; scope expanded 2026-07-11 (CS-02 pre-approval dry-run, CS-24 post-mitigation re-validation) |
| Epic | EP-07 | Safety & governance | BR-003/005/009 | F01–F02 | Defined | SEC | pre-launch blocker set |
| Feature | EP-07-F01 | Audit trail & exposure delta | EP-07 | US-006 | Defined | TS-2 | hash chain + /audit/verify |
| Story | US-006 | Audit & Exposure Delta | EP-07-F01 | T01–T06 | To Do | TS-2 | 5 pts; +T05/T06 daily-count + calendar nav |
| Feature | EP-07-F02 | Kill switch, kernel & HITL (enabler) | EP-07 | T01–T09, T10a–d, T11 (enabler) | Defined | SEC | KS-007 Week-2 spike; kernel before any tool dispatch; range corrected 2026-07-12 (old T10 split into T10a–d + new T11 citations gate) |
| Epic | EP-08 | Programmatic platform | BR-006/011 | F01–F03 | Defined | TS-2 | webhooks P1; /v1 Seed |
| Feature | EP-08-F01 | Outbound webhooks (enabler) | EP-08 | T01–T04 (enabler) | Defined | TS-2 | NATS JetStream durable queue, not BullMQ |
| Feature | EP-08-F02 | Public data API /v1 | EP-08 | US-022, US-024 | **Deferred (Seed)** | — | openapi.yaml skeleton ready (BS-17a) |
| Feature | EP-08-F03 | Outbound MCP server | EP-08 | US-026 | **Deferred (no near-term trigger)** | — | added 2026-07-11, CS-03; gates through EP-07 kernel when scheduled |
| Epic | EP-09 | Triage disposition | BR-012 | F01 | **Deferred (Gate-1 capacity fallback, D-19)** | TS-2 | — |
| Feature | EP-09-F01 | Acknowledgment lifecycle | EP-09 | US-023 | **Deferred (Gate-1 capacity fallback, D-19)** | TS-2 | — |
| Story | US-023 | Acknowledge Vulnerability Instance | EP-09-F01 | T01–T04 | Deferred | TS-2 | 3 pts |
| Epic | EP-10 | Personalization | BR-002 | F01 | **Deferred (Gate 2c)** | — | needs behavioral data volume |
| Feature | EP-10-F01 | Preference engine | EP-10 | US-009 | **Deferred (Gate 2c)** | — | spec ready |

## Hour rollup vs capacity (re-baselined 2026-07-09, [D-7 R1 + D-8](../00-meta/decisions-log.md); competitor-scan additions 2026-07-11; traceability-audit gap-fill 2026-07-12)

| Epic | Base | Costed spine (+174) | H-fold (+42, D-8) | Competitor-scan (+120, 2026-07-11) | Traceability-audit (+52, 2026-07-12) | CI-07 closure (+38, 2026-07-16) | K8s stack (+98, 2026-07-19, D-33/D-34) | Total | Share |
|------|------|--------------------|-------------------|--------------------------------------|----------------------------------------|-------------------------------------|-----------------------------------|-------|-------|
| EP-01 Platform & auth (+foundations, +legal lane) | 260 | +12 FE scaffolding | — | — | — | — | +98 K8s(EKS)/CloudNativePG/NATS/Valkey/MinIO/Temporal + Firecracker (Bifrost P0 spike retired, D-34, −16h) | 370 | 17% |
| EP-02 Data ingestion | 304 | +16 WM versioning · +10 cred encryption | +12 H3 scoped action creds | +14 CS-16 asset fields · +18 CS-25 auto-tagging · +44 CS-23/US-027 | +16 Splunk connector (US-002-T06) | — | — | 434 | 20% |
| EP-03 Assessment engine (+LLM gateway) | 286 | +48 LLM gateway · +16 calibration bands · +12 sandbox cap/kill-path | +20 H1 (CVE × env) eval | +14 CS-15 WAF/IPS/segmentation | +8 `search_msrc` MCP tool (US-001-T13) | +10 A1 golden-set validation · +8 B5 sandbox quota model · +8 E4 spec placeholder | — | 426 | 20% |
| EP-04 Continuous | 68 | — | — | — | — | — | — | 68 | 3% |
| EP-05 Analyst surfaces | 288 | +24 API completeness (rate-limiter, checkpoint/resume, Redoc) | — | — | +14 remediation-option ranking (US-008-T08) | +12 C1/C2 H9 outcome instrumentation | — | 322 | 15% |
| EP-06 Write path | 156 | — | +10 H4 impact preview | +14 CS-20 compensating controls · +16 CS-27 severity routing | — | — | — | 196 | 9% |
| EP-07 Safety & governance | 212 | +18 MCP split (T10a–d) · +18 citations gate + OWASP artifacts | — | — | +14 audit daily-count/calendar (US-006-T05/T06) | — | — | 262 | 12% |
| EP-08 Programmatic | 40 | — | — | +0 (EP-08-F03 deferred, no tasks) | — | — | — | 40 | 2% |
| EP-09 Acknowledgment | 0 (deferred) | — | — | — | — | — | — | 0 | — |
| EP-10 Personalization | 0 (deferred) | — | — | — | — | — | — | 0 | — |
| **Total** | **1,614 h** | **+174 h** | **+42 h** | **+120 h** | **+52 h** | **+38 h** | **+98 h** | **2,118 h** | 100% |

**Capacity check (D-7 R1 envelope; D-19 fallback applied 2026-07-15; CI-07 closure tasks added 2026-07-16; envelope re-baselined 2026-07-16, D-23):** envelope = 5 eng × **16 wk** × 25 focused h = **2,000 h**. The backlog carried 2,073 h (73 h over) until Sagi picked the documented fallback lever (D-19): EP-09 (30 h), US-015 (11 h), and US-005 (30 h) are deferred to the Gate-2 backlog — removing 71 h, landing at 2,002 h (~100.1%). The claims-alignment directive (CI-07) then added 38 h of engineering-to-close commitments — A1 golden-set cache-hit validation (10 h), B5 sandbox burst-tier quota model (8 h), C1/C2 H9 outcome instrumentation (12 h), E4 predictive-risk-forecasting spec placeholder (8 h) — none deferrable, since CI-07 binds live marketing claims — landing the backlog at **2,040 h (~102%, 40 h over the 2,000 h envelope)**.

**D-23 (2026-07-16): the envelope is re-baselined, not the scope.** Rather than cut 40 h of CI-07 claims-binding or D-19-deferred work, Sagi raised the focused-hours assumption from 25 to **26 h/week** — one additional focused hour/day across the 16-week window, still 5 engineers, still no sixth hire (the D-7 R1 headcount constraint is unchanged). New envelope: 5 eng × 16 wk × 26 h = **2,080 h**. The 2,040 h backlog landed at **~98% (40 h buffer)** — the same buffer shape D-8 used for the original 1,901 h/2,000 h re-baseline. **[OI-32](../00-meta/open-items.md) is resolved by this re-baseline**, same disposition as OI-05 before it.

**New overrun (2026-07-19, D-33): the self-hosted-Kubernetes stack replacement adds 114 h to EP-01** (K8s cluster bring-up, CloudNativePG, NATS/JetStream, Valkey, MinIO, self-hosted Temporal, Vault, plus the Bifrost and Firecracker work pulled forward from Gate-2/3 by ADR-010 R4/ADR-015 R4) — see [backlog-ep01](backlog-ep01.md). **The backlog stood at 2,154 h against the 2,080 h envelope (~103.5%, 74 h over)**, regressing the OI-32/D-23 buffer.

**Narrowed (2026-07-19, D-34): net −16 h against the D-33 figure, before unestimated net-new scope.** The DO/Linode LKE → EKS cluster-CSP swap (T02a) is hour-neutral — provisioning an EKS cluster via Pulumi is not materially more or less work than the DO/Linode equivalent it replaces. The P0 Bifrost bake-off spike (T10, 16 h) is retired outright, since LiteLLM removal ([ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer)) leaves no primary-slot proxy to bake off against — Bifrost becomes a Gate-2-only spike, not a Gate-1 line item. This lands EP-01's K8s-stack addition at 98 h and the portfolio at **2,138 h (~102.8%, 58 h over the 2,080 h envelope)** — narrower than D-33's overrun, still over.

**Corrected (2026-07-19, this review): Total column reconciled to 2,118 h, down from 2,138 h.** Two changes, neither touching the per-wave columns above (each stays each wave's originally-stated contribution, as a historical record): **EP-03 430→426 h** — a pre-existing 4 h arithmetic slip inside the "costed spine" sub-total (`EP-03-F01-T09`'s D-34 rewrite, LiteLLM-gateway integration → direct-Bedrock-SDK integration, lowered its own row estimate 32→28 h without the footer cascading), found by re-running the restored `scripts/validate-playbooks.py`. **EP-05 338→322 h** — D-35's −16 h Week-6 Mastra/LangGraph.js decision-sprint closure (`US-008-T02`, already 0 h at the row level) had never been cascaded into this table. Net: the true portfolio total is **2,118 h against the 2,080 h envelope — 38 h over (~101.8%)**, not the previously-stated 58 h/2,138 h. See [backlog-ep03](backlog-ep03.md), [backlog-ep05](backlog-ep05.md), and [decisions-log](../00-meta/decisions-log.md).

**Envelope raised a third time (2026-07-20, D-40): 26 → 27 focused h/week.** New envelope: 5 eng × 16 wk × 27 h = **2,160 h**. The corrected 2,118 h backlog lands at **~98.1% (42 h buffer)** — the same buffer shape D-8 and D-23 used for the prior two re-baselines. **[OI-39](../00-meta/open-items.md) is resolved by this re-baseline**, same disposition as OI-32/OI-05 before it.

**Not fully re-costed: Agentic RAG, Apache AGE, and the Gate-2 vLLM+Phi-4 S-LLM path are net-new scope this pass did not estimate.** These land in EP-03 (Assessment engine) and possibly EP-01 (Apache AGE extension install), not EP-01's K8s-stack line above — a real engineering estimate is needed before they can be added to this table, consistent with this file's own "no invented hours" discipline. **This is explicitly not covered by the D-40 buffer above** — per D-40's own rationale, absorbing an unknown net-new number into a fourth consecutive envelope raise is rejected as a precedent; once sized, this scope needs its own funding decision (a real re-baseline or an actual scope cut), tracked as a fresh open item once the estimate exists.

Legal/PM lane (EU AI Act opinion Wk 6, **ZDR with OpenAI/Anthropic (H2)**, Langfuse DPA Wk 8, SLA counsel, claims checklist) carries 0 eng hours but two hard external deadlines.

**Gate-2 / fast-follow register (D-8, not in the Gate-1 sum):** H5 pre-approved-scope policy ~12 h · H8 connector-freshness + live contract tests ~16 h · H11 full SBOM/SLSA ~8 h (H6/H9 are runbook/instrumentation, low-eng). If additional 🔴-class scope lands in Phase 1, re-look at the Week-12 date.

**Task-week convention after re-baseline:** existing task target-weeks are unchanged (work lands in the same build order); only gate-anchored dates moved — Gate 1 review Wk 10→12, minimal HITL UI →Wk 12, `chat_write_tools` →Wk 14, exit Wk 14→16. Spine/H tasks schedule into the freed Wk 9–12 window at sprint planning.

**Discipline load (rough):** Backend ~52% · Frontend ~17% · QA ~13% · DevOps ~10% · Security ~8%. TS-1/PY-1 are critical-path-loaded Weeks 3–8; SEC is a single point of failure for kernel+MCP+identity.
