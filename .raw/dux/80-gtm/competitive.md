---
owner: GTM
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-25, H5]
---

# Competitive Positioning & POC

**Purpose:** the analyst anchor, the feature-availability matrix, competitor counters, press errata, and the POC framework.

**Parents:** BR-010 · **Owner:** GTM lead · **Review:** quarterly.

Category sizing comes from [CybersecTools](https://cybersectools.com/tools/dux-security) (85 exposure-management tools). **It is not a product authority.**

## 1. Analyst validation

**Do not lead with this, and do not cite it anywhere, pending re-verification (D-48).**

**A quote previously attributed to "Gartner (Nunez, Mar 2026)" — wording/date/attribution unconfirmed:**

> "By 2028, organizations that prioritize exposures using threat intelligence, asset context, exploitability modeling and security control validation will reduce breach likelihood by at least 70% vs peers relying on CVSS-based prioritization."

**Status: unconfirmed, not "primary research."** A 2026-07-21 web pass could not independently corroborate this quote's exact wording, date, or attribution to Jonathan Nunez against any public or secondary source — the nearest real, indexed Gartner statement is a differently-worded 2022 CTEM prediction ("a two-thirds reduction in breaches" by 2026, no named analyst). This supersedes the 2026-07-09 decisions-log call that the quote was "confirmed primary research" (see [D-48](../00-meta/decisions-log.md)) — nobody on the team can currently point to the actual primary Gartner document it was sourced from. **Do not use this quote internally or externally — in decks, sales enablement, or RFPs — until Legal or the Founder locates and confirms the primary source.** Jonathan Nunez is a real Gartner analyst covering Exposure Management, but that does not confirm this specific quote.

**The broader positioning claim does not depend on this quote and stays valid:** "CVSS is not enough" is the mainstream Gartner, Rapid7, and Zafran position, not a contrarian take — Dux's capabilities #1 (exploitability modeling), #2 (asset context and security-control validation), and #7 (threat intelligence) map onto that direction regardless of whether this specific quote is ever confirmed. See [product-overview §2](../10-product/product-overview.md#2-core-capabilities).

> **No reprint/syndication licence exists (confirmed 2026-07-16, resolves [OI-10](../00-meta/open-items.md)).** Even if the quote is later confirmed, it remains internal-use-only unless Legal secures an actual Gartner reprint licence — this restriction is independent of, and in addition to, the unconfirmed-sourcing restriction above.

**Supporting category statistics — never Dux SLAs:**

| Statistic | Note |
|-----------|------|
| Testing exploitability reduces false urgency by **up to 84%** — Picus Security's own reported figure, not independent research (D-53) | CTEM-validation research. **Validation is the CTEM stage most teams skip — and it matters most.** |
| Only **16%** of organizations have operationalized CTEM, though **84%** call it important — Reflectiz survey, n=128 (D-53) | market timing. A small, single-vendor-run survey, not a broad industry finding |
| **~5% of published CVEs have a known exploit in the wild** | EPSS (FIRST.org), corroborated by an independent secondary analysis of the CVE/EPSS dataset (~5–6%, resilientcyber.io, 2024-08) and in the same range as other vendors' independent estimates (Tenable ~3%, Fortinet ~5.7%, Kenna/Cyentia <2%, NopSec ~3.4%) — the empirical basis for "if it isn't exploitable, why fix it?" |

**CTEM five-stage mapping** — the analyst-buyer grammar:

| CTEM stage | Dux surface |
|------------|-------------|
| Scoping | Connector Hub (US-013) |
| Discovery | multi-source ingest (EP-02) |
| Prioritization | exploitability bands + factor cards |
| **Validation** | **executed investigation code + trace (US-017) — this is the Dux wedge** |
| Mobilization | unattended mitigation + routed ticket (US-004, US-018) |

The outcome metric "thousands → tens" (the actionable-queue ratio) **stays illustrative until it is measured on real partner data at N ≥ 10.**

> **Planned, and not yet claim-safe.** The 84% figure is **Picus Security's own reported figure, not a Dux-measured number** (D-53).
>
> Dux already runs an internal DeepEval + 250-CVE golden-set eval harness for its own regression testing. **Once those results are stable and validated, surface them here as an owned, measured proof point** — replacing or supplementing the cited category statistic.
>
> **No number is published here until it is real and validated. Do not fabricate one.**

## 2. Feature availability matrix

**Attach this to every RFP and every pre-Gate-3 contract.**

**The "Live" cells describe what the *capability* does at that gate — not what every tier includes.** A Starter or Professional prospect must not read this matrix as full access; check the minimum-tier column ([pricing-packaging](pricing-packaging.md)).

| Capability | Gate 1 | Gate 2c | Gate 3 | Claim-safe? | Min. tier |
|-----------|--------|---------|--------|-------------|-----------|
| Exploitability analysis (queue + drill-down) | Live (US-010, US-011) | — | — | Yes — on-demand and continuous | Starter |
| Continuous re-assessment | Live (US-021, ADR-016) | — | — | Yes | Starter |
| Executed investigation code (trace) | Live (US-017, self-hosted Firecracker) | — | — | Yes | Starter |
| Asset context / protection breakdown | Live (US-002, US-003) | richer vendor data | — | Yes | Starter |
| Lightweight mitigations | **Live, unattended by default** (US-004, US-016) | — | closed-loop validation (US-019) | Yes | **Professional+** |
| Remediation ticket create + route | Live (US-018) | — | closed-loop automation | Yes | **Enterprise only** |
| Ownership inference / auto-tagging | Live (US-007) | — | — | Yes | Starter |
| Preference learning | — | US-009 | — | Gate 2c only | Professional+ |
| Public REST data API | — | Seed trigger | — | Seed+ | Enterprise |
| Optional physical residency | — | — | Gate 5 | roadmap only | Enterprise |

## 3. Competitor positioning

| Competitor | Their pitch | The Dux counter | Caveat |
|-----------|-------------|-----------------|--------|
| **ZEST Security** | owns the phrase "Agentic Exposure Management"; AI maps risks to resolution pathways | Dux **proves** exploitability per environment — agent-written and executed code, a CaMeL safety boundary, a claims firewall — *before* it routes a fix | Same category phrase. **Differentiate on validation depth, not on the label** |
| **Konvu** | "deterministic checks to confirm exploitability" | per-environment agent reasoning with executed code and evidence traces, not fixed checks. Dux also owns the governed write path — unattended by default, kill-switch-covered | Rising visibility (RSAC 2026 Launch Pad finalist; won Infosecurity Europe's inaugural Cyber Startup competition, Jun 2026) — no new funding round since its $5M seed (Jun 2024) |
| **Ethiack / SecRecon / Securifera** (agentic pentest, CART) | "prove what's exploitable" — by attacking | **Dux is defensive only.** It reasons about exploitability from evidence, and never attacks. **That is the wedge, not a limitation** | buyers who conflate validation with pentesting |
| **Tenable Hexa AI** (GA May 2026, 40 K customers) | agentic orchestration across Tenable One; remediation workflows — GA build adds multistep reasoning and MCP (Model Context Protocol) support for custom agent building | prerequisite decomposition, per-source citations, executed-code traces (US-017); connector-backed live context, versus a cloud-side Exposure Data Fabric | buyers already bundled into Tenable One |
| **Strobes AI** (Mar 2026 — 4.2 s per finding, 100+ integrations, 95% noise reduction; Apr 2026 "AI Harness" claims 2–4 week pentests compressed to <48 h) | fast triage, noise reduction | exploitability-validated buckets **plus a reasoning trace** — not speed-only triage. Customer-environment code artifacts, behind a CaMeL boundary | Analyze is live at Gate 1 |
| **Wiz** (part of Google Cloud's portfolio — acquisition closed March 2026) | risk graphs, exposure scores | environmental exploitability and control-aware paths, over a posture score | Wiz / Google Cloud standardization; Google has stated Gemini AI integration is forthcoming — watch for deeper Google Cloud platform bundling as integration proceeds |
| **Tenable / Qualys VM** (Qualys "Agent Val" GA March 2026, claiming 90%+ remediation-noise reduction, 70% faster time-to-remediate, 1,600+ CVEs covered) | scanner breadth, now with Qualys shipping its own agentic validation layer (Agent Val) | **enrich** scanner findings with environmental exploitability — Dux ingests Qualys and Wiz as input | a scanner vendor shipping a comparable agentic layer inside the renewal window — Qualys Agent Val is now that case, not just a hypothetical |
| **Armis / Averlon / RunSybil / IONIX** | AI validation with PoC evidence | a unified integration layer, preference learning, a CaMeL security boundary, and inspectable reasoning | **Disclosure: an Armis executive is also a Dux angel investor** (BusinessWire, Dec 2025 — [vision-reference](../00-meta/vision-reference.md)). The positioning here is independent of that relationship; noted for transparency. **2026 developments:** ServiceNow agreed (Dec 2025) to acquire Armis for $7.75B cash (Armis ARR $340M, +50% YoY), expected to close H2 2026 — Armis may reposition as a ServiceNow-platform capability rather than a standalone vendor. Averlon shipped "Precog" (May 2026, pre-production/CI-integrated exploitability evaluation ahead of merge) and joined Anthropic's Cyber Verification Program (Jun 2026). RunSybil raised a $40M round (Mar 2026, led by Khosla Ventures, incl. Anthropic/Menlo's Anthology Fund); valuation undisclosed |
| **Prioritization layers (CVSS + EPSS)** | rank the backlog | per-environment exploitability reasoning, plus lightweight mitigation paths | — |

**Honest competitive gaps.** Say these plainly: broad scanner replacement (out of scope) · PTaaS (rejected — defensive only) · OT/IoT (Phase 2+) · on-prem and air-gapped (Gate 5) · financial-impact quantification (Phase 3) · native mobile (Series A).

## 4. Press errata

Attach when a prospect cites the December-2025 press.

| Press claim | Gate-safe response |
|-------------|--------------------|
| Full pipeline at machine speed | **True** — Analyze → Mitigate → Remediate is live at Gate 1, unattended by default. Closed-loop validation is Gate 3 |
| "Continuous exploitability analysis" | **True** — continuous re-assessment ships at Gate 1 (ADR-016) |
| "Major U.S. enterprises" / millions of vulnerabilities | **Design partners under NDA.** No scale claims until the Gate-2 re-baseline |
| CEO: agents "write and run" code | **True** — sandboxed execution at Gate 1 (self-hosted Firecracker) |
| "Dux Technologies Inc." (SiliconANGLE) | **The contracting entity is Dux, Inc.** — confirmed (D-51). Dux's own Privacy Policy PDF names "Dux Technologies Inc." in error; that page needs a live-site correction (outside this repo, external follow-up), independent of the ZoomInfo/PitchBook corrections already in flight |
| "fastest safe fix" (BusinessWire) | **The phrase is retired, and stays retired** (V-13). Say **"fastest path to protection"** — the hero canon. Fix-*safety* validation is closed-loop, at Gate 3 (MC-05) |
| PR subhead: "shuts them down before they're used in an attack" | "Dux identifies exploitable paths and closes critical gaps instantly, reaching full protection at machine speed" — the [gtm-guardrails](gtm-guardrails.md) wording (MC-05) |
| FinSMEs interview: "already supporting major U.S. enterprises … hundreds of thousands of assets and millions of vulnerabilities" | **Design partners under NDA** (V-7). **No scale claims until the Gate-2 re-baseline** (V-9) — cite design-partner scale only (MC-06) |

> **Confirmed not Dux-supplied (2026-07-16, resolves [OI-11](../00-meta/open-items.md)).** The CybersecTools listing publishes NIST CSF 2.0 coverage percentages — ID 72% / PR 85% / DE 60% / RS 45% / RC 38% / GV 55% — that Dux never provided. Directory-generated, third-party fabrication. **These numbers must never be cited anywhere as Dux's own** — they don't match `compliance-program.md` §8's actual CSF crosswalk. Remaining action: request removal or sourcing from CybersecTools directly — tracked as an external-surface item under [OI-26](../00-meta/open-items.md).

## 5. Proof of concept (14 days)

| Phase | Days | Success criteria |
|-------|------|------------------|
| Onboard | 1–3 | AWS connector live; NDA and design-partner MSA executed |
| Assess | 4–10 | ≥10 exploitability assessments queued (US-010); trace export reviewed (US-017) |
| Review | 11–14 | CISO readout — the reduction delta (US-006), the top 3 validated findings, and the gate roadmap |

**The security excerpt** — a 1–2 page PDF covering tenant isolation (BR-001), the kill switch (BR-003), a data-flow diagram, and subprocessors — **ships before the first enterprise POC.**

**POC exit:** convert to a paid pilot ([pricing-packaging](pricing-packaging.md)), or record a documented disqualification.

## 6. ROI calculator

**Inputs:** critical findings per month; remediation capacity (%); engineer hourly rate.

**Phase-1 outputs:** triage time saved, and false-urgency reduction — **illustrative, from design-partner N = 3. Validate at N ≥ 10 before it enters signed collateral.** Plus a build-versus-buy comparison.

**MTTR reduction is a Gate 3+ output. Product MTTR is not measured in Phase 1** — keep it out of the calculator.
