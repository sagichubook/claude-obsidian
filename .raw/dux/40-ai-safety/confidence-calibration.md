---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [H5]
---

# Confidence Calibration

**Purpose:** how an assessment's `confidence_score` is produced, calibrated, and turned into a verdict label or an abstention. **Parents:** BR-002 (BRD-EXP-003 — exploitability trust → calibration pipeline → false-positive-rate KPI).

Extends `EXPLOITABILITY_ASSESSMENT.confidence_score`, which is Platt-scaled.

## 1. Three-signal ensemble

| Signal | Weight | When available |
|--------|--------|----------------|
| Mean top-1 logprob, over claim-bearing tokens | 0.40 | when the API exposes logprobs (`calibration.use_logprobs` flag) |
| Semantic entropy, over meaning-clustered completions | 0.35 | always (multi-sample, max 3 on the calibration path) |
| Verbalized confidence, from structured output | 0.25 | always |

When logprobs are unavailable, the remaining two signals renormalize to **entropy 0.54 / verbalized 0.46**.

**Verbalized confidence alone is not sufficient.** RLHF rewards confident-sounding responses over accurate uncertainty, so a model's stated confidence is the weakest of the three signals and never stands on its own. Semantic-entropy token overhead counts against CostCap.

## 2. Platt scaling and abstention

`CalibrationRecord` is global, with fields `model_version`, `prompt_version`, `platt_params` (jsonb), `brier_score`, `ece`, `training_set_size`, `fitted_at`, `active`.

**ECE gate:** ≤0.15 on a golden-set holdout of n ≥ 50, enforced by `pnpm test:calibration --ece-threshold 0.15`. The `platt_scaling` flag requires an active `CalibrationRecord` with `ece ≤ 0.15`; without one, the runtime guard returns `CALIBRATION_MISSING` and raises a P2 alert.

**Abstention mapping.** Lower bound inclusive, upper bound exclusive. Unit-tested in `calibration_check`; application layer only.

| Calibrated confidence | Label | Human review |
|-----------------------|-------|--------------|
| [0.85, 1.00] | `exploitable` | if a critical asset |
| [0.70, 0.85) | `likely` | always (HITL queue) |
| [0.40, 0.70) | `unlikely` | always (HITL queue) |
| [0.00, 0.40) | `not_exploitable` | if a critical asset |
| Uncertain | `insufficient_data` | always. `insufficient_data_reason` enum: `asset_gap`, `intel_gap`, `context_limit` (US-011 UX; not exposed in the public OpenAPI) |

**This table governs review of the *analysis verdict*, which is separate from write-action approval.** Write actions execute unattended by default ([kill-switch-hitl](kill-switch-hitl.md)); the bands above still route critical-asset verdicts and abstentions to human review.

The public API projection of these labels is in [taxonomy §1](../10-product/taxonomy.md).

## 3. Critic rules

Rule-based from Month 1; an ML critic runs in shadow from seed.

| Rule | Check | Severity |
|------|-------|----------|
| Schema compliance | Zod against `assessmentSchema`, generated from the same OpenAPI as the MCP JSON Schema (`test:schema-parity`) | Blocking |
| Confidence in range | a value in 0.0–1.0 is present | Blocking |
| Prerequisite consistency | with no affected assets, the conclusion cannot be `exploitable` | Blocking |
| Source traceability | reasoning references permitted sources only | Escalation |
| Self-contradiction | no X and not-X in the same chain | Escalation |
| Tool result injection | match against `tests/fixtures/tool-injection-patterns.json`; `test:prompt-injection` runs last-pass in CI | Escalation |

## 4. Sample-size power analysis (resolves OI-21)

**Platt scaling fits 2 parameters** (A, B — a 1-D logistic regression over the raw ensemble score). Standard practice for logistic-regression calibration is a minimum of 10–15 held-out samples per parameter per class outcome (positive/negative) — a well-established rule of thumb (Peduzzi et al.'s "events per variable" heuristic, applied to calibration rather than the original model fit). For 2 parameters × 2 outcome classes, that floor is **40–60 samples overall** — comfortably below the existing **ECE gate's own n ≥ 50 holdout** ([§2](#2-platt-scaling-and-abstention)), which was already set above this floor independently.

**Per-stratum minimum.** The golden set's CVSS-decile stratification ([ci-cd-testing §3](../50-engineering/ci-cd-testing.md)) puts ~25 CVEs per decile bin, each crossed with environment fixtures — comfortably above the 40–60 floor once crossed. The binding constraint is **exploit-maturity strata** (30% functional / 50% PoC / 20% theoretical of 250 ≈ 75 / 125 / 50), since `test:calibration --stratified` fits Platt parameters per stratum where per-stratum drift is suspected. The 20% theoretical-maturity stratum (n≈50) sits at the floor, not above it — **it is the stratum to watch first for calibration instability**, and the one that would trigger a re-sample first.

**Re-sampling schedule.** Tied to the existing quarterly model-pin review ([catalogs §3](../10-product/catalogs.md)): re-fit Platt parameters whenever `model_version` or `prompt_version` changes (already required — a `CalibrationRecord` is keyed on both), **and** on a quarterly cadence regardless of pin changes, to catch calibration drift from distribution shift in incoming CVEs rather than only from model changes. A stratum whose Brier score regresses more than 10% between fits is flagged for a targeted re-sample (additional golden-set cases added to that stratum specifically) rather than a full 250-case re-collection.
