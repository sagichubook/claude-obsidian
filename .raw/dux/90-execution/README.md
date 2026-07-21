---
owner: Founder
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-7, D-19, D-23, D-33, D-34, D-35]
---

# Execution Backlog — Framework & Rules

**Purpose:** the Epic → Feature → Story → Task decomposition of the Phase-1 build, and the rules that govern it.

**The specs stay canonical in `10-product/` through `70-governance/`. This folder adds the execution layer only.**

**Live status tracking belongs in Linear.** These files are the planning source that gets imported there — **they are not a status mirror.**

## 1. ID scheme

This extends the corpus canon. **It does not create a second ID universe.**

| Level | ID form | Source |
|-------|---------|--------|
| Epic | `EP-01` … `EP-10` | canonical — [traceability-matrix §1](../00-meta/traceability-matrix.md) |
| Feature | `EP-xx-Fyy` | **new here** — groups an epic's FRs into shippable capabilities |
| Story | `US-001` … `US-028` | canonical, and a **closed set**. **Stories are never invented in this folder.** US-025/026/027 and US-028 (BR-013, Gate 2) were extended into the set at the true canonical layer (`traceability-matrix.md`); this table only tracks it |
| Task | `US-xxx-Tzz` (story tasks) · `EP-xx-Fyy-Tzz` (enabler tasks) | **new here** |

## 2. Framework fields

| Level | Fields |
|-------|--------|
| **Epic** | business objective (BR + KPI link), success metrics, target release (gate/week), priority, technical constraints |
| **Feature** | parent epic, acceptance criteria (citing the feature spec + FR verification), dependencies, API/service boundary, performance SLA (NFR crosswalk) |
| **Story** | parent feature, points (Fibonacci), persona, acceptance criteria, DoD, spikes |
| **Task** | parent, discipline (Backend/Frontend/DevOps/QA/Security/Data/Design), type (Code/Test/Review/Deploy/Config/Research), estimated hours, target week, assignee |

## 3. Standing answers

Resolved from canon. **Cited, not restated.**

| Question | Answer |
|----------|--------|
| Business objectives | BR-001–012, plus the [KPI table](../80-gtm/pricing-packaging.md) |
| Personas | security engineer (primary), CISO, tenant admin, AI Safety Lead, API consumer |
| Timeline | **16-week Phase 1. Gate 1 review at Week 12; exit at Week 16.** `phase_1_epoch` is 2026-06-23 (D-7 R1) — see the [gate model](../10-product/product-overview.md) |
| Tech stack | ADR-001–016 ([adr-index](../20-architecture/adr-index.md)) |
| Definition of Done | every [merge gate](../50-engineering/ci-cd-testing.md) green **+** the story's verification command ([traceability-matrix §4](../00-meta/traceability-matrix.md)) **+** deployed to staging **+** demoed |
| Capacity | 5 engineers (3 TypeScript, 2 Python); a **2,080 h planning envelope** — 16 weeks × 26 focused hours (re-baselined D-23, was 25 h/week) |
| Ownership | PO = Founder · Tech Lead = CTO · PM = product sign-off · Security / AI-Safety Lead = the safety gate |

**Capacity (D-19, D-23; re-opened by the 2026-07-19 D-33/D-34/D-35 stack replacement, arithmetic-corrected this review).** The backlog now totals 2,118 h against the 2,080 h envelope (~101.8%, 38 h over) — Agentic RAG and Apache AGE remain unestimated net-new scope. See the capacity check in [traceability.md](traceability.md) and [OI-39](../00-meta/open-items.md).

## 4. Documented deviations (D-EX)

Each of these breaks a strict-framework rule **on purpose**. The reason matters more than the rule.

| # | The rule | The deviation | Why |
|---|----------|---------------|-----|
| D-EX-1 | A Feature must have 3–12 stories | some carry 1–2 | **US-001–024 is a closed canonical set.** Inventing stories to hit a quota would break corpus authority. Task granularity compensates |
| D-EX-2 | An Epic decomposes into 2–8 features | EP-04, EP-09, EP-10 have 1 | Single-FR epics. Splitting them would be artificial |
| D-EX-3 | Every Task parents a Story | enabler tasks (kill switch, governance kernel, webhooks, CI/infra) parent their **Feature** (`EP-xx-Fyy-Tzz`) | The corpus **deliberately has no user story** for platform enablers — BR-003 covers "all agent paths", and BR-009 is CI-only. **Synthetic stories were rejected** |
| D-EX-4 | Assignee is a named person | role placeholders — **TS-1..3** (TypeScript), **PY-1..2** (Python), **SEC**, **PM**, **CTO** | The team is not yet named. Replace these at sprint planning |
| D-EX-5 | Story status "Ready" | everything ships as **Defined / To Do** at t0 | **Nothing has been groomed by the real team yet** |
| D-EX-6 | A Feature must have ACs, dependencies, an API/service boundary, and a performance SLA | deferred and draft features (EP-02-F04, EP-03-F03, EP-06-F04, EP-08-F02/F03, EP-10-F01) carry only Parent / FR / Story-in-prose | **There is no scheduled build target.** Specifying ACs, an API boundary, and an SLA for work with no committed date would fake precision — the same failure as inventing stories. **Backfill at the gate that un-defers them** |
| D-EX-7 | Story DoD = merge gates + verification command + staging deploy + demo | per-story `DoD:` lines state only "merge gates + verification command" | Staging deploy and demo are enforced at the **Epic and Sprint** level. **The full four-part bar still applies before a Feature ships** — it is simply not repeated across 40+ story lines |

**Deferred stories carry no tasks**, because they are gate-fenced: US-009 (Gate 2c) · US-019 (Gate 3) · US-020 (Gate 5) · US-022, and the public half of US-024 (Seed) · US-025 (a post-Gate-3 candidate) · US-026 (no near-term trigger) · US-028 (Gate 2 — spec-authoring is closed, but sizing the build's hours into the backlog is separate, unbudgeted work).

**Decomposing them now would fake precision.**

## 5. Validation rules

Enforced on every change:

1. **No orphans** — every Feature parents an Epic; every Story a Feature; every Task a Story **or** a Feature (D-EX-3).
2. **No widows** — every Epic has ≥1 Feature; every non-deferred Feature has ≥1 Story or enabler task.
3. **ID uniqueness**, portfolio-wide.
4. **Bidirectional traversal**, via [traceability.md](traceability.md).
5. **Status cascade** — a parent is Done only when every child is Done.
6. **Tech-lead gate** — every task carries an assignee, hours, discipline, type, and target week.

## 6. Files

[traceability.md](traceability.md) — the portfolio matrix and the hour rollups.

`backlog-ep01.md` … `backlog-ep10.md` — one file per epic.
