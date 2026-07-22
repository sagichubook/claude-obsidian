---
type: resource
title: "Dux Decisions & Traceability Reference"
topic: "dux/governance"
created: 2026-07-22
updated: 2026-07-22
tags: [resource, dux, dux/governance]
status: mature
sources: [".raw/dux/00-meta/decisions-log.md", ".raw/dux/00-meta/traceability-matrix.md", ".raw/dux/00-meta/open-items.md", ".raw/dux/00-meta/quick-reference.md"]
related: ["[[Dux]]", "[[Dux Architecture Guide]]", "[[Dux Governance & Compliance Guide]]"]
---

# Dux Decisions & Traceability Reference

Navigation: [[Dux]] | [[Dux Architecture Guide]] | [[Dux Governance & Compliance Guide]]

This page is the audit trail: every decision that shaped the product, every open item still unresolved, the quick-reference governance facts, and the join table connecting business requirements down to individual verification commands. It's reference material by design — dense and lookup-oriented rather than narrative, because that's what an audit trail is actually for. Specs elsewhere state current truth in the present tense with no change history; this is the deliberate, sole exception, and it's the most cross-referenced page in the entire corpus.

---

# Part 1 — Decisions Log

**Purpose:** the history home. Every decision that shaped this corpus is recorded here, with its date and its rationale — and it is recorded *only* here.

**Started:** 2026-07-05 · **Participants:** Founder (Sagi) + the docs sessions.

These decisions bind the `docs/` tree. The original playbooks live in git history as the historical record (moved to `archive/` in `ed146a3`, removed from the working tree in `84fec31`).

> **This file is the exception to D-12.** Specs state current truth in the present tense and carry no change history — **this log carries all of it.** If you want to know what a document used to say, and why it changed, the answer is here, or in `git log`. **It is not in the spec.**

## Process decisions

| # | Decision |
|---|----------|
| P-1 | Scope: all four playbooks rewritten into `docs/`; originals kept (zero-data-loss via preservation + coverage verification) |
| P-2 | Canon: **Gap Closure & Ideal-State v2.2** (authority precedence #0) is product truth. Promoted Gate-1 scope applied everywhere; superseded statements preserved as history notes, never silently dropped |
| P-3 | Audience: engineers building Phase 1 first; PM/compliance content preserved as supporting reference |
| P-4 | Structure: numbered domain folders (00-meta … 80-gtm) mirroring the product's feature structure |
| P-5 | Contradictions resolved by the corpus's own authority hierarchy (GCIS → OpenAPI 3.1 for `/v1/*` → BR→FR→US → Claims map → agent registry); scope/audience questions asked, judgment calls documented |

## Foundational tech-stack decisions (2026-07-05)

| # | Area | Decision | Rationale |
|---|------|----------|-----------|
| D-1 | Sandbox vendor | **E2B default, Modal fallback** behind `SandboxPort`; Week-2 checklist covers code-residency review, latency/cost benchmarks, SOC 2 posture | E2B purpose-built for agent code execution; port keeps the swap cheap |
| D-2 | Temporal tenancy | Temporal Cloud, **single namespace per environment** (dev/staging/prod), tenant-scoped task queues (`assessment-{tenant_id}`); namespace-per-tenant deferred to a scale-out TBD box | Matches ADR-007 R2 child-workflow-per-tenant blast-radius model without namespace sprawl |
| D-3 | Cost gates | Re-derived proportionally to the R2 envelope: **soft circuit breaker $0.675** (10% under $0.75 hard), **CI cost gate blocks >$0.55 staging avg** (design target), **Gate 1 criterion <$0.75/workflow**; $0.28–0.32 retained as stretch-only | Preserves original intent (breaker 10% under ceiling; CI at design target) at the honest baseline |
| D-4 | HITL timing | **Approve/deny surface is a Gate-1 blocker for write actions**: API approval Week 8 → minimal approve/deny UI by Gate 1 review (Week 12 under D-7 R1) → full chat HITL UI Week 14 (`chat_write_tools`). Write actions do not enable before their approval surface exists. *(Superseded in part 2026-07-13: the surface no longer gates every write — it serves the anomaly-escalation path; timing unchanged.)* | ADR-012 R2 ships HITL writes at launch; an approval path must launch with them (resolves BS-16); weeks shifted with the D-7 re-baseline |
| D-5 | Secrets | **AWS SSM Parameter Store** on ECS Fargate from Gate 1; Vault optional later (transit for OAuth refresh tokens may stay per ADR-011) | Follows the ECS promotion; native IAM integration |
| D-6 | Rate limits | Two explicit planes: **application API** 1,000 / 5,000 / 10,000 req/min (Starter/Pro/Enterprise) and **public data API** 60 / 300 / negotiated req/min | Resolves BS-25; both tables were plane-specific, never reconciled |
| D-7 | Plan re-baseline | *(Original 2026-07-05)* the 5-engineer × 14-week calendar was sized for Analyze-only; re-baseline required at build start. **R1 (RESOLVED 2026-07-09): Gate 1 moves to Week 12, Phase-1 exit to Week 16 (16-week / 2,000 h envelope).** | GCIS promoted scope without re-baselining effort (blind spot A-4); quantified by the [execution backlog](../90-execution/traceability.md) |
| D-8 | Adversarial-audit disposition (H1–H11, 2026-07-09) | 🔴 **H1–H4 fold into Gate-1 scope now** (+42 eng h) → Gate-1 backlog ≈**1,901 h (95% of 2,000 h, ~99 h buffer)**. 🟠 H5–H9 = Gate-2 hardening register. 🟡 H10–H11 = fast-follows | seam audit (marketing↔product↔engineering); the four 🔴s are enterprise-review blockers |
| D-9 | Sandbox budget & partial-failure semantics | Per-tenant **300 sandbox-seconds/hr + 5 concurrent microVMs**, enforced in the governance kernel (`budget_exceeded` → L2); partial-failure semantics BLOCKED → no retry / TIMEOUT → 1 retry then HITL T3 / OOM → no retry + HITL T3 | closes TBD E-3 (uncapped-execution hole) |

## Update 2026-07-09 — canonical-v3 absorption

The ClickUp knowledge export (canonical v3 corpus + deep-research validations + adversarial audit + claim-safe patch set) was absorbed into this tree. Claim-safe wording applied; **Gartner sourcing reversal** — the Mar-2026 Gartner (Nunez) 2028/70% quote is confirmed primary research; EU AI Act status updated to Digital-Omnibus-Council-adopted/OJ-pending; 2026 citations added corpus-wide.

## Update 2026-07-11 — competitor-scan promotion

A hypothetical/unverified competitor spec ("Dux Plus") was scanned for feature insights. 32 items checked, 16 real findings surfaced (5 claims-critical, 3 medium, 8 opportunistic). Sagi reviewed and accepted all 16, promoting them into the real backlog:

- **Backlog additions:** `US-002-T05`/`US-001-T12` (asset criticality/environment fields, EP-02/EP-03), `US-018-T06` (severity-tiered remediation routing, EP-06), `US-004-T07` (IAM/segmentation compensating controls, EP-06), `US-007-T06` (broader auto-tagging, EP-02). New Stories: `US-025` Outcome Learning (EP-03-F03, deferred post-Gate-3), `US-026` Outbound MCP Server (EP-08-F03, deferred), `US-027` Proactive Tool Discovery (EP-02-F05, Gate 2 candidate). New FRs: `FR-027`/`FR-028`/`FR-029`.
- **GTM copy:** `gtm-guardrails.md` qualifier tightened; `competitive.md` gets a forward-looking note about the DeepEval + 250-CVE golden-set eval harness.
- **Explicitly not done:** no "Dux Plus" row added to `competitive.md` (not a real competitor).

## Update 2026-07-12 — full BR→Epic→US→Task traceability audit

Findings and fixes: matrix bugs fixed (BR-009, BR-010, US-003 reference gaps); hour-math bug found and fixed at its root; 4 features documented as already-live with zero backing engineering task added as real scheduled tasks (+52h); corrected grand total: 2,073h against the 2,000h envelope — a 73h overrun; 3 stale traceability ranges fixed; file-removal check: nothing removable.

## Update 2026-07-12 (later session) — closed audit files removed

Six point-in-time audit/checklist artifacts deleted. `stack-review-2026-07.md` kept (19 findings still awaiting disposition). Two defects fixed: `backlog-ep03.md` footer total corrected; stale `public-data-api.md` line fixed.

## Update 2026-07-12 (third session) — multi-expert docs review pass

Five-reviewer panel over the full `docs/` tree. Resolutions: US-004/US-006 dual-spec ownership resolved; stale Railway references fixed; `architecture-overview.md` infra-tree comment corrected; dangling M-10 reference fixed; `validate-playbooks.py` tracked in git; panel-discovered corrections applied (D-4 timing drift, etc.).

## Update 2026-07-13 — H5/unattended-write escalation resolved

**HITL requirement lifted for Gate-1 launch.** Vendor write actions execute by default at Gate 1 without waiting for human approval. Compensating controls stay fully intact: kill switch (KS-L1–L4, <5 s propagation), governance-kernel budget/effect/DLP gates, least-privilege scoped action credentials, hash-chained audit, blast-radius tiering. HITL retained only as an **anomaly-escalation path** (low-confidence verdicts, sandbox TIMEOUT/OOM per D-9, extreme-blast-radius outliers).

## Update 2026-07-13 (second session) — re-gating propagation + claims-alignment

Two outcomes: (1) **Claims-alignment directive (Sagi): live marketing claims bind the corpus 100%.** (2) **Re-gating propagation completed.** The first pass missed the authority and execution layers — fixed across 20+ files.

## Documentation discipline decisions (2026-07-14)

| # | Decision | Rationale |
|---|----------|-----------|
| D-10 | **The claims-alignment directive is narrowed.** The Marketing Claims map binds GTM copy, product naming, and UI strings. It does not bind safety posture, control design, gate criteria, or SLOs. | an engineering document must be able to state what the system actually does |
| D-11 | **Open questions leave prose and enter a register.** `open-items.md` is the single list (`OI-##`), each with an owner, a severity, and what it blocks; the originating ID is preserved in an Origin column | 39 TBD boxes with no owner had turned the corpus into an issue tracker with no assignee column |
| D-12 | **Change history leaves prose and stays in this file.** Specs state current truth in the present tense. This log plus git history is the record | a reader had to reconstruct the current state by mentally applying a chain of dated diffs |

## Update 2026-07-14 (second session) — Bedrock inference path; write-path control gaps closed

| # | Decision | Rationale |
|---|----------|-----------|
| D-13 | **AWS Bedrock is the Claude/Anthropic inference path; the Anthropic direct API is retired.** | AWS is already the primary cloud and IAM/network trust boundary |
| D-14 | **`VendorActionGate` specified as GOV-014, with a `GOV-TOOL-*` risk matrix.** Resolves OI-01. | the write path shipped unattended by default on the premise this gate would make it safe |
| D-15 | **A rollback catalog is authored for the 5 canonical write actions** (R-01…R-05). Resolves OI-03. | `rollbackProcedure` was a required field pointing at nothing |
| D-16 | **Temporal Cloud transport/payload security specified:** mTLS client certs via AWS SSM, per-tenant-DEK `PayloadCodec`. | ADR-007 R2 left this as "still to be specified at build" |

## Update 2026-07-15 — architecture-panel recommendations disposed

| # | Decision | Rationale |
|---|----------|-----------|
| D-17 | **S1 applied: earned per-action-class write autonomy.** `ticket.create_remediation`, `network.blocklist_add`, `policy.deploy_device_config` stay unattended by default. `endpoint.isolate` and `patch.deploy_special_devices` require mandatory HITL on every call. | The panel's central finding: the 2026-07-13 unattended-by-default posture is out of step with the 2026 market's tiered-autonomy consensus |
| D-18 | **OI-04 resolved: tenant offboarding soft-deletes at day 0.** `multi-tenancy.md` §5 confirmed as authority. | Three-way GDPR/DPA-relevant conflict resolved |
| D-19 | **OI-05 resolved: documented fallback lever applied.** EP-09 (30h), US-015 (11h), US-005 (30h) deferred to Gate-2. Gate-1 total: 2,002h against 2,000h envelope (~100.1%). | Closest available without further scope cut or envelope re-baseline |

## Update 2026-07-16 — stack-review quick wins applied

SR-01 (cache scope), SR-06 (PgBouncer/Neon), SR-08 (Temporal harness citation), SR-09 (model pins), SR-10 (OTel GenAI semconv), SR-15 (Temporal versioning wording) — all applied.

## Update 2026-07-16 (second session) — full audit-remediation pass

| # | Decision | Rationale |
|---|----------|-----------|
| D-20 | **EU AI Act Annex III: Dux does not trigger Annex III.** Dux is a security tool used *by* operators, not infrastructure or access-control itself. | An EU AI Act compliance program was being built without ever stating what triggered it |
| D-21 | **ASI02/ASI09 re-scored (resolves OI-28).** ASI02: Implemented/Low → Implemented/Low-Medium. ASI09: Partial/Medium → Partial/Medium-High. | D-17's mixed-autonomy posture invalidated the original rating's premise |
| D-22 | **OI-06 reconciled.** `checkCostCap` is one mechanism, not two. The 40–80 action-budget range kept; p95 ≈ 58, consistent with sold p95<60 KPI. | The contradiction had already been sold as a customer-facing SLO |

Also: OI-02 closed (MCP write-tool catalog authored); DA/DF/panel findings closed; Quick Reference Card authored.

## Update 2026-07-16 (third session) — engineering-owned open-items pass: 16 items closed

OI-07, 08, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 29 closed. Key decisions:

- **OI-17: AWS CDK (TypeScript)** for ECS deploy tooling
- **OI-12: per-tier re-assessment rate caps** (Design Partner 50/hr, Starter 200/hr, Professional 2,000/hr, Enterprise 10,000/hr)
- **OI-13/SR-11: burst-tier degradation model**
- **OI-22/D-2: namespace-per-tenant trigger** — 15,000 total active task queues (75% of Temporal Cloud ceiling)
- **OI-29: Gate-3 remediation-orchestration evaluation framework** — LiteLLM native A2A as default candidate
- **OI-24: `engineering-standards.md` authored**
- **OI-20: `DuxOnboardingHealthCheckMissed` alert added**

## Update 2026-07-16 (fourth session) — SR-02, SR-17 closed

Better Auth security review (3 findings closed) and vendor corporate-event currency verified. OI-25 fully closed.

## Update 2026-07-16 (fifth session) — OI-32 resolved

| # | Decision | Rationale |
|---|----------|-----------|
| D-23 | **Gate-1 envelope re-baselined from 2,000h to 2,080h** (25 → 26 focused h/week). 5 engineers, no sixth hire. | Every deferrable item was already deferred; the 38h CI-07 is claims-binding work |

## Update 2026-07-16 (sixth session) — OI-09 resolved

| # | Decision | Rationale |
|---|----------|-----------|
| D-24 | **E4 specced as evidence-trend computation, not ML model or composite score.** BR-013 → EP-03 (F04) → US-028 → FR-030, Gate 2. | A trend over real evidence answers "which assets are likely to become risky" literally, without a black box |

## Update 2026-07-16 (seventh session) — MC-11 sourced, MC-14 drafted

LinkedIn RSA-2026 "65-day MTTR" claim sourced (Edgescan Vulnerability Statistics Report). "How it works" section drafted in `gtm-guardrails.md` §8.

## Update 2026-07-16 (eighth session) — OI-10, OI-11 closed

| # | Decision | Rationale |
|---|----------|-----------|
| D-25 | **Gartner quote internal-use only** until reprint licence secured. Resolves OI-10. | No licence exists |
| — | **CybersecTools CSF percentages confirmed third-party fabrication.** Resolves OI-11. | Never Dux-supplied |

## Update 2026-07-17 — drafted deliverables not requiring live-system access

A1 and C1/C2 (OI-27) specced. OI-26 external actions drafted in `external-corrections-2026-07.md`.

## Update 2026-07-17 (second session) — OI-26, OI-27 closed

Both closed as Founder calls. Corpus/spec-side work complete; remaining items are pure execution tracked outside the register.

## External best-practice review adopted (2026-07-18)

| # | Decision | Rationale |
|---|----------|-----------|
| D-26 | **EU AI Act deadline corrected.** Annex III high-risk obligations: 2 Dec 2027. Art. 50 transparency: 2 Aug 2026. | Verified external fact correction |
| D-27 | **LLM-gateway primary slot goes to Gate-2 isolation bake-off (ADR-010 R3).** LiteLLM remains Phase-1 proxy; Bifrost and Portkey promoted to required Gate-2 bake-off. | LiteLLM's cross-tenant semantic-cache defect and PyPI backdoor are a poor fit |
| D-28 | **Self-hosted Firecracker sandbox pulled forward from Gate-5 to Gate-2/3.** | E2B sees the customer's environment being probed — a heavier subprocessor |
| D-29 | **Frontend: headless-adopt (ADR-018).** React Aria Components for data-dense surfaces; Radix/shadcn elsewhere. | Headless owns visual identity and deepest accessibility for grid-heavy surfaces |
| D-30 | **Dataviz: headless charts + SVG (ADR-019).** Visx for donut/trend/distributions; custom SVG for attack path. | Headless keeps charts on same token system and accessibility discipline |
| D-31 | **Post-Phase-1 retrieval: stay on Postgres** (pgvector + pgvectorscale) until ~100M vectors. Hybrid search + cross-encoder reranker. | One RLS-enforced store keeps the tenancy story unified |

## Update 2026-07-18 (second session) — external TDD re-evaluation

| # | Decision | Rationale |
|---|----------|-----------|
| D-32 | **Canon holds on every conflicting axis; external TDD's "confirmed" stack not adopted.** No ADR or safety-spine change. | Both TDDs are outsider reconstructions; each divergence is either premature-scaling or positioning-breaking |

## Update 2026-07-19 — full stack replacement

| # | Decision | Rationale |
|---|----------|-----------|
| D-33 | **Full stack replacement to self-hosted Kubernetes.** ECS Fargate → K8s (DO/Linode LKE); CDK → Pulumi; Neon → CloudNativePG; Upstash → NATS + Valkey; Temporal Cloud → self-hosted; E2B → self-hosted Firecracker (Gate-1 default); AWS SSM → Vault. ADR-017 reversed (Bedrock-only → multi-provider). OI-36 resolved (Better Auth + Unleash + Grafana LGTM kept). | Finance/healthcare ICPs demand portability, auditability, no single-vendor dependency |

## Update 2026-07-19 (third session) — v4.0 unified architecture adopted

| # | Decision | Rationale |
|---|----------|-----------|
| D-34 | **Hosting reverts to EKS.** LiteLLM removed entirely → direct Bedrock SDK behind `LLMProviderPort`. Agentic RAG re-enabled with constrained decoding. Apache AGE added as graph layer. vLLM + Phi-4 S-LLM path at Gate-2. | v4.0 is the new locked source of truth; AWS GovCloud/FedRAMP outweighs single-vendor concern |

## Update 2026-07-19 (fourth session) — Mastra/LangGraph.js removed

| # | Decision | Rationale |
|---|----------|-----------|
| D-35 | **Remove Mastra and LangGraph.js entirely.** Agent reasoning loop is Temporal workflow calling Bedrock Converse API directly. Retires Gate-2 vLLM+Phi-4 S-LLM path. | Both frameworks are abstraction layers over capabilities Temporal and Bedrock Converse already own |

## Update 2026-07-19 (fifth session) — strict end-to-end review

Validator restored. Hour-reconciliation bugs fixed (EP-01 386→370h, EP-03 430→426h, EP-05 338→322h, grand total 2,138→2,118h). FR-ID for Agentic RAG split into OI-44. OI-45 raised (traceability.md present-tense exception). Full 65-file corpus read completed.

## Update 2026-07-20 (second session) — Founder closes eight open items

| # | Decision | Rationale |
|---|----------|-----------|
| D-36 | **OI-33 closed: E4/US-028 funded at Gate-2.** | Already a public roadmap anchor |
| D-37 | **OI-34 closed: Airbyte not adopted.** | One ingestion path, one operational model |
| D-38 | **OI-35 closed: per-tenant DB isolation ships as Enterprise-tier add-on**, gated on enterprise-RFP trigger. | Ties build trigger to real procurement signal |
| D-39 | **OI-38 closed: FedRAMP stays gated on untriggered federal-RFP condition.** | Too large a bet against an untriggered condition |
| D-40 | **OI-39 closed: Gate-1 envelope raised to 2,160h** (26 → 27 focused h/week). 2,118h backlog at ~98.1% (42h buffer). Fourth consecutive raise explicitly rejected as precedent. | Known 38h gap only; buffer not for unestimated scope |
| D-41 | **OI-44 closed: `FR-031` assigned to Agentic RAG + Apache AGE.** | Next available uncollided number |
| D-42 | **OI-45 closed: `traceability.md` added as third present-tense exception.** | Running capacity ledger needs visible history |
| D-43 | **OI-46 closed: Apache AGE supersedes Neo4j** as first-line graph-latency migration trigger. | AGE fulfills the NFR-004 3-hop-CTE need without leaving Postgres |

## Update 2026-07-21 — best-practice review + web-research refresh

140 sourced facts gathered. 10/10 web-research discovery angles completed. Product/API folders reviewed. 12 mechanical fixes applied. 5 new open items registered (OI-50 through OI-54).

## Update 2026-07-21 (second session) — remaining folders reviewed

40-ai-safety, 20-architecture, 60-operations, 70-governance, 80-gtm, 90-execution, 00-meta-consistency reviewed. 3 more open items (OI-55, OI-56, OI-57). Vision-reference §7 updated with headcount (24), leadership hire, and Israel Security Prize detail.

## Update 2026-07-21 (third session) — substantive facts written

`vision-reference.md` §6 gains V-14 (unearned certification badges, OI-57 P0). `compliance-program.md` §4 gains Privacy Policy disclosure paragraph. `competitive.md` §1 gains corroborating CVE-exploit stat.

## Update 2026-07-21 (fourth session) — Founder works the register live

| # | Decision | Rationale |
|---|----------|-----------|
| D-44 | **OI-57 closed: dux.io footer badges confirmed unearned.** Compliance-program.md is correct. Live-site fix tracked externally. | Documented programme status is the true one |
| D-45 | **OI-51 closed: bare `admin`/`member`/`viewer` canonical everywhere.** Full per-endpoint role matrix added to api-overview.md §3. | Sagi's call on each ambiguous tier |
| D-46 | **OI-48 closed: `agt_` is the Public Data API credential.** agent-identity.md corrected. | Weight of evidence from 4 files + OpenAPI security scheme |
| D-47 | **OI-50 closed: "MVP connector set" table wins**, superseded table removed. New narrower gap: OI-58 (Wiz/Intune cadence). | Surviving table is vendor-specific and rate-limit-grounded |
| D-48 | **OI-52 closed: Gartner quote downgraded to unconfirmed**, pulled from active sales use. | Sagi could not confirm anyone holds the primary document |
| D-49 | **OI-56 closed: LinkedIn RSA-2026 post confirmed real.** URL + wording added. Two adjacent false claims recorded as standing guardrail. | Founder located the post directly |
| D-50 | **OI-55 closed: `crowdstrike` named as Gate-1 executor for `network.blocklist_add.** Via IOC/indicator blocklisting. | CrowdStrike already a Gate-1-floor, action-capable connector |
| D-51 | **OI-54 closed: "Dux, Inc." confirmed correct legal entity.** Privacy Policy PDF is the error; external follow-up. | Sagi's call as the signing entity |
| D-52 | **OI-49 closed: series-b-scale.md Ready row corrected.** | Mechanical fix |
| D-53 | **OI-53 closed: all three stats corrected with honest attribution.** | Keep real numbers with honest attribution rather than discard |
| D-54 | **OI-37 closed: all 33 roadmap vendors assigned a real `role`**, grouped into 7 rows. | Categorized from known real-world product function |
| D-55 | **OI-41 mostly closed: embedding-signing spec written, LLM04/LLM08 reassessment closed.** New narrower item: OI-59 (live HNSW test). | Reuse existing hash-based integrity pattern |
| D-56 | **OI-42 closed: both "missing" endpoints already exist under other names.** Cross-references added. | Naming/cross-reference gap, not missing product surface |
| D-57 | **OI-47 closed: local-development.md §4a first-run setup confirmed accurate.** | Sagi confirmed matches team's real workflow |

## Footer conventions

1. Gates cited are ideal-state canon; where a gate moved, the doc says "promoted from X (GCIS §B)" once, in place.
2. Model IDs and prices are **pins dated 2026-06** — the mechanism (`models.json` + AI-BOM + CI pin gate) is the durable fact.
3. Later-stage content states deltas; canonical definitions live in exactly one file, linked elsewhere.
4. Every feature spec carries its BR/Epic/US/FR IDs from the [traceability matrix](./traceability-matrix.md).
5. **Open engineering questions live in [open-items.md](open-items.md), never as `> **TBD**` boxes inside a spec** (D-11).
6. **Specs state current truth in the present tense** (D-12). Change history belongs in this log.
7. **Every file carries front-matter:** `owner`, `status`, `gate`, `last_reviewed`.

---

# Part 2 — Open Items Register

**Purpose:** the single list of every unresolved question in the corpus. Specs state what is true today; anything still undecided lives here with an ID, an owner, and the thing it blocks.

## How this register works

1. **One ID scheme.** Every open question is `OI-##`. The original ID is kept in the Origin column, never dropped.
2. **A spec never carries an open question in its body.** It states current truth and links here. The exception is `decisions-log.md`.
3. **Nothing here is resolved by editing prose.** An item closes when its owner makes the call; the decision is recorded in [decisions-log.md](decisions-log.md) and the affected spec is updated to state it as fact.
4. **Severity is about consequence, not effort.**
   - **P0** — blocks a Gate-1 ship, or is a live legal/compliance exposure.
   - **P1** — blocks a plan, a date, or an external commitment already made.
   - **P2** — spec debt: real, but nothing is currently unsafe or over-promised because of it.

## P0 — blocking

**None open.**

*(OI-36 closed 2026-07-19; OI-02 closed 2026-07-16; OI-57 closed 2026-07-21 via D-44; OI-51 closed 2026-07-21 via D-45.)*

## P1 — blocks a plan, date, or live commitment

**None open.**

*(OI-33 closed 2026-07-20 via D-36; OI-39 closed 2026-07-20 via D-40; OI-07–14, OI-32 closed 2026-07-16; OI-48 closed 2026-07-21 via D-46; OI-50 closed 2026-07-21 via D-47; OI-52 closed 2026-07-21 via D-48; OI-56 closed 2026-07-21 via D-49; OI-55 closed 2026-07-21 via D-50; OI-54 closed 2026-07-21 via D-51.)*

## P2 — spec debt

| ID | Item | Origin | Blocks | Owner |
|----|------|--------|--------|-------|
| **OI-40** | **Temporal namespace-per-tenant trigger needs re-deriving for self-hosted.** The prior 15,000-active-task-queue trigger was keyed to Temporal Cloud's ~20K-poller-per-namespace SaaS ceiling. Self-hosted Temporal's scaling ceiling depends on cluster resource allocation — the numeric trigger needs re-deriving against real self-hosted capacity planning. No Phase-1 impact (single-namespace model unaffected). **Recommendation:** load-test actual matching/history/frontend pod allocation against synthetic workflow-task volume, find the poller-per-task-queue count where p99 schedule-to-start latency degrades, apply ~75% safety margin. | D-33, D-34 | Namespace-per-tenant migration planning | Engineering |
| **OI-59** | **Live execution of the ISO-012 ANN adversarial-neighbor test against the real, deployed tenant-scoped HNSW index has never been run.** Narrowed out of OI-41 (D-55). A docs-only repo cannot run a test against a live index. | opened closing OI-41, 2026-07-21 | Confirmation the tenant-scoped HNSW index resists the documented ANN-adversarial-neighbor attack in practice | Engineering / Security |
| **OI-43** | **Test-scenario checklist exists (ci-cd-testing §4); the test code does not.** All 7 scenarios enumerated: happy path, max-turns(10) exit, budget-exceeded ($0.75) abort, tool failure/retry, Bedrock timeout, malformed tool response, HITL signal timeout (30 min). Implementation lives in the separate application repo. | D-35 | Gate-1 test-coverage sign-off for `ExploitabilityAssessmentWorkflow` | Engineering |
| **OI-58** | **Wiz and Intune have no rate-limit-derived sync cadence**, unlike the other 8 MVP connectors. The surviving "MVP connector set" table (D-47) only covers Tenable, Qualys, CrowdStrike, AWS Security Hub, Rapid7, Splunk, Jira, ServiceNow, Okta, and Azure AD. | opened closing OI-50, 2026-07-21 | Wiz and Intune sync-cadence figures on the same rate-limit-research basis | Engineering |

## Standing review registries

*(none open — OI-26, OI-27 closed 2026-07-17.)*

## Closing an item

An item leaves this register only when its owner decides it. On close:

1. Record the decision in [decisions-log.md](decisions-log.md) with the date and the person who made the call.
2. Update the affected spec to state the outcome as present-tense fact.
3. Delete the row here.

---

# Part 3 — Quick Reference Card

A pure compilation of facts already documented elsewhere — no new claims, no new numbers. If a figure here disagrees with its source doc, the source doc wins.

## Write-action autonomy (5 canonical actions)

| Action | HITL posture (D-17) | Rollback |
|---|---|---|
| `endpoint.isolate` | mandatory HITL — every call | R-01 |
| `network.blocklist_add` | unattended by default, anomaly-only escalation | R-02 |
| `policy.deploy_device_config` | unattended by default, anomaly-only escalation | R-03 |
| `patch.deploy_special_devices` | mandatory HITL — every call | R-04 |
| `ticket.create_remediation` | unattended by default, anomaly-only escalation | R-05 |

See [[Dux Feature Reference]] for the write surfaces, [[Dux AI Safety Guide#Governance Kernel]], and D-17 in this document.

## Kill-switch levels

| Level | Scope | Propagation target |
|---|---|---|
| KS-L1 Session | single assessment workflow | ≤30 s |
| KS-L2 Tenant assessment | all sessions + queue, one tenant | p99 <5 s (KS-001) |
| KS-L3 Tenant platform | all tenant agent features, dashboard read-only | p99 <5 s |
| KS-L4 Global | entire platform agent fleet | p99 <5 s, two-person approval |

**Execution-interruption SLO:** p99 <30 s (one heartbeat interval).

See [[Dux AI Safety Guide#Kill Switch & HITL]].

## Governance gates (GOV-001–014)

**Chain:** `IntentGate → BudgetGate → EffectGate → VendorActionGate → HITLGate`

| Gate | Threshold |
|---|---|
| GOV-003 ActionBudget | warn >100, halt ≥200 weighted actions/assessment (LLM=1, MCP-read=2, MCP-write=5) |
| GOV-007 CostCap | $25/hour per-tenant hard cap |
| GOV-010 Loop | max 50 P-LLM iterations/assessment, max 10 MCP calls/iteration |
| GOV-014 VendorActionGate | write action blocked unless confidence floor met AND a rollback procedure is on file |

**Cost-threshold evaluation order** (first match wins): $0.675/assessment soft breaker → $25/hour CostCap → 2× rolling-baseline circuit breaker.

See [[Dux AI Safety Guide#Governance Kernel]].

## Confidence bands

| Band | Label |
|---|---|
| [0.85, 1.00] | `exploitable` |
| [0.70, 0.85) | `likely` |
| [0.40, 0.70) | `unlikely` |
| [0.00, 0.40) | `not_exploitable` |
| abstain | `insufficient_data` |

**Scoring mechanism:** 3-signal ensemble (logprob 0.40 / semantic entropy 0.35 / verbalized confidence 0.25) → Platt scaling. This is confidence-in-exploitability, not a composite CVSS×EPSS×criticality×exposure risk score.

See [[Dux Taxonomy & Catalogs#Confidence scoring methodology]], [[Dux AI Safety Guide#Confidence Calibration]].

## Compliance control-ID families

| Framework | Where mapped |
|---|---|
| SOC 2 (AICPA TSC: CC6/7/8, A1, Confidentiality, PI, Privacy) | compliance-program §2 |
| ISO/IEC 27001:2022 (internal IDs + Annex A numbers) | compliance-program §3 |
| ISO 42001 / EU AI Act Art. 9 | compliance-program §3 |
| NIST AI RMF + NIST CSF 2.0 | compliance-program §8 |
| CIS Controls v8 | compliance-program §8 |
| OWASP Agentic (ASI01–10) / LLM (LLM01–10) / MCP Top 10 / NHI Top 10 | owasp-assessments, compliance-program §5 |

See [[Dux Governance & Compliance Guide]].

## Gate-1 exit criteria (selected)

- Zero cross-tenant fuzz reads (`pnpm test:fuzz-tenant-id` merge-blocking).
- Zero Critical findings in the adversarial suite.
- RLS `FORCE` on every tenant-scoped table, CI-verified (`check-rls.sh`).

See [[Dux Feature Reference#Product Overview|the product overview]].

---

# Part 4 — Traceability Matrix

**Purpose:** the join table for the whole corpus. It binds BR ↔ Epic ↔ US ↔ FR ↔ NFR/TR-NFR ↔ Gate ↔ feature spec ↔ verification command.

## The chain

```
Vision anchor → BR (business requirement)
              → Epic (delivery theme; groups the corpus's FRs)
                → US (user story)
                  → FR / NFR → TR-NFR (technical spec)
```

Gates reflect the ideal-state canon (GCIS v2.2).

**Every level references its parent, and the feature specs in `10-product/` cite these IDs back.**

> **The references are bidirectional by design, and the validator does not check them.** A US dropped from a BR's story list, or a BR dropped from an epic's parent list, **silently breaks the chain** — that exact defect has been found here before, by hand. **Keep both directions in sync.**

**This file owns the closed US set.** Stories are defined here, and never invented in `90-execution/`.

## 1. Epics

| Epic | Name | FRs | Parent BRs |
|------|------|-----|-----------|
| EP-01 | Multi-tenant platform & auth | FR-001, FR-009 | BR-001 |
| EP-02 | Environmental data ingestion (connectors & feeds) | FR-002, FR-003, FR-015, FR-016, FR-020, FR-014, FR-029 | BR-004 |
| EP-03 | Exploitability assessment engine | FR-004, FR-017, FR-026, FR-027, FR-030 | BR-002, BR-007, BR-013 |
| EP-04 | Continuous re-assessment | FR-025 | BR-002 |
| EP-05 | Analyst surfaces & APIs (dashboards, drill-down, chat) | FR-006, FR-007 | BR-002, BR-006, BR-008, BR-010 |
| EP-06 | Mitigation & remediation write path (HITL) | FR-011, FR-012, FR-013, FR-019 | BR-002, BR-003 |
| EP-07 | Safety & governance (kill switch, kernel, audit) | FR-005, FR-008 | BR-003, BR-005, BR-009 |
| EP-08 | Programmatic platform (webhooks, public data API) | FR-010, FR-021, FR-022, FR-023, FR-028 | BR-006, BR-011 |
| EP-09 | Triage disposition (acknowledgment) | FR-024 | BR-012 |
| EP-10 | Personalization (preference learning) | FR-018 | BR-002 |

## 2. Business requirements → epics → user stories

| BR | Business requirement | Vision anchors | Epics | User stories | Gate | Verification |
|----|----------------------|----------------|-------|--------------|------|--------------|
| BR-001 | Zero cross-tenant data leakage for findings, assets, assessments, users, exports | D5; tenant trust | EP-01 | US-014 (+ every tenant-scoped view) | Gate 1 | `pnpm test:fuzz-tenant-id`; ISO-001–010; ISO-FUZZ-001–005 |
| BR-002 | Agentic exploitability analysis distinguishing truly exploitable from noise; evidence-backed prioritization, mitigation paths, remediation acceleration | A1–A9, B2–B7, C1–C7, D1–D3, E2 | EP-03, EP-04, EP-05, EP-06, EP-10 | US-001, US-003–005, US-008–012, US-016–017, US-021; US-018–019; US-025 (draft, deferred) | Gate 1 (Analyze + writes + continuous); Gate 2c (US-009); Gate 3 (US-019); post-Gate-3 (US-025) | Golden set; trace export; DeepEval CI |
| BR-003 | Emergency agent halt (kill switch) with governance-kernel enforcement; <5 s L2–L4 | B1, D4, D5 (primary) | EP-06, EP-07 | US-014 (visibility); all agent paths | Pre-launch | KS-001; `test:kill-switch`; `test:governance-kernel` |
| BR-004 | Multi-source environmental data ingestion (AWS, NVD/KEV/EPSS, CSV, vendor connectors) | B2 step 1, B6 | EP-02 | US-002, US-003, US-007, US-011, US-013; US-020 (draft); US-027 | Gate 1 (AWS + intel + ≥3 connectors); Gate 5 (FR-014); Gate 2 candidate (US-027) | Connector sync tests; CONN-001 |
| BR-005 | Tamper-evident hash-chained audit trail + export | C6 | EP-07 | US-006, US-014, US-017 | Gate 1 | `/audit/verify`; trace export |
| BR-006 | Programmatic access — REST, SSE chat, webhooks, rate limits | B2 step 4 | EP-05, EP-08 | US-008, US-014, US-015; US-026 (draft, deferred) | Gate 1; no near-term trigger (US-026) | Rate-limit + webhook HMAC tests |
| BR-007 | AI-BOM manifest accuracy (agent tools, models, MCP catalog) | D4 | EP-03 | US-008 | Gate 1 | `aibom-validate` CI |
| BR-008 | Platform availability SLOs for enterprise trust | C1–C2 | EP-05 | US-006, US-012 | Gate 1 | SLO burn-rate alerts |
| BR-009 | AI-BOM drift detection in CI — no undeclared agent capabilities | D4 | EP-07 (CI gate; no US) | — | Ongoing | Manifest drift merge gate |
| BR-010 | Design partner cohort (2+ NDA) for Phase 1 validation | E1 | EP-05 | US-010 | Gate 1 | CRM; ≥10 assessments in first 14 days |
| BR-011 | Tenant-defined custom metrics over World Model via DQL; public read API | B2 step 4 | EP-08 | US-022, US-024 | Seed | OpenAPI contract tests; DQL validation |
| BR-012 | Vulnerability-instance risk acknowledgment (create/expire/revoke) with audit | Triage outcomes | EP-09 | US-023 | Phase 1 write; Seed public read | Ack lifecycle tests; audit events |
| BR-013 | Predictive risk forecasting — surface assets trending toward higher risk via evidence-trend deltas (EPSS/finding-count/control-coverage), not a new ML model or composite score | E4 | EP-03 | US-028 | Gate 2 | Trend-computation tests against fixture deltas |

## 3. User story index (US → parent epic/BR → spec)

| US | Title (screen) | Epic | BR | FR | Gate (canon) | Feature spec |
|----|----------------|------|----|----|--------------|--------------|
| US-001 | Prerequisites Analysis | EP-03 | BR-002 | FR-004 | Gate 1 live | `security-stepper.md` |
| US-002 | Asset Context Evidence | EP-02 | BR-004 | FR-020 | **Gate 1** (read-only, promoted) | `security-stepper.md` |
| US-003 | Protection Breakdown | EP-02 | BR-002/004 | FR-002 | **Gate 1** (read-only, promoted) | `security-stepper.md` |
| US-004 | Action Cards | EP-06 | BR-002 | FR-011 | **Gate 1, unattended by default** | `mitigation-write-path.md` |
| US-005 | Control Refinements | EP-06 | BR-002 | FR-019 | **Gate 1** (promoted) | `security-stepper.md` |
| US-006 | Audit & Exposure Delta | EP-07 | BR-005/008 | FR-008 | Gate 1 live | `dashboard-audit.md` |
| US-007 | Ownership Inference | EP-02 | BR-004 | FR-016 | **Gate 1** (read-only, promoted) | `security-stepper.md` |
| US-008 | Chat Guidance | EP-05 | BR-002/006/007 | FR-007 | Gate 1 (read-only tools; write tools per HITL schedule) | `chat-guidance.md` |
| US-009 | Preference Learning | EP-10 | BR-002 | FR-018 | Gate 2c (not promoted — needs behavioral data) | `security-stepper.md` |
| US-010 | Research Dashboard (Mitigation nav) | EP-05 | BR-002/010 | FR-004/006 | Gate 1 live | `research-dashboard.md` |
| US-011 | Exposure Analysis (Exposure nav) | EP-05 | BR-002/004 | FR-004/006 | Gate 1 live | `exposure-analysis.md` |
| US-012 | Dashboard Home | EP-05 | BR-008 | FR-006 | Gate 1 live | `dashboard-audit.md` |
| US-013 | Connector Hub (Apps nav) | EP-02 | BR-004 | FR-002/003/015 | Gate 1 live (AWS P0; CSV P1; vendor connectors per ADR-011 R2) | `connector-hub.md` |
| US-014 | Tenant Settings | EP-01/07/08 | BR-001/003/005/006 | FR-001/005/009/010 | Gate 1 (users + agent policy P0; keys/webhooks P1) | `tenant-settings.md` |
| US-015 | Help & Support | EP-05 | BR-006 | — | Phase 1 exit (P1; not a Gate 1 blocker) | `tenant-settings.md` |
| US-016 | Fast Actions | EP-06 | BR-002 | FR-011 | **Gate 1, `POST /fast-actions` unattended by default** | `mitigation-write-path.md` |
| US-017 | Assessment Trace (+ execution results) | EP-03 | BR-002/005 | FR-004/026 | **Gate 1 incl. `execution_results`** (promoted) | `assessment-trace.md` |
| US-018 | Remediation Ticket Panel | EP-06 | BR-002 | FR-013 | **Gate 1 create+route, unattended by default**; closed-loop Gate 3 (US-019) | `mitigation-write-path.md` |
| US-019 | Mitigation Validation Panel (draft) | EP-06 | BR-002 | FR-012 | Gate 3 | `mitigation-write-path.md` |
| US-020 | Physical Residency Admin (draft) | EP-02 | BR-004 | FR-014 | Gate 5 (future/roadmap) | `connector-hub.md` |
| US-021 | Continuous Re-assessment Scheduler | EP-04 | BR-002 | FR-025 | **Gate 1** (promoted; ADR-016) | `continuous-assessment.md` |
| US-022 | Custom Metrics Configuration | EP-08 | BR-011 | FR-021 | Seed | `public-data-api.md` |
| US-023 | Acknowledge Vulnerability Instance | EP-09 | BR-012 | FR-024 | Phase 1 write; Seed public read | `research-dashboard.md` |
| US-024 | Vulnerability Instances by CVE | EP-08 | BR-011 | FR-022/023 | UI Gate 1 (`by_instance`); public API Seed | `public-data-api.md` |
| US-025 | Outcome Learning (draft) | EP-03 | BR-002 | FR-027 | Post-Gate-3 candidate (draft, deferred) | `backlog-ep03.md` §EP-03-F03 |
| US-026 | Outbound MCP Server (draft) | EP-08 | BR-006 | FR-028 | No near-term trigger (draft, deferred) | `backlog-ep08.md` §EP-08-F03 |
| US-027 | Proactive Tool Discovery | EP-02 | BR-004 | FR-029 | Gate 2 candidate | `backlog-ep02.md` §EP-02-F05 |
| US-028 | Asset Risk Trend Forecast | EP-03 | BR-013 | FR-030 | Gate 2 | `predictive-risk-forecasting.md` |

## 4. FR → implementation (component / verification / gate)

| FR | Component / ADR | Verification | Gate (canon) |
|----|------------------|--------------|--------------|
| FR-001 | ADR-001/002; Better Auth via `AuthPort`; multi-tenancy spec | `test:fuzz-tenant-id`; AUTH-001–005; ISO-001–010 | Gate 1 |
| FR-002 | ADR-004 `AWSConnector`; ADR-011 R2 vendor connectors | Connector sync integration tests; <48 h onboarding metric | Gate 1 |
| FR-003 | `NVDConnector` (delta ≤2 h, ≥6 s sleep), `CISAConnector` (6 h; 1 h Enterprise), `EPSSConnector` (daily) | CONN-001 staleness | Gate 1 |
| FR-004 | ADR-007 R3 `ExploitabilityAssessmentWorkflow` (self-hosted Temporal); CaMeL plane; `POST /research/queue` idempotent | Golden set; DeepEval; TR-NFR-005 | Gate 1 |
| FR-005 | `KillSwitchRelay`; kill-switch spec | KS-001–007; `test:kill-switch-idempotency` | Pre-launch |
| FR-006 | `ExposureProjection` / `ProtectionProjection` / `ActionCardProjection`; SSE streams | US-010 SSE <1 s; US-012 <5 s | Gate 1 |
| FR-007 | Chat SSE+POST; `chat_interface` flag; MCP gateway | US-008 phased delivery; `aibom-validate` | Gate 1 |
| FR-008 | `AuditVerificationService`; hash-chained partitions | `/audit/verify` | Gate 1 |
| FR-009 | `TenantExportWorkflow`; GDPR delete | TR-NFR-009 (<24 h) | Gate 1 (P1) |
| FR-010 | ADR-005 R2 `WebhookDeliveryWorker` (NATS JetStream-backed durable queue) | HMAC tests; DLQ replay | Gate 1 (P1) |
| FR-011 | `QuickMitigationWorkflow` via `VendorActionGate`; T2/T3 tiers (ADR-012 R3) | `test:vendor-action-hitl`; `test:governance-kernel` | **Gate 1, unattended by default**; closed-loop Gate 3 |
| FR-012 | `ClosedLoopValidationWorkflow` saga | `closed_loop_validation` flag | Gate 3 |
| FR-013 | `RemediationWorkflow` create+route (T1 tier; unattended); ServiceNow/Jira | `test:remediation-ticket-create`; audit event | **Gate 1 create+route**; closed loop Gate 3 |
| FR-014 | `dux-resident-agent` DaemonSet | `test:physical-resident-agent-isolation` | Gate 5 |
| FR-015 | `CSVConnector` (UTF-8; 50 K rows; unique tenant+hostname) | Schema validation tests | Gate 1 (P1) |
| FR-016 | `OwnershipInferenceWorkflow`; ServiceNow/Entra (ADR-011 R2) | `test:ownership-inference`; `ownership.inferred` webhook | **Gate 1** |
| FR-017 | `EvaluationAgent` — 10% stratified drift sample vs 30-day baseline | Week-10 sprint; 2σ alert | Gate 1 |
| FR-018 | `PreferenceEngine`; `POST /preferences` | Gate 2c acceptance | Gate 2c |
| FR-019 | `ControlRefinementQuery` (Specification pattern); `ControlRefinementAggregate` | Qualys/Wiz ingest live | **Gate 1** |
| FR-020 | `GET /assets/{id}/context`; CrowdStrike/Intune ingest | Standalone US-002 screen | **Gate 1** |
| FR-021 | `GET /v1/custom-metrics*`; DQL engine | OpenAPI parity; size ≤200 | Seed |
| FR-022 | `GET /v1/vulnerability-instances/{cve_id}` cursor pagination | limit ≤5000; `expand=asset`; cross-tenant 404 | Seed |
| FR-023 | `POST /v1/cve-research` batch 1–50 | `cve_research.completed` webhook | Seed |
| FR-024 | `VulnerabilityInstanceAckService`; auto-expire job | Create/revoke/expire tests | Phase 1 |
| FR-025 | `ReassessmentSchedulerWorkflow` (self-hosted Temporal, ADR-016 R2); NATS core pub/sub; `ReassessmentDebouncer` (15-min coalesce); evidence-hash dirty-check | `test:reassessment-trigger`; `test:reassessment-dirtycheck` | **Gate 1** |
| FR-026 | `SandboxPort` → `SelfHostedFirecrackerAdapter` (self-hosted, K8s, ADR-015 R4); `ScriptSecurityScanner` AST pre-scan | `test:sandbox-execution-isolation`; `test:script-ast-scan-reject`; `execution_results` assertion | **Gate 1** |
| FR-027 | Outcome-learning feedback loop (draft) — track remediation outcome per `asset_vuln` | Not yet specced (deferred) | Post-Gate-3 candidate |
| FR-028 | Outbound MCP server (draft) — expose Dux capabilities to third-party MCP clients | Not yet specced (deferred) | No near-term trigger |
| FR-029 | `GET /connectors/discovered` (draft) — proactive discovery of tenant's installed tools | Discovery-suggestion integration tests | Gate 2 candidate |
| FR-030 | `GET /assets/risk-trend` — per-asset `rising`/`stable`/`falling` trend from EPSS/finding-count/control-coverage deltas | Trend-computation tests against fixture deltas | Gate 2 |
| FR-031 | Agentic RAG + Apache AGE graph retrieval (ADR-020 R2, D-34). FR ID assigned 2026-07-20 (D-41). | — | — |

## 5. NFR crosswalk (PRD → TRD)

| NFR | Requirement | TR-NFR | Target / measurement |
|-----|-------------|--------|----------------------|
| NFR-001 | Zero cross-tenant reads | TR-NFR-002 | ISO-001–010, 100% CI |
| NFR-002 | API latency | TR-NFR-004 | p95 < 300 ms |
| NFR-003 | Assessment start | TR-NFR-005 | p95 < 2 s |
| NFR-004 | Graph query | TR-NFR-006 | 3-hop CTE p95 < 200 ms at >2 K assets (migration trigger: >150 ms @ >1 K assets, 7 days) |
| NFR-005 | Kill switch | TR-NFR-003 | < 5 s p99 (L2–L4) |
| NFR-006 | GDPR export/delete | TR-NFR-009 | export < 24 h; purge ≤ 90 d |
| NFR-007 | Accessibility | TR-NFR-010 | WCAG 2.2 AA; axe-core 0 violations |
| NFR-008 | Assessment quality | TR-NFR-007/014 | golden-set regression < 2% (P0 block); 100% instrumented LLM paths |
| NFR-009 | Code-backed audit retention | TR-NFR-011 | trace + code (+ execution results at Gate 1) per assessment |
| NFR-010 | Agent context depth | TR-NFR-012 | 128 K tokens; checkpoint at 80% |
| NFR-011 | Per-tenant LLM cost cap | TR-NFR-013 | enforced before intervention ($25/h default) |
| NFR-012 | API availability | TR-NFR-001 | 99.5% monthly excl. LLM providers |
| NFR-013 | Exposure drill-down | TR-NFR-015 | p95 < 500 ms at 1 K assets |
| — (ops) | Feature-flag eval | TR-NFR-008 | SDK p99 < 20 ms; Unleash API 99.9% |
| — (ops) | Valkey cache effectiveness | TR-NFR-016 | `dux_valkey_hit_rate` (`llm_response` type) ≥ 0.6 |
| — (ops) | NATS JetStream backlog | TR-NFR-017 | `dux_nats_consumer_lag` < 1,000 (`VULNERABILITIES`); < 5,000 (any stream) |

## 6. Non-FR traceability rows

| Capability | BR | Component | Gate |
|-----------|----|-----------|------|
| Trust/status portals | BR-005 (comms trust) | Cloudflare DNS + static shell | **Launch blocker** (GCIS §E) |
| Governance kernel GOV-001–013 | BR-003 | `packages/core/governance/` | Pre-launch / Gate 1 |
| Agent identity + registry | BR-003/007 | JWT SPIFFE claims; agent-as-directory (ADR-009) | Gate 1; SPIRE PoC Month 3 |
| Compliance programs | — | SOC 2 (seed trigger → Type I M9–12 → Type II Series A), ISO 27001/42001 (Series A), EU AI Act Art. 9 (Aug 2026) | Stage triggers |

## Bidirectional reference warning

The chain is bidirectional in principle but the validator only checks the forward direction. A US quietly dropped from a BR's story list, or a BR dropped from an epic's parent list, **silently breaks the chain**. This exact defect has been caught by hand during review passes. **Keep both directions in sync.**

---

## Sources

- `.raw/dux/00-meta/decisions-log.md`
- `.raw/dux/00-meta/traceability-matrix.md`
- `.raw/dux/00-meta/open-items.md`
- `.raw/dux/00-meta/quick-reference.md`
