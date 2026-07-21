---
owner: Founder
status: draft
gate: 2
last_reviewed: 2026-07-21
decisions: [D-24, D-36]
---

# Feature — Predictive Risk Forecasting (US-028, E4)

**Purpose:** answer "which assets are likely to become risky in the future" — the literal E4 claim (FinSMEs CEO interview, Dec 2025; committed roadmap capability per the 2026-07-13 claims-alignment directive). **Parents:** BR-013.

**Nav:** Dashboard · **Surface:** a new panel on Dashboard Home (US-012), "Rising Risk" — a cross-asset aggregate view, distinct from US-011's per-CVE Exposure Analysis · **Epic:** EP-03 (F04) · **BR:** BR-013 · **FR-030** · **Gate:** 2 · **Flag:** `risk_trend_forecasting` (Gate 2, default off).

## Design principle: trend, not a new model

**This is a velocity computation over evidence Dux already ingests, not a new ML model and not a composite risk score.** That constraint is deliberate, not a limitation stated apologetically: `taxonomy.md`'s confidence-scoring section already documents that Dux does not compute a composite CVSS×EPSS×criticality×exposure score, because a fabricated single number misrepresents what the system actually knows. Predictive forecasting inherits that discipline — it surfaces **which signals are moving and in what direction**, not a black-box probability. This also means it ships mostly on existing infrastructure (ADR-016's Continuous Assessment Engine, the World Model, existing EPSS ingest) plus one new lightweight history table — not a new subsystem.

## US-028 Asset Risk Trend Forecast

**Gate 2**, read-only.

**Job.** A security engineer or CISO sees which assets are trending toward higher risk — before a new CVE is confirmed exploitable against them — so hardening work can be prioritized ahead of an incident, not just in reaction to one.

**Orchestration.** `AssetRiskTrendWorkflow` (Temporal), scheduled weekly per tenant (reuses ADR-016's scheduled-sweep mechanism; a lower cadence than the 24 h continuous-assessment default, since a trend signal doesn't need daily granularity). It computes a `RiskTrendScore` per asset from three signals, each already backed by existing or newly-added data:

| Signal | Weight | Source |
|--------|--------|--------|
| EPSS 30-day delta, summed across the asset's open findings | 0.4 | **new:** `EPSS_SCORE_HISTORY` (below) — EPSS is currently ingested but not retained as a series |
| Open-finding count, 30-day delta | 0.35 | existing `FINDING.state` transitions, via `ASSESSMENT_STATE_TRANSITION` |
| Control-coverage delta (`Protected` → `Partially Mitigated`/`Exposed` transitions, US-003) | 0.25 | existing `CONTROL_ASSET_MAPPING` |

**Direction, not a probability.** Output is `rising` / `stable` / `falling`, plus the contributing-factor breakdown — never a percentage framed as a likelihood. This avoids the false-precision trap a composite score would carry, and matches the confidence-calibration discipline already applied to exploitability verdicts (bands, not bare numbers).

**Data — new entity.** `EPSS_SCORE_HISTORY` *(global, like `EPSS_SCORE`)*: `cve_id`, `epss_score`, `percentile`, `snapshot_date`. Appended daily alongside the existing `EPSS_SCORE` upsert (ADR-016's ingest path); retained 90 days rolling, matching the trend window this feature needs. This is the one net-new piece of infrastructure the feature requires.

**API.** `GET /assets/risk-trend` → `AssetRiskTrendDto[]`:

| Field | Shape |
|-------|-------|
| `asset_id`, `hostname` | denormalized `ASSET` fields |
| `trend_direction` | `rising` \| `stable` \| `falling` |
| `trend_score` | 0.0–1.0, magnitude only — not exposed as a probability |
| `contributing_factors[]` | `{signal, delta, weight}` — the three rows above, so the ranking is inspectable, not a black box |
| `as_of` | ISO 8601 |

Sorted `rising` first, by `trend_score` descending.

**Safety.** No write action is ever triggered by a trend alone — `rising` surfaces a prioritization signal on the dashboard; it does not enqueue an assessment, open a ticket, or feed `VendorActionGate`. A rising trend is a prompt for a human to *request* research (US-010), same as any other queue entry.

**Marketing reconciliation.** "Which assets are likely to become risky" (E4) is claim-safe once this ships — it is answered literally, using Dux's own reasoning/evidence discipline rather than a claimed predictive model the corpus elsewhere explicitly says Dux doesn't build. GTM copy must not describe this as "AI predicts your next breach" or similar — it is a trend surfaced from real evidence, described the same way the rest of this corpus describes every other output: inspectable, not magic.

## Funding status

**Funded at Gate-2 (2026-07-20, D-36).** The 8 h `EP-03-F04-T01` task funded *authoring this spec*; the feature itself is committed for the Gate-2 backlog. Sizing implementation hours for `backlog-ep03.md` is Engineering's Gate-2 planning-pass task — a scheduling step, not an open funding question.
