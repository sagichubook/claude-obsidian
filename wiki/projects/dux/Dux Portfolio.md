---
type: project
title: "Dux Portfolio"
created: 2026-07-21
updated: 2026-07-23
status: active
deadline: "Phase 1 exit, Week 16 (Gate 1 review Week 12)"
tags: [project, dux, dux/execution]
related: ["[[Dux]]", "[[Dux Decisions & Traceability Reference]]", "[[Dux AI Safety Guide]]", "[[Dux Product Guide]]"]
sources: [".raw/dux/90-execution/README.md", ".raw/dux/90-execution/traceability.md", ".raw/dux/90-execution/backlog-ep01.md", ".raw/dux/90-execution/backlog-ep02.md", ".raw/dux/90-execution/backlog-ep03.md", ".raw/dux/90-execution/backlog-ep04.md", ".raw/dux/90-execution/backlog-ep05.md", ".raw/dux/90-execution/backlog-ep06.md", ".raw/dux/90-execution/backlog-ep07.md", ".raw/dux/90-execution/backlog-ep08.md", ".raw/dux/90-execution/backlog-ep09.md", ".raw/dux/90-execution/backlog-ep10.md"]
---

# Two Thousand One Hundred Eighteen Hours: Inside the Dux Phase-1 Backlog

### How ten epics, twenty-eight canonical stories, and a 42-hour buffer turn a product vision into a 16-week build plan

Navigation: [[Dux]] | [[Dux Product Guide]] | [[Dux Decisions & Traceability Reference]]

Every product roadmap eventually has to answer an unglamorous question: *who is doing what, in what order, and does it fit in the calendar we've got?* For Dux — a platform whose entire pitch rests on the claim that it can tell a security team which vulnerabilities are actually exploitable in their environment — that question got answered in a document that is equal parts spreadsheet and constitution. This is that document, told as a walk through the build.

The shape of it: **10 epics, 22 features, 28 canonical user stories, and roughly 150 tasks**, spread across five engineering roles over a 16-week engagement that started 2026-06-23. The plan lives here for reference; it was imported into the team's actual tracker (Linear) as the working source, and this page is not a live status mirror of that tracker.

What makes it worth reading as a narrative rather than skimming as a table is the arithmetic underneath it. The backlog totals **2,118 engineering hours** against a **2,160-hour capacity envelope** — 5 engineers × 16 weeks × 27 focused hours a week — which lands at **~98.1% utilization with a 42-hour buffer**. That envelope wasn't set once and left alone. It was re-baselined three separate times (decisions D-19, D-23, D-40), and each time the team's answer to "we're over capacity" was to raise the envelope rather than cut scope. A manual audit later caught a 20-hour arithmetic slip that had quietly accumulated in the rollup table — fixed by the same reconciliation script, `scripts/validate-playbooks.py`, that keeps the rest of this corpus honest.

The framework itself bends in seven documented, deliberate places — never to invent a story to hit a quota (the 28-story set is closed and canonical), but to accommodate placeholders for team members not yet hired, or to admit that some scope genuinely isn't ready to be estimated. Three pieces of work — Agentic RAG, the Apache AGE graph database layer, and the Gate-2 vLLM+Phi-4 S-LLM path — are deliberately left unestimated as net-new scope and sit outside the 42-hour D-40 buffer entirely.

## The ID scheme that holds it together

Before the epics, the skeleton. Every level of the plan maps to a fixed ID format, and only two of those levels are allowed to mint new IDs at this planning layer:

| Level | Format | Source |
|-------|--------|--------|
| Epic | `EP-01` … `EP-10` | Canonical, defined in [[Dux Decisions & Traceability Reference]] |
| Feature | `EP-xx-Fyy` | New at this planning layer |
| Story | `US-001` … `US-028` | Canonical, closed set — never invented here |
| Task | `US-xxx-Tzz` or `EP-xx-Fyy-Tzz` | New at this planning layer |

**Capacity:** 5 engineers (3 TypeScript: TS-1, TS-2, TS-3; 2 Python: PY-1, PY-2), plus Security (SEC), PM, and CTO role placeholders. Definition of Done: every merge gate green + verification command passing + deployed to staging + demoed — no partial credit.

## The portfolio, at a glance

Ten epics carry the weight unevenly. Three of them — data ingestion, the assessment engine, and multi-tenant auth — account for well over half the hours. Two more (triage disposition and personalization) carry zero Gate-1 hours at all; they exist in the plan as placeholders, not as work in flight.

| Epic | ID | Focus | BR(s) | Priority | Hours | Share | Gate |
|------|----|-------|-------|----------|-------|-------|------|
| Multi-tenant platform & auth | EP-01 | Zero cross-tenant leakage | BR-001 | P0 | 370 | 17% | Gate 1 |
| Environmental data ingestion | EP-02 | Connectors and feeds | BR-004 | P0 | 434 | 20% | Gate 1 |
| Exploitability assessment engine | EP-03 | Core reasoning pipeline | BR-002/007 | P0 | 426 | 20% | Gate 1 |
| Continuous re-assessment | EP-04 | Keeping verdicts fresh | BR-002 | P0 | 68 | 3% | Gate 1 |
| Analyst surfaces & APIs | EP-05 | Dashboards, drill-down, chat | BR-002/006/008/010 | P0 | 322 | 15% | Gate 1 |
| Mitigation & remediation write path | EP-06 | Action surfaces | BR-002/003 | P0 | 196 | 9% | Gate 1 |
| Safety & governance | EP-07 | Kill switch, kernel, audit | BR-003/005/009 | P0 | 262 | 12% | Gate 1 |
| Programmatic platform | EP-08 | Public API, webhooks | BR-006/011 | P1 | 40 | 2% | Gate 1 (webhooks) |
| Triage disposition | EP-09 | Acknowledgment lifecycle | BR-012 | P1 | 0 | — | Deferred (Gate 2) |
| Personalization | EP-10 | Preference learning | BR-002 | P2 | 0 | — | Deferred (Gate 2c) |
| **Total** | | | | | **2,118** | **100%** | |

Laid out across the calendar, the dependency chain becomes visible: platform and safety work runs from day one because everything else needs a foundation to stand on; ingestion and the assessment engine start in parallel because the assessment engine needs evidence to reason about; and the analyst-facing surfaces only really get going once there's something to display.

```mermaid
gantt
    title Dux Phase 1 (16 weeks, Gate 1 review Week 12)
    dateFormat YYYY-MM-DD
    axisFormat W%W
    section Platform
    Multi-tenant auth (370h)      :active, ep01, 2026-06-23, 8w
    section Core
    Data ingestion (434h)         :ep02, 2026-06-23, 9w
    Assessment engine (426h)      :ep03, 2026-06-23, 9w
    Continuous re-assessment (68h) :ep04, after ep03, 3w
    section Surfaces
    Analyst surfaces (322h)       :ep05, 2026-07-07, 8w
    Write path (196h)             :ep06, 2026-07-07, 6w
    section Safety
    Safety & governance (262h)    :crit, ep07, 2026-06-23, 9w
    section Deferred
    Programmatic (40h)            :ep08, 2026-07-21, 3w
    Triage disposition (deferred) :done, ep09, 2026-08-01, 1d
    Personalization (deferred)    :done, ep10, 2026-08-01, 1d
```

## Where the hours actually came from

The 2,118-hour total isn't a single estimate handed down at kickoff — it's a base figure that absorbed six separate waves of correction and addition as the team learned more about what "done" would actually require: a costed-spine pass to price out infrastructure plumbing, an "H-fold" wave (D-8), a competitor-scan wave that added claims the product needed to be able to back up, a traceability-audit gap-fill, a CI-07 closure pass that funded claims-alignment work, and — the largest single jolt — a self-hosted Kubernetes stack replacement in mid-July (D-33/D-34) that added 98 hours to EP-01 alone.

| Epic | Base | Costed Spine | H-Fold (D-8) | Competitor-Scan | Traceability Audit | CI-07 Closure | K8s Stack (D-33/D-34) | Total |
|------|------|-------------|-------------|-----------------|-------------------|---------------|----------------------|-------|
| EP-01 Platform & auth | 260 | +12 | — | — | — | — | +98 | 370 |
| EP-02 Data ingestion | 304 | +26 | +12 | +76 | +16 | — | — | 434 |
| EP-03 Assessment engine | 286 | +72 | +20 | +14 | +8 | +26 | — | 426 |
| EP-04 Continuous | 68 | — | — | — | — | — | — | 68 |
| EP-05 Analyst surfaces | 288 | +24 | — | — | +14 | +12 | — | 322 |
| EP-06 Write path | 156 | — | +10 | +30 | — | — | — | 196 |
| EP-07 Safety & governance | 212 | +36 | — | — | +14 | — | — | 262 |
| EP-08 Programmatic | 40 | — | — | — | — | — | — | 40 |
| EP-09 Acknowledgment | 0 | — | — | — | — | — | — | 0 |
| EP-10 Personalization | 0 | — | — | — | — | — | — | 0 |
| **Total** | **1,614** | **+174** | **+42** | **+120** | **+52** | **+38** | **+98** | **2,118** |

Stack that total against what five people can actually deliver in sixteen weeks, and the margin is thin but real:

| Metric | Value |
|--------|-------|
| Envelope (D-40) | 5 eng × 16 wk × 27 h = **2,160 h** |
| Corrected backlog | **2,118 h** |
| Buffer | **42 h (~1.9%)** |
| Deferred to Gate 2 (D-19) | EP-09 (30 h) + US-015 (11 h) + US-005 (30 h) = **71 h** |
| Unestimated net-new scope | Agentic RAG, Apache AGE, Gate-2 vLLM+Phi-4 — deliberately not in buffer |

That buffer isn't evenly distributed across the team, either. **Discipline load:** Backend ~52% · Frontend ~17% · QA ~13% · DevOps ~10% · Security ~8%. TS-1 and PY-1 carry the critical-path load through Weeks 3–8, and Security is a single point of failure across the kernel, the MCP gateway, and identity — three separate concerns owned by one role.

## The gates the work has to clear

Nothing in this plan ships in a vacuum — it ships against five checkpoints, and Gate 1 is the one that matters most for the next twelve weeks.

| Gate | Week | Key Milestones |
|------|------|----------------|
| Gate 1 Review | Week 12 | Isolation harness (EP-01), ≥3 vendor connectors (EP-02), golden set ≥80% (EP-03), continuous claim true (EP-04), design-partner demos (EP-05), unattended writes + HITL approve/deny (EP-06), kill switch + kernel + audit (EP-07), webhooks live (EP-08) |
| Gate 1 Exit | Week 16 | Full Gate-1 scope shippable; chat writes (US-008-T06 Week 14) |
| Gate 2 | Post-Gate 1 | Closed-loop validation (US-019), public data API (US-022/024), preference learning trigger, Agentic RAG, Apache AGE |
| Gate 2c | Post-Gate 2 | Personalization (EP-10) — needs behavioral data volume |
| Gate 3 | Post-Gate 2c | Outcome learning (EP-03-F03), closed-loop validation (EP-06-F04) |
| Seed | Post-Gate 3 | Public data API `/v1` (US-022/024), outbound MCP server (US-026) |

With the scaffolding in place, here's the epic-by-epic walk — roughly in build order, following the dependency chain rather than the numbering.

## EP-01 — Multi-Tenant Platform & Auth (370h, P0)

Everything else in this plan sits on top of tenant isolation, so EP-01 goes first not because it's numbered first but because nothing downstream is safe to build without it. Its objective is BR-001: zero cross-tenant leakage, measured by 100% isolation in CI. **Metrics:** `test:fuzz-tenant-id` 100%, ISO-001–010 + ISO-FUZZ-001–005 green, AUTH-001–005 green. **Target:** Gate 1 (Week 12); the isolation harness itself is due Week 4 and blocks Gate 1 outright. **Constraints:** ADR-001 (Better Auth via `AuthPort`), ADR-002 R2 (shared-schema row-level security FORCEd on CloudNativePG; PgBouncer transaction mode), composite foreign keys, fail-closed GUC.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-01-F01** Tenant platform: auth, isolation, settings, lifecycle | US-014 (13 pts) + 2 enabler tasks | Better Auth integration (24h), RLS FORCE migrations (24h), PgBouncer fuzz spike (12h), fuzz-tenant-id suite (16h), settings UI (20h), API keys/webhooks (12h), GDPR delete (16h), impersonation guardrails (8h), check-rls.sh (6h), ISO suite (16h) | 154 |
| **EP-01-F02** Platform foundations & launch blockers (enabler) | 0 stories — 14 enabler tasks | Monorepo+CI (20h), K8s EKS cluster (16h), CloudNativePG (16h), NATS+JetStream (8h), Valkey (4h), MinIO (8h), self-hosted Temporal (20h), Vault (12h), observability stack (28h), python-eval container (8h), local dev (8h), trust/status portals (12h), on-call (8h), models.json (10h), FE scaffolding (14h), Firecracker/Kata (24h) | 216 |

**EP-01 total: 370h** — includes +98h from the 2026-07-19 self-hosted Kubernetes stack replacement (D-33/D-34). The P0 Bifrost bake-off spike (16h) was retired by D-34.

## EP-02 — Environmental Data Ingestion (434h, P0)

The moment tenants can be trusted to stay in their own lane, the platform needs something to reason about — which is where EP-02 picks up. This is the largest single epic in the portfolio by hours, and its objective, BR-004 multi-source ingestion, comes with two sharp KPIs: time-to-value under 48 hours from AWS connect, and at least three vendor connectors live by Gate 1. **Target:** Gate 1 (Week 12); AWS lands as the P0 vertical slice by Week 6. **Constraints:** ADR-004 (SDK v3, cross-account IAM + external ID), ADR-011 R2 (`vendor-contract.ts`, role interfaces, RLS-scoped), ADR-006 R4.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-02-F01** Connector Hub & first-party ingest | US-013 (13 pts) + 1 enabler | AWS SDK v3 discovery (32h), cross-account IAM (12h), throttling backoff (8h), NVD/CISA-KEV/EPSS workers (20h), CSV upload (10h), Connector Hub UI (20h), K8s sync service (12h), sync integration tests (14h), World Model versioning (16h) | 144 |
| **EP-02-F02** Vendor connector framework & evidence screens | US-002 (5 pts) + US-003 (8 pts) + 2 enablers | AssetContextDto spike (6h), GET /assets/{id}/context (14h), asset context screen (12h), contract tests (8h), criticality+environment fields (14h), Splunk connector (16h), vendor-contract.ts (16h), CrowdStrike connector (20h), Wiz connector (16h), ProtectionProjection UI (14h), vendor RLS tests (10h), credential encryption (10h), scoped action credentials (12h) | 168 |
| **EP-02-F03** Ownership inference | US-007 (8 pts) | ServiceNow/Entra connector (18h), OwnershipInferenceWorkflow (16h), OWNERSHIP_EVIDENCE schema (8h), ownership UI (10h), test suite (8h), broader auto-tagging (18h) | 78 |
| **EP-02-F04** Physical residency | US-020 (deferred) | No tasks — needs signed on-prem contract | 0 |
| **EP-02-F05** Proactive tool discovery | US-027 (8 pts) | Tool-signature detection (12h), GET /connectors/discovered (14h), suggestion card UI (10h), discovery integration tests (8h) | 44 |

**EP-02 total: 434h** — includes +76h competitor-scan additions (CS-16, CS-23, CS-25) and +16h Splunk connector (US-002-T06).

## EP-03 — Exploitability Assessment Engine (426h, P0)

This is the core product. Everything upstream — tenancy, evidence — exists to feed this pipeline, and everything downstream — dashboards, chat, the write path — exists to act on what it decides. Its objective is twofold: BR-002 (the core product itself) and BR-007 (AI-BOM accuracy), with KPIs that read like a service-level agreement for judgment: mean time to exploitability verdict under 15 minutes per CVE, a golden-set accuracy floor of 80% by Gate 1 with under 2% regression tolerance, and a cost ceiling of $0.75 per assessment. **Target:** Gate 1 (Week 12); vertical slice by Week 6. **Constraints:** ADR-007 R3 (self-hosted Temporal, orchestrator-worker, child-workflow-per-tenant), ADR-008 R2 (CaMeL routing, domicile rules), ADR-015 R4 (self-hosted Firecracker microVM + AST pre-scan).

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-03-F01** Assessment workflow & CaMeL plane | US-001 (13 pts) | Temporal namespace setup (16h), ExploitabilityAssessmentWorkflow skeleton (32h), CaMeL split S-LLM/P-LLM (32h), prerequisite-extractor subagent (20h), llguidance spike (8h), POST /research/queue (12h), golden set v1 + DeepEval (32h), prompt-injection corpus (16h), env fixtures (20h), search_msrc MCP tool (8h), LLMProviderPort direct Bedrock (28h), InstrumentedLLMClient (16h), confidence bands (16h), WAF/IPS/segmentation controls (14h), golden-set validation (10h) | 280 |
| **EP-03-F02** Sandbox execution & quality | US-017 (13 pts) + 2 enablers | Firecracker operational checklist (12h), SandboxPort + SelfHostedFirecrackerAdapter (24h), ScriptSecurityScanner AST pre-scan (16h), investigation code generation (24h), Trace API + hash-chain (16h), EvaluationAgent drift sampling (14h), sandbox isolation tests (12h), sandbox tenant cap + kill-path (12h), burst-tier quota model (8h) | 138 |
| **EP-03-F03** Outcome learning | US-025 (deferred) | No tasks — post-Gate-3 candidate | 0 |
| **EP-03-F04** Predictive risk forecasting (E4) | Spec authored, build unscheduled | BR/FR/US spec authoring (8h) | 8 |

**EP-03 total: 426h** — includes +72h costed spine (LLM gateway, calibration bands, sandbox cap), +20h H1 env fixtures, +14h CS-15 WAF/IPS, +8h search_msrc, +26h CI-07 closure. EP-03-F03 adds 0h (deferred).

## EP-04 — Continuous Re-assessment (68h, P0)

The assessment engine's output is only as good as its freshness — a "continuous, 24/7" claim in the product marketing has to be engineering-true, not aspirational, by Gate 1. That's the whole reason EP-04 exists, and it's deliberately small: BR-002's continuous claim, measured against the GCIS G1 gate. **Metrics:** `test:reassessment-trigger` + `test:reassessment-dirtycheck`; the design goal is that most triggers resolve without a fresh LLM call at all. **Target:** Gate 1 (Week 12). **Constraints:** ADR-016 R2 (event + scheduled + dirty-check triggers), NATS core pub/sub, a `ReassessmentDebouncer` that coalesces on a 15-minute window.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-04-F01** Continuous assessment engine | US-021 (8 pts) | ReassessmentSchedulerWorkflow (16h), NATS event bus wiring (14h), ReassessmentDebouncer (8h), evidence-hash dirty-check (12h), schedule API + settings UI (8h), test suites (10h) | 68 |

## EP-05 — Analyst Surfaces & APIs (322h, P0)

With the assessment engine producing verdicts and re-assessment keeping them fresh, EP-05 is where a human actually sees the work — dashboards, drill-down, and the chat interface that turns queue triage from thousands of items down to tens. Objective: BR-002/006/008/010, with KPIs that are as much about perceived speed as raw throughput — sub-1-second SSE patches to the queue, and at least two committed design partners by Gate 1. **Target:** Gate 1 (Week 12); the chat spike happens Week 4, chat writes not until Week 14. **Constraints:** ADR-014 R2 (React + Vite/TanStack Router, SSE+POST rather than WebSocket), read-only MCP in Phase 1, and a CaMeL boundary drawn around chat specifically.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-05-F01** Dashboards & drill-down | US-010 (8 pts) + US-011 (13 pts) + US-012 (5 pts) + 2 enablers | ResearchDashboardDto (14h), SSE queue_row_update (10h), dashboard UI (20h), vulnerability aggregates (8h), E2E + SSE tests (8h), CVEDetailQuery (20h), attack-path CTEs (14h), drill-down UI (24h), ActionCardProjection (10h), perf test 1K assets (10h), DashboardHomeDto (10h), home UI (16h), degraded states (6h), snapshot tests (6h), API completeness (24h), H9 outcome instrumentation (12h) | 212 |
| **EP-05-F02** Chat guidance | US-008 (13 pts) | Week-4 chat spike (16h), chat session service + SSE (20h), MCP tool wiring + citations (16h), remediation-option ranking (14h), chat UI panel (18h), chat_write_tools + HITL UI (16h), injection suite (10h) | 110 |
| **EP-05-F03** Help & support | US-015 (deferred) | No tasks — deferred to Gate 2 (D-19 capacity fallback; 11h) | 0 |

**EP-05 total: 322h** — net of −11h US-015 deferral and −16h US-008-T02 decision sprint closure (D-35/ADR-021).

## EP-06 — Mitigation & Remediation Write Path (196h, P0)

Seeing the risk is only half the product; EP-06 is where Dux starts acting on it. Objective: BR-002/003 — the GCIS G3 claim that the full pipeline runs "at machine speed," Analyze→Mitigate→Remediate, unattended by default at Gate 1. **Metrics:** `test:vendor-action-hitl`, `test:governance-kernel`, `test:remediation-ticket-create`. **Target:** Gate 1 (Week 12) ships unattended action cards, fast actions, and ticket create/route; closed-loop validation (US-019) waits for Gate 3. **Constraints:** ADR-012 R3 (`VendorActionGate`, canonical→native mapping, T1–T3 tiers), and a gate chain — Intent→Budget→Effect→VendorAction→HITL — that every write has to pass through.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-06-F01** Action cards & fast actions | US-004 (8 pts) + US-016 (5 pts) | VendorActionGate (16h), QuickMitigationWorkflow (16h), rollback catalog (10h), action-card UI (14h), test suite (10h), HITL impact preview (10h), compensating-control discovery (14h), manual content engine (12h), Fast Actions UI (10h), mitigation webhooks (6h), manual-flow E2E test (6h) | 118 |
| **EP-06-F02** Control refinements | US-005 (deferred) | No tasks — deferred to Gate 2 (D-19 capacity fallback; 30h) | 0 |
| **EP-06-F03** Remediation tickets | US-018 (8 pts) | RemediationWorkflow create+route (16h), ServiceNow adapter (12h), ticket panel UI (12h), inbound ticket-status webhooks (8h), test suite (8h), severity-tiered routing (16h) | 72 |
| **EP-06-F04** Closed-loop validation | US-019 (deferred) | No tasks — Gate 3 (scope expanded CS-02, CS-24) | 0 |

**EP-06 total: 196h** — net of −30h US-005 deferral (D-19). Includes +10h H4 impact preview, +14h CS-20 compensating controls, +16h CS-27 severity routing.

## EP-07 — Safety & Governance (262h, P0)

Running alongside all of the above — not after it — is the epic that makes the rest of the plan trustworthy enough to ship. EP-07 covers three business requirements at once: BR-003 (a halt in under 5 seconds), BR-005 (tamper-evident audit), and BR-009 (AI-BOM drift caught in CI), with the headline KPI being a kill-switch p99 under 5 seconds before launch. **Metrics:** KS-001–007; GOV-001–013 via `test:governance-kernel`; `/audit/verify`; the AI-BOM drift gate. **Target:** pre-launch/Gate 1, with the KS-007 spike due Week 2 and gating the Gate-1 release itself. **Constraints:** a `KillSwitchRelay` on NATS core pub/sub with a CloudNativePG LISTEN/NOTIFY fallback, a fail-closed worker breaker, and a governance kernel implemented as a Chain of Responsibility that runs before every LLM call or tool dispatch — hash-chained audit partitions underneath all of it.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-07-F01** Audit trail & exposure delta | US-006 (5 pts) | Hash-chained audit partitions (16h), audit read/export API (10h), exposure-delta view (10h), chain-tamper tests (8h), daily-count aggregation (6h), calendar/date-picker nav (8h) | 58 |
| **EP-07-F02** Kill switch, governance kernel & HITL (enabler) | 0 stories — 11 enabler tasks | KillSwitchRelay (20h), KS-007 LISTEN/NOTIFY fallback (10h), governance kernel gates GOV-001–013 (28h), HITL contract SSE+POST (16h), approve/deny UI (14h), KS test suite (16h), governance kernel tests (12h), AIBOM manifest + drift gate (12h), agent identity SPIFFE (16h), MCP gateway core (14h), MCP schema integrity (10h), MCP egress proxy (10h), MCP circuit breaker (8h), LLM09 citations gate (18h) | 204 |

**EP-07 total: 262h** — includes MCP gateway split into T10a–d (+18h) and LLM09 citations gate (+18h). Single point of failure: SEC owns kernel + MCP + identity concurrently.

## EP-08 — Programmatic Platform (40h, P1)

The smallest funded epic, and deliberately so — EP-08 is the seam where Dux starts talking to the outside world programmatically, but only the P1 slice (webhooks) is scheduled for Gate 1. Objective: BR-006 programmatic access at Gate 1, BR-011 public data API held for Seed. **Metrics:** HMAC + DLQ replay tests; OpenAPI contract tests at Seed. **Target:** webhooks at Gate 1; public `/v1` waits for the Seed trigger. **Constraints:** ADR-005 R2 (NATS JetStream durable queues), HMAC-SHA256 signing plus an `Idempotency-Key`, and 5 delivery attempts with exponential backoff before falling into `webhook_dead_letter`.

| Feature | Stories | Key Tasks | Hours |
|---------|---------|-----------|-------|
| **EP-08-F01** Outbound webhooks (enabler) | 0 stories — 4 enabler tasks | WebhookDeliveryWorker on NATS JetStream (16h), Gate-1 event emitters (10h), DLQ replay CLI (6h), HMAC verification tests (8h) | 40 |
| **EP-08-F02** Public data API `/v1` | US-022, US-024 (deferred) | No tasks — Seed trigger; openapi.yaml skeleton ready (BS-17a) | 0 |
| **EP-08-F03** Outbound MCP server | US-026 (deferred) | No tasks — no near-term trigger; gates through EP-07 kernel | 0 |

## EP-09 and EP-10 — the two epics with nothing scheduled

Two epics round out the ten, and both carry zero Gate-1 hours by design rather than by oversight.

**EP-09 — Triage Disposition (0h, Deferred).** Objective: BR-012, the risk-acknowledgment lifecycle with audit. **Target:** a Gate-2 write path, with a Seed-stage public read of `is_acknowledged`. **Status:** deferred to the Gate-2 backlog as a D-19 capacity fallback (30 hours parked there). The spec is ready — `VulnerabilityInstanceAckService` — it simply isn't funded in this pass.

**EP-10 — Personalization (0h, Deferred).** Objective: BR-002's per-customer triage personalization. **Target:** Gate 2c — and deliberately not promoted any earlier, because it needs a volume of behavioral data the product won't have generated yet (it sits on the GCIS §B still-staged list). **Status:** spec ready — `PreferenceEngine` — with one invariant already locked in regardless of when it ships: it will never train across tenants. That's the isolation invariant EP-01 exists to guarantee, and EP-10 inherits it.

## The user story index

Twenty-eight stories, closed set, no exceptions. Reading this table left to right tells you not just what each story is, but which epic funds it, how many points it carries, who it's for, and whether it's actually scheduled or waiting on a gate.

| Story | Epic | Feature | Points | Persona | Status | Gate |
|-------|------|---------|--------|---------|--------|------|
| US-001 | EP-03 | EP-03-F01 | 13 | Security engineer | To Do | Gate 1 |
| US-002 | EP-02 | EP-02-F02 | 5 | Security engineer | To Do | Gate 1 |
| US-003 | EP-02 | EP-02-F02 | 8 | Security engineer | To Do | Gate 1 |
| US-004 | EP-06 | EP-06-F01 | 8 | Security engineer | To Do | Gate 1 |
| US-005 | EP-06 | EP-06-F02 | 5 | Security engineer | Deferred | Gate 2 |
| US-006 | EP-07 | EP-07-F01 | 5 | CISO / auditor | To Do | Gate 1 |
| US-007 | EP-02 | EP-02-F03 | 8 | Security engineer | To Do | Gate 1 |
| US-008 | EP-05 | EP-05-F02 | 13 | Security engineer | To Do | Gate 1 |
| US-009 | EP-10 | EP-10-F01 | — | Security engineer | Deferred | Gate 2c |
| US-010 | EP-05 | EP-05-F01 | 8 | Security engineer | To Do | Gate 1 |
| US-011 | EP-05 | EP-05-F01 | 13 | Security engineer | To Do | Gate 1 |
| US-012 | EP-05 | EP-05-F01 | 5 | CISO | To Do | Gate 1 |
| US-013 | EP-02 | EP-02-F01 | 13 | Tenant admin | To Do | Gate 1 |
| US-014 | EP-01 | EP-01-F01 | 13 | Tenant admin | To Do | Gate 1 |
| US-015 | EP-05 | EP-05-F03 | 2 | Tenant admin | Deferred | Gate 2 |
| US-016 | EP-06 | EP-06-F01 | 5 | Security engineer | To Do | Gate 1 |
| US-017 | EP-03 | EP-03-F02 | 13 | Security engineer | To Do | Gate 1 |
| US-018 | EP-06 | EP-06-F03 | 8 | Security engineer | To Do | Gate 1 |
| US-019 | EP-06 | EP-06-F04 | — | Security engineer | Deferred | Gate 3 |
| US-020 | EP-02 | EP-02-F04 | — | Tenant admin | Deferred | Gate 5 |
| US-021 | EP-04 | EP-04-F01 | 8 | Security engineer | To Do | Gate 1 |
| US-022 | EP-08 | EP-08-F02 | — | API consumer | Deferred | Seed |
| US-023 | EP-09 | EP-09-F01 | 3 | Security engineer | Deferred | Gate 2 |
| US-024 | EP-08 | EP-08-F02 | — | API consumer | Deferred | Seed |
| US-025 | EP-03 | EP-03-F03 | — | Security engineer | Deferred | post-Gate 3 |
| US-026 | EP-08 | EP-08-F03 | — | Platform partner | Deferred | No trigger |
| US-027 | EP-02 | EP-02-F05 | 8 | Security engineer | To Do | Gate 2 candidate |
| US-028 | EP-03 | EP-03-F04 | — | Security engineer | Spec authored | Gate 2 |

## Sequencing: what actually blocks what

None of the epic-by-epic order above is arbitrary — it follows a real dependency chain, and eight links in that chain are the ones that would actually stall the schedule if missed.

| Dependency | Blocks | Week |
|------------|--------|------|
| EP-01 RLS + isolation harness | EP-02 (tenant-scoped writes), EP-03 (assessments) | Week 4 |
| EP-01-F02-T02f self-hosted Temporal | EP-03-F01 (orchestration) | Week 4 |
| EP-02 F01 Connector Hub | EP-02 F02 vendor connectors, EP-04 (delta events) | Week 5 |
| EP-03 F01 assessment workflow | EP-04 (re-assessment), EP-05 F01 (dashboards), EP-05 F02 (chat) | Week 6 |
| EP-01-F02-T11 Firecracker/Kata | EP-03-F02 (sandbox execution) | Week 6 |
| EP-07 F02 governance kernel + HITL | EP-06 F01 (action cards), EP-06 F03 (remediation tickets) | Week 8 |
| EP-02 CrowdStrike/Wiz connectors | EP-06-F02 control refinements (deferred) | Week 7 |
| EP-05 F02 chat spike (Week 4) | EP-05 F02 chat writes (Week 14) | Week 4 → 14 |

## Where the plan breaks its own rules — on purpose

A strict Epic/Feature/Story framework would fail this backlog in seven places, and rather than paper over the gaps, the team documented every one of them as a deliberate exception with a rationale attached.

| # | Rule | Deviation | Rationale |
|---|------|-----------|-----------|
| D-EX-1 | Feature must have 3–12 stories | Some carry 1–2 | US-001–024 is a closed canonical set. Inventing stories to hit quota breaks corpus authority |
| D-EX-2 | Epic decomposes into 2–8 features | EP-04, EP-09, EP-10 have 1 | Single-FR epics; splitting would be artificial |
| D-EX-3 | Every Task parents a Story | Enabler tasks parent their Feature (`EP-xx-Fyy-Tzz`) | Corpus has no user story for platform enablers — synthetic stories were rejected |
| D-EX-4 | Assignee is a named person | Role placeholders: TS-1..3, PY-1..2, SEC, PM, CTO | Team not yet named; replace at sprint planning |
| D-EX-5 | Story status "Ready" | Everything ships as Defined / To Do at t0 | Nothing groomed by real team yet |
| D-EX-6 | Feature must have ACs, dependencies, API boundary, SLA | Deferred/draft features carry only Parent / FR / Story-in-prose | No scheduled build target — specifying ACs/SLA would fake precision |
| D-EX-7 | Story DoD = merge gates + verification + staging + demo | Per-story DoD states only merge gates + verification | Full four-part bar enforced at Epic/Sprint level, not repeated across 40+ story lines |

## The rules that hold the whole thing together

None of the above works without automated enforcement. `scripts/validate-playbooks.py` runs on every change and checks six invariants:

1. **No orphans** — every Feature has a parent Epic, every Story a Feature, every Task a Story or Feature (D-EX-3)
2. **No widows** — every Epic has ≥1 Feature; every non-deferred Feature has ≥1 Story or enabler task
3. **ID uniqueness** — portfolio-wide
4. **Bidirectional traversal** — via [[Dux Decisions & Traceability Reference]]
5. **Status cascade** — parent is Done only when every child is Done
6. **Tech-lead gate** — every task carries assignee, hours, discipline, type, and target week

## What could still go wrong

A 42-hour buffer is not a large margin for error, and the team's own risk register names exactly where the pressure points are.

| Risk | Impact | Mitigation |
|------|--------|------------|
| Agentic RAG + Apache AGE unestimated | Unknown hours, not covered by D-40 buffer | Tracked as open item (OI-39); requires own funding decision before adding to table |
| SEC single point of failure | Kernel + MCP + identity all owned by one role | Named before Gate 1; consider splitting if on-call conflicts arise |
| TS-1/PY-1 critical-path load Weeks 3–8 | Both backend leads fully loaded during core build | Task-week scheduling into freed Wk 9–12 window at sprint planning |
| Self-hosted K8s operational complexity | 5 services to operate vs managed cloud tiers | Firecracker/Kata pulled forward (D-33); observability stack (28h) for SLO burn-rate alerts |
| Vendor sandbox accounts | Needed Week 3 for CrowdStrike/Wiz/ServiceNow | PM owns procurement; escalation path through on-call |
| EU AI Act counsel opinion | Blocks EU provisioning if not delivered Week 6 | Legal/PM lane — 0 eng hours but hard external deadline |
| ZDR with OpenAI + Anthropic | Subprocessor-listing precondition, H2 deadline | Legal/PM lane — 0 eng hours, tracked in executive summary |

## What's deliberately left out of the Gate-1 sum

Not everything the team wants to build lives in the 2,118-hour total. A small Gate-2/fast-follow register (D-8) sits outside it on purpose:

| Item | Est. Hours | Notes |
|------|-----------|-------|
| H5 pre-approved-scope policy | ~12h | — |
| H8 connector-freshness + live contract tests | ~16h | — |
| H11 full SBOM/SLSA | ~8h | — |
| H6/H9 runbook/instrumentation | Low | Not separately called out |

## The takeaway

Strip away the table formatting and what this document really is becomes clear: a record of a team repeatedly choosing honesty over comfort. When the numbers didn't fit the calendar, they raised the calendar rather than quietly trim scope three times running. When a framework rule didn't fit reality, they wrote down why they broke it instead of hiding the break. When an arithmetic error crept into a 2,000-plus-hour rollup, a script caught it and the record was corrected in the open. That's what a 42-hour buffer against 2,118 hours of committed work actually buys a team at Week 0: not safety margin, but the discipline to know exactly how thin the margin is.

## Sources

- `.raw/dux/90-execution/README.md`
- `.raw/dux/90-execution/traceability.md`
- `.raw/dux/90-execution/backlog-ep01.md` through `backlog-ep10.md`
