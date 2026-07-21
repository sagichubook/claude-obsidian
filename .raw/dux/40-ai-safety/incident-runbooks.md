---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-3, H6]
---

# AI Safety Incident Runbooks

**Purpose:** the twelve canonical agentic failure modes and the procedure for each. **Parents:** BR-003, BR-005.

**This file is the source of truth for these twelve procedures.** [Seed runbooks](../60-operations/runbooks.md) add stage deltas — PagerDuty IDs, admin CLI, and a thirteenth agent-quota mode. Series A and B link here. **Do not duplicate the step tables downstream.**

## 1. Runbook template (12 sections)

Every runbook below follows the same structure:

| § | Section | § | Section |
|---|---------|---|---------|
| 1 | Trigger | 7 | Pre-conditions |
| 2 | Service catalog | 8 | Automation gate |
| 3 | Business impact (MRR-at-risk formula) | 9 | Execution steps — command / expected / timeout / human fallback |
| 4 | User impact (PM) | 10 | AI safety check |
| 5 | System/agent boundary | 11 | Verification |
| 6 | Incident roles | 12 | Post-incident |

**§10 AI safety check — all eight items, every incident:**

| # | Check |
|---|-------|
| a | agent loop ≤50 iterations |
| b | `admin:agent-tool-diff` clean |
| c | prompt-injection CI passing |
| d | HITL queue drained |
| e | cross-tenant `foreign_tenant_refs: 0` |
| f | shadow AI `undeclared_count: 0` |
| g | AI-BOM valid |
| h | cost-cap state confirmed |

Plus the OWASP triple: the [LLM and Agentic assessments](owasp-assessments.md) and the [MCP crosswalk](mcp-security.md).

**The AI Safety Lead (`@ai-safety-oncall`) holds 60-second halt authority, and that role cannot be merged with the Incident Commander.**

## 2. The twelve runbooks

| # | Runbook | Type | Severity | Trigger | Core containment |
|---|---------|------|----------|---------|------------------|
| 1 | [Cross-tenant context leak](#r1--cross-tenant-context-leak) | COMPOSITE | P0-C | `DuxCrossTenantContextDetected` (`foreign_tenant_refs > 0` + isolation SLO burn ≥5%/1h), or a fuzz failure | `admin:platform-contain` (L4); agent halt ≤60 s; context audit; engage counsel if PII (GDPR/DORA, 72 h) |
| 2 | [Token cost runaway](#r2--token-cost-runaway) | COMPOSITE | P0-C / P1 | spend >3× the 7-day baseline, or >$25/hour/tenant | `admin:cost-cap enforce`; L2 kill switch; halt the top-spend agent |
| 3 | [Model provider outage](#r3--model-provider-outage) | COMPOSITE | P1 | OpenAI status ≠ operational, or `DuxLLMAvailabilityFastBurn` | `admin:model-route --fallback on` (≤60 s); golden-set spot check, halting if regression >5% |
| 4 | [MCP dependency failure](#r4--mcp-dependency-failure) | AGENTIC-SAAS | P2 | MCP errors >50% over 5 min, or a health-check failure | `admin:mcp-circuit-breaker --open`; verify no hallucinated success |
| 5 | [Rate limit cascade](#r5--rate-limit-cascade) | AGENTIC-SAAS | P2 | `DuxRateLimitCascade` — model 429 and customer 429 in the same window | jitter backoff; halt the runaway agent; escalate provider quota |
| 6 | [Context window exhaustion](#r6--context-window-exhaustion) | AGENTIC-SAAS | P2 | `DuxContextWindowExhausted` (128 K) | checkpoint at 80%, abandon at 100%; halt the runaway retry loop |
| 7 | [Prompt cache invalidation](#r7--prompt-cache-invalidation) | AGENTIC-SAAS | P2 | `DuxPromptCacheHitRateDrop` (>15% over 5 min) | identify the cause (deploy or pin); roll back, or `admin:llm-cache-warm` |
| 8 | [Hallucinated-CVE citation](#r8--hallucinated-cve-citation) | AGENTIC-SAAS | P1 | EXP-CIT-001 fails at generation time — a citation's CVE ID doesn't resolve against NVD, or its CVSS/description diverges from the resolved record | `admin:verdict-quarantine`; re-verify against NVD; re-run `test:exposure-citations` |
| 9 | [Alert fatigue](#r9--alert-fatigue) | AGENTIC-SAAS | P2 | HITL queue backlog with a rubber-stamp approval pattern (>95% approved, sub-baseline median review time) | triage mandatory vs anomaly-escalation queues; temporary confidence-floor raise; add reviewer capacity |
| 10 | [Memory / context poisoning](#r10--memory--context-poisoning) | AGENTIC-SAAS | P1 | `camel.output_audit_failed` spike or a `connector_drift_anomaly` tied to one tenant/session | agent halt ≤60 s; `admin:agent-context-audit`; isolate the suspect connector |
| 11 | [Coordination overhead](#r11--coordination-overhead) | AGENTIC-SAAS | P2 | assessment p95 latency regression traced to worker-to-worker handoff, with GOV-010 loop counters within limits | `admin:workflow-trace`; batch/cache redundant MCP calls across subagent handoffs |
| 12 | [Prompt brittleness](#r12--prompt-brittleness) | AGENTIC-SAAS | P2 | golden-set regression >2% (NFR-008) traced to a prompt/schema edit, not a provider EOL migration | `admin:prompt-pin --rollback`; re-run golden set; re-baseline the hash after sign-off |

## R1 — Cross-tenant context leak

**§8 Automation gate:** `admin:platform-contain --confirm`, 60 s. On timeout, page the CTO and the Founder.

**§9 Execution steps:**

1. Discover active agents.
2. `admin:agent-halt --id $ID --confirm` — within 60 s.
3. `admin:agent-context-audit` — must read `foreign_tenant_refs: 0` after the fix. **This scan is post-hoc, not real-time** — it runs on demand against logged context, not inline on every request — a known detection-latency tradeoff; a leak can persist between occurrence and the next audit run.
4. `admin:export-session` — evidence to MinIO.
5. **Engage counsel if PII is involved.**
6. Root-cause fix, then `test:isolation` — every ISO case must pass.
7. PM approves the customer notification.
8. `admin:platform-contain --release`.

**§3 Business impact:** `monthly_MRR × (affected ÷ total) × (hours ÷ 730)`. **Regulatory: GDPR Art. 33 and DORA Delegated Regulation 2025/301 Art. 5 — 72 hours if PII is involved.**

**§11 Verification:** `sum(foreign_tenant_refs) == 0`; `rate(dux_assessment_errors[15m]) < 0.01`; API p95 <0.3 s.

## R2 — Token cost runaway

Cost evaluation order (D-3): **$0.675/assessment breaker → $25/hour CostCap → 2× baseline.**

**§9 Execution steps:**

1. `admin:token-spend`.
2. `admin:cost-cap enforce --max-usd-per-hour 25`.
3. L2 kill switch.
4. Discover and halt the top-spend agent.
5. Export the session.
6. Fix the configuration — iteration limit, or model downgrade.
7. Verify spend is below 1.5× baseline **before** releasing L2.

## R3 — Model provider outage

The fallback model is `claude-sonnet-4-6` (+28% latency).

**§9 Execution steps:**

1. Confirm the outage.
2. `admin:model-route --fallback on` — within 60 s.
3. PM sets the status page to `degraded_performance`.
4. Run the golden set against the fallback. **Halt if regression exceeds 5%.**
5. Verify the queue is draining.
6. `--fallback off` once the primary is restored.

**Low-traffic guard:** force HITL T3 on fallback verdicts until a spot check of ≥5 assessments per 15 min passes.

### R3b — Model EOL / forced deprecation (H6, Gate-2 hardening)

Planned work, not incident-triggered. **Provider models are EOL-dated dependencies.** In 2026, ChatGPT-surface models have been retired on two weeks' notice; the API minimum is 6 months GA and **3 months for variants** — and the S-LLM workhorse is a variant.

A forced migration shifts the golden-set baseline, and **schema-parity checks alone will not catch quality drift**: "Drift or Dice" (2026) documents a schema gate reporting zero regressions while output shrank 15.7%, and a tier failed 25% of hard tasks.

Procedure:

1. Subscribe to provider deprecation feeds; review weekly.
2. On an EOL notice, run the migration eval protocol — the golden set **plus** output-length, latency, and tier-failure deltas. Never schema parity alone.
3. Pin the successor via `models.json` and the CI pin gate.
4. Keep a pinned fallback, so a forced sunset cannot take the assessment path down.
5. Re-baseline the golden-set hash after sign-off.

## R4 — MCP dependency failure

**The risk is a silent stall** — assessments appear to be running but produce no results.

**§9 Execution steps:**

1. `admin:mcp-circuit-breaker --open --server $ID`.
2. **Verify the audit shows `tool_unavailable`, not a hallucinated success.**
3. Notify tenants if the outage exceeds 1 hour.
4. Half-open: test a single tool.
5. `--close` once the error rate is below 5%.

## R5 — Rate limit cascade

The signature is a **dual 429** — OpenAI (60 RPM/tenant) and the NestJS throttler in the same window.

**§9 Execution steps:**

1. Confirm the dual 429.
2. `admin:rate-limit-backoff --jitter on`.
3. Discover the runaway agent; optionally L1-halt it.
4. Verify spend is within cap.
5. Notify if the stall exceeds 1 hour.
6. Reset the backoff.

**Seed depth:** dynamic throttle at `max(1, floor(baseline × 0.5))`.

## R6 — Context window exhaustion

The ceiling is 128 K, with L1–L3 compression selected by asset count.

**§9 Execution steps:**

1. `admin:agent-halt`.
2. `admin:export-session`.
3. Checkpoint or abandon, per governance limits.
4. Root-cause: context configuration, or asset count.
5. `admin:platform-contain --tenant` if a retry loop is running.
6. Resume: `test:e2e --flow assessment`.

**Checkpoint at 80%** into `assessment_checkpoints` (JSONB). **Abandon at 100%** (`abandoned_with_notify`). Maximum 3 resume attempts.

## R7 — Prompt cache invalidation

**§9 Execution steps:**

1. Confirm the hit-rate drop exceeds 15% against the 7-day baseline.
2. Identify the cause — a deploy, or a pin change.
3. If deploy-caused, evaluate a rollback.
4. `admin:llm-cache-warm --env production` — IC-approved, after staging parity.
5. Verify the hit rate is within 10% of the 7-day baseline.
6. PM updates the status page if p95 stays above 2× for more than 1 hour.

## R8 — Hallucinated-CVE citation

**§9 Execution steps:**

1. `admin:citation-audit --assessment $ID` — confirm which `exploitable`/`likely` claim's citation failed NVD resolution, or diverged on CVSS/description, per the extended EXP-CIT-001 check ([owasp-assessments §2](owasp-assessments.md)).
2. `admin:verdict-quarantine --assessment $ID` — hide the exposure verdict from the customer surface pending manual verification.
3. Manually re-verify against NVD. If the CVE genuinely doesn't resolve, downgrade confidence and re-run the citation check.
4. Root-cause: model hallucination vs. a stale NVD cache vs. an upstream NVD outage (cross-reference the [NVD fallback priority](../10-product/catalogs.md#nvd-enrichment-fallback-priority)).
5. `pnpm test:exposure-citations` green before re-enabling the verdict.

**§11 Verification:** all `exploitable`/`likely` claims on the affected assessment show `nvd_resolved: true`; zero repeat EXP-CIT-001 failures for the tenant over 24 h.

## R9 — Alert fatigue

**§9 Execution steps:**

1. `admin:hitl-queue-depth` — confirm a sustained backlog or a rubber-stamp pattern (>95% approved, median review time below baseline).
2. Triage: separate the 2 mandatory-HITL actions (`endpoint.isolate`, `patch.deploy_special_devices`) from the 3 anomaly-escalation-only actions — mandatory items are never paused or floor-adjusted.
3. `admin:confidence-floor --raise` — a time-boxed, logged increase on the anomaly-escalation actions only, to cut marginal-confidence escalations.
4. Add reviewer capacity or redistribute load across the HITL roster.
5. Root-cause: confidence-floor miscalibration vs. a genuine incident surge driving real escalations.
6. Revert the raised floor once median time-to-decision is back under target.

**§11 Verification:** HITL median time-to-decision under the target baseline; approval rate back below the rubber-stamp threshold.

## R10 — Memory / context poisoning

**§9 Execution steps:**

1. `admin:agent-halt --id $ID` — within 60 s.
2. `admin:agent-context-audit` — inspect the `sources[].content_hash` provenance chain for the poisoned span ([camel-plane §3](camel-plane.md)).
3. `admin:mcp-circuit-breaker --open --server $ID` on the suspect connector or research source.
4. Cross-check: confirm `getAssetsForCVE` / native `query_controls` disagree with the poisoned claim (CaMeL CDA #3 and #5, [camel-plane §4](camel-plane.md)).
5. Purge World Model rows sourced from the poisoned span; re-sync clean.
6. `pnpm test:camel-benchmark` green before resuming the connector.

**§11 Verification:** `connector_drift_anomaly` count returns to 0; no repeat `camel.output_audit_failed` for the tenant over 24 h.

## R11 — Coordination overhead

**§9 Execution steps:**

1. `admin:workflow-trace --assessment $ID` — break latency down by workflow subagent (`prerequisite-extractor`, `asset-context-worker`, `control-mapping-worker`).
2. Confirm GOV-010 loop counters are within limits — this rules out a runaway loop, a distinct failure mode from handoff overhead.
3. Identify redundant or serial MCP calls (`query_assets`/`query_controls` chains) that should batch or parallelize.
4. Patch: batch calls, or cache `summarizeContext` output across subagent handoffs.
5. Re-run the golden set to check for a regression.
6. Verify p95 assessment duration is back within the governance latency budget ([governance-kernel §3](governance-kernel.md)).

**§11 Verification:** p95 assessment duration within budget; MCP call count per assessment not elevated versus baseline.

## R12 — Prompt brittleness

**§9 Execution steps:**

1. Confirm the golden-set regression traces to a prompt or schema edit — not a provider EOL migration ([R3b](#r3b--model-eol--forced-deprecation-h6-gate-2-hardening)) and not a hallucinated citation ([R8](#r8--hallucinated-cve-citation)).
2. `admin:prompt-pin --rollback` — to the last golden-set-passing version.
3. Re-run the golden set; confirm the regression drops under the 2% NFR-008 threshold.
4. Root-cause: an overly narrow enum or wording that fails on edge-case CVE phrasing.
5. Stage the fix behind a shadow run before re-promoting.
6. Re-baseline the golden-set hash after sign-off.

**§11 Verification:** golden-set regression under 2%; no repeat failure over 3 consecutive assessment runs.

## 3. Runbook map

The twelve above are canonical here: cross-tenant leak (P0-C), token cost (P0-C/P1), model outage (P1), MCP failure (P2), rate limit (P2), context exhaustion (P2), prompt cache (P2), hallucinated-CVE citation (P1), alert fatigue (P2), memory/context poisoning (P1), coordination overhead (P2), prompt brittleness (P2).

Seed adds two more in [seed runbooks](../60-operations/runbooks.md): agent quota exhaustion (seed-only) and shadow AI. The seed routing matrix lives there too.
