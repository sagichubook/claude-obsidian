---
owner: Engineering
status: canonical
gate: 2
last_reviewed: 2026-07-19
decisions: [D-34]
---

# Operations Overview

**Purpose:** what activates the seed operational stage, what triggers each programme, and who is on call.

**Stage:** operational maturity 2 (Days 90–365).

**This is an operational-maturity stage. It is not the December-2025 funding round.**

**Activation.** Gate 2's consolidated criteria. The **mandatory** sub-gates are **Gate 2a** (production infrastructure) and **Cross-cutting** (kill switch L1–L4 tested; `SandboxPort` microVM live, or a CTO-signed defer; SLO alerts; cross-tenant isolation green). **Gate 2b** (GTM and product) and **Gate 2c** (connectors) are **optional** for seed activation.

## 1. Gate 2 consolidated criteria

| Tier | Sub-gates | Blocks seed | Unlocks |
|------|-----------|-------------|---------|
| **Mandatory** | **Gate 2a** — production K8s deploy; observability and on-call named; DR posture. **Plus Cross-cutting** — kill switch tested; **self-hosted Firecracker live** (or a CTO-signed defer); SLO alerts; isolation green | **Yes** | production ops, on-call, runbooks |
| Optional | Gate 2b GTM — Stripe SKUs, list pricing, counsel-approved MSA/SLA. Gate 2b product — Fast Actions manual | No | GTM collateral; manual UI |
| PLG only | Gate 2b PLG (a)–(e) — list pricing + Stripe SKUs; automated provisioning with RLS verification; KS-L3 tested in production; Stripe Tax for EU/UK | No — **blocks public signup only** | self-serve signup |
| Connectors | Gate 2c | No | vendor-dependent screens (already Gate 1 for the W1 set) |

**The hiring plan — roughly 30 developers within 6 months of Gate 2a, 40–50 employees total — is aspirational. It is never a seed-activation blocker.**

## 2. Pre-seed build triggers

Carried from the pre-seed playbook:

| Build-phase event | Activates |
|-------------------|-----------|
| Gate 1 in reach | milestone review ([gate model](../10-product/product-overview.md)) |
| SLO burn during the build | verify alerting before Gate 2 ([observability-slo](observability-slo.md)) |
| An agentic failure is observed | the [incident runbooks](../40-ai-safety/incident-runbooks.md) |
| **An isolation test fails** | **BLOCKS Gate 1** ([multi-tenancy](../20-architecture/multi-tenancy.md)) |
| Cost anomaly, provider outage, MCP spike, 429 cascade, context exhaustion, or cache drop | the named [incident runbook](../40-ai-safety/incident-runbooks.md) |
| An MCP tool registration changes | `aibom-validate` + `mcp-scan` in CI ([mcp-security](../40-ai-safety/mcp-security.md)) |
| Gate 2 passes | activate this seed stage |
| Quarterly | the gap-closure workshop — a claims ↔ capability review |

## 3. Seed triggers

Reconstructed from cross-references (BS-21): the original seed playbook referenced "seed triggers" dozens of times without ever tabulating them.

| Trigger | Starts work on |
|---------|----------------|
| The first enterprise prospect requiring SOC 2 evidence, **or** a Gate 2 consolidated pass | SOC 2 Type I gap assessment and readiness (audit engagement, months 9–12) |
| The first SSO/SAML RFP, or enterprise IdP federation | the [SSO onboarding runbook](runbooks.md) |
| Public API demand from a developer or partner | the Public Data API + developer portal (`api.dux.io/docs`) |
| The first churn-risk signal — a health score below 50 for 2 weeks | CSM EBR + [customer-lifecycle](customer-lifecycle.md) |
| NHI / agent inventory above 500 (`nhi_threshold_500`) | formalize the [NHI policy](../70-governance/compliance-program.md) — Series A |
| The first $100 K ACV, or an enterprise security questionnaire | trust portal content-complete — **a procurement blocker** |
| Gate 3 approval | closed-loop mitigation validation (US-019); amend the SOC 2 scope to cover the validation surface. **Unattended writes are already live at Gate 1** |
| The first EU prospect | EU AI Act Art. 9 (ISO 42001); Azure OpenAI EU routing |
| Throughput ≥500 assessments/day, **or** Temporal spend above $500/month for 30 days | the `WorkflowPort` graduate spike — Restate / Hatchet / DBOS |

## 4. Founder checklist

**P0 — before the first paying customer:**

- PagerDuty on-call, and `#incidents`.
- **All 13 failure modes tested in staging** — the 12 [canonical runbooks](../40-ai-safety/incident-runbooks.md), plus agent quota.
- The shadow-AI runbook (`DuxShadowAI`, 5-minute SLA).
- Platform cost cap, and a quota dry-run.
- Service-catalog Tier 1 owners named.
- **Self-hosted Firecracker live**, or a CTO-signed defer.

**P1 — before the first enterprise RFP:**

- SOC 2 Type I evidence automated.
- Status page live.
- Agent registry complete — `shadow-ai-reconcile` reports `undeclared_count: 0`.
- Pentest completed, or scheduled within 30 days.

**P2 — before Gate 3 preparation:**

- Red-team backlog triaged.
- OpenAPI and the developer portal published.

## 5. Incident roles

The seed table. At 16+ FTE, the Series A delta applies.

| Role | Responsibility | Default assignee |
|------|----------------|------------------|
| Incident Commander | timeline, severity | `@platform-oncall` (primary) |
| Technical Lead | mitigation, rollback | on-call secondary |
| **AI Safety Lead** | **agent halt within 60 s — this role cannot be merged with the IC** | `@ai-safety-oncall` |
| Comms Lead | status page, customer email | Founder or PM before Gate 2; `@product-oncall` after |
| Scribe | timeline and evidence | rotating engineer |

**Closure standard: any P0 or P1 logs all five DORA MTTR phases in PagerDuty before it closes.**

AI incident-response activation is phased: seed covers §1–6; Series A adds the §7–12 deltas; Series B runs the full 12-section template.

## 6. Service catalog

| Service | PagerDuty | On-call | RTO | RPO |
|---------|-----------|---------|-----|-----|
| `dux-api` | `dux-platform` | `@platform-oncall` | 4 h | 1 h |
| `dux-connector-sync` / `dux-workflow` | `dux-platform` | `@platform-oncall` | 4 h | 1 h |
| `dux-web` | `dux-platform` | `@platform-oncall` | 4 h | 4 h |
| `dux-notifications` | `dux-platform` | `@platform-oncall` | 8 h | 4 h |
| `dux-agent` | `dux-ai` | `@ai-safety-oncall` | 8 h | 4 h |
| `dux-sandbox` | `dux-ai` | `@ai-safety-oncall` | 8 h | 4 h |

Service names follow the Kubernetes (EKS) Deployment topology of ADR-006 R4.

**Gate 2 activation checklist:** verified PagerDuty service-directory URLs for every row, plus SRE and CTO sign-off before any enterprise RFP.

## 7. Sub-documents

[Runbooks](runbooks.md) · [Observability & SLO](observability-slo.md) · [DR / BCP](dr-bcp.md) · [Customer lifecycle](customer-lifecycle.md)
