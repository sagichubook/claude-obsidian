---
owner: Founder
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-3, D-4, D-7, D-17, D-36, H1, H4, H5]
---

# Product Overview

**Purpose:** what Dux is, what ships at Gate 1, and what is deliberately fenced beyond it. **Parents:** all BRs · **Canon:** ideal-state v2.2 · **Marketing language:** [vision-reference](../00-meta/vision-reference.md).

## 1. Thesis

Dux is a **per-environment, attacker-minded reasoning system**. It decides what is *actually exploitable here*, and the *fastest path to protection*.

It runs at machine speed for analysis and re-assessment. It is personalized per customer through coding agents rather than static rules. It rides third-party frontier models. **It is defensive only.** And it takes governed write actions on the customer's security stack. Three of five actions are unattended by default, with human review as the anomaly-escalation path; the other two (`endpoint.isolate`, `patch.deploy_special_devices`) are gated to mandatory human approval until each earns unattended execution via a field-proven safety record (D-17). Of the three, `network.blocklist_add` and `ticket.create_remediation` are live at Gate 1; `policy.deploy_device_config`'s unattended posture activates only once its Gate-3/W2 Intune connector ships.

Five delivery pillars:

| Pillar | Delivered by | Canonical spec |
|--------|-------------|----------------|
| **A** — Moat: World Model, eval, personalization | World Model, investigation-code artifacts, preference engine, golden-set eval | [data-model](../20-architecture/data-model.md), [taxonomy](taxonomy.md), US-017 / US-009 |
| **B** — Safety scales with autonomy | governance kernel, kill switch, CaMeL, anomaly-escalation HITL | [40-ai-safety](../40-ai-safety/safety-overview.md) |
| **C** — Claim ↔ capability firewall | claims traceability, gate-safe copy | [gtm-guardrails](../80-gtm/gtm-guardrails.md) |
| **D** — Isolation + compliance invariants | RLS FORCE, composite keys, SOC 2 / ISO | [multi-tenancy](../20-architecture/multi-tenancy.md), [compliance-program](../70-governance/compliance-program.md) |
| **E** — Extensibility spine | the 8-part contract, catalogs-as-manifest | [catalogs](catalogs.md) |

## 2. Core capabilities

All eight are live at Gate 1. Write actions execute unattended by default for 3 of 5 canonical actions; `endpoint.isolate` and `patch.deploy_special_devices` require mandatory HITL until they earn unattended execution (D-17). Only **preference learning** and **physical residency** remain fenced.

**The operating principle (GCIS v2.2):** close every claim↔capability gap by **raising the design to deliver the claim, not by narrowing the claim**. A promoted capability ships in Phase 1 with best-practice architecture plus governance-kernel, kill-switch, and audit controls. Staging is retained only where earlier delivery is physically impossible — insufficient data volume, an unsigned contract, or a safety record that does not yet exist.

| # | Capability | BR | Claims | Gate-1 delivery | Fenced beyond Gate 1 |
|---|-----------|----|--------|-----------------|----------------------|
| 1 | AI-driven exploitability analysis — exploitable vs merely reachable | BR-002 | A1, A2, A4, A9, D1 | Full: prerequisite analysis, environmental evidence, executed investigation code in traces | — |
| 2 | Vulnerability → asset → control relationship mapping | BR-002 | B6, C5 | Attack paths + AWS security groups + vendor control panels (CrowdStrike live; Intune at Gate 3/W2) | — |
| 3 | Lightweight mitigation using the existing stack | BR-002 | C4 | Unattended-by-default action cards (US-004, US-016) | — |
| 4 | Configuration-change recommendations vs patching | BR-002 | C3, C4 | Control refinements live (US-005) | Closed-loop validation → Gate 3 |
| 5 | Remediation acceleration via AI agents | BR-002 | C3, B2 step 4 | Ticket create + route, unattended (US-018) | Closed-loop automation → Gate 3 (US-019) |
| 6 | Automated asset tagging and ownership | BR-004 | B2 | Ownership inference live — ServiceNow, Entra (US-007) | Preference-driven refinement → Gate 2c |
| 7 | Multi-source data aggregation | BR-004 | B2 step 1, B6, B3 | AWS + NVD/KEV/EPSS + CSV + ≥3 vendor connectors | Full 42-source taxonomy → waves W2/W3 |
| 8 | Exploitability-based prioritization | BR-002 | D1, C7 | Mitigation queue + exposure states | Preference learning → Gate 2c |

Claim-ID verdicts against delivery status: recorded in [decisions-log](../00-meta/decisions-log.md) (claims-implementation audit findings absorbed 2026-07-16; source file removed, see [open-items](../00-meta/open-items.md) OI-27).

**The agent's operational loop — four steps, all Gate 1:**

1. Continuously analyze vulnerabilities across connected environments.
2. Determine whether existing tools and configuration already block the attack path.
3. Surface lightweight mitigations that are faster than a full patch.
4. Route focused remediation to the right stakeholders.

Steps 3 and 4 execute unattended by default for `network.blocklist_add`/`policy.deploy_device_config`/`ticket.create_remediation`, human review firing only on anomaly escalation; `endpoint.isolate`/`patch.deploy_special_devices` require mandatory HITL on every call (D-17).

## 3. Personas

**The agent persona — Dux Agent.** An "Aggressive Exposure Management Specialist": calm, logical, humble, transparency-focused, and **citation-first** — every exploitability claim references a permitted source (NVD, asset inventory, control evidence).

**Human personas:**

| Persona | Goal | Primary stories | Degraded path without connectors |
|---------|------|-----------------|----------------------------------|
| Security engineer (primary user) | Cut the actionable queue from thousands to tens | US-001, US-010, US-011, US-008 | AWS + NVD live; vendor panels show connector-degraded empty states |
| CISO / security leader (buyer) | Board-ready, validated risk reduction | US-006, US-012 | Donut and delta metrics only |
| AI Safety Lead | Agent halt authority (<5 s kill switch) | US-014 | — |
| DevOps / SRE | Fix without breaking deploys | US-007, US-018 | Webhooks delayed |
| Tenant admin | Users, connectors, agent policy, export | US-013, US-014 | AWS wizard + CSV fallback; SSO deferral note |
| API consumer | Phase 1: assessments and webhooks (JWT). Seed: public data API | US-014, US-024 | Poll `GET /v1/vulnerability-instances` once the Seed API is live |

## 4. Navigation → user-story map

The eight-icon sidebar:

| Nav | Page title | User stories |
|-----|-----------|--------------|
| Dashboard | Home / Exposure Overview | US-012 (+ US-006 widgets) |
| Apps | Connector Hub | US-013 |
| Security | Investigation workflow (stepper) | US-001; US-002–007, US-009 |
| Exposure | Exposure Analysis / CVE Detail | US-011 (+ US-017 panel) |
| Mitigation | Research Dashboard / Vulnerability Reduction | US-010 |
| Fast Actions | One-Click Mitigations | US-016 |
| Settings | Tenant Administration | US-014 |
| Help | Help & Support | US-015 |

Feature specs live in `features/`. **The nav-label vs page-title split, and the Mitigation-nav vs Mitigate-stage distinction, are canonical in [taxonomy](taxonomy.md)** — they are easy to conflate, and must not be.

## 5. Phase-1 gate model

| Gate | Meaning | Exit criteria |
|------|---------|---------------|
| Vertical slice (Week 6) | One CVE → one agent → one conclusion → one UI view | SIGKILL durability (zero duplicate external effects; the LLM step is not re-sampled); governance gates live (Intent, Budget, ActionBudget, CostForecast); 2+ design partners onboarded; 10 test CVEs with traces |
| **Gate 1 review (Week 12)** | Full Phase-1 pipeline, hardened | >80% golden-set accuracy on a held-out set of **(CVE × environment) pairs** (H1); zero cross-tenant fuzz reads; zero Critical findings in the adversarial suite; **<$0.75 per workflow** (D-3); 2+ committed design partners; anomaly-escalation approve/deny surface live with impact preview (H4); LLM09 citation gate green |
| Phase-1 exit (Week 16) | Production beta | Customer data flowing for 2+ weeks; sustained >80% held-out accuracy; false negatives <5%; OWASP LLM and Agentic assessments at Partial or better; actionable-queue ratio and MTTP measured on real partner data |
| Gate 2a / 2b / 2c | Seed activation / GTM / vendor-screen expansion | See [operations-overview](../60-operations/operations-overview.md) |
| Gate 3 | **Closed-loop mitigation validation (US-019)** | A field-proven action-safety record. Unattended write *execution* is already Gate 1 |
| Gate 5 | Optional physical residency | A signed on-prem contract |

**Release milestones (16-week calendar, D-7 R1):**

| Week | Milestone |
|------|-----------|
| 2 | Infrastructure skeleton + isolation harness |
| 4 | PgBouncer pooling fuzz; `test:fuzz-tenant-id` becomes merge-blocking |
| 6 | Vertical slice + **the EU AI Act counsel opinion, before any EU prospect outreach.** This blocks EU tenant provisioning until the opinion and classification memo are on file. At `phase_1_epoch` 2026-06-23, Week 6 falls near 2026-07-28 — ahead of the 2 Aug 2026 Article 50 transparency deadline ([compliance-program](../70-governance/compliance-program.md), D-26) |
| 8 | Internal dogfood (2 tenants, isolation pass); internal `/api/docs` (Redoc, shared with NDA partners); HITL approval API; **the Langfuse DPA — this blocks production traces** |
| 12 | Gate 1 review + minimal HITL approve/deny UI |
| 14 | `chat_write_tools` + full chat HITL UI |

**Abort rule:** switch the inner framework if the golden set regresses by more than 2%.

## 6. Capacity

**Capacity resolved (D-19, D-23).** The backlog consumes 2,040 h against the re-baselined 2,080 h envelope (~98%, a 40 h buffer) — 26 focused h/week instead of 25, same 5-engineer team, no sixth hire. Gate-1 Week 12 and exit Week 16 timing are unchanged. See the capacity check in the [execution backlog](../90-execution/traceability.md).

## 7. Explicitly out of scope for Phase 1

| Item | Where it lands |
|------|----------------|
| Closed-loop mitigation validation / post-fix re-verification (US-019) | Gate 3 — note that unattended write *execution* is in scope at Gate 1 |
| Preference **learning** refinement | Gate 2c |
| Azure / GCP discovery | Phase 2 |
| Enterprise SSO / SCIM | seed trigger |
| OT / IoT discovery | Phase 2+ |
| On-prem / air-gapped physical residency | Gate 5 |
| Predictive risk forecasting (US-028, BR-013) | Gate 2 (funded, D-36) |
| Financial-impact quantification | Phase 3 |
| Native mobile app | Series A |
| **Scanner replacement, and PTaaS** | **permanent non-goals** |

## 8. Canonical end-to-end path (demo / POC)

US-012 Dashboard → US-013 AWS connector (or CSV) → US-010 Request Research → US-011 Exposure Analysis → US-017 trace, showing the reasoning and the executed code → optionally US-008 Chat Guidance.

The story it tells is "thousands → tens": scanner findings become a small set of evidence-backed action groups.
