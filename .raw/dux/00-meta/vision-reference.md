---
owner: GTM
status: process-record
gate: n/a
last_reviewed: 2026-07-21
decisions: [D-10, D-34]
---

# Vision Reference — Marketing & Scraped Language

**Purpose:** the single home for all marketing, press, and vision language captured into the corpus. **Preserved verbatim, with sources and retrieval dates.**

> **Nothing here is an engineering commitment.**
>
> This file records **what was said in public, and when.** Every claim maps to the [traceability-matrix](traceability-matrix.md), and to the gap list in §6.
>
> **Scope of the claims map (D-10).** It binds **GTM copy, product naming, and UI strings.** It does **not** bind safety posture, control design, gate criteria, or SLOs.
>
> Where a claim here and engineering reality diverge, that divergence is raised in [open-items.md](open-items.md). **It is never resolved by editing a spec to match the claim.**

---

## 1. Positioning & taglines

- **Category:** Agentic Exposure Management
- **Primary tagline:** *Safety at Machine Speed*
- **Alternate marketing copy:** *Safety Faster than the Speed of Strike*
- **Stage headline (dux.io):** *If it's not exploitable, why fix it?*
- **Contact-section header (dux.io):** *Faster Safety Starts Here* (added 2026-07-13, marketing-claims review MC-N-02 — file removed 2026-07-16, findings absorbed into [decisions-log](decisions-log.md) and [open-items](open-items.md) OI-26)
- **Naming:** Public "AI-workers" / "Dux AI agents" = **Dux Agent** (canonical product name). Legal entity: **Dux, Inc.** (press "Dux Technologies Inc." is non-canonical) — confirmed (D-51). Dux's own Privacy Policy PDF names "Dux Technologies Inc." in error; needs a live-site correction, tracked outside this repo.

**Canonical hero (gate-safe copy):**
> Dux is an agentic exposure management platform built for a world where vulnerability exploitation windows are collapsing from weeks to minutes. Use **Dux Agent** to determine what's actually exploitable in your environment, identify the fastest path to protection, and eliminate exposure before it's used against you. We bring exploitability, context, and action together — so you achieve **safety at machine speed**.

**Hero / CVE-drop scenario:**
> When a zero-day drops, your scanners flag thousands of instances. Teams spin up **Dux Agent** to investigate which are **actually exploitable in your environment** — with evidence for controls already blocking the path. Stop drowning in alerts. Start focusing on what actually matters.

**Gate-safe context line:** *Not exploitable = not urgent. Context changes everything.* (CVSS 9.8 doesn't mean critical in **your** environment; EPSS is an input badge, not the verdict.)

**Problem narrative (long-form):**
> For years, vulnerability management was optimized for visibility — more dashboards, better reports, cleaner ticket flows, and incremental "rank your backlog better" tools that still leave teams drowning in noise. When vulnerabilities move from disclosure to exploitation in hours, visibility isn't the problem anymore; **speed is**. Security teams fall behind because they can't quickly tell what's actually exploitable, what's already blocked, and what really needs action. Moving data around doesn't solve that. Understanding true exposure does. Attackers weaponize vulnerabilities in hours, not days; autonomous AI agents are now documented executing attacks end-to-end (Anthropic, 2025); most teams still work through week-old backlogs — a structural problem, not a process problem.

## 2. Three-stage public framing (dux.io)

| Stage | Public headline | Buyer questions on site |
|-------|-----------------|-------------------------|
| Exploitability Analysis | If it's not exploitable, why fix it? | What's needed for the exploit? Can it actually be exploited? Where am I already protected? |
| Lightweight Mitigations | Apply mitigations that get you to safety | Can I mitigate this quickly? What can I do with my existing stack? Am I still protected? |
| Remediation Acceleration | Speed up remediation when required | Who should remediate? What should I remediate? How can I customize this? |

**Public outcomes (BusinessWire, dux.io):** materially smaller attack surface; far shorter path from vulnerability discovery to resolution; less time chasing noise in the research queue; reactive exposure triage → proactive defensive strategy.

## 3. Article-sourced product claims inventory (Dec 2025 press, canonical IDs)

North star (attacker-minded defense):

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

How (agent platform + loop):

| ID | Claim | Source |
|----|-------|--------|
| B1 | **Agent-based threat-management platform** | all four Hebrew-press articles |
| B2 | **Four-step loop:** analyze → check controls block paths → lightweight mitigations → route stakeholders | bizportal |
| B3 | **Periodic scans + manual triage → continuous agent-based research** | bizportal; tv10; calcalist |
| B4 | **Autonomous analyst** investigating vulnerabilities at **machine speed** | geektime |
| B5 | **Zero-day investigated across environment within minutes** | bizportal (Geva) |
| B6 | Three pillars: **exploit-path analysis**, **tuning existing defense tools**, **continuous threat management** | bizportal; tv10 |
| B7 | **Machines that think like humans** enable a fundamentally **different solution** than ranking | bizportal |

Outcomes:

| ID | Claim | Source |
|----|-------|--------|
| C1 | **Much smaller attack surface** | bizportal — illustrative/unmeasured until Phase-1 exit (H9); see [gtm-guardrails](../80-gtm/gtm-guardrails.md) §4 |
| C2 | **Shorter path** from vulnerability identification to solution | bizportal — illustrative/unmeasured until Phase-1 exit (H9); see [gtm-guardrails](../80-gtm/gtm-guardrails.md) §4 |
| C3 | **Fastest and safest fix:** config change → existing tool update → full patch | bizportal |
| C4 | **Lightweight mitigations faster than full patch** | bizportal |
| C5 | **Tuning existing defense tools** in customer context (not replacing stack) | bizportal; tv10 |
| C6 | **Game changer** — exploitability vs vulnerability lists | tv10; Segev |
| C7 | **Breakthrough** agentic capability in cyber defense | bizportal |

What we are not:

| ID | Claim | Source |
|----|-------|--------|
| D1 | **Not another "rank your backlog better" tool** | bizportal |
| D2 | **No unified prioritization logic for all customers** — per-customer via coding agents | bizportal |
| D3 | **No static rules** — per-environment personalization ("secret sauce") | geektime |
| D4 | Agents on **third-party AI models** (not a proprietary-model claim) | geektime |
| D5 | **Defensive platform only** — not PTaaS / not offensive execution | playbook invariant |

Aspirational:

| ID | Claim | Source |
|----|-------|--------|
| E1 | Orgs **feel like experts working for them 24/7** | bizportal |
| E2 | Solve exposure management **from the root** — different than how the industry thought about it | bizportal |
| E3 | **Long-term:** expand beyond patching-only "1000 lb hammer" | Redpoint + articles |
| E4 | **Predictive risk forecasting** — "which assets are likely to become risky in the future" | FinSMEs CEO interview (Dec 2025); **committed roadmap capability per the 2026-07-13 claims-alignment directive** (live press claim binds the corpus, per MC-07) |

**E4 specced (resolves [OI-09](open-items.md), D-24, 2026-07-16).** BR-013 → EP-03 → US-028 → FR-030, Gate 2. Canonical spec: [predictive-risk-forecasting](../10-product/features/predictive-risk-forecasting.md). It answers the claim literally, as an evidence-trend computation (EPSS/finding-count/control-coverage deltas) — not a new ML model or a composite score, consistent with how the rest of this corpus treats risk scoring. **Implementation hours are not yet in the backlog** — the spec-authoring task (`EP-03-F04-T01`) is closed by this; sizing the build is separate, unbudgeted work.

Why now (market context only — never product capability claims):

| ID | Claim | Source |
|----|-------|--------|
| F1 | Mandiant mean time-to-exploit is negative and worsening: **~−1 day (2024), ~−7 days (2025)** | Confirmed verbatim in Mandiant's own M-Trends 2026 (Google Cloud blog). **The previously-paired "32 → 5 day" framing is dropped (D-53)** — that starting point traced only to a different, non-Mandiant source (CyberMindr) describing "weaponization" time, a different metric, conflated here under one Mandiant attribution |
| F2 | Anthropic-documented **fully autonomous AI cyber-espionage attack** (planning through execution) | Anthropic 2025, cited in tv10/bizportal/calcalist |
| F3 | AI-driven attack **speed** outpaces patch cycles | bizportal |
| F4 | Industry MTTR for critical-severity vulnerabilities remains **~65 days** | **Edgescan's 2023 Vulnerability Statistics Report (8th edition, published 2023-04-11, analyzing 2022 data) is confirmed, verbatim, as a real published source for exactly this "65 days" Critical-severity MTTR figure** (2026-07-21 re-verification). **Dux's own RSA-2026 LinkedIn post is confirmed** (Founder-located and re-fetched 2026-07-21): [linkedin.com/posts/duxsecurity_rsac2026-ctem-vulnerabilitymanagement-activity-7440388981793546240-GjjN](https://www.linkedin.com/posts/duxsecurity_rsac2026-ctem-vulnerabilitymanagement-activity-7440388981793546240-GjjN), posted ~2026-03 (RSA 2026 timeframe; exact date not pinned down). Exact wording: "40,000 security pros. 600+ vendors. Endless acronyms. And somehow, the average MTTR is still 65 days." (D-49) |

Additional press-sourced claims: per-customer autonomous research; **coding agents write and run deterministic investigation code**; **evolving research playbook** per customer; major U.S. enterprises at **hundreds of thousands of assets / millions of vulnerabilities**; shift toward **mean time to protection (MTTP)**; zero-day response, ad-hoc threat investigations, continuous exposure analysis use cases; U.S. and Europe medium-term GTM (FinSMEs CEO interview). Redpoint interview: autonomous researcher in customer environment; breaks CVEs into real-world exploitation requisites; gathers runtime/identity/network/controls evidence; code-backed investigations (consistent, inspectable, repeatable); operational maturity in a few weeks; thousands of alerts → small human-readable groups with evidence; exposure management as a **reasoning problem**; 10-year vision to close the loop beyond patching.

**Additional press source (added 2026-07-13, MC-16):** ice.co.il (`ice.co.il/digital-140/news/article/1095806`, Dec 2025, Hebrew) — same launch-press claim set (three pillars, exploitability focus, funding facts); one unqualified line "resolving them before attacks occur" (V-13 class, press-immutable). Also first press source stating **20 employees** (bizportal corroborates; LinkedIn shows 24 as of 2026-07).

**Additional dux.io site copy (added 2026-07-11, coverage audit):** **"Not every gap needs a patch"** (Lightweight Mitigations stage framing, pairs with C4); **compensating controls** / **stopgap measures** that close critical gaps instantly while waiting on a patch (maps to existing C4 "lightweight mitigations faster than full patch" — same claim, site's own wording); **"works with your existing stack, no new purchases"** (maps to existing C5 "tuning existing defense tools... not replacing stack" — site's stronger framing, no new purchase required); Remediation Acceleration stage explicitly names **bottleneck elimination** — "blows past spotty data, unclear ownership, legacy systems" — and **legacy system handling** as a named capability, not just implied by B2's four-step loop.

## 4. Market validation quotes (verbatim, with attribution)

Usage policy: approved for sales collateral and RFP attachments until **2026-09-30**; owner GTM lead; refresh quarterly from dux.io and primary press sources.

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

## 5. Illustrative marketing numbers (never contractual)

Funnel (harmonized across banner/body/social; maps to US-010 Vulnerability Reduction buckets; *illustrative only — not production targets, customer SLAs, or guaranteed outcomes; validate with design partners before signed collateral*):

| Metric | Value | Bucket |
|--------|-------|--------|
| Total researched via Dux | 8,341 | Total Researched |
| Unexploitable | 6,198 (74.3%) | Unexploitable / Protected |
| Partially mitigated | 1,301 (15.6%) | Partially Mitigated |
| Mitigation required | 842 (10.1%) | Mitigation Required |
| Exposed | 0 (0.0%) | Exposed |
| Need attention | 2,143 (25.7%) | Partial + Required + Exposed |

Other reference numbers: ownership-inference certainty example 78%; Missing Blocklist / Protected By Policy examples 307 / 1,030; CTEM benchmarks: testing exploitability reduces false urgency by **up to 84%** (Picus Security's own reported figure, not independent research — see [competitive](../80-gtm/competitive.md); a category stat, not a Dux SLA, D-53); design-partner aggregate pain: ~1,247 critical findings/mo at ~15% remediation capacity (N=3 anonymized, 2026-06); "10+ tools" = contractual integration-catalog connectors (not the 42-value OpenAPI `Sources` attribution enum). Market sizing (all forward-looking hypotheses): TAM $8–12B, SAM $800M–1.2B, SOM Y1–2 $5–15M; category sizing CybersecTools 85 exposure-management tools / 459 attack-surface.

## 6. Vision-vs-implementation gap list

Under the ideal-state canon (GCIS v2.2), most launch-press claims are now **engineering-true at Gate 1**. Remaining gaps:

| # | Vision/marketing language | Implementation reality (ideal-state canon) | Status |
|---|---------------------------|--------------------------------------------|--------|
| V-1 | "Experts working for you 24/7" (E1) | Continuous re-assessment (US-021/FR-025/ADR-016) + on-demand queue ship Gate 1; autonomous write actions execute unattended by default at Gate 1 | **Closed** |
| V-2 | "Agents write **and run** investigation code" | True at Gate 1 via self-hosted Firecracker on K8s (FR-026/ADR-015 R4); `NoOpSandboxAdapter` retained as emergency kill path → claim false only during a kill-path event | Closed w/ caveat |
| V-3 | "Ties together **all** data sources / auto-tags **every** asset" | Gate 1 = AWS + intel feeds + 3 vendor connectors + ownership inference; 42-source taxonomy spans waves W1/W2/W3 | **Closed 2026-07-13** — dux.io wording accepted verbatim as GTM baseline; no qualifier required in messaging (integration-catalog wave coverage still tracked separately as an engineering roadmap item, unaffected) |
| V-4 | "Zero-day investigated across environment **within minutes**" (B5) | MTXV target <15 min/CVE is an internal SLO; hero scenario citable as design-partner anecdote only, never an SLA | Open (framing rule) |
| V-5 | "Agents learn how the org thinks about risk **in weeks**" / evolving research playbook | Preference *learning* is Gate 2c (needs behavioral data volume); Gate 1 has per-tenant investigation artifacts + session routing prefs (24h TTL) | Open until Gate 2c |
| V-6 | "Skilled analyst / thinks like a human" (A6–A8, B7) | Vision language — engineering delivers evidence-backed traces and dual-triage outcomes, not anthropomorphic guarantees | Permanent framing rule |
| V-7 | "Major U.S. enterprises" as customers | Design partners under NDA; named references only with written permission | Permanent GTM rule |
| V-8 | Homepage screenshots of Mitigation/Remediation screens | Backed by shipped Gate 1 capability under GCIS — unattended by default | **Closed** |
| V-9 | "Hundreds of thousands of assets / millions of vulnerabilities" | Public reference metric from press — not an engineering SLO; no scale claims until Gate 2 re-baseline (≥100 assessments/day) | Open (framing rule) |
| V-10 | trust.dux.io / status.dux.io, privacy policy hosting, © year, ZoomInfo entity name, CybersecTools listing | Reclassified **launch blockers** with owners (GCIS §E) — not accepted gaps. **2026-07-13 probe (MC-01, marketing-claims review, file removed 2026-07-16 — see [open-items](open-items.md) OI-26): trust/status/docs.dux.io no DNS; © 2025 footer; privacy still Google Drive; ZoomInfo slug uncorrected — all still open. CybersecTools now clean ✓** | Open-ops until HTTP 200 / corrections land |
| V-11 | "Lives inside your environment" | **Logical residency** (SaaS-to-SaaS, read-only APIs/OAuth) — physical residency is Gate 5 roadmap only | Permanent framing rule |
| V-12 | Never claim | Scanner replacement, PTaaS/offensive execution, OT/IoT discovery, on-prem resident agent pre-Gate 5, financial impact quantification | Permanent |
| V-13 | "Safety at Machine Speed" / "shuts them down before they're used" / "instant" | dux.io hero copy ("instant exploitability analysis... full protection at machine speed"; "close critical gaps instantly") is the accepted GTM baseline — no human-approved qualifier required in customer messaging. "Fastest safe fix" (retired press phrase, BusinessWire-only, not on live dux.io) stays retired — unrelated to this decision, see competitive.md §3 errata | **Accepted as GTM baseline** |
| V-14 | dux.io's own footer displays an "ISO 27001 Certified — Information Security Management" seal and a generic AICPA "SOC for Service Organizations" mark, unlabeled as Type I/II, with no certificate number, issuer name, or verification link anywhere on the site (2026-07-21 site check) | [compliance-program.md §1–2](../70-governance/compliance-program.md) states SOC 2 and ISO 27001 are **Series A-stage, event-triggered programs that have not started** — no readiness letter, no audit engagement, no ISMS scope has been initiated at the Phase-1 stage this corpus otherwise describes | **Confirmed unearned claim (D-44).** compliance-program.md is the accurate side; the live badges are wrong. Fix is removing or relabeling them on the live site — outside this repo, tracked as an external follow-up, not a doc change |

**Claim firewall mechanics preserved:** every public claim must trace through the Marketing Claims → Technical Achievement map to a BR/FR/US at the same gate; pre-publication checklist (PM + Counsel signed, stored in CRM) is a blocking process gate for RFPs/order forms/decks.

## 7. Company facts (press-verified, added 2026-07-11)

Promoted from the now-removed `00-meta/coverage/inventory.md` audit (rows INV-0067, INV-0071, INV-0084, INV-0148; recoverable from git history) into this GTM-facing file so they're visible outside the meta layer. Historical facts, **not used for gate thresholds/ARR models/KPI targets** (per the source audit's INV-0084 disposition).

- **Founded:** 2024.
- **HQ:** Tel Aviv, **Israel** (R&D) + New York, **USA** (GTM).
- **Funding:** $9M seed round (Dec 2025) — Redpoint, TLV Partners, Maple Capital. Funding complete, no undisclosed SAFEs (counsel-confirmed 2026-06-30).
- **Angel investors:** individuals from CrowdStrike, Okta, and Armis participated as angels (BusinessWire, Dec 2025). Distinct from the CrowdStrike/AWS/NVIDIA 2026 Accelerator cohort membership (`gtm-guardrails.md` §4) — that's ecosystem validation, this is investor cap-table fact; don't conflate the two CrowdStrike mentions. **Note:** Armis is also listed as a named competitor in [competitive.md](../80-gtm/competitive.md) §2 — see the disclosure note there.
- **Founders:** Or Latovitz (CEO), Amit Nir (CPO), Nadav Geva (CTO) — Talpiot alumni (shared cohort adjacent to Wiz/Cyera founders), and per Hebrew press (Geektime, Calcalist), recipients of the Israel Security Prize and the Prime Minister's Prize for cyber/AI work at the PMO and Unit 8200. **The CEO's "deep-learning MSc TAU" credential could not be re-corroborated (2026-07-21 web pass)** — every LinkedIn fetch attempt for Or Latovitz returned a blocked response, and the claim surfaced only as an unsourced summary paraphrase, never a quoted primary sentence; treat as unverified pending a citable source. (Amit Nir's TAU degree, 2018–2021, is separately confirmed via his own LinkedIn — do not conflate the two.)
- **Headcount:** 20 employees at the Dec-2025 launch, per the CEO (bizportal); Dux's own LinkedIn company page showed 24 as of 2026-07-20.
- **Team (beyond the three founders):** Eden Amitai joined as Head of Growth and Alliances (self-announced on LinkedIn, undated) — the only post-launch leadership hire confirmed via a self-authored primary source as of 2026-07-21; several other names surfaced only in low-confidence aggregator listings (RocketReach) and are not included here pending a primary-source confirmation.
- **Sectors (design-partner ICP):** finance, healthcare, technology.
- **Markets:** U.S.-first GTM; Europe medium-term (FinSMEs CEO interview); Israel as the R&D base is not itself a served market. "North America" beyond the U.S. is not separately documented — treat as unverified beyond "U.S."
- **Revenue model:** subscription SaaS, B2B, tiered plans ([pricing-packaging.md](../80-gtm/pricing-packaging.md)); ACV language implies annual contracts — explicit "multi-year contracts" framing is not otherwise documented in the corpus, noted here as a press-sourced fact pending confirmation.
