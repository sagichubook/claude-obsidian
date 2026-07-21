---
type: project
title: "Dux Portfolio"
created: 2026-07-21
status: active
deadline: "Phase 1 exit, Week 16 (Gate 1 review Week 12)"
tags: [project, dux, dux/execution]
related: ["[[Dux Overview]]", "[[Dux Traceability Matrix]]", "[[Dux Decisions Log]]"]
sources: [".raw/dux/90-execution/README.md", ".raw/dux/90-execution/traceability.md", ".raw/dux/90-execution/backlog-ep01.md", ".raw/dux/90-execution/backlog-ep02.md", ".raw/dux/90-execution/backlog-ep03.md", ".raw/dux/90-execution/backlog-ep04.md", ".raw/dux/90-execution/backlog-ep05.md", ".raw/dux/90-execution/backlog-ep06.md", ".raw/dux/90-execution/backlog-ep07.md", ".raw/dux/90-execution/backlog-ep08.md", ".raw/dux/90-execution/backlog-ep09.md", ".raw/dux/90-execution/backlog-ep10.md"]
---

# Dux Portfolio

## Objective

The Epic -> Feature -> Story -> Task decomposition of the Phase-1 build (16 weeks, `phase_1_epoch` 2026-06-23), and the rules governing it. Owner: Founder (PO), CTO (Tech Lead). Status: canonical. Gate: 1. This is the planning source imported into Linear — not a live status mirror; live status tracking belongs there.

## Executive Summary

The backlog has been re-baselined three times against a moving capacity envelope rather than cut scope: 2,000h (D-7) -> 2,080h (D-23, 25->26 focused h/week) -> 2,160h (D-40, 26->27 focused h/week), with the corrected backlog landing at **2,118h (~98.1%, 42h buffer)** against the final envelope — a 4 h/effective-hour manual audit (2026-07-19) also caught and fixed a 20h arithmetic slip that had accumulated silently in the rollup table (EP-03 430->426h, EP-05 338->322h) via the restored `scripts/validate-playbooks.py` reconciliation. D-40 explicitly rejects a fourth consecutive envelope raise as precedent — Agentic RAG, Apache AGE, and the Gate-2 vLLM+Phi-4 path remain deliberately unestimated net-new scope, tracked as a fresh open item rather than silently absorbed. Seven documented deviations (D-EX-1 through D-EX-7) from the strict Epic/Feature/Story framework are recorded on purpose — most notably, stories are never invented to hit a quota, since US-001-024 is a closed canonical set, and role placeholders (TS-1..3, PY-1..2, SEC) stand in for a not-yet-named team.

## Specification

### ID scheme and framework fields

| Level | ID form | Source |
|---|---|---|
| Epic | `EP-01`...`EP-10` | canonical, [[Dux Traceability Matrix]] |
| Feature | `EP-xx-Fyy` | new in this layer |
| Story | `US-001`...`US-028` | canonical, closed set — never invented here |
| Task | `US-xxx-Tzz` / `EP-xx-Fyy-Tzz` (enabler) | new in this layer |

Capacity: 5 engineers (3 TypeScript, 2 Python). Definition of Done: every merge gate green + the story's verification command + deployed to staging + demoed.

### Portfolio matrix and hour rollup (corrected 2026-07-19)

| Epic | Title | Status | Hours | Share |
|---|---|---|---|---|
| EP-01 | Multi-tenant platform & auth | Defined | 370h | 17% |
| EP-02 | Environmental data ingestion | Defined | 434h | 20% |
| EP-03 | Exploitability assessment engine | Defined | 426h | 20% |
| EP-04 | Continuous re-assessment | Defined | 68h | 3% |
| EP-05 | Analyst surfaces & APIs | Defined | 322h | 15% |
| EP-06 | Mitigation & remediation write path | Defined | 196h | 9% |
| EP-07 | Safety & governance | Defined | 262h | 12% |
| EP-08 | Programmatic platform | Defined | 40h | 2% |
| EP-09 | Triage disposition | **Deferred** (D-19 capacity fallback) | 0h | - |
| EP-10 | Personalization | **Deferred** (Gate 2c) | 0h | - |
| **Total** | | | **2,118h** | 100% |

### Epic highlights

**EP-01 Multi-Tenant Platform & Auth (370h, P0).** Objective: BR-001 zero cross-tenant leakage. The self-hosted-K8s stack replacement (D-33/D-34) added 98h net to this epic — cluster provisioning, CloudNativePG, NATS/JetStream, Valkey, MinIO, self-hosted Temporal, Vault — replacing the original ECS Fargate baseline outright rather than a 1:1 swap, since self-hosted install/ops of five services is strictly more work than pointing services at managed AWS tiers.

**EP-04 Continuous Re-assessment (68h, P0 claim-safety blocker).** Makes the "continuous/24-7" marketing claim engineering-true at Gate 1. Most triggers must resolve without an LLM call (ADR-016 cost control) via a 15-minute debounce coalesce and evidence-hash dirty-check.

**EP-07 Safety & Governance (262h, P0 pre-launch blocker).** Kill switch, governance kernel (GOV-001-013), audit trail, MCP gateway. The MCP gateway core alone splits into 4 sub-tasks (T10a-d, 42h) after a 2026-07-12 traceability audit found the original single "T10" task under-scoped for policy engine + schema integrity + egress proxy + circuit breaker.

**EP-08 Programmatic Platform (40h, P1).** Outbound webhooks ship at Gate 1; the Public Data API (`/v1`) and an outbound MCP server are both deferred — the latter would invert Dux's current MCP-client-only posture into also being an MCP server for third parties, an ecosystem play with no near-term trigger.

**EP-09/EP-10 (0h, deferred).** EP-09 (acknowledgment lifecycle) deferred via the D-19 capacity fallback lever when the backlog first ran 73h over the original 2,000h envelope. EP-10 (preference learning) is deliberately not promoted — it needs behavioral-data volume that doesn't exist pre-launch.

### Documented deviations (D-EX-1 to D-EX-7)

The framework is broken on purpose in seven named ways, all justified by preserving corpus authority over hitting a template quota — e.g., D-EX-1: some Features carry only 1-2 stories rather than the "3-12" rule, because inventing stories to fit the template would fabricate scope; D-EX-6: deferred/draft features carry no acceptance criteria or SLA, because specifying precision for unscheduled work would fake commitment.

### Validation rules (enforced on every change)

No orphans (every Feature parents an Epic, every Story a Feature, every Task a Story or Feature); no widows (every Epic has >=1 Feature); ID uniqueness portfolio-wide; bidirectional traversal via the traceability matrix; status cascade (a parent is Done only when every child is Done).

## Diagram

```mermaid
gantt
    title Dux Phase 1 (16 weeks, Gate 1 review Week 12)
    dateFormat YYYY-MM-DD
    axisFormat W%W
    section Platform
    EP-01 Multi-tenant auth (370h)      :active, ep01, 2026-06-23, 8w
    section Core
    EP-02 Data ingestion (434h)         :ep02, 2026-06-23, 9w
    EP-03 Assessment engine (426h)      :ep03, 2026-06-23, 9w
    EP-04 Continuous re-assessment (68h) :ep04, after ep03, 3w
    section Surfaces
    EP-05 Analyst surfaces (322h)       :ep05, 2026-07-07, 8w
    EP-06 Write path (196h)             :ep06, 2026-07-07, 6w
    section Safety
    EP-07 Safety & governance (262h)    :crit, ep07, 2026-06-23, 9w
    section Deferred
    EP-08 Programmatic (40h)            :ep08, 2026-07-21, 3w
    EP-09 Triage disposition (deferred) :done, ep09, 2026-08-01, 1d
    EP-10 Personalization (deferred)    :done, ep10, 2026-08-01, 1d
```

## Entities & Concepts

- [[Dux Traceability Matrix]] — the BR->Epic->US chain this portfolio decomposes
- [[Dux Decisions Log]] — D-7/D-19/D-23/D-33/D-34/D-40 capacity re-baseline history
- [[Governance Kernel]] — EP-07's GOV-001-013 implementation

## Related

- [[Dux Overview]]
- [[Dux Product Area]]

## Review cadence

Weekly, or on any capacity re-baseline.

## Sources

- `.raw/dux/90-execution/README.md`
- `.raw/dux/90-execution/traceability.md`
- `.raw/dux/90-execution/backlog-ep01.md` through `backlog-ep10.md` (all 10 epic files individually reviewed)
