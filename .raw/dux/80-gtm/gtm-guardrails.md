---
owner: GTM
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-10, H5, H9]
---

# GTM Guardrails — the Claims Firewall

**Purpose:** what may be said publicly, what may not, and the process gate that enforces it.

**Parents:** BR-010 · Pillar C.

**Every public claim must trace through the Marketing Claims → Technical Achievement map to a BR/FR/US at the same gate.**

**Scope of authority (D-10).** The claims map binds **GTM copy, product naming, and UI strings** — which is this document. It does **not** bind safety posture, control design, gate criteria, or SLOs; those are stated by engineering. A divergence between a live claim and engineering reality is raised in [open-items.md](../00-meta/open-items.md) — it is never resolved by editing a spec.

Data contracts are OpenAPI 3.1. Product behavior is BR → FR → US. Vision language is preserved in [vision-reference](../00-meta/vision-reference.md).

## 1. Why the guardrails inverted

The original GTM guidance **suppressed** claims — code execution was "Gate 2+", continuous was "Gate 3", Mitigate and Remediate were "Gate 3" — because the product was Analyze-only.

Under GCIS v2.2 those capabilities ship at Gate 1. **The claims became true, so the guardrails inverted: what was once suppressed is now safe to say.** Only two fences remain.

## 2. The two remaining fences

**Do not imply either has shipped before its gate.**

| Fenced capability | Gate | Guardrail |
|-------------------|------|-----------|
| Preference **learning** refinement | Gate 2c | It needs behavioral-data volume. The Gate-1 substitute is per-instance acknowledgment plus session routing preferences |
| Optional physical residency (an in-VPC agent) | Gate 5 | **"Lives inside your environment" means *logical* residency** — read-only APIs and OAuth — for Phases 1–4 |

## 3. Claim-safe at Gate 1

The live dux.io copy is the accepted GTM baseline.

| Claim | Status |
|-------|--------|
| "Continuous exploitability analysis / experts 24/7" | **True.** Continuous re-assessment (US-021, ADR-016), plus the on-demand queue |
| "Agents write **and run** investigation code" | **True.** Self-hosted Firecracker microVM execution at Gate 1; code artifacts and executed results appear in traces |
| "Ties together all data sources / auto-tags every asset" | **True, unqualified.** Integration-catalog wave coverage is tracked separately, as an engineering roadmap item — not as a messaging constraint |
| "Lightweight mitigations / rapid remediation"; "AI agents blow past the usual bottlenecks" | **True, unqualified.** No HITL caveat is required in customer-facing messaging |
| The full Analyze → Mitigate → Remediate pipeline | **True end to end at Gate 1** |
| "Machine-speed analysis" / "full protection at machine speed" | **True, unqualified, end to end** |
| "Instant" — "instant exploitability analysis", "close critical gaps instantly" | **Claim-safe as stated.** Both are accepted verbatim |
| **"Zero-day investigated in minutes"** | **Qualified — this row is deliberately unchanged.** A design-partner anecdote, carrying the FR-004 MTXV qualifier. **Never an SLA** |

> **The "zero-day in minutes" qualifier still stands, and it is not the same claim as "instant".**
>
> It is true **per CVE**. **Environment-wide sweeps are queue-paced, not minutes-scale** — the D-9 sandbox tenant caps (300 sandbox-seconds/hour, 5 concurrent microVMs) bound execution-backed sweeps to **hours** at fleet width (SR-11; competitor-scan CS-19).
>
> **Never imply that a full-environment zero-day sweep completes in minutes.** The live dux.io copy makes no environment-wide-minutes claim, so nothing is being suppressed here.

## 4. Permanent rules

**Customer references.** Reference only **"enterprise design partners under NDA"**, unless written permission exists for a named reference. **Never use "major U.S. enterprises" as a stand-in proof claim.**

**Scale.** **Never use generalized scale language** — "hundreds of thousands of assets", "millions of vulnerabilities" — in signed collateral until the Gate-2 re-baseline (≥100 assessments/day) is complete. **Cite design-partner scale only.**

**Residency.** Logical, not physical, for Phases 1–4. Gate 5 is physical, and remains roadmap.

**Never claim, at any gate:** scanner replacement · PTaaS or offensive execution · OT/IoT discovery · an on-prem resident agent before Gate 5 · financial-impact quantification.

**The CTEM benchmark.** "Testing exploitability reduces false urgency by up to 84%" is anchored to CTEM-validation research (see [competitive](competitive.md)). **It is a category statistic, never a Dux SLA.**

**Illustrative numbers.** The 8,341 → 2,143 funnel and the reduction statistics are **illustrative**. Validate them with design partners before they enter signed collateral.

**Outcome claims (C1 / C2).** "Materially smaller attack surface" and "far shorter path from vulnerability discovery to resolution" are accepted baseline messaging.

**But the *measured* numbers do not exist yet.** They arrive with Phase-1 exit instrumentation (H9 — cache-hit and MTTP distributions). **Until then, make the claims without attaching specific figures** — the same numeric discipline as the illustrative funnel.

**Ecosystem mentions.** The CrowdStrike / AWS / NVIDIA 2026 Accelerator (Dux among 35 startups), and InfraRed-100-style listings, are **ecosystem validation only**, used with GTM approval. **Never present them as product-feature claims.**

**Naming.** Public/press materials have used "AI-workers" — never use it in Dux-authored copy. Canonical is **Dux Agent** (historical usage recorded in [vision-reference.md](../00-meta/vision-reference.md)).

## 5. Customer qualification

| Customer asks | Say | Do not say |
|---------------|-----|------------|
| "Do you run agents in our VPC?" | "No — Dux runs in our cloud, and connects via read-only APIs and OAuth. Optional physical residency is a Gate 5 roadmap option." | "Yes, Dux lives inside your network." |
| "Is this continuous and real-time?" | "Yes — continuous re-assessment and connector polling. Real-time webhooks for supported integrations." | "Everything is real-time." |
| "Can you see runtime behavior?" | "Yes, if you connect CrowdStrike or SentinelOne and grant read scope. Otherwise we reason from asset and network context." | "We see all runtime behavior." |
| "Can Dux remediate automatically?" | "Yes — Dux agents identify and deploy lightweight mitigations and accelerated remediation, blowing past the usual bottlenecks to get urgent fixes done fast." | **Do not understate it.** This is the accepted baseline |
| "Do you shut exposures down automatically?" | "Dux identifies exploitable paths and closes critical gaps instantly, reaching full protection at machine speed." | **Do not understate it.** |
| "Is the analysis instant?" | "Yes — instant exploitability analysis, reaching full protection at machine speed." | **Do not add a "starts / streams / completes-in-minutes" hedge.** That qualifier is retired |
| "Do you think like an attacker?" | "We apply an attacker-minded lens to determine real-world exploitability in your environment — **defensive analysis only, not PTaaS**." | "We hack your environment." / "We run pen tests." |

## 6. Process gate

**The pre-publication claims checklist is blocking.** Signed by PM and Counsel, stored with the CRM opportunity, and required for **every RFP response, order form, and sales-deck revision**. **An RFP answer without a stored checklist is a blocked send.**

**The checklist also covers founder interviews, social posts, and conference talks** (MC-06, MC-11). **Any statistic in a public post must carry its source URL and date.** The trigger case, now resolved (D-49): the "65-day MTTR" RSA-2026 LinkedIn post is confirmed real — [linkedin.com/posts/duxsecurity_rsac2026-...-GjjN](https://www.linkedin.com/posts/duxsecurity_rsac2026-ctem-vulnerabilitymanagement-activity-7440388981793546240-GjjN), ~2026-03 — citable going forward with that URL attached, per the rule this trigger case itself established.

**Known false claims about Dux, caught 2026-07-21 while re-verifying the post above — never repeat these, in any external or internal material, unless independently re-confirmed:** (1) a claimed Dux-hosted RSA 2026 reception — the actual host was the unrelated "Dune Security"; (2) a claimed Dux placement on Redpoint's 2026 InfraRed 100 list — Dux does not appear on the real list. Both are plausible-sounding, AI-search-summary-shaped claims about Dux with no primary source underneath — the same failure mode the LinkedIn-post re-verification above was checking for. Neither had entered this corpus before being caught.

Attach the [feature-availability matrix](competitive.md) to every RFP and contract.

### Public-surface launch blockers (GCIS §E)

| Item | Required state |
|------|----------------|
| `trust.dux.io`, `status.dux.io` | HTTP 200 |
| Privacy policy | served from `dux.io/privacy`, **not Google Drive** |
| Footer | current year |
| ZoomInfo entity | Dux, Inc. |
| CybersecTools listing | corrected |
| cyberdb.co | corrected — it listed "Establishment 2024 / Company Stage Mature / $3M revenue", which is inaccurate for a Dec-2025 stealth launch. **The listing is no longer locatable and may have been removed — verify, then close** |

### Listing-correction sweep

| Surface | Correction |
|---------|-----------|
| PitchBook profile 771870-88 (MC-09) | says "headquartered in New York, NY", **dropping the Tel Aviv R&D HQ** — submit a correction |
| clawandtalon.capital (MC-10) | founding year reads "2025"; **the actual year is 2024** |
| cybersecurityintelligence.com (MC-17) | also drops the Tel Aviv R&D HQ — New York only |

**Funding databases and directory descriptions are part of the §E correction checklist**, not just the original ZoomInfo / CybersecTools / cyberdb trio.

### Archive step (MC-15)

**The Wayback Machine holds zero 2025–2026 captures of dux.io.**

After every site revision — **and once now** — trigger Save Page Now on `https://dux.io/`, and store the capture URL in [vision-reference](../00-meta/vision-reference.md) provenance.

**Without it there is no independent record of what the public surface claimed, and when.** For a company that treats its live claims as binding on GTM copy, that record is not optional.

## 7. Claim-safe short-form block

Reusable collateral. Matches the dux.io hero copy.

> Dux is an agentic exposure management platform, built for a world where vulnerability exploitation windows are collapsing from weeks to minutes. Use Dux Agent for instant exploitability analysis, rapid mitigation, and accelerated remediation, so you reach full protection at machine speed. Dux ties together all your data sources and auto-tags every asset, to make sure urgent fixes get done fast.

**Scale and customer-reference claims stay in design-partner / NDA framing** until formally re-baselined or permissioned.

## 8. "How it works" short-form block (resolves MC-14)

**Draft copy, ready to paste onto the site if Sagi approves the site-strategy call.** Surfaces the differentiators the corpus says win deals — executed-code traces (US-017), the CaMeL dual-LLM boundary, the kill switch, and HITL — without adding a single claim not already true at Gate 1.

> **Proof, not a guess.** Dux doesn't score a CVE and hope — it writes and runs real investigation code in an isolated sandbox to prove whether a vulnerability is actually exploitable in your environment, and shows you the code and the result.
>
> **A safety boundary between the internet and your environment.** Untrusted data from public CVE feeds is reasoned over in a separate, sandboxed layer from the one that has access to your environment's context — so a poisoned advisory can't manipulate what the agent does with your data.
>
> **A kill switch you control.** Any agent, any session, any customer — halted in under 5 seconds, on your command, not just ours.
>
> **Human approval where the blast radius is highest.** Isolating an endpoint or patching a firmware-only device always waits for your sign-off. Lower-risk actions — like updating a blocklist or opening a ticket — execute automatically, fully audited, so your team isn't the bottleneck on the easy calls.

**Framing rule:** this block describes mechanisms (sandbox execution, CaMeL boundary, kill switch, HITL tiers), not outcomes or speed — it does not need the "instant"/"machine speed" fences in §3, and must not be edited to add them.
