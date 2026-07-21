---
owner: Engineering
status: canonical
gate: 2
last_reviewed: 2026-07-14
decisions: [D-18]
---

# Customer Lifecycle & Comms

**Purpose:** onboarding, offboarding, billing, status-page copy, and health monitoring.

**Parents:** BR-001, BR-006. Aligned to the [tenant lifecycle](../20-architecture/multi-tenancy.md).

## 1. Onboarding

**Provisioning.** Create the tenant through the NestJS API (idempotent) → validate the AWS cross-account IAM role → send the welcome email and admin training video → run the RLS verification auto-test → health check: **a first asset sync and assessment within 7 days**.

**7-day onboarding health-check alert (resolves OI-20).** `DuxOnboardingHealthCheckMissed` fires at exactly 7 days post-provisioning if the tenant has **no** completed first sync **and** no completed first assessment — checked by a scheduled job (`admin:onboarding-health-sweep`, daily), not the 2-week CSM cadence in §7. Routing:

1. Page `@customer-success-oncall` (not engineering — this is a customer-motion gap, not a platform incident).
2. CSM runbook: check `aws_role_status` and `connector_configs.status` for a stuck `aws_role_failed`/`credential_revoked` state (§ provisioning, §7); if clean, reach out to the tenant admin directly.
3. If unresolved by day 10, escalate to the day-14 CSM health-score churn-risk trigger early rather than waiting for the regular cadence.

This closes the up-to-14-day detection gap the CSM-only cadence left: worst case is now 7 days, not 14.

**The 9-step onboarding procedure:**

1. Provision, and verify RLS.
2. Land the customer on the Dashboard (US-012).
3. Admin training — a 30-minute video.
4. Connect AWS, via Apps (US-013).
5. Run the first assessment.
6. Demonstrate the kill switch (L1 and L3).
7. Test the audit-log export.
8. Optionally, test an API key and a webhook.
9. Document the escalation path.

## 2. Offboarding

Soft-delete at day 0 → days 0–30, export bundle available (JSON/CSV for the customer, Parquet for the internal archive; 24 h SLA), revoke sessions/keys/agent credentials → days 31–90, legal-hold retention (`legal_hold` flag blocks the purge and notifies Legal) → day 90, hard purge across MinIO, the database, and backups → email the destruction certificate. This matches [multi-tenancy §5](../20-architecture/multi-tenancy.md), the lifecycle authority.

## 3. Trust and status portal gates

| Tier | Requirement | Blocks |
|------|-------------|--------|
| **P0** | `status.dux.io` (uptime + incident copy) and `trust.dux.io` (home, `/subprocessors`, security contact, status link, SOC 2 readiness note) **return HTTP 200** | **onboarding the first NDA design partner** |
| P1 | portal shell complete — engineering milestone 2026-09-30 | — |
| **P2** | content-complete: SOC 2 summary, pentest executive summary, security FAQ | **procurement — the first $100 K ACV, or an enterprise questionnaire** |

**Interim exception.** Provisioning a partner before P0 requires a **CEO and Security Officer signed risk acceptance, per tenant** — artifact `RA-TRUST-INTERIM-{tenant_id}-{YYYYMMDD}`, filed in the provisioning ticket.

## 4. Billing metering

Stripe meter events for tokens and agent runs. Daily reconciliation of platform usage against the Stripe meter. Invoice line items must match the usage dashboard. Overage alerts fire at 80%, 100%, and 120% of quota.

**Drift above 5% → the [billing reconciliation drift runbook](runbooks.md).**

## 5. Status page and incident comms

`https://status.dux.io` — **a launch blocker. It must return 200 before the first design partner.**

Approver: Founder or PM before Gate 2; `@product-oncall` after. **SLA: 15 minutes.** Subscribe-by-email is available, and 99.5% historical uptime is visible.

**Literal copy:**

| Severity | Status page | In-app banner | Email subject |
|----------|-------------|---------------|---------------|
| P0 platform outage | "Dux is experiencing a platform-wide issue affecting dashboard and API access. Our team is investigating." | "Platform issue — some features unavailable. See status.dux.io." | "Dux — service disruption (investigating)" |
| P1 tenant incident | "Some customers may experience delayed assessments. We are investigating." | "Mitigation queue delayed — your data is safe." | "Dux — temporary delay in exploitability analysis" |
| P0 AI safety (KS-L4) | "We detected a potential data isolation issue and paused AI analysis as a precaution." | "AI features temporarily unavailable — your data is being reviewed." | "Dux — precautionary pause of AI analysis (action may be required)" |
| Degraded AI (fallback) | "AI analysis is running in reduced-capability mode. Results may take slightly longer." | "Exploitability analysis using backup AI provider. No action required." | "Dux — temporary AI provider switch (no data impact)" |
| KS-L2 (tenant assessment) | "Agent features paused for your organization pending security review." | "AI analysis paused by your administrator or Dux safety controls." | "Dux — AI analysis paused for your organization" |
| KS-L3 (tenant platform) | "Your organization's dashboard is in read-only mode while we review a billing or security matter. Existing data remains accessible." | "Dashboard read-only — new assessments paused. Contact your administrator." | "Dux — read-only mode for your organization" |

## 6. Support tiers

An add-on, **separate from product packaging**.

| Tier | Response | Channels | Price |
|------|----------|----------|-------|
| Standard | 24 h, business days | email, docs | included with Starter |
| Professional | 8 h, business days | email, Slack Connect | ~$500/month add-on (model) |
| Enterprise | 2 h; 24×7 for P1 | Slack, email, phone | custom, ~$2 K+/month. Named CSM, QBRs |

## 7. Health monitoring

**Signals.** Product usage (weekly assessments per tenant); support (ticket volume, time to resolve, CSAT); expansion (quota utilization at 80% signals expansion); risk (**14 days with no assessment raises a churn alert**).

**Three health formulas exist. They do not merge.**

| Formula | Composition | Owner | Escalation |
|---------|-------------|-------|------------|
| **CSM health score** (0–100) | assessment frequency 40% · CSAT 20% · connector health 20% · quota trend 20% | CSM | **Below 50 for 2 weeks** → churn-risk trigger + EBR: usage trends, open incidents, expansion blockers, a 90-day plan |
| **TenantHealthScore** (Series A) | reliability 40% · cost 30% · safety 30% | AI Safety Lead | below 50 routes to **Security and FinOps — not CSM** |
| **Governance dashboard v1** (Seed Month 3, Grafana `governance-dashboard-v1`) | queue depth 30% (>50 pending per tenant → CSM outreach) · reduction delta 25% (a negative 2-week trend → exec review) · kill-switch activations 25% (**any L3 or L4 → mandatory audit**) · connector health 20% (AWS sync stale >24 h → P2) | PM + Security | **Green ≥70 · Yellow 50–69** (CSM outreach) **· Red <50** (exec review, optional L2 freeze) |

## 8. Triage-value hypothesis

**A GTM input, not an SLO.** Practitioners report that the majority of vulnerability-management time goes to triage rather than remediation.

**This is qualitative and unvalidated.** It awaits design-partner validation at N ≥ 5, with a decision target of Gate 2b GTM sign-off. ROI-calculator inputs live in [competitive](../80-gtm/competitive.md).
