---
owner: Founder
status: canonical
gate: n/a
last_reviewed: 2026-07-21
decisions: [D-10, D-24, D-32, D-33, D-34, D-35, D-36, D-37, D-38, D-39, D-40, D-41, D-42, D-43, D-44, D-45, D-46, D-47, D-48, D-49, D-50, D-51, D-52, D-53, D-54, D-55, D-56, D-57]
---

# Open Items Register

**Purpose:** the single list of every unresolved question in the corpus. Specs state what is true today; anything still undecided lives here with an ID, an owner, and the thing it blocks.

Before this register existed, open questions were `> **TBD**` blockquotes scattered across ~20 files under seven unrelated ID schemes (`E-`, `BS-`, `SR-`, `MC-`, `H-`, `CI-`, `AI-`), most with no owner and no stated consequence. The **Origin** column preserves the old ID so every existing cross-reference still resolves.

## 1. How this register works

1. **One ID scheme.** Every open question is `OI-##`. The original ID is kept in the Origin column, never dropped.
2. **A spec never carries an open question in its body.** It states current truth and links here. The exception is `decisions-log.md`, which is the historical record and keeps its text verbatim.
3. **Nothing here is resolved by editing prose.** An item closes when its owner makes the call; the decision is then recorded in [decisions-log.md](decisions-log.md) and the affected spec is updated to state it as fact.
4. **Severity is about consequence, not effort.**
   - **P0** — blocks a Gate-1 ship, or is a live legal/compliance exposure.
   - **P1** — blocks a plan, a date, or an external commitment already made.
   - **P2** — spec debt: real, but nothing is currently unsafe or over-promised because of it.

## 2. P0 — blocking

*(OI-36 closed 2026-07-19, see [decisions-log](decisions-log.md): Better Auth, Unleash, and Grafana LGTM confirmed as canon; the live Descope/LaunchDarkly/Sentry `demo` deployment was environment drift, not an adopted alternative.)*

*(OI-02 closed 2026-07-16, see [decisions-log Update 2026-07-16 (second session)](decisions-log.md))*

*(OI-57 closed 2026-07-21 via D-44 (badges confirmed unearned; compliance-program.md is correct; live-site fix tracked externally, outside this repo), see [decisions-log](decisions-log.md).)*

*(OI-51 closed 2026-07-21 via D-45 (bare `admin`/`member`/`viewer` canonical everywhere; full per-endpoint role matrix added to api-overview.md §3), see [decisions-log](decisions-log.md).)*

**None open.**

## 3. P1 — blocks a plan, date, or live commitment

*(OI-33 closed 2026-07-20 via D-36 (fund at Gate-2); OI-39 closed 2026-07-20 via D-40 (envelope raised to 2,160 h); see [decisions-log](decisions-log.md).)*

*(OI-07, OI-08, OI-09, OI-10, OI-11, OI-12, OI-13, OI-14, OI-32 all closed 2026-07-16; see [decisions-log](decisions-log.md) for each disposition)*

*(OI-48 closed 2026-07-21 via D-46 (`agt_` is the Public Data API credential; agent-identity.md corrected to match; openapi.yaml TBD resolved), see [decisions-log](decisions-log.md).)*

*(OI-50 closed 2026-07-21 via D-47 (MVP connector set table wins, superseded table removed; narrower Wiz/Intune cadence gap reopened as OI-58), see [decisions-log](decisions-log.md).)*

*(OI-52 closed 2026-07-21 via D-48 (quote downgraded to unconfirmed, pulled from active use, 2026-07-09 "confirmed" call superseded), see [decisions-log](decisions-log.md).)*

*(OI-56 closed 2026-07-21 via D-49 (LinkedIn post confirmed real by Founder, URL + wording added; two adjacent false claims recorded as a standing gtm-guardrails.md guardrail, never entered the corpus), see [decisions-log](decisions-log.md).)*

*(OI-55 closed 2026-07-21 via D-50 (`crowdstrike` named as Gate-1 executor via IOC/indicator blocklisting), see [decisions-log](decisions-log.md).)*

*(OI-54 closed 2026-07-21 via D-51 ("Dux, Inc." confirmed correct; Privacy Policy PDF is the error, external follow-up; drafted correction requests no longer held), see [decisions-log](decisions-log.md).)*

**None open.**

## 4. P2 — spec debt

*(OI-49 closed 2026-07-21 via D-52 (Ready row corrected to "ERM risk appetite; data-room index"), see [decisions-log](decisions-log.md).)*

*(OI-53 closed 2026-07-21 via D-53 (all three stats corrected with honest attribution rather than dropped; Mandiant pairing split), see [decisions-log](decisions-log.md).)*

*(OI-37 closed 2026-07-21 via D-54 (all 33 roadmap vendors assigned a real `role`, grouped into 7 rows), see [decisions-log](decisions-log.md).)*

*(OI-41 closed 2026-07-21 via D-55 (embedding-signing spec written, LLM04/LLM08 reassessment closed; narrower live-test gap reopened as OI-59), see [decisions-log](decisions-log.md).)*

*(OI-42 closed 2026-07-21 via D-56 (both "missing" endpoints already exist under other names; cross-references added, no new endpoints), see [decisions-log](decisions-log.md).)*

*(OI-47 closed 2026-07-21 via D-57 (Founder confirmed §4a's first-run setup notes match reality; "unverified" caveat removed), see [decisions-log](decisions-log.md).)*

| ID | Item | Origin | Blocks | Owner |
|----|------|--------|--------|-------|
| **OI-40** | **Temporal namespace-per-tenant trigger needs re-deriving for self-hosted.** The prior 15,000-active-task-queue trigger ([ADR-007 R3](../20-architecture/adr-index.md#adr-007-r3--durable-execution-engine)) was keyed to Temporal Cloud's ~20K-poller-per-namespace SaaS ceiling. Self-hosted Temporal's actual scaling ceiling depends on cluster resource allocation instead — the numeric trigger needs re-deriving against real self-hosted capacity planning before it can be trusted again. No Phase-1 impact (single-namespace model is unaffected either way at current scale). EKS's own node/pod scaling characteristics don't change this axis — the trigger is CSP-agnostic, tied to cluster resource allocation generally, not the DO/Linode vs. EKS distinction. **Recommendation (2026-07-20, research pass):** can't derive the real number without live cluster specs (no invented figure here), but the re-derivation methodology per Temporal's own self-hosted scaling guidance is: (1) load-test the actual matching/history/frontend service pod allocation against synthetic workflow-task volume, (2) find the poller-per-task-queue count where p99 schedule-to-start latency starts degrading, (3) apply the same safety margin the original ~15K/20K figures used (roughly 75% of measured ceiling) as the new trigger. This is a bounded Engineering exercise once EKS node sizing is finalized — flag as blocked on that sizing, not on unknowns. | D-33, D-34 | Namespace-per-tenant migration planning | Engineering |
| **OI-59** | **Live execution of the ISO-012 ANN adversarial-neighbor test against the real, deployed tenant-scoped HNSW index has never been run.** Narrowed out of the former OI-41 (D-55) — the embedding-signing spec and LLM04/LLM08 reassessment that made up the rest of that item are done. This is a pure execution artifact: a docs-only repo cannot run a test against a live index. Opened 2026-07-21 closing OI-41. | opened closing OI-41, 2026-07-21 | Confirmation the tenant-scoped HNSW index resists the documented ANN-adversarial-neighbor attack in practice, not just by architectural design | Engineering / Security |
| **OI-43** | **Narrowed 2026-07-20 (research pass) — a test-scenario checklist now exists ([ci-cd-testing §4](../50-engineering/ci-cd-testing.md)); the test code itself does not, and can't exist in this docs-only repo.** All 7 scenarios (happy path, max-turns(10) exit, budget-exceeded ($0.75) abort, tool failure/retry, Bedrock timeout, malformed tool response, HITL signal timeout) are enumerated with their spec citations. **Correction:** the HITL signal timeout is **30 min** (kill-switch-hitl.md's T4 escalation timeout) — the previously-stated "24 h" was never corroborated by any spec in this corpus and has been dropped. Remaining gap is implementation-only: writing the actual `pnpm test:*` suite lives in the separate application repo, not here. | D-35 | Gate-1 test-coverage sign-off for `ExploitabilityAssessmentWorkflow` (implementation, separate repo) | Engineering |
| **OI-58** | **Wiz and Intune have no rate-limit-derived sync cadence, unlike the other 8 MVP connectors.** [connector-hub.md](../10-product/features/connector-hub.md)'s "MVP connector set" table — now the sole authoritative cadence source per [D-47](decisions-log.md) — only covers Tenable.io, Qualys, CrowdStrike, AWS Security Hub, Rapid7 InsightVM, Splunk, Jira, ServiceNow, Okta, and Azure AD. Wiz and Intune are both named Gate-1-relevant connectors (Wiz is one of ADR-011 R2's ≥3 Gate-1 floor connectors) but were only ever covered by the now-superseded "sync cadence defaults" table, whose figures for them (6 h each) were never rate-limit-researched the way the surviving table's entries were — not carried forward, to avoid keeping an unresearched number on a table whose authority has been retired. Opened 2026-07-21 while closing OI-50. | opened closing OI-50, 2026-07-21 | Wiz and Intune sync-cadence figures, on the same rate-limit-research basis as the other 8 MVP connectors | Engineering |

*(OI-34 closed 2026-07-20 via D-37 (Airbyte not adopted); OI-35 closed 2026-07-20 via D-38 (isolation tier offered, gated); OI-38 closed 2026-07-20 via D-39 (FedRAMP stays gated); OI-44 closed 2026-07-20 via D-41 (FR-031 assigned); OI-45 closed 2026-07-20 via D-42 (traceability.md added as 3rd present-tense exception); OI-46 closed 2026-07-20 via D-43 (AGE supersedes Neo4j as first-line trigger response); see [decisions-log](decisions-log.md) for each disposition.)*

*(OI-15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 29 closed 2026-07-16, all engineering-owned, see [decisions-log](decisions-log.md))*

*(Older IDs not otherwise re-keyed above: OI-01 closed 2026-07-14 via D-14; OI-03 closed 2026-07-14 via D-15; OI-04 closed 2026-07-15 via D-18; OI-05 closed 2026-07-15 via D-19 — see OI-32 for its later regression and closure; OI-06 closed 2026-07-16 via D-22; OI-28 closed 2026-07-16 via D-21; OI-25 fully closed 2026-07-16; OI-30 and OI-31 substantively closed 2026-07-16 — see [decisions-log](decisions-log.md) for each disposition.)*

## 5. Standing review registries

Three point-in-time reviews carry their own numbered findings and are recorded as **awaiting disposition** in full. They are tracked here as blocks rather than re-keyed into `OI-##`, because each finding already has an ID and a verdict inside its own file.

*(none open — OI-26, OI-27 closed 2026-07-17: all corpus/spec-side work complete; remaining items are execution-only (DNS, pasting/submitting drafted corrections, unbuilt code) and tracked outside this register — see [decisions-log](decisions-log.md).)*

## 6. Closing an item

An item leaves this register only when its owner decides it. On close:

1. Record the decision in [decisions-log.md](decisions-log.md) with the date and the person who made the call.
2. Update the affected spec to state the outcome as present-tense fact — not as a diff against what it used to say.
3. Delete the row here. The decisions-log entry and git history are the record; this register tracks only what is still open.
