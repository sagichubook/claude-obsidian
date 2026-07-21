---
owner: Engineering
status: canonical
gate: 2
last_reviewed: 2026-07-19
decisions: [D-3, D-33, D-34, H8, H9]
---

# Observability & SLO

**Purpose:** the MELT stack, the LLM instrumentation contract, burn-rate alerting, and the SLO/SLA ladder.

**Parents:** BR-008, NFR-012. Cost figures are derived from the R2 envelope (D-3).

## 1. MELT stack

**Rewritten 2026-07-19 (D-33): self-hosted Grafana LGTM stack, on Kubernetes, replaces Grafana Cloud.** This also closes the observability leg of [OI-36](../00-meta/open-items.md) — the live `demo` deployment's Sentry usage was environment drift, not adopted; Grafana LGTM's Loki (logs) + Tempo (traces) + Prometheus (metrics) + Grafana dashboards already cover the MELT surface Sentry would have partially duplicated (browser/frontend error tracking), so no separate error-tracking tool is added.

| Layer | Tool | Purpose |
|-------|------|---------|
| Metrics | Prometheus + Grafana (self-hosted, LGTM stack) / Langfuse (self-hosted) | API latency, error rates, kill-switch propagation, LLM cost, tenant quota |
| Events | PostHog | product analytics, funnel, adoption |
| Logs | Loki (self-hosted, LGTM stack) | structured JSON carrying `tenant_id`, `request_id`, `correlation_id` |
| Traces | OTel → Langfuse (self-hosted) + Tempo (self-hosted, LGTM stack) | agent-session traces; LLM spans via the OTel GenAI semantic conventions |
| Runtime security | Falco (in-cluster) | anomalous syscalls, sandbox-escape attempts — see §3 |

**Retention and sampling:**

| Data | Retention |
|------|-----------|
| Audit telemetry | 90 days hot; 2 years in the MinIO archive. (The canonical hash-chained audit is 7 years, cold, in MinIO) |
| Application logs | 30 days |
| Agent traces | 14 days, sampled at 100% |
| API traces | 7 days — 10% head sampling, plus 100% of errors |

`ops:verify-observability-ttl` runs monthly.

**Self-hosted Langfuse (2026-07-19, D-33) — no DPA required**, since trace data never leaves the platform boundary; this removes the AI-20 Langfuse-DPA prerequisite entirely rather than satisfying it. **Langfuse moved its backend to ClickHouse in January 2026** — informational only, no Dux-side change; the `trace_id`-keyed query pattern this corpus relies on (§2, §4) is unaffected, and self-hosted Langfuse runs the same ClickHouse-backed version.

**Self-hosted Grafana LGTM sizing (supersedes the Grafana-Cloud-paid-tier line, resolves part of SR-17).** The retention figures in the table above (90 days audit telemetry hot, 30 days app logs, 14 days agent traces at 100% sampling) now drive in-cluster storage sizing (Loki/Tempo/Prometheus PVCs) rather than a Grafana Cloud plan tier. **Self-hosted storage capacity is a Gate-1 cost-model line item** — tracked alongside the other infra costs in [pricing-packaging](../80-gtm/pricing-packaging.md), now against the self-hosted-K8s cost baseline rather than a per-seat/per-GB SaaS rate.

## 2. LLM instrumentation contract

**Every LLM call routes through `InstrumentedLLMClient`** (`packages/observability/`). **No ad-hoc SDK calls** — enforced in CI by `no-direct-llm-sdk`.

**OTel GenAI.** The semantic convention is pinned at a specific version (`OTEL_SEMCONV_STABILITY_OPT_IN`) and tracked as **Development-status upstream — not yet stable**; a version bump is a deliberate, tested change, not a silent dependency update. Span name: `chat {provider}`. Required attributes:

`gen_ai.provider.name` (renamed from `gen_ai.system` in semconv v1.37.0) · `gen_ai.request.model` · `gen_ai.request.temperature` · `gen_ai.response.model` · `gen_ai.usage.input_tokens` · `gen_ai.usage.output_tokens` · `tenant.id` · `agent.id` · `session.id` · `prompt_version`

**`prompt_version` (DA-11).** Carried on `CalibrationRecord` and on every trace and span alongside the other required attributes above, so a prompt-registry version can be correlated against accuracy and cost the same way `gen_ai.request.model` is — a prerequisite for extending the existing model-version-pin discipline ([ci-cd-testing §2](../50-engineering/ci-cd-testing.md)) to prompts once a prompt registry exists.

**Prompts and completions are opt-in span *attributes*** (`gen_ai.input.messages` / `gen_ai.output.messages`) **, never unconditional span events** — the span-events pattern is deprecated upstream. The wrapper auto-sanitizes API keys, email addresses, JWTs, and sensitive hostnames regardless of which capture mode is enabled.

**Sampling:**

| Environment | Rate |
|-------------|------|
| Dev | 100% |
| Production LLM | 10–30% head, plus tail sampling on errors and high latency |
| Any span above 10 K tokens | **always sampled** |
| Golden set | 100% |
| Agent APM | 100%, with parent `workflow_id` links |

Langfuse in production runs with `hide_inputs` and `hide_outputs`. **A single `trace_id` spans both Langfuse and Grafana.**

Coverage target: **100%** (TR-NFR-014).

**`replay_trace_id` (DA-12).** Given the hash-chained audit trail (§1) and the shared `trace_id` above, a full agent run — reasoning steps, tool calls, and memory-access events — can be reconstructed from that single `trace_id`: it replays the OTel span tree (agent, tool, and LLM spans, in their original parent/child order) and joins it against the corresponding LLM call records in Langfuse, keyed by the same `trace_id`. This is the capability the `GET /assessments/{id}/replay` endpoint ([application-api §1](../30-api/application-api.md)) surfaces at the API layer; it is a read-time reconstruction, not a stored artifact, so it stays valid for the full trace retention window in §1.

## 3. Cost and safety dashboard

| Panel | Threshold |
|-------|-----------|
| LLM cost per assessment | **above $0.75 → P2.** Sustained **above $0.55 → P2 early warning** |
| Workflow actions per assessment (p95) | SLO <60. Fast burn above 2× baseline within 1 h |
| Workflow actions per tenant per day | a cap breach raises `WORKFLOW_TENANT_BUDGET_EXCEEDED`; L2 at 2× the hourly baseline |
| Golden-set accuracy trend | a regression above 2% is a **P0 merge block** |
| Model structural drift (10% stratified) | above 2σ against the 30-day baseline → P2 |
| Model cost spike, per tenant | above 3× the 7-day baseline → P2 |
| Kill-switch rate (safety anomaly) | above 1% → P1 |
| Abstention labor cost | above $10 K/month → P1 |
| Self-hosted Temporal cluster cost (`dux_cost_temporal_cents`) | added 2026-07-19, D-33 — closes a gap where no SaaS reversal trigger existed to monitor. Panel tracks cluster compute cost against the infra cost model instead |
| Bedrock vs. direct-Anthropic latency (`dux_llm_bedrock_latency_p95`, `dux_llm_anthropic_baseline_p95`) | added 2026-07-19, D-33 — makes the [ADR-017 R3](../20-architecture/adr-index.md#adr-017-r3--multi-provider-claude-inference-path) multi-provider fallback trigger (Bedrock p95 >2× baseline for 7 days) actually enforceable; the pre-cutover baseline was never preserved before this panel existed |
| Valkey cache hit rate (`dux_valkey_hit_rate`) | Gauge, `keyspace_hits / (keyspace_hits + keyspace_misses)` from Valkey `INFO stats`, labeled `tenant_id` + `cache_type` (`llm_response`, `session`, `rate_limit`, `graph_path`, `temporal_activity`). `llm_response` type below 0.6 → P2 — the LLM response cache is the primary cost-reduction mechanism (target 60–80% hit rate); a drop here compounds directly into the LLM cost-spike alert above |
| NATS JetStream consumer lag (`dux_nats_consumer_lag`) | Gauge, `num_pending` from JetStream `CONSUMER.INFO`, labeled `stream` (`VULNERABILITIES`, `ASSETS`, `INVESTIGATIONS`) + `consumer_name` + `tenant_id`. `VULNERABILITIES` stream above 1,000 → P1 (investigation backlog, SLA breach imminent); any stream above 5,000 → P1 (worker starvation, scale immediately) |
| Falco runtime-security alerts (Gate 2) | sandbox-escape attempts, anomalous in-cluster syscalls — any alert is P1, triaged same as a kill-switch anomaly |

**Prometheus metrics:** `dux_cost_llm_cents` · `dux_cost_workflow_actions` · `dux_cost_infrastructure_cents` · `kill_switch_propagation_seconds` · `dux_cost_llm_cents_per_tenant{tenant_id}` · `dux_cost_sandbox_seconds_per_tenant` (Gate 2+) · `dux_cost_temporal_cents` · `dux_llm_bedrock_latency_p95` · `dux_llm_anthropic_baseline_p95` · `dux_valkey_hit_rate` · `dux_nats_consumer_lag`.

`DuxTenantCostCapApproach` fires when hourly spend exceeds `monthly_cap / 720 × 14.4` (the MWMBR form), **or** when raw `rate(dux_cost_llm_cents_per_tenant[1h]) > 2500` — that is, $25/hour.

> **Two divisors, deliberately different.** `720` is a 30-day calendar month, used for the **hourly cap**. The `/730` average-month divisor in the MRR-at-risk formulas ([kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md), [incident-runbooks](../40-ai-safety/incident-runbooks.md)) is for **revenue math**. **Do not unify them.**

## 4. MTTP — time to protection (H9)

Instrumented by Phase-1 exit.

**The marketed outcome is MTTP:** assess → the vendor action executes → exposure drops. **MTXV (<15 min) covers only the first leg.**

Measure it on real partner data, **as a measured metric — not an SLA**:

| Leg | Metric |
|-----|--------|
| Assessment | `assessment_latency` (MTXV) |
| Action | `action_latency` — verdict → `mitigation.executed`, or ticket routed |
| Approval | `approval_latency` — `hitl_request` → `hitl_response`. **Anomaly-escalation path only** |

**Expect a bimodal MTTP distribution, and report the escalated tail separately.** Writes are unattended by default, so the approval leg exists only on anomaly escalation.

**The MTTP distribution is the number a CISO will actually ask for.** It is the same class of gap as the "thousands → tens" measurement: **no end-to-end speed claim is safe without it.**

Connector-freshness and degraded-evidence alerting feed this same dashboard ([taxonomy — H8](../10-product/taxonomy.md)).

**H9 outcome-instrumentation contract (C1/C2, CI-07 closure spec).** `EP-05-F01-T07` backs the "smaller attack surface" (C1) and "shorter path to solution" (C2) claims with the MTTP pipeline above. This is the instrumentation methodology, not the implementation:

| Leg | Start event | End event | Correlation |
|-----|-------------|-----------|-------------|
| Assessment | `assessment.queued` | `assessment.completed` | `assessment_id` |
| Action | `assessment.completed` (verdict crosses an action threshold) | `mitigation.executed` \| `ticket.created` \| `mitigation.blocked` | `assessment_id` → `action_id` (1:N — one assessment can produce multiple actions) |
| Approval | `hitl_request` | `hitl_response` | `assessment_id` → `hitl_request_id`, **anomaly-escalation path only** |

**All three legs share `assessment_id` as the stitching key** — the same ID already on `EXPLOITABILITY_ASSESSMENT` (data-model §2) and the same `trace_id` correlation the replay capability (§2 above) already relies on. No new ID scheme; MTTP is a **read-time join** across three existing event streams (`AUDIT_EVENT`, webhook delivery log, HITL request/response), not a new stored record — same non-duplication principle as `replay_trace_id`.

**Computation:** `MTTP = action_latency` when no approval leg fires (the 3 earned-autonomy actions); `MTTP = action_latency + approval_latency` when it does (`endpoint.isolate`/`patch.deploy_special_devices`, or an anomaly escalation on the other three). **Report both the unconditional distribution and the bimodal split** (§4 above already requires this) — C1/C2 must never be sold as a single blended number, since the two modes differ by whichever HITL wait time dominates.

**Verification:** a fixture test asserting the three-leg join resolves correctly across all 5 canonical actions (mandatory-HITL and earned-autonomy alike), plus a check that `action_id`/`hitl_request_id` never orphan from their parent `assessment_id`.

## 5. SLO burn-rate alerts (MWMBR)

| Alert | SLO | Window | Runbook |
|-------|-----|--------|---------|
| `DuxSLOAvailabilityFastBurn` | API availability | 1 h + 5 m (14.4×) | [Rollback](runbooks.md) |
| `DuxSLOAvailabilitySlowBurn` | API availability | 6 h + 30 m (6×) | Incident response |
| `DuxAssessmentLLMAvailabilityFastBurn` | assessment / LLM path | 1 h + 5 m | [Model outage](../40-ai-safety/incident-runbooks.md) |
| `DuxPromptCacheHitRateDrop` | cache hit rate | >15% drop in 5 m | [Prompt cache](../40-ai-safety/incident-runbooks.md) |
| `DuxTenantCostCapApproach` | per-tenant spend | above 80% of the daily cap | [Token cost](../40-ai-safety/incident-runbooks.md) |
| `DuxWorkflowActionsFastBurn` | actions per assessment | MWMBR | [Token cost](../40-ai-safety/incident-runbooks.md) |
| `DuxAgentBehaviorAnomaly` | behavioral baseline | 2σ in 5 m | [Shadow AI](runbooks.md) |
| `DuxApiLatencyFastBurn` / `SlowBurn` | API p95 <300 ms (TR-NFR-004) | 1 h + 5 m (14.4×) / 6 h + 30 m (6×) | [Rollback](runbooks.md) |
| `DuxAssessmentStartLatencyFastBurn` | assessment start p95 <2 s (TR-NFR-005) | 1 h + 5 m | [Coordination overhead](../40-ai-safety/incident-runbooks.md#r11--coordination-overhead) |
| `Dux3HopCteLatencyFastBurn` | 3-hop CTE p95 <200 ms above 2 K assets (TR-NFR-006) | 1 h + 5 m | [Neo4j reconciliation failure](runbooks.md) — the `neo4j_graph` fallback path |
| `DuxExposureDrilldownLatencyFastBurn` | exposure drill-down p95 <500 ms at 1 K assets (TR-NFR-015 / `ExposureProjection` NFR-013) | 1 h + 5 m | [Coordination overhead](../40-ai-safety/incident-runbooks.md#r11--coordination-overhead) |

**[OI-07](../00-meta/open-items.md) resolved 2026-07-16.** The four latency p95 targets (TR-NFR-004/005/006/015) now page through the four burn-rate alerts above, using the same MWMBR windows as the availability alerts. `DuxAssessmentStartLatencyFastBurn` and `DuxExposureDrilldownLatencyFastBurn` route to R11 because a sustained p95 breach on either is, in practice, the same worker-handoff or redundant-MCP-call pattern R11 already diagnoses; `Dux3HopCteLatencyFastBurn` routes to the existing Neo4j-fallback runbook since a CTE latency breach above the 2 K-asset threshold is the documented trigger for the `neo4j_graph` flag flip (runbooks.md §11).

Recording rules use `rate()` over raw counters — a 15-minute scrape is sufficient for the 1 h / 5 m MWMBR windows. A rollback and kill-switch drill runs monthly against these alert types.

**Low-traffic advisory (AI-82).** Sustained 5xx above 1% over 1 h fires an advisory **regardless of request volume**, validated in staging. **Burn-rate math alone can miss it at pre-seed traffic.**

### Legacy alert names

Renamed in this rewrite. Kept grep-able for old dashboards and runbooks:

| Legacy name | Canonical name |
|-------------|----------------|
| `DuxSLOAvailabilitySlowDrift` (3 d + 6 h windows) | `DuxSLOAvailabilitySlowBurn` |
| `DuxPromptCacheFastBurn` | `DuxPromptCacheHitRateDrop` |
| `DuxWorkflowTenantBudgetExceeded` (hourly tenant actions >2× the 7-day average) | `DuxTenantCostCapApproach` |
| `DuxTokenSpendAnomaly` (>3× the 7-day baseline) | a `DuxTenantCostCapApproach` trigger, in the [token-cost runbook](../40-ai-safety/incident-runbooks.md) |
| `DuxTenantNoisyNeighbor` | the PromQL noisy-neighbor throttle in [multi-tenancy](../20-architecture/multi-tenancy.md) |
| `DuxAssessmentDedupFailure` | dedup and queue-segmentation alert routing (`AssessmentDeduplicationService`) |
| `DuxPhysicalResidentCacheAnomaly` (>15% cache drop, Gate-5 resident agents) | the per-agent-type baseline diff in [agent-identity](../40-ai-safety/agent-identity.md) |
| AWS CloudWatch `AWSThrottledRequests` | connector sync-health alerts in [connector-hub](../10-product/features/connector-hub.md) |

## 6. TR-NFR targets

| ID | Requirement | Target |
|----|-------------|--------|
| TR-NFR-001 | API availability (excluding LLM) | 99.5% monthly |
| TR-NFR-002 | tenant isolation | **zero cross-tenant reads** |
| TR-NFR-003 | kill switch | <5 s p99 |
| TR-NFR-004 | API p95 latency | <300 ms |
| TR-NFR-005 | assessment start p95 | <2 s |
| TR-NFR-006 | 3-hop CTE p95 | <200 ms above 2 K assets |
| TR-NFR-007 | golden-set regression | <2% |
| TR-NFR-008 | feature-flag evaluation | SDK p99 <20 ms; API 99.9% |
| TR-NFR-009 | GDPR export and delete | <24 h |
| TR-NFR-010 | WCAG 2.2 AA | 0 axe-core violations |
| TR-NFR-011 | code-backed audit retention | trace + code, plus execution results at Gate 1 |
| TR-NFR-012 | max agent context | 128 K; checkpoint at 80% |
| TR-NFR-013 | per-tenant LLM cost cap | enforced before intervention |
| TR-NFR-014 | OTel GenAI instrumentation | 100% of LLM paths |
| TR-NFR-015 | exposure drill-down p95 | <500 ms at 1 K assets |

> **TR-NFR-004, TR-NFR-005, TR-NFR-006, and TR-NFR-015 — the four latency p95 targets — now page through the burn-rate alerts in §5** (OI-07, resolved 2026-07-16).
>
> Note also that [pricing-packaging](../80-gtm/pricing-packaging.md) sells figures derived from these targets as a customer SLO — see [OI-06](../00-meta/open-items.md).

## 7. SLA ladder — contractual versus operational

| Tier | Contractual SLA | Operational SLO | Enforceable when |
|------|-----------------|-----------------|------------------|
| Starter | 99.5% (excluding LLM) | 99.5% at seed launch | the per-tenant SLO object is live |
| Professional | 99.9% | 99.9% per tenant, after Gate 2+ | the object is live, and ≥2 SLA contracts exist (SOC 2 A1) |
| Enterprise | 99.99% | 99.99%, with capacity headroom | the object is live; status-page credits if A1 is adopted |

**Rules:**

- A contractual SLA **may** appear in an order form before the operational objects exist — **Legal attaches an activation date tied to Gate 2+ MELT.**
- **No 99.9% figure enters a signed contract until counsel signs** (review AI-226a, target 2026-07-31).
- **LLM provider outages are excluded from the availability numerator** (NFR-012).
- Assessment p95 is <120 s end to end at 1 K assets (AI-217), with a burn-rate alert at 5% of the error budget in 1 h.
