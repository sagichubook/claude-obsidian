---
owner: GTM
status: process-record
gate: n/a
last_reviewed: 2026-07-17
decisions: []
---

# External Corrections — 2026-07

**Purpose:** ready-to-use text for external-surface actions (site copy Sagi/GTM can paste directly, and correction-request text for third-party listings) that no doc edit inside this repo can close by itself. **Nothing here is published by this repo** — it's collateral, prepared so acting on it is copy-paste, not research.

**OI-26 closed 2026-07-17** (see [decisions-log](../00-meta/decisions-log.md)) — the register entry is gone, but the work isn't done. This file is now the action-item tracker of record for what's left: the DNS work below, physically pasting/submitting the text below, and the MC-14 publish decision (still an open Founder/GTM call, not decided here).

**Facts used below** (already established elsewhere, not invented here): legal entity is **Dux, Inc.** (not "Dux Technologies Inc." — [vision-reference §7](../00-meta/vision-reference.md)); HQ is **Tel Aviv, Israel (R&D) + New York, USA (GTM)** dual-HQ (vision-reference §7); founded **2024** (IVC listing, IVC-verified); seed round co-led by **Redpoint, TLV Partners, and Maple Capital** (competitive.md §7 funding facts).

## 1. Site copy (dux.io, Framer) — paste directly

**MC-13 — banner omits a co-lead.**

> Was: "led by Redpoint and TLV Partners"
> Fix: "led by Redpoint, TLV Partners, and Maple Capital"

**Footer year.**

> Was: "© 2025 Dux, Inc." (or equivalent)
> Fix: "© 2026 Dux, Inc." — better long-term: a `{{current_year}}` template token if Framer supports it, so this doesn't go stale again next January.

**MC-08 — seed-round banner links to the SiliconANGLE article, which carries the wrong entity name ("Dux Technologies Inc.").**

> Retarget the banner's hyperlink from the SiliconANGLE URL to the BusinessWire launch PR (`businesswire.com/news/home/20251216193951/en/...`) — the canonical, correctly-named source — or to an owned blog post once one exists. Do not ask SiliconANGLE to change their own headline; retarget the link instead, since that's fully in Dux's control.

**Privacy policy — currently a raw Google Drive link (MC-01).**

**This is a real legal document. The draft below is a structural skeleton for Legal to fill in and approve — do not publish it as-is.** It exists so the page has real content instead of a Drive link, not to skip legal review:

> ## Privacy Policy
>
> **Effective date:** [Legal to set]
>
> **1. Who we are.** Dux, Inc. ("Dux," "we," "us") operates an agentic exposure-management platform. Contact: privacy@dux.io.
>
> **2. Data we collect.** Account and authentication data (name, email, role); connected-environment metadata via your configured connectors (asset inventories, vulnerability findings, control state) — see [connector-hub](../10-product/features/connector-hub.md) for what each connector reads; product-usage telemetry.
>
> **3. How we use it.** To operate the exposure-assessment and mitigation features you configure; to secure and audit the platform (hash-chained audit trail, retained per our [retention policy](../20-architecture/data-model.md)); to communicate service and security notices.
>
> **4. Subprocessors.** See `trust.dux.io/subprocessors` for the current list [Legal/GTM to confirm this page is live before publishing this policy — it's also an MC-01 blocker].
>
> **5. Your rights.** Export and deletion requests: `POST /tenants/{id}/export` and account-level GDPR deletion, per our [data lifecycle](../60-operations/customer-lifecycle.md) — a one-month export window, then a 90-day purge. EU/UK data-subject rights requests: privacy@dux.io.
>
> **6. Security.** Encryption at rest and in transit; see our [compliance program](../70-governance/compliance-program.md) for the current control posture.
>
> **7. Changes to this policy.** [Legal to set the notice mechanism and cadence.]
>
> **[Legal must review and approve before this replaces the Google Drive link. Sections 4–6 point at real, already-true corpus facts; sections 1, 5, and 7 need Legal's actual wording for effective date, jurisdiction, and change-notice process.]**

## 2. Third-party listing corrections — submit via each site's correction/dispute form

**PitchBook** (`pitchbook.com/profiles/company/771870-88`):

> Correction request: (1) "automated remediation" overstates current capability — 3 of 5 canonical write actions execute unattended with full audit, but the 2 highest-impact actions (endpoint isolation, firmware patching) always require human approval before executing. Please revise to "AI-driven exploitability assessment and mitigation, with human approval on the highest-impact actions." (2) HQ is listed as "New York, NY" only — please add Tel Aviv, Israel as the R&D headquarters (dual-HQ: Tel Aviv R&D + New York GTM).

**ZoomInfo** (`zoominfo.com/c/dux-technologies-inc/547149997`):

> Correction request: the listed entity name/slug "Dux Technologies Inc." is incorrect. The legal entity is **Dux, Inc.** Please update the company name and, if possible, the URL slug to match.

**clawandtalon.capital** (`startup-dux-security.html`):

> Correction request: (1) founding year is listed as 2025; the company was founded in **2024**. (2) "remediation automation" — please revise to match the current, more precise description: AI-driven exploitability assessment with automated mitigation on lower-risk actions and human approval on the highest-impact ones.

**cybersecurityintelligence.com** (`dux-security-12258.html`):

> Correction request: HQ is listed as "New York, New York, USA" only. Dux is dual-HQ'd — Tel Aviv, Israel (R&D) and New York, USA (GTM). Please update to reflect both.

**SiliconANGLE** (`siliconangle.com/2025/12/16/...`):

> This is a live, immutable press article — SiliconANGLE is unlikely to edit a published piece. Lower priority than the four above. If pursued: request a correction note appended acknowledging the entity is **Dux, Inc.**, not "Dux Technologies Inc." Otherwise, handle via the site-side banner retarget in §1 instead of chasing an article edit.

**CybersecTools** (`cybersectools.com/tools/dux-security`):

> Correction/removal request: the listing publishes NIST CSF 2.0 coverage percentages (ID 72% / PR 85% / DE 60% / RS 45% / RC 38% / GV 55%) attributed to Dux that Dux never supplied — directory-generated, not sourced from us, and inconsistent with our own published CSF crosswalk. Please either remove these percentages from the listing or clearly label them as third-party estimates, not Dux-sourced data.

## 3. What's still not covered here

- **trust.dux.io / status.dux.io / docs.dux.io DNS** (MC-01) — infrastructure work (point DNS, deploy pages), not copy. Out of scope for this file.
- **MC-14 "how it works" section** — drafted separately in [gtm-guardrails §8](gtm-guardrails.md#8-how-it-works-short-form-block-resolves-mc-14).
