---
owner: Founder
status: backlog-shell
gate: n/a
last_reviewed: 2026-07-19
decisions: [D-34]
---

# Series B Scale Programs

**Purpose:** the Series B governance surface — ERM, TPRM, data sovereignty, multi-region DR, and the enterprise platform.

**Stage:** Series B (Year 3–5+).

> **This document is a backlog shell.** Most sections are planning placeholders. **Inherit the Series A artifacts until a section is content-complete.**
>
> **Do not execute** the regulated-industry annex, the multi-region game-day gates, or the 12-section AI incident-response playbook from a stub section alone. **Where a P0 gate is inherited, counsel must sign a risk acceptance.**

## 1. Section maturity

Check this table before relying on anything below.

| Maturity | Sections |
|----------|----------|
| **Ready** | ERM risk appetite; data-room index |
| **Partial** | TPRM; internal audit; data residency; global privacy; multi-region DR; the AI IR outline; SOX ITGC; certifications |
| **Stub** | company signals; triggers; board reporting; enterprise platform; CCM; chaos; vulnerability disclosure; purple team; continuous red team; model drift; the AI ethics board; EU AI Act |

## 2. Governance and risk

### ERM risk appetite — *ready*

Current policy. **Exempt from the backlog-shell caveat.**

| Domain | Appetite |
|--------|----------|
| Innovation, in non-production | **medium** risk accepted |
| Tenant isolation, agent safety, financial integrity | **low** |
| **Unmitigated cross-tenant access, or uncontrolled agent autonomy** | **none. No risk accepted, at any level.** |

**Risk scoring** is Impact × Likelihood, on a 1–25 scale:

| Score | Handling |
|-------|----------|
| 20–25 | critical — goes to the board |
| 15–19 | high — a 30-day plan |
| 8–14 | medium — monitored |
| 1–7 | low — accepted |

Internal audit runs an annual Q1–Q4 plan, with a findings template. The audit-committee charter is still open.

### TPRM — *partial*

Inherits Series A until content-complete.

| Tier | Definition | Vendors |
|------|-----------|---------|
| Tier 0 | no data; annual attestation | — |
| Tier 1 | tenant or financial data | `V-OAI` OpenAI (GPT models, direct API) · `V-STR` Stripe · `V-CF` Cloudflare (DNS/CDN/edge WAF only) · `V-ANT` Anthropic (direct API, multi-provider fallback leg — ADR-017 R3) |
| Tier 2 | critical path; reviewed quarterly | `V-AWS` — full-platform runtime subprocessor; Kubernetes runs on Amazon EKS ([ADR-006 R4](../20-architecture/adr-index.md#adr-006-r4--deployment-topology)). `V-GRAF` Grafana Cloud/Langfuse Cloud are **not** subprocessors — both are self-hosted in-cluster, so neither receives external data. Full revision history: [decisions-log](../00-meta/decisions-log.md). |

Vendor register IDs carry over from the interim subprocessor list, reviewed annually.

## 3. International and data sovereignty — *partial*

**Inherits the Series A pseudonymization and topology interim. This is not yet adopted policy.**

**Flagged, not resolved this pass (D-34 judgment call b).** The Azure OpenAI EU/APAC/LATAM rows below predate [ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer)'s LiteLLM removal, and the v4.0 source doc that drove this pass's other changes never mentions Azure OpenAI — it may be an orphaned regional-routing leg now that LiteLLM's multi-provider routing is gone, or an intentionally-retained detail out of that doc's scope. Left as-is pending explicit Founder confirmation; see [decisions-log D-34](../00-meta/decisions-log.md).

| Region | Default | Topology | LLM | Sign-off gate |
|--------|---------|----------|-----|---------------|
| US | default | US-East primary | OpenAI US (GPT) / AWS Bedrock US (Claude) | Gate 2+ |
| EU | tenant pin (GDPR) | EU primary + replica | Azure OpenAI EU | **before the first EU-pinned tenant** |
| APAC | deferred | Singapore / Australia target | Azure OpenAI APAC | before the first APAC tenant; inherit the Series A pseudonymization interim |
| LATAM | deferred | Brazil shortlist + an LGPD DPIA | Azure OpenAI Brazil | 30 days before any LATAM contract. **No clause without Legal and Engineering pre-approval** |

Tenants are pinned at provisioning. **No cross-region read without a legal basis.** Adequacy is revalidated annually.

**Global privacy.** A RoPA per jurisdiction — CCPA, GDPR, UK, APPI, LGPD, PDPA-SG.

**Breach notification clocks:**

| Jurisdiction | Clock |
|--------------|-------|
| EU / UK | 72 h |
| US | per state |
| LGPD (Brazil) | "reasonable" |
| PDPA (Singapore) | 72 h |
| India DPDP | 72 h |
| Australia | 30 days |

**Procedure:** Legal is engaged within 30 minutes; PM signs off; **the regulatory and contractual clocks are tracked separately**; forensics land in R2; counsel confirms the filing.

## 4. Multi-region DR — *partial*

**The RTO/RPO ladder below is a target. It is not yet operational.**

Stage ladder: Seed at 4 h / 1 h → **Series B targets <1 h active-active**, and **<15 min at scale**.

**EU and APAC go-live are blocked on a P0 game day, or a signed risk acceptance.**

Under ADR-006 R4, Kubernetes on EKS ships from Gate 1, so the former "Railway → AWS cutover before FedRAMP / Fortune 500" gate **is already satisfied**. The Series B trigger is region topology and active-active capacity — **not a hosting migration**.

## 5. Enterprise platform and AI governance at scale — *stub*

**A planning placeholder. Not adopted policy.**

**Outcome-based pricing** becomes the Enterprise default: a base fee, plus a charge per validated true positive, plus an unexploitable credit. **Flag it for revenue recognition and SOX.**

**Partner requirements:** SOC 2 or ISO minimum; a DPA; blast-radius alignment; an annual questionnaire.

**Marketplace ISV:** 3% for AWS public, 1.5–3% private, plus 0.5% CPPO. Support tiering: the ISV holds T1; Dux holds T2 and T3. Requires a `third-party-isv` row in the agent catalog.

**Continuous red teaming:** CI fuzzing on every config change; monthly novel-injection testing; a quarterly compromise simulation; an annual external report.

**Continuous control monitoring thresholds:**

| Signal | Severity |
|--------|----------|
| RLS check — any failure | P2 |
| Access review older than 90 days | P1 |
| A critical CVE open more than 7 days | P1 |
| Kill-switch burn of 5% in 1 h | **P0** |
| MCP denial burn | per burn-rate policy |

The eBPF escape-detection runbook is still open.

> **A gVisor-based sandbox (or an interim, signed risk acceptance) is P0 compliance debt at Series B.** It requires the CTO and CISO to **re-sign**, and any slip must be disclosed.

## 6. AI incident response — *partial, reference-only*

The 12-section outline:

detection → triage → containment (tenant scope) → agent halt (KS-L2/L4) → forensics → internal comms and war room → customer notification → recovery and safe restart → postmortem → evidence and audit → regulatory notification → retest and golden-set regression.

**This stays reference-only until all four hold:**

1. The Series B triggers have fired.
2. The sections are content-complete — **not stubs**.
3. **One full tabletop has passed, with S3 evidence.**
4. The PagerDuty escalation URLs are real, **not placeholders**.

## 7. IPO / M&A / certifications

**Data-room index — *ready*:** corporate, financial, technical and security, SOX ITGC, EU AI Act (conditional), people.

**SOX ITGC — *partial*:** readiness links to the Series A observation window.

**EU AI Act — *stub*:** Art. 9 via ISO 42001 (Annex III horizon, 2 Dec 2027 — D-26); Annex IV deferred; quarterly counsel review.

**Certification triggers:**

| Certification | Trigger |
|---------------|---------|
| ISO 42001 | an EU RFP |
| FedRAMP | a federal RFP. **Board approval is required before responding.** Needs an SSP and a boundary diagram |
| HIPAA | a PHI prospect. Data-flow mapping within 60 days |
| PCI | cardholder data beyond Stripe tokenization |
| ISO 27001 | enterprise demand |

**Runbook reference legend:** Execute · Reference-only · SSoT.
