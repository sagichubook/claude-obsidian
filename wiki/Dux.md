---
type: area
title: "Dux"
created: 2026-07-22
updated: 2026-07-22
review_cadence: weekly
tags: [area, dux, dux/hub]
status: evergreen
related: ["[[index]]", "[[Dux Product Guide]]", "[[Dux Architecture Guide]]", "[[Dux AI Safety Guide]]"]
sources: [".raw/dux/README.md", ".raw/dux/00-meta/vision-reference.md", ".raw/dux/00-meta/decisions-log.md", ".raw/dux/10-product/product-overview.md"]
---

# Dux

Navigation: [[index]]

---

## What Dux is

**Dux** is an agentic exposure management platform built for a world where vulnerability exploitation windows are collapsing from weeks to minutes. Use **Dux Agent** to determine what's actually exploitable in your environment, identify the fastest path to protection, and eliminate exposure before it's used against you. We bring exploitability, context, and action together — so you achieve **safety at machine speed**.

The pipeline is three stages — **Analyze → Mitigate → Remediate** — live end to end at Gate 1, **unattended by default**. Human review is an anomaly-escalation path, not a gate on every write. Agents **write and execute** investigation code in self-hosted Firecracker microVM sandboxes, and re-assess continuously as threat intel and the environment change.

**Defensive only. Never PTaaS. Never a scanner replacement.**

This page is the single front door into the full knowledge base: what the product does, how it's built, how it stays safe while acting with real autonomy, and how the business around it runs.

---

## Positioning & taglines

- **Category:** Agentic Exposure Management
- **Primary tagline:** *Safety at Machine Speed*
- **Alternate marketing copy:** *Safety Faster than the Speed of Strike*
- **Stage headline (dux.io):** *If it's not exploitable, why fix it?*
- **Contact-section header (dux.io):** *Faster Safety Starts Here*
- **Naming:** Public "AI-workers" / "Dux AI agents" = **Dux Agent** (canonical product name). Legal entity: **Dux, Inc.** (press "Dux Technologies Inc." is non-canonical) — confirmed (D-51).

### Canonical hero copy

> Dux is an agentic exposure management platform built for a world where vulnerability exploitation windows are collapsing from weeks to minutes. Use **Dux Agent** to determine what's actually exploitable in your environment, identify the fastest path to protection, and eliminate exposure before it's used against you. We bring exploitability, context, and action together — so you achieve **safety at machine speed**.

### Hero / CVE-drop scenario

> When a zero-day drops, your scanners flag thousands of instances. Teams spin up **Dux Agent** to investigate which are **actually exploitable in your environment** — with evidence for controls already blocking the path. Stop drowning in alerts. Start focusing on what actually matters.

### Gate-safe context line

*Not exploitable = not urgent. Context changes everything.* (CVSS 9.8 doesn't mean critical in **your** environment; EPSS is an input badge, not the verdict.)

### Problem narrative

For years, vulnerability management was optimized for visibility — more dashboards, better reports, cleaner ticket flows, and incremental "rank your backlog better" tools that still leave teams drowning in noise. When vulnerabilities move from disclosure to exploitation in hours, visibility isn't the problem anymore; **speed is**. Security teams fall behind because they can't quickly tell what's actually exploitable, what's already blocked, and what really needs action. Moving data around doesn't solve that. Understanding true exposure does. Attackers weaponize vulnerabilities in hours, not days; autonomous AI agents are now documented executing attacks end-to-end (Anthropic, 2025); most teams still work through week-old backlogs — a structural problem, not a process problem.

---

## The "thousands to tens" story

The story Dux tells is **"thousands → tens"**: scanner findings become a small set of evidence-backed action groups. The agent takes a CVE plus a customer's live environment evidence — assets, runtime, identity, network, existing controls — and works out what's genuinely exploitable and the fastest way to shut the door on it.

---

## Company facts

**Dux, Inc.** — founded 2024, dual-headquartered in Tel Aviv, Israel (R&D) and New York, USA (go-to-market).

| Fact | Detail |
|------|--------|
| **Founded** | 2024 |
| **HQ** | Tel Aviv, Israel (R&D) + New York, USA (GTM) |
| **Funding** | $9M seed round (Dec 2025) — Redpoint, TLV Partners, Maple Capital. Funding complete, no undisclosed SAFEs (counsel-confirmed 2026-06-30) |
| **Angel investors** | Individuals from CrowdStrike, Okta, and Armis (BusinessWire, Dec 2025) |
| **Founders** | Or Latovitz (CEO), Amit Nir (CPO), Nadav Geva (CTO) — Talpiot alumni (shared cohort adjacent to Wiz/Cyera founders), recipients of the Israel Security Prize and the Prime Minister's Prize for cyber/AI work at the PMO and Unit 8200 |
| **Headcount** | 20 employees at Dec-2025 launch (CEO, bizportal); 24 as of 2026-07-20 (LinkedIn) |
| **Leadership hires** | Eden Amitai — Head of Growth and Alliances (self-announced on LinkedIn, undated) |
| **Design-partner ICP** | Finance, healthcare, technology |
| **Markets** | U.S.-first GTM; Europe medium-term (FinSMEs CEO interview); Israel is the R&D base, not a served market |
| **Revenue model** | Subscription SaaS, B2B, tiered plans ([pricing-packaging.md](../80-gtm/pricing-packaging.md)); ACV language implies annual contracts |

> **Note:** Armis is also listed as a named competitor in [[Dux GTM Guide]] — see the disclosure note there. The CEO's "deep-learning MSc TAU" credential could not be re-corroborated (2026-07-21 web pass) — treat as unverified pending a citable source.

---

## Three-stage public framing

The dux.io site is structured around three buyer stages, each answering a different question:

| Stage | Public headline | Buyer questions on site |
|-------|-----------------|-------------------------|
| **Exploitability Analysis** | If it's not exploitable, why fix it? | What's needed for the exploit? Can it actually be exploited? Where am I already protected? |
| **Lightweight Mitigations** | Apply mitigations that get you to safety | Can I mitigate this quickly? What can I do with my existing stack? Am I still protected? |
| **Remediation Acceleration** | Speed up remediation when required | Who should remediate? What should I remediate? How can I customize this? |

**Public outcomes (BusinessWire, dux.io):** materially smaller attack surface; far shorter path from vulnerability discovery to resolution; less time chasing noise in the research queue; reactive exposure triage → proactive defensive strategy.

**Additional dux.io site copy:** "Not every gap needs a patch" (Lightweight Mitigations stage framing); compensating controls / stopgap measures that close critical gaps instantly while waiting on a patch; "works with your existing stack, no new purchases" (no new purchase required); Remediation Acceleration stage explicitly names bottleneck elimination — "blows past spotty data, unclear ownership, legacy systems" — and legacy system handling as a named capability.

---

## Product thesis

Dux is a **per-environment, attacker-minded reasoning system**. It decides what is *actually exploitable here*, and the *fastest path to protection*.

It runs at machine speed for analysis and re-assessment. It is personalized per customer through coding agents rather than static rules. It rides third-party frontier models. **It is defensive only.** And it takes governed write actions on the customer's security stack. Three of five actions are unattended by default, with human review as the anomaly-escalation path; the other two (`endpoint.isolate`, `patch.deploy_special_devices`) are gated to mandatory human approval until each earns unattended execution via a field-proven safety record (D-17).

---

## Five delivery pillars

| Pillar | Delivered by | Canonical spec |
|--------|-------------|----------------|
| **A** — Moat: World Model, eval, personalization | World Model, investigation-code artifacts, preference engine, golden-set eval | [[Dux Architecture Guide]], [[Dux Taxonomy & Catalogs]] |
| **B** — Safety scales with autonomy | governance kernel, kill switch, CaMeL, anomaly-escalation HITL | [[Dux AI Safety Guide]] |
| **C** — Claim ↔ capability firewall | claims traceability, gate-safe copy | [[Dux GTM Guide]] |
| **D** — Isolation + compliance invariants | RLS FORCE, composite keys, SOC 2 / ISO | [[Dux Architecture Guide]], [[Dux Governance & Compliance Guide]] |
| **E** — Extensibility spine | the 8-part contract, catalogs-as-manifest | [[Dux Taxonomy & Catalogs]] |

---

## Core capabilities (Gate 1)

All eight capabilities are live at Gate 1. Write actions execute unattended by default for 3 of 5 canonical actions; `endpoint.isolate` and `patch.deploy_special_devices` require mandatory HITL until they earn unattended execution (D-17). Only **preference learning** and **physical residency** remain fenced.

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

### The agent's operational loop — four steps, all Gate 1

1. Continuously analyze vulnerabilities across connected environments.
2. Determine whether existing tools and configuration already block the attack path.
3. Surface lightweight mitigations that are faster than a full patch.
4. Route focused remediation to the right stakeholders.

Steps 3 and 4 execute unattended by default for `network.blocklist_add`/`policy.deploy_device_config`/`ticket.create_remediation`, human review firing only on anomaly escalation; `endpoint.isolate`/`patch.deploy_special_devices` require mandatory HITL on every call (D-17).

---

## Claims inventory (Dec 2025 press, canonical IDs)

> **Nothing here is an engineering commitment.** This file records what was said in public, and when. Every claim maps to the [[Dux Decisions & Traceability Reference]], and to the gap list in §Vision-vs-implementation gaps.
>
> **Scope of the claims map (D-10).** It binds **GTM copy, product naming, and UI strings.** It does **not** bind safety posture, control design, gate criteria, or SLOs.
>
> Where a claim here and engineering reality diverge, that divergence is raised in [[Dux Governance & Compliance Guide]]. **It is never resolved by editing a spec to match the claim.**

### North star (attacker-minded defense)

| ID | Claim | Source |
|----|-------|--------|
| A1 | Help teams understand **what attackers can exploit in the real world** | bizportal; tv10; calcalist |
| A2 | Equally: what is **not exploitable** and **wastes team time** | bizportal |
| A3 | **Fix weakness before it is exploited** | bizportal |
| A4 | Determine **what is actually possible for an attacker** in a specific environment | bizportal |
| A5 | Close gap: generic *"there's a vulnerability"* → *"dangerous in our environment + how to prevent"* | geektime |
| A6 | **Imitate a skilled analyst** who knows how weaknesses behave and cleans backlogs | bizportal CPO |
| A7 | **Think like a human would solve it** — not another backlog ranker | bizportal vision |
| A8 | Apply **logic across every vulnerability and asset, every time** | bizportal (Nir) |
| A9 | Orgs want **what an attacker can actually exploit** — decades of "rank better" failed | geektime |

### How (agent platform + loop)

| ID | Claim | Source |
|----|-------|--------|
| B1 | **Agent-based threat-management platform** | all four Hebrew-press articles |
| B2 | **Four-step loop:** analyze → check controls block paths → lightweight mitigations → route stakeholders | bizportal |
| B3 | **Periodic scans + manual triage → continuous agent-based research** | bizportal; tv10; calcalist |
| B4 | **Autonomous analyst** investigating vulnerabilities at **machine speed** | geektime |
| B5 | **Zero-day investigated across environment within minutes** | bizportal (Geva) |
| B6 | Three pillars: **exploit-path analysis**, **tuning existing defense tools**, **continuous threat management** | bizportal; tv10 |
| B7 | **Machines that think like humans** enable a fundamentally **different solution** than ranking | bizportal |

### Outcomes

| ID | Claim | Source |
|----|-------|--------|
| C1 | **Much smaller attack surface** | bizportal — illustrative/unmeasured until Phase-1 exit (H9) |
| C2 | **Shorter path** from vulnerability identification to solution | bizportal — illustrative/unmeasured until Phase-1 exit (H9) |
| C3 | **Fastest and safest fix:** config change → existing tool update → full patch | bizportal |
| C4 | **Lightweight mitigations faster than full patch** | bizportal |
| C5 | **Tuning existing defense tools** in customer context (not replacing stack) | bizportal; tv10 |
| C6 | **Game changer** — exploitability vs vulnerability lists | tv10; Segev |
| C7 | **Breakthrough** agentic capability in cyber defense | bizportal |

### What we are not

| ID | Claim | Source |
|----|-------|--------|
| D1 | **Not another "rank your backlog better" tool** | bizportal |
| D2 | **No unified prioritization logic for all customers** — per-customer via coding agents | bizportal |
| D3 | **No static rules** — per-environment personalization ("secret sauce") | geektime |
| D4 | Agents on **third-party AI models** (not a proprietary-model claim) | geektime |
| D5 | **Defensive platform only** — not PTaaS / not offensive execution | playbook invariant |

### Aspirational

| ID | Claim | Source |
|----|-------|--------|
| E1 | Orgs **feel like experts working for them 24/7** | bizportal |
| E2 | Solve exposure management **from the root** — different than how the industry thought about it | bizportal |
| E3 | **Long-term:** expand beyond patching-only "1000 lb hammer" | Redpoint + articles |
| E4 | **Predictive risk forecasting** — "which assets are likely to become risky in the future" | FinSMEs CEO interview (Dec 2025); **committed roadmap capability per the 2026-07-13 claims-alignment directive** (live press claim binds the corpus, per MC-07) |

**E4 specced** (resolves OI-09, D-24, 2026-07-16). BR-013 → EP-03 → US-028 → FR-030, Gate 2. It answers the claim literally, as an evidence-trend computation (EPSS/finding-count/control-coverage deltas) — not a new ML model or a composite score, consistent with how the rest of this corpus treats risk scoring. **Implementation hours are not yet in the backlog** — the spec-authoring task (`EP-03-F04-T01`) is closed by this; sizing the build is separate, unbudgeted work.

### Why now (market context — never product capability claims)

| ID | Claim | Source |
|----|-------|--------|
| F1 | Mandiant mean time-to-exploit is negative and worsening: **~−1 day (2024), ~−7 days (2025)** | Confirmed verbatim in Mandiant's own M-Trends 2026 (Google Cloud blog). The previously-paired "32 → 5 day" framing is dropped (D-53) — that starting point traced only to a different, non-Mandiant source (CyberMindr) describing "weaponization" time, a different metric |
| F2 | Anthropic-documented **fully autonomous AI cyber-espionage attack** (planning through execution) | Anthropic 2025, cited in tv10/bizportal/calcalist |
| F3 | AI-driven attack **speed** outpaces patch cycles | bizportal |
| F4 | Industry MTTR for critical-severity vulnerabilities remains **~65 days** | Edgescan's 2023 Vulnerability Statistics Report (8th edition, published 2023-04-11, analyzing 2022 data); Dux's own RSA-2026 LinkedIn post confirmed: "40,000 security pros. 600+ vendors. Endless acronyms. And somehow, the average MTTR is still 65 days." (D-49) |

### Additional press-sourced claims

Per-customer autonomous research; **coding agents write and run deterministic investigation code**; **evolving research playbook** per customer; major U.S. enterprises at **hundreds of thousands of assets / millions of vulnerabilities**; shift toward **mean time to protection (MTTP)**; zero-day response, ad-hoc threat investigations, continuous exposure analysis use cases; U.S. and Europe medium-term GTM (FinSMEs CEO interview).

**Redpoint interview highlights:** autonomous researcher in customer environment; breaks CVEs into real-world exploitation requisites; gathers runtime/identity/network/controls evidence; code-backed investigations (consistent, inspectable, repeatable); operational maturity in a few weeks; thousands of alerts → small human-readable groups with evidence; exposure management as a **reasoning problem**; 10-year vision to close the loop beyond patching.

**Additional press source (MC-16):** ice.co.il (Dec 2025, Hebrew) — same launch-press claim set (three pillars, exploitability focus, funding facts); one unqualified line "resolving them before attacks occur" (V-13 class, press-immutable). Also first press source stating **20 employees** (bizportal corroborates; LinkedIn shows 24 as of 2026-07).

---

## Market validation quotes

> Usage policy: approved for sales collateral and RFP attachments until **2026-09-30**; owner GTM lead; refresh quarterly from dux.io and primary press sources.

> "These attacks don't wait for patch cycles. Defenders need rapid insight into what's actually exploitable and the means to reduce those exposures effectively, at the pace modern attacks demand." — **Or Latovitz, Co-Founder and CEO, Dux** (BusinessWire, Dec 2025)

> "Attackers are moving faster than ever, and defenders need a platform built for that pace. Dux puts vulnerabilities in the context of their actual threat to a business, and then uses AI agents exactly where speed and precision matter most to resolve them. At last, and with Dux, vulnerability management is something teams can finally get ahead of — overcoming what was an insurmountable hurdle during my time at GitHub." — **Erica Brescia, Managing Director, Redpoint** (BusinessWire, Dec 2025)

> "Most security tools show you what's vulnerable. Dux shows you what attackers can actually use — and that's a game changer. Their AI agents bring a perspective that's been missing from exposure management, and the Dux team has precisely the kind of experience you want steering a shift of this magnitude." — **Rona Segev, Co-Founder and Managing Partner, TLV Partners** (BusinessWire, Dec 2025)

> "Most scanner findings aren't exploitable once you account for real context. But discovering that manually takes expert judgment and deep knowledge of the environment. Agentic AI lets teams apply that level of reasoning across every vulnerability and asset, every time." — **Amit Nir, Co-Founder and CPO, Dux** (BusinessWire, Dec 2025)

> "Every time a zero-day drops or a critical vulnerability hits the news, teams need answers fast. Our customers spin up AI-workers to investigate those vulnerabilities across their environment within minutes. That's a level of rapid, environment-specific research that simply wasn't possible before." — **Nadav Geva, Co-Founder and CTO, Dux** (BusinessWire, Dec 2025)

> "CISOs will not turn over a rock to find another risk unless they have a solution for it. Dux solves the tale as old as time — too many vulns, not enough resources — allowing Cybersecurity teams to focus their finite resources where it matters most — on those exploits they are actually vulnerable to." — **Andrew Wilder, CSO at Vetcor, ex-Nestlé** (dux.io)

> "After nearly a decade in exposure management, determining true exploitability always felt like the holy grail — but out of reach. Dux is the first approach I've seen that uses modern AI to actually make it practical in real environments." — **Mille Gandelsman, CPO @ Opti, ex-VP at Tenable** (dux.io)

> "The biggest gap in exposure management today isn't a lack of data — it's the inability to determine what actually matters amidst constant change. Dux closes that gap by focusing on exploitability, context, and the fastest path to safety. It transforms exposure management from reactive into a proactive defensive strategy." — **Rinki Sethi, CSO at Upwind, ex-BILL** (dux.io)

> "The reality today is that attackers move faster than traditional security workflows. Dux changes that dynamic by helping teams reason about exposure and respond at the pace modern threats demand." — **Karl Mattson, Squared Circle, ex-PennyMac** (dux.io)

> "Dux agents act like an autonomous researcher inside each customer's environment — breaking vulnerabilities into real-world exploitation requisites, gathering runtime, identity, network, and controls evidence, and backing every investigation with agent-written code. In a matter of weeks, agents learn how the organization thinks about risk — turning thousands of alerts into a small number of clear, human-readable groups with evidence." — **Or Latovitz** (Redpoint YouTube, 2026)

> "We chat with a lot of CISOs at Redpoint, and every single one bemoans the millions of vulnerabilities across their tooling. Most aren't actually exploitable — teams waste time on low-priority noise. We were excited by Dux's ability to close this gap by leveraging AI agents to perform rapid exploitability analysis." — **Erica Brescia** (Redpoint YouTube, 2026)

> "Up until now, vulnerability management teams mainly had one 1000 lb hammer — patching. For the first time, we're expanding that arsenal so VM teams can close the loop and fix problems themselves, not only orchestrate remediation with IT." — **Or Latovitz** (Redpoint YouTube, 2026)

---

## Illustrative marketing numbers

> **Never contractual.** Illustrative only — not production targets, customer SLAs, or guaranteed outcomes; validate with design partners before signed collateral.

### Funnel (harmonized across banner/body/social; maps to US-010 Vulnerability Reduction buckets)

| Metric | Value | Bucket |
|--------|-------|--------|
| Total researched via Dux | 8,341 | Total Researched |
| Unexploitable | 6,198 (74.3%) | Unexploitable / Protected |
| Partially mitigated | 1,301 (15.6%) | Partially Mitigated |
| Mitigation required | 842 (10.1%) | Mitigation Required |
| Exposed | 0 (0.0%) | Exposed |
| Need attention | 2,143 (25.7%) | Partial + Required + Exposed |

### Other reference numbers

- **Ownership-inference certainty example:** 78%
- **Missing Blocklist / Protected By Policy examples:** 307 / 1,030
- **CTEM benchmarks:** testing exploitability reduces false urgency by **up to 84%** (Picus Security's own reported figure, not independent research — a category stat, not a Dux SLA, D-53)
- **Design-partner aggregate pain:** ~1,247 critical findings/mo at ~15% remediation capacity (N=3 anonymized, 2026-06)
- **"10+ tools"** = contractual integration-catalog connectors (not the 42-value OpenAPI `Sources` attribution enum)
- **Market sizing (all forward-looking hypotheses):** TAM $8–12B, SAM $800M–1.2B, SOM Y1–2 $5–15M; category sizing CybersecTools 85 exposure-management tools / 459 attack-surface

---

## Personas

### The agent persona — Dux Agent

An "Aggressive Exposure Management Specialist": calm, logical, humble, transparency-focused, and **citation-first** — every exploitability claim references a permitted source (NVD, asset inventory, control evidence). **Dux Agent** is the only name that ever appears in front of a customer for the AI system doing the work.

### Human personas

| Persona | Goal | Primary stories | Degraded path without connectors |
|---------|------|-----------------|----------------------------------|
| Security engineer (primary user) | Cut the actionable queue from thousands to tens | US-001, US-010, US-011, US-008 | AWS + NVD live; vendor panels show connector-degraded empty states |
| CISO / security leader (buyer) | Board-ready, validated risk reduction | US-006, US-012 | Donut and delta metrics only |
| AI Safety Lead | Agent halt authority (<5 s kill switch) | US-014 | — |
| DevOps / SRE | Fix without breaking deploys | US-007, US-018 | Webhooks delayed |
| Tenant admin | Users, connectors, agent policy, export | US-013, US-014 | AWS wizard + CSV fallback; SSO deferral note |
| API consumer | Phase 1: assessments and webhooks (JWT). Seed: public data API | US-014, US-024 | Poll `GET /v1/vulnerability-instances` once the Seed API is live |

---

## Navigation → user-story map

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

The nav-label vs page-title split, and the Mitigation-nav vs Mitigate-stage distinction, are canonical in [[Dux Taxonomy & Catalogs]] — they are easy to conflate, and must not be.

---

## Canonical end-to-end path (demo / POC)

US-012 Dashboard → US-013 AWS connector (or CSV) → US-010 Request Research → US-011 Exposure Analysis → US-017 trace, showing the reasoning and the executed code → optionally US-008 Chat Guidance.

The story it tells is **"thousands → tens"**: scanner findings become a small set of evidence-backed action groups.

---

## Phase-1 gate model

| Gate | Meaning | Exit criteria |
|------|---------|---------------|
| Vertical slice (Week 6) | One CVE → one agent → one conclusion → one UI view | SIGKILL durability (zero duplicate external effects; the LLM step is not re-sampled); governance gates live (Intent, Budget, ActionBudget, CostForecast); 2+ design partners onboarded; 10 test CVEs with traces |
| **Gate 1 review (Week 12)** | Full Phase-1 pipeline, hardened | >80% golden-set accuracy on a held-out set of **(CVE × environment) pairs** (H1); zero cross-tenant fuzz reads; zero Critical findings in the adversarial suite; **<$0.75 per workflow** (D-3); 2+ committed design partners; anomaly-escalation approve/deny surface live with impact preview (H4); LLM09 citation gate green |
| Phase-1 exit (Week 16) | Production beta | Customer data flowing for 2+ weeks; sustained >80% held-out accuracy; false negatives <5%; OWASP LLM and Agentic assessments at Partial or better; actionable-queue ratio and MTTP measured on real partner data |
| Gate 2a / 2b / 2c | Seed activation / GTM / vendor-screen expansion | See [[Dux Operations Guide]] |
| Gate 3 | **Closed-loop mitigation validation (US-019)** | A field-proven action-safety record. Unattended write *execution* is already Gate 1 |
| Gate 5 | Optional physical residency | A signed on-prem contract |

### Release milestones (16-week calendar, D-7 R1)

| Week | Milestone |
|------|-----------|
| 2 | Infrastructure skeleton + isolation harness |
| 4 | PgBouncer pooling fuzz; `test:fuzz-tenant-id` becomes merge-blocking |
| 6 | Vertical slice + **the EU AI Act counsel opinion, before any EU prospect outreach.** This blocks EU tenant provisioning until the opinion and classification memo are on file. At `phase_1_epoch` 2026-06-23, Week 6 falls near 2026-07-28 — ahead of the 2 Aug 2026 Article 50 transparency deadline |
| 8 | Internal dogfood (2 tenants, isolation pass); internal `/api/docs` (Redoc, shared with NDA partners); HITL approval API; **the Langfuse DPA — this blocks production traces** |
| 12 | Gate 1 review + minimal HITL approve/deny UI |
| 14 | `chat_write_tools` + full chat HITL UI |

**Abort rule:** switch the inner framework if the golden set regresses by more than 2%.

---

## Capacity

**Capacity resolved (D-19, D-23).** The backlog consumes 2,040 h against the re-baselined 2,080 h envelope (~98%, a 40 h buffer) — 26 focused h/week instead of 25, same 5-engineer team, no sixth hire. Gate-1 Week 12 and exit Week 16 timing are unchanged.

---

## Explicitly out of scope for Phase 1

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

---

## Vision-vs-implementation gap list

Under the ideal-state canon (GCIS v2.2), most launch-press claims are now **engineering-true at Gate 1**. Remaining gaps:

| # | Vision/marketing language | Implementation reality (ideal-state canon) | Status |
|---|---------------------------|--------------------------------------------|--------|
| V-1 | "Experts working for you 24/7" (E1) | Continuous re-assessment (US-021/FR-025/ADR-016) + on-demand queue ship Gate 1; autonomous write actions execute unattended by default at Gate 1 | **Closed** |
| V-2 | "Agents write **and run** investigation code" | True at Gate 1 via self-hosted Firecracker on K8s (FR-026/ADR-015 R4); `NoOpSandboxAdapter` retained as emergency kill path → claim false only during a kill-path event | Closed w/ caveat |
| V-3 | "Ties together **all** data sources / auto-tags **every** asset" | Gate 1 = AWS + intel feeds + 3 vendor connectors + ownership inference; 42-source taxonomy spans waves W1/W2/W3 | **Closed 2026-07-13** — dux.io wording accepted verbatim as GTM baseline |
| V-4 | "Zero-day investigated across environment **within minutes**" (B5) | MTXV target <15 min/CVE is an internal SLO; hero scenario citable as design-partner anecdote only, never an SLA | Open (framing rule) |
| V-5 | "Agents learn how the org thinks about risk **in weeks**" / evolving research playbook | Preference *learning* is Gate 2c (needs behavioral data volume); Gate 1 has per-tenant investigation artifacts + session routing prefs (24h TTL) | Open until Gate 2c |
| V-6 | "Skilled analyst / thinks like a human" (A6–A8, B7) | Vision language — engineering delivers evidence-backed traces and dual-triage outcomes, not anthropomorphic guarantees | Permanent framing rule |
| V-7 | "Major U.S. enterprises" as customers | Design partners under NDA; named references only with written permission | Permanent GTM rule |
| V-8 | Homepage screenshots of Mitigation/Remediation screens | Backed by shipped Gate 1 capability under GCIS — unattended by default | **Closed** |
| V-9 | "Hundreds of thousands of assets / millions of vulnerabilities" | Public reference metric from press — not an engineering SLO; no scale claims until Gate 2 re-baseline (≥100 assessments/day) | Open (framing rule) |
| V-10 | trust.dux.io / status.dux.io, privacy policy hosting, © year, ZoomInfo entity name, CybersecTools listing | Reclassified **launch blockers** with owners (GCIS §E) — not accepted gaps. trust/status/docs.dux.io no DNS; © 2025 footer; privacy still Google Drive; ZoomInfo slug uncorrected — all still open. CybersecTools now clean ✓ | Open-ops until HTTP 200 / corrections land |
| V-11 | "Lives inside your environment" | **Logical residency** (SaaS-to-SaaS, read-only APIs/OAuth) — physical residency is Gate 5 roadmap only | Permanent framing rule |
| V-12 | Never claim | Scanner replacement, PTaaS/offensive execution, OT/IoT discovery, on-prem resident agent pre-Gate 5, financial impact quantification | Permanent |
| V-13 | "Safety at Machine Speed" / "shuts them down before they're used" / "instant" | dux.io hero copy ("instant exploitability analysis... full protection at machine speed"; "close critical gaps instantly") is the accepted GTM baseline — no human-approved qualifier required in customer messaging. "Fastest safe fix" (retired press phrase, BusinessWire-only, not on live dux.io) stays retired | **Accepted as GTM baseline** |
| V-14 | dux.io footer displays an "ISO 27001 Certified" seal and a generic AICPA "SOC for Service Organizations" mark, unlabeled as Type I/II, with no certificate number, issuer name, or verification link | compliance-program.md states SOC 2 and ISO 27001 are **Series A-stage, event-triggered programs that have not started** — no readiness letter, no audit engagement, no ISMS scope has been initiated | **Confirmed unearned claim (D-44).** Fix is removing or relabeling them on the live site — tracked as an external follow-up |

---

## Claim firewall mechanics

Every public claim must trace through the Marketing Claims → Technical Achievement map to a BR/FR/US at the same gate. The pre-publication checklist (**PM + Counsel signed, stored in CRM**) is a blocking process gate for RFPs, order forms, and decks.

**Authority order:** decisions-log → GCIS v2.2 → OpenAPI 3.1 (`/v1/*` wire contract only) → BR→FR→US → agent registry (manifests).

**Claims-alignment rule (D-10, 2026-07-14, Sagi):** the Marketing Claims map binds **GTM copy, product naming, and UI strings**. It does **not** bind safety posture, control design, gate criteria, or SLOs — an engineering doc states what the system does. Where a live public claim and engineering reality diverge, the divergence is raised as a claim-integrity item; it does not bend the spec. The Gate/build-status record in [[Dux Decisions & Traceability Reference]] tracks actual delivery.

**What this means in practice:** a claim is only "GCIS-true" when the underlying BR/FR/US is at the same gate as the claim's stated capability. If the engineering delivery lags the marketing language, the gap is documented, never papered over by editing the spec to match the claim.

---

## How this knowledge base is organized

Ten source domains (product, architecture, API, AI safety, engineering, operations, governance, GTM, and the execution backlog) have been consolidated into a small set of dense, purpose-built guides rather than left as dozens of narrow pages. Each domain gets an **explanation-and-how-to guide** written to be read start to finish, and where a domain also needs fast lookup (API contracts, decision history, the controlled vocabulary), it gets a companion **reference** page built for scanning, not reading.

### Reading paths by role

| You are | Start here | Then |
|---|---|---|
| An engineer building a feature | [[Dux Product Guide]] | [[Dux Feature Reference]] → [[Dux API Reference]] |
| An architect | [[Dux Architecture Guide]] | [[Dux AI Safety Guide]] for the safety spine built on top of it |
| A security reviewer | [[Dux AI Safety Guide]] | [[Dux AI Safety Operations Reference]] |
| On-call / SRE | [[Dux Operations Guide]] | [[Dux AI Safety Operations Reference]] for the incident runbooks |
| Founder, PM, or GTM | [[Dux GTM Guide]] | [[Dux Product Guide]] for the roadmap it has to stay honest against |
| Customer success | [[Dux Customer Success Guide]] | [[Dux Operations Guide]] for the incident-communication mechanics |
| Auditor or compliance reviewer | [[Dux Governance & Compliance Guide]] | [[Dux Decisions & Traceability Reference]] |
| Anyone, before starting real work | [[Dux Governance & Compliance Guide]] for open items | whatever guide the open item touches |

### The domain map

**Product**
- [[Dux Product Guide]]: thesis, the five delivery pillars, write-action autonomy, personas, the gate model, the roadmap
- [[Dux Feature Reference]]: every screen and API surface behind the eight-icon sidebar and chat
- [[Dux Taxonomy & Catalogs]]: the controlled vocabulary, the confidence model, and the nine extension registries

**Architecture**
- [[Dux Architecture Guide]]: system context, the technology stack, how an assessment actually runs, the data model, tenant isolation, and the key architecture decisions behind all of it

**AI safety**
- [[Dux AI Safety Guide]]: the governance kernel, the kill switch, the CaMeL dual-LLM boundary, sandbox execution, MCP security
- [[Dux AI Safety Operations Reference]]: agent identity, confidence calibration, OWASP maturity ratings, the twelve incident runbooks

**Engineering**
- [[Dux Engineering Guide]]: local setup, coding standards, code review, and the full CI/CD pipeline including the golden set

**Operations**
- [[Dux Operations Guide]]: seed-stage activation criteria, observability and SLOs, deploy/rollback runbooks, disaster recovery
- [[Dux Customer Success Guide]]: onboarding, support tiers, incident communication, tenant health

**Governance**
- [[Dux Governance & Compliance Guide]]: the open-items discipline, SOC 2/ISO/EU AI Act, and the Series B governance roadmap

**Go-to-market**
- [[Dux GTM Guide]]: the claims firewall, the business model, pricing, competitive positioning, and live external corrections

**API**
- [[Dux API Reference]]: the three REST planes, every DTO contract, DQL, events and webhooks

**Reference**
- [[Dux Decisions & Traceability Reference]]: the full decision history and the business-requirement-to-verification-command join table

**Execution**
- [[Dux Portfolio]]: the ten-epic Phase-1 backlog, hour rollup, and the rules governing how it's allowed to change

---

## Document contract

Every file in this tree carries front-matter:

```yaml
---
owner: Engineering | Founder | GTM | Security
status: canonical | draft | backlog-shell | process-record
gate: 1 | 2 | 3 | n/a
last_reviewed: YYYY-MM-DD
decisions: [D-4, H5]     # optional: the decisions that govern this file
---
```

Then: an `# H1`, a **Purpose** line, and numbered `##` sections.

### What `status` means

| Status | Meaning |
|--------|---------|
| `canonical` | current truth. Build from it |
| `draft` | being written; do not build from it yet |
| `backlog-shell` | a planning placeholder. **Inherit the earlier stage's artifacts** — do not execute from a stub |
| `process-record` | dated audit evidence. **Not a spec, and not on any reading path** |

### Authority & conventions

1. **Authority order:** decisions-log → GCIS v2.2 → OpenAPI 3.1 (`/v1/*` wire contract only) → BR→FR→US → agent registry (manifests).
2. **Stage model:** playbook names = operational maturity, not funding rounds. Pre-seed = Phase 1 build (Days 0–90); seed = post-Gate-2 ops (Days 90–365); Series A = certifications/governance; Series B = global-scale programs (backlog shell). The company's $9M seed round (Dec 2025) is unrelated to "seed playbook" activation.
3. **Terminology:** **Dux Agent** = only customer-facing agent name; *assessment agent* internal role only; *Mitigation nav* = research queue (Analyze), distinct from the *Mitigate* pipeline stage; kill switch (noun) / kill-switch (adjective); World Model is a proper noun; API/code say `tenant_id`, UI may say organization.
4. **Open questions:** every unresolved question lives in [[Dux Governance & Compliance Guide]] as `OI-##`, with an owner, a severity, and what it blocks (D-11). A spec links to the item rather than restating it, and never carries a `> **TBD**` box of its own.
5. **Current truth only:** specs are written in the present tense (D-12). **Change history belongs in [[Dux Decisions & Traceability Reference]] — never in body prose.** No "re-gated 2026-07-13", no "superseded — was: …", no dated parentheticals. A reader should learn the current state by reading, not by mentally applying a chain of diffs.

**Three exceptions to the present-tense rule, all deliberate:**
- **[[Dux Decisions & Traceability Reference]]** is the history home. It carries all of it.
- **ADR index** keeps its `Superseded` lines. An ADR is a decision record — carrying its own supersession history is the format working as intended.
- **[[Dux Portfolio]]**'s hour-rollup section keeps its dated historical paragraphs — a running capacity ledger needs its history visible the same way an ADR does, to show which lever fired and why.

---

## A note on how this knowledge base itself is maintained

This corpus was ingested with a genuinely unusual amount of rigor: four independent verification passes cross-checked citation coverage, dense ID ranges, dollar figures and diagram coverage, and inline-code spans against the source material, catching and fixing nineteen real content gaps along the way. The full account of that process lives in `wiki/migration-audit.md` and `wiki/validation-checklist.md`, preserved as a historical record from before this knowledge base was consolidated into the guides above. The source material itself remains mirrored in full at `.raw/dux/` and is never modified: every fact in every guide above traces back to it.

### Validation

After editing docs, run `python3 scripts/validate-playbooks.py`. **Exit 0 is required to merge.** It checks:
- dead links **and anchors** (renaming a heading changes its anchor — the validator catches the resulting dead links, but keep anchors stable where you can);
- duplicate canonical H2 IDs (`## US-`, `## EP-`, `## ADR-`, …) — each may be defined in exactly one file;
- task → epic → portfolio hour reconciliation, including the grand total;
- the front-matter contract below;
- **change history in spec prose** — a reintroduced "re-gated …" or "superseded — was: …" fails the gate.

---

## Sources

- `.raw/dux/README.md`
- `.raw/dux/00-meta/vision-reference.md`
- `.raw/dux/00-meta/decisions-log.md`
- `.raw/dux/10-product/product-overview.md`
