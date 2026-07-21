---
owner: Founder
status: canonical
gate: n/a
last_reviewed: 2026-07-19
decisions: [D-3, D-7, D-34, H5]
---

# Lean Canvas

**Purpose:** the one-page business model, with every entry traced to its canonical source.

**No new claims originate here.** Each box links out to the document that owns the fact.

**Tags:** `[V]` = validated · `[H]` = hypothesis. **Review:** quarterly, at the gap-closure workshop.

## 1. Problem

**`[V]`** Security teams drown in vulnerability noise. The design-partner aggregate (N = 3) shows roughly **1,247 critical findings per month against ~15% remediation capacity**, while exploitation windows collapse (M-Trends reports a negative mean time-to-exploit).

**`[H]`** The majority of vulnerability-management time goes to triage rather than remediation. **Validate at N ≥ 5, by Gate 2b.**

→ [vision-reference](../00-meta/vision-reference.md), [customer-lifecycle](../60-operations/customer-lifecycle.md)

## 2. Customer segments

**`[V]`** Primary user: the **security engineer** — the person turning a queue of thousands into tens. Buyer: the **CISO**.

Also: the **AI Safety Lead** (halt authority), and **DevOps/SRE** (who owns the remediation).

Early adopters: 2+ NDA design partners. GTM is US-first.

→ [product-overview](../10-product/product-overview.md), [gtm-guardrails](gtm-guardrails.md)

## 3. Unique value proposition

**`[V]`** **"Safety at Machine Speed."**

Thousands of alerts become a small set of evidence-backed action groups, each with a defensible reasoning chain: *what is actually exploitable here*, and *the fastest path to protection* — with the write path unattended by default.

→ [vision-reference](../00-meta/vision-reference.md)

## 4. Solution

**`[V]`** A three-stage agentic pipeline — **Analyze → Mitigate → Remediate** — live end to end at Gate 1, unattended by default.

Agents **write and execute** investigation code in microVM sandboxes, and re-assess continuously.

→ [product-overview](../10-product/product-overview.md)

## 5. Channels

**`[V]`** Phase 1: Contact-Us, plus the NDA design-partner flow.

**`[H]`** Self-serve PLG opens at Gate 2b — Stripe SKUs, automated provisioning, and KS-L3 tested in production.

→ [operations-overview](../60-operations/operations-overview.md)

## 6. Revenue streams

**`[H]`** Design partner at $500/month or usage-based → Starter $2,500/month and Professional $8,000/month at Gate 2b list → Series A ACV of $50–150 K, or $150 K+.

Outcome-based pricing — per validated true positive, plus an unexploitable credit — becomes the Enterprise default at Series B.

→ [pricing-packaging](pricing-packaging.md)

## 7. Cost structure

**`[V]`**

| Line | Figure |
|------|--------|
| LLM cost per assessment | **≤$0.75 hard, ≤$0.55 design.** D-3 gates: breaker $0.675, CI $0.55 |
| Self-hosted K8s infrastructure (Amazon EKS, incl. CloudNativePG/NATS/Valkey/MinIO/self-hosted Temporal) | **~$1,300–1,800/month for the MVP-scale 3-node cluster** (3× m6i.2xlarge, ~$800–1,200/mo compute + CloudFront/DNS ~$100, per the v4.0 source doc's Gate-1 cost table; EKS control-plane fee ~$73/month additive on top). Replaces the separate Temporal Cloud and ECS Fargate line items entirely — full revision history in [decisions-log](../00-meta/decisions-log.md) |
| Bedrock/LLM tokens (usage-dependent) | **~$500–1,000/month at MVP scale** (v4.0 source doc, direct-Bedrock-SDK cost model) |
| Team | 5 engineers; a 16-week / **2,080 h** envelope (D-7 R1, re-baselined D-23) — see the capacity line below; full backlog history in [decisions-log](../00-meta/decisions-log.md) |

**Capacity.** Backlog stands at **2,118 h against the 2,080 h envelope (~101.8%, 38 h over)** — see [traceability capacity check](../90-execution/traceability.md) and [OI-39](../00-meta/open-items.md), which also tracks the still-unestimated Agentic RAG/Apache AGE net-new scope. Not silently absorbed. Full re-baseline history: [decisions-log](../00-meta/decisions-log.md).

→ [decisions-log](../00-meta/decisions-log.md), [adr-index](../20-architecture/adr-index.md)

## 8. Key metrics

**`[V]`** Targets:

| Metric | Target |
|--------|--------|
| MTXV | <15 min per CVE |
| Time to value | <48 h from connector |
| Queue reduction | thousands → tens |
| Golden set | ≥80% by Gate 1; <2% regression |
| Kill switch | p99 <5 s |
| Design partners | 2+ committed by the Gate-1 review (Week 12) |

→ [pricing-packaging §5](pricing-packaging.md)

## 9. Unfair advantage

**`[H]`** Per-customer **executed investigation code** — consistent, inspectable, repeatable — on top of the CaMeL dual-LLM boundary, RLS isolation, and a hash-chained audit spine.

**Competitors rank findings. Dux proves exploitability, per environment.**

→ [competitive](competitive.md)

## 10. Market sizing

**All forward-looking hypotheses:** TAM $8–12 B · SAM $800 M–$1.2 B · SOM years 1–2 $5–15 M.

**Cite a source URL and a date in any external deck** ([pricing-packaging §3](pricing-packaging.md)).
