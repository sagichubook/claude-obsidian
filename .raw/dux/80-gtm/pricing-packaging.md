---
owner: GTM
status: canonical
gate: 2
last_reviewed: 2026-07-20
decisions: [D-22, D-38, H5, H9]
---

# Pricing & Packaging

**Purpose:** the tier structure, the outcome-pricing model, and the Phase-1 KPIs.

**Parents:** BR-010.

The tier structure is approved in Phase 1. **List prices activate at Gate 2b GTM sign-off.** Until Stripe SKUs publish, design-partner pricing applies.

## 1. Tiers

| Tier | USD/month | Active from | Asset band | App API rate | SSO | SLA | Support |
|------|-----------|-------------|-----------|--------------|-----|-----|---------|
| Design Partner | $500, or usage-based | Phase-1 pilots (NDA) | — | — | — | beta — **no SLA** | email |
| Starter | $2,500 | Gate 2b list | up to ~1 K | 1,000 req/min | — | 99.5% | email |
| Professional | $8,000 | Gate 2b list | up to ~10 K | 5,000 req/min | included¹ | 99.9%² | email + chat |
| Enterprise | custom | enterprise RFP | unlimited | 10,000 req/min | included | 99.99%³ | dedicated CSM |

**Per-tenant database isolation (2026-07-20, D-38).** A dedicated-CloudNativePG-per-tenant option is offered to Enterprise buyers who require physical isolation beyond the default shared-schema RLS FORCE model ([multi-tenancy.md](../20-architecture/multi-tenancy.md)) — priced and scoped at deal time, on the same enterprise-RFP trigger as the FedRAMP path ([compliance-program.md](../70-governance/compliance-program.md)), not a self-serve SKU.

**¹ SSO entitlement is not SSO delivery.** The contract entitles it; the technical SAML/OIDC work is a **seed trigger**, and US-014 shows a deferral note in Phase 1.

**² 99.9% is a contractual target, not an operating SLO.** Per-tenant operational SLO objects activate at Gate 2+. **LLM provider outages are excluded** (NFR-012). **Counsel must approve the order-form SLA language** (AI-226a).

**³ Enterprise's 99.99% target is a commercial commitment.** The warm-pool / provisioned-concurrency capacity planning needed to underwrite it at the infrastructure layer is tracked in [architecture-overview](../20-architecture/architecture-overview.md) — see that doc for the current state of that work, not here.

**The public data API is a separate plane, with its own limits:** Starter 60, Professional 300, Enterprise negotiated — req/min. See [api-overview §4](../30-api/api-overview.md).

**Pipeline stages by tier.** Starter is Analyze only; Professional adds Mitigate; Enterprise gets all three.

**But that gating is commercial, not technical.** The Analyze pipeline *and* Mitigate/Remediate all ship at Gate 1, for every tier, **unattended by default**. HITL is an anomaly-escalation path — **it is not a tier-gated feature.**

## 2. Data and GDPR

**Export:** Parquet by default, or JSON, via `POST /tenants/{id}/export`. 30-day retention.

**GDPR deletion:** `DELETE /tenants/{id}` → `GDPRDeletionWorkflow`. A one-calendar-month read-only export window (Art. 12(3), Art. 17), then a 90-day hard purge.

**The 90 days is operational retention, not a statutory maximum.**

Subprocessors are listed at `trust.dux.io/subprocessors`.

## 3. Market context

**Forward-looking hypotheses. These are not projections.**

| Measure | Range |
|---------|-------|
| TAM | $8–12 B |
| SAM | $800 M – $1.2 B |
| SOM, years 1–2 | $5–15 M — 50–150 enterprises × $50–100 K ACV |

Category sizing: CybersecTools lists 85 exposure-management tools.

**Every range here is a stage-model illustration. Any external deck must cite a source URL and a date.**

## 4. Outcome-based pricing

**Seed pilot (pre-Gate 2b).** Design partners at $500/month, or usage-based.

**Meters:**

| Meter | Definition |
|-------|-----------|
| `dux.validated_true_positive` | a validated exploitable finding, per the golden-set criteria. **Deduplicated per CVE + asset, per 30 days** |
| `dux.unexploitable_credit` | a finding reclassified as unexploitable after assessment → issues a meter credit |

Finance reconciles monthly, with a 14-day dispute window.

**Gate 2b readiness:** PM and Finance sign off on the algorithm, and **at least one design-partner LOI exists**, before any Stripe SKU publishes.

> **AI-10 — before pricing sign-off.** Finance and the CTO must validate the cost model against **real partner telemetry**, define usage caps and metering, and **confirm that Starter is actually profitable — or explicitly reposition it as a loss-leader with strict limits.** Telemetry collection starts during Phase 1.

**Series A enterprise pricing.** Professional at $50–150 K ACV ($8 K/month list). Enterprise at $150 K+ — the full 3-stage offering post-Gate 3, with an outcome-based option: base + per-validated-true-positive + unexploitable credit.

The professional-services add-on `PS-ONBOARD-001` **blocks $100 K ACV until the trust-portal minimum page set is live.**

**Series B.** Outcome pricing becomes the Enterprise default. **Flag it for revenue recognition and SOX.**

**Renewal drivers.** A TenantHealthScore below 50 triggers a Security/FinOps review. Golden-set drift triggers a contract-amendment discussion.

## 5. Phase-1 KPIs

| KPI | Target |
|-----|--------|
| MTXV | <15 min per CVE |
| Actions per assessment, p95 | **<60** — the canonical SLO ([observability-slo](../60-operations/observability-slo.md)). Governance warns above 100, and halts at 200 |
| MTTR | <72 h (Gate 3) |
| **MTTP** | measured end to end on partner data by Phase-1 exit (H9). **A metric, not an SLA — and distinct from MTTR** |
| Time to value | <48 h |
| Kill switch | p99 <5 s |
| Golden-set regression | <2% |
| Design partners | 2+ by the Gate-1 review (Week 12) |
| Cross-tenant isolation | 100% in CI |

**Product MTTR is not DORA MTTR.** DORA MTTR is incident recovery, targeted under 1 h.

**Reconciled against the engineering figures.** [workflows §5](../20-architecture/workflows.md) records an observed range of **40–80 actions (p50 ≈ 55, p95 ≈ 58)** — the 70–80 tail is a sub-5% case driven by multi-hop attack-path traversal on large asset inventories, and does not move the p95. The p95 <60 figure sold here is consistent with that engineering data.
