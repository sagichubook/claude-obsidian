---
owner: Founder
status: canonical
gate: n/a
last_reviewed: 2026-07-21
decisions: [D-20, D-33, D-34, D-39, H5]
---

# Compliance Program

**Purpose:** the certification programmes — SOC 2, ISO 27001/42001, EU AI Act — plus policies, evidence automation, and AI governance.

**Stage:** Series A (Year 2+).

**Format: delta only.** Canonical RBAC, incident-response, and tenant controls link to the earlier documents. Counsel-owned policy prose is version-controlled separately, and referenced by version rather than inlined.

**Activate on events, not on the calendar.**

## 1. Triggers

**Company signals:** roughly 30+ engineers (40–50 employees); 16+ FTE unlocks dedicated incident roles; NHI (non-human identity, e.g. service accounts/API keys) inventory above 500; the first $100 K+ ACV; board risk reporting; AI vendor security reviews; EU enterprise buyers requiring ISO 27001 and 42001.

| Trigger | Starts work on |
|---------|----------------|
| An enterprise renewal or RFP requiring Type II | SOC 2 Type II observation begins |
| An EU enterprise RFP | ISO 27001 + ISO 42001 gap assessment |
| An AI vendor security review | AIBOM governance, an AI red-team report, model-card updates |
| NHI inventory above 500 | NHI policy formalization; OWASP NHI Top 10 assessment |
| The first $100 K+ ACV | the customer self-service security portal — **a procurement blocker** |
| Gate 3 passes | amend the SOC 2 scope to cover the closed-loop validation surface (US-019). **The Mitigate and Remediate write surfaces are in scope from program start — they are live at Gate 1** |
| A federal RFP, or a FedRAMP path | the NIST AI RMF crosswalk. **The AWS-hosting infrastructure prerequisite is restored (2026-07-19, D-34)** — the platform runs on **Amazon EKS** ([ADR-006 R4](../20-architecture/architecture-overview.md)), a FedRAMP-authorized CSP with GovCloud availability. Per the v4.0 source doc's FedRAMP compliance matrix: EKS ✅ Moderate / High (GovCloud); Bedrock ✅ Moderate / ⚠️ High (limited regions); CloudNativePG, Valkey, NATS, MinIO, self-hosted Temporal, and NestJS ✅ (self-hosted, in-boundary). This reverses D-33's brief window where DigitalOcean/Linode Kubernetes closed the FedRAMP path. **FedRAMP is not a near-term target (2026-07-20, D-39)** — it stays gated on an untriggered federal-RFP condition, not a committed claim. The EKS/GovCloud-capable infrastructure keeps the option open cheaply; a dedicated FedRAMP authorization push is not committed ahead of a real federal deal. |

## 2. SOC 2

**Type I timeline**, where seed activation is Month 0:

| Months | Work |
|--------|------|
| M1–3 | controls; evidence into S3; gap assessment |
| M4–5 | questionnaire responses |
| **M6** | **readiness letter** — not a report |
| **M9–12** | **Type I audit engagement** |

**Type II observation starts only after the Type I report is issued** — not at the readiness letter.

> **"Seed activation" is itself the qualifying event that starts the Month-0 clock.** That is consistent with "activate on events, not calendar", not an exception to it. The "Gate 3 passed" trigger in §1 is a separate, later **scope-amendment** event on top of an already-running programme — **not a second timeline anchor.**

**Type II Trust Services Criteria:**

| Criterion | Included when |
|-----------|---------------|
| Security (CC6/7/8) | **always — the minimum** |
| Availability (A1) | ≥2 SLA contracts exist. **A1 controls must operate for ≥3 months before the letter** |
| Confidentiality | NDA classification applies |
| Processing Integrity | scores are used for compliance decisions |
| Privacy | personal data beyond telemetry is handled |

Observation windows: 6 months for the first; 3–12 months auditor-scoped; 12 months on renewal.

**Continuous evidence:** quarterly access reviews; weekly vulnerability scans; an annual external pentest plus semi-annual internal ones; annual policy reviews; quarterly backup tests.

**SLA language is counsel-gated** (AI-226a).

## 3. ISO 27001 / 42001

### ISO 27001 (ISMS)

**Scope:** the Dux SaaS platform, multi-tenant isolation, Dux Agent governance, MCP and CaMeL, connectors via the Unified Integration Layer, the Mitigation and Exposure UI, auth, and support.

**Exclusions:** Gate-5 physical residency, and third-party MCP server internals — covered by a customer responsibility letter.

**Both ISO 27001 and SOC 2 include the Mitigate and Remediate write surfaces from program start**, because those surfaces are live at Gate 1. The "Gate 3 passed" amendment in §1 covers **only** the closed-loop validation surface (US-019).

**Risk treatment:**

| Risk | Treatment | ISO/IEC 27001:2022 Annex A |
|------|-----------|------------------------------|
| Cross-tenant leak | RLS; ISO-001–010 | A.8.3 (information access restriction), A.8.16 (monitoring activities) |
| Agent incident | kill switch; MCP controls | A.5.24 (incident management planning), A.8.16 (monitoring activities) |
| Supply chain | model pins; DPAs | A.5.19 (supplier relationships), A.5.20 (addressing security in supplier agreements) |
| Admin access | MFA; break-glass | A.5.15 (access control), A.8.5 (secure authentication) |

**Annex A numbers are cited at the internal-control-ID level (ISO-001–010), not per sub-clause** — the full clause-by-clause Statement of Applicability is a §3 ISO 42001 M3–6 deliverable, not this table.

**Internal audit** is quarterly and thematic — Q1 access and NHI, Q2 change, Q3 AI and MCP, Q4 tenant isolation — reaching full clause coverage across 12 months. Management review is annual. Kickoff: Series A Month 1.

### ISO 42001 (AI management system)

**Scope:** the Dux Agent lifecycle (on-demand queue, connector sync, and continuous re-assessment per ADR-016 — **all Gate 1**; ownership inference at Gate 1; preference learning at Gate 2c), assessment governance, golden-set calibration, the kill switch, model management, and the AI impact assessment.

**Milestones:** gap M1–3 · risk and Statement of Applicability M3–6 · internal audit M6–9 · certification M9–15. Quarterly management review.

**Roadmap artifact pulled forward (architecture-panel-review-2026-07 §5.3 M2).** Certification itself stays fenced to Series A, but the roadmap artifact — this milestone table, dated and board-presentable — ships now, ahead of certification, because ~40% of EU AI RFPs already screen on a stated ISO 42001 path and competitors (AWS, Anthropic, OpenAI, ServiceNow) are already certified. **Target: M2 (Board pack §6) — the same board pack that already carries risk, certifications, AI governance, and DORA.** This closes the readiness gap without pulling certification spend forward.

**Run ISO 42001 as a 60–70% extension of the SOC 2 / ISO 27001 evidence base** — a unified control map, 4–6 months with guidance. **Parallel programmes fail roughly 60% of the time, on duplicate evidence and control gaps.**

**Annex III high-risk classification determination (owners: Founder + Counsel).** Annex III §5 covers AI systems used for critical-infrastructure management and operation — the category most plausibly in play, since Dux's agent makes autonomous security-remediation decisions on customer infrastructure. **Working position: Dux does not trigger Annex III.** Reasoning: (1) Dux is a security tool sold to and operated by the infrastructure operator — it is not itself the critical-infrastructure system, nor does it control the infrastructure's core function (network routing, power, water, transport) the way Annex III targets; it acts on security posture (endpoints, network policy, tickets), not on operational-technology control loops. (2) Of the two highest-blast-radius write actions, `endpoint.isolate` and `patch.deploy_special_devices`, **mandatory human-in-the-loop approval is preserved on every call regardless of confidence** ([kill-switch-hitl.md §7](../40-ai-safety/kill-switch-hitl.md), D-17) — there is no autonomous decision loop over infrastructure without a human in it for the actions with the largest impact. (3) The remaining three canonical write actions are reversible, low-blast, and tenant-scoped (`network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`). **This is a working position, not a filed determination** — it has not been counsel-confirmed, and a customer whose own system Dux touches (e.g. an EU operator of energy or transport infrastructure) could still pull Dux into that customer's own Annex III assessment as a component supplier. Track under the Week-6 counsel opinion below; revisit if Annex III's own scope guidance (pending CEN-CENELEC harmonized standards) narrows or widens the "management and operation" language.

**EU AI Act Art. 9** risk management runs through the ISO 42001 impact assessments (owners: Security + Counsel). As a high-risk-system obligation it tracks the Annex III **2 Dec 2027** horizon (D-26), and — since Dux's working position is that it does not trigger Annex III — is run as best practice rather than a triggered requirement. **The live near-term obligation is Art. 50 transparency (2 Aug 2026), below.** Annex IV conformity is deferred to Series B, or to the first EU high-risk contract.

**Art. 9 sub-items, mapped to existing Dux mechanisms** (in scope regardless of the Annex III determination above — Art. 9 discipline is good practice, and the determination could still change):

- **Data governance and data quality** → [data-model.md](../20-architecture/data-model.md) — schema, retention matrix, referential integrity, tenant purge order.
- **Technical documentation** → this corpus plus the ADR set ([adr-index.md](../20-architecture/adr-index.md)) — architecture, design rationale, and control decisions kept current and versioned.
- **Record-keeping** → the hash-chained audit trail, 7-year MinIO Object Locking retention with a deny-delete break-glass role (§4 Logging and monitoring above; §7 below).
- **Transparency and provision of information to users** → the chat interface's agent framing ([chat-guidance.md](../10-product/features/chat-guidance.md)) — "not a general-purpose security chatbot," every turn routed through governed agent workflows and surfaced with citations, so users interact with a clearly-scoped AI agent rather than an unlabeled assistant.
- **Human oversight** → [kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md) §7 HITL contract, D-17 — mandatory live approval on `endpoint.isolate` and `patch.deploy_special_devices`, anomaly-escalation HITL on the other three canonical actions.
- **Accuracy, robustness, and cybersecurity** → [confidence-calibration.md](../40-ai-safety/confidence-calibration.md) — Platt scaling, abstention bands, confidence-floor escalation; the golden-set eval suite (250 CVE×environment pairs, sha256-locked, stratified).

**AI impact assessment procedure:** an intake form; a 5-business-day Legal SLA; **it blocks EU contracts until sign-off**, filed in `s3://dux-evidence-prod/ai-impact-assessments/`.

> **Status (D-26).** The EU **Digital Omnibus on AI is in force** — Parliament adopted it 16 Jun 2026, Council 29 Jun 2026, and it entered into force in July 2026 on Official Journal publication. **Annex III high-risk obligations are now 2 Dec 2027** (product-integrated high-risk 2 Aug 2028). **Article 50 transparency obligations did not move — they apply from 2 Aug 2026**, independent of any high-risk classification.
>
> **The near-term live gate is Article 50, not Annex III.** The Annex III cliff is 16 months out; the obligation that binds now is chat-surface transparency — disclosing that a user is interacting with an AI system. The **Week-6 counsel opinion is scoped to Art. 50 compliance on the chat surface** as the immediate item; the Annex III determination below runs on the longer 2 Dec 2027 horizon.

> **ISO 42001 is not EU AI Act conformity.** ISO 42001 certifies the *organization's* AI management system; the Act regulates the *system and the product*.
>
> The harmonized standards that grant presumption of conformity — **prEN 18286** (QMS), **prEN 18228** (risk), **prEN 18282** (cybersecurity) — are CEN-CENELEC JTC 21 work, targeted for late 2026. **That is the actual conformity path. Never imply that ISO 42001 alone means EU-compliant.**
>
> **Procurement framing:** SOC 2 alone is now "necessary but not sufficient" for an AI vendor. Lead with **SOC 2 + AI governance** — AI-BOM, model cards, and the ISO 42001 roadmap.

## 4. Policies (Series A deltas)

**Access control.** Dual approval for `platform_admin`. **Break-glass and provisioning approvers must be different people.** Break-glass is two-person, 4 h maximum. Quarterly reviews. The RBAC matrix crosses tenant roles `admin` / `member` / `viewer` ([`USER.role`](../20-architecture/data-model.md#2-core-entities)) plus the separate `platform_admin` (Dux internal staff) against the KS-L scopes — see the endpoint-level matrix in [api-overview.md §3](../30-api/api-overview.md#3-auth).

**Change management.** A standard RFC needs 1 approval; 2 for anything security-sensitive. An emergency change may be verbal, with a retroactive RFC within 24 h.

**A model-pin change requires all of: security review + AIBOM + golden set + Promptfoo + 48 h of monitoring.** A violation is a P1, and a SOX ITGC weakness.

**SOX ITGC observation.** The clock starts at the first $100 K+ ACV, or on IPO exploration. It runs 12+ months continuously, with an ITGC ↔ SOC 2 mapping.

**NHI policy.** A creation ticket plus approval. Rotation: keys and agent credentials every 90 days; CI/CD every 180. Revocation within 24 h. Quarterly inventory. The OWASP NHI Top 10 appendix (NHI1–10) applies — see §5.

**Data retention.** Cross-region pseudonymization: EU and APAC admin logs go to a US aggregate, with SHA-256 salted actor and IP, and email stripped. The salt rotates annually, with a 7-day dual window. SCCs apply. **Month 6 deadline.**

**GDPR Art. 17 (right to erasure).** The tenant purge order ([data-model.md §3](../20-architecture/data-model.md)) is the mechanism: halt workflows and trip the kill switch → delete the S3 prefix → `DELETE FROM tenants` (cascade) → revoke Vault/SSM secrets → write the `tenant.purged` audit record. Export and delete complete within **<24 h** ([TR-NFR-009](../60-operations/observability-slo.md)); `chat_messages` (PII in `content`) and `mcp_tool_invocations` are purged on hard delete per the retention matrix.

**Live public-surface privacy commitment (dux.io Privacy Policy, effective 2025-12-01) — already binding today, ahead of the Series A programme above.** Dux already represents itself as a GDPR "Processor" and CCPA "Service Provider" for customer information, and already discloses that personal information is stored via AWS in both the US and EU regions. This is a present, external legal commitment independent of when the Series A SOC 2/ISO 27001 programme itself kicks off — the two should not be conflated: the privacy commitments above are already live; the certification programme is not. The same page also displays "ISO 27001 Certified" and SOC marks that are **not** backed by this programme — confirmed as an unearned claim requiring a live-site fix, outside this repo (D-44).

**GDPR Art. 32 (security of processing / encryption).** Encryption at rest and in transit is already implemented, not newly added here: per-tenant envelope encryption on MinIO object storage (a DEK wrapped by the platform KEK in Vault — [multi-tenancy.md §5](../20-architecture/multi-tenancy.md)); connector credentials AES-256 in JSONB with Vault transit for OAuth ([adr-index.md](../20-architecture/adr-index.md)); self-hosted Temporal workflow/activity payloads encrypted per-tenant via a custom `PayloadCodec` before leaving the worker process, cert-manager/Vault-issued mTLS on the transport itself (adr-index.md D-16 R2). Self-hosted Temporal never holds plaintext payloads.

**Personnel.** Background checks; a 14-day security module.

**Vendor management.** Questionnaire, DPA/SCC, annual review, and a subprocessor register.

**Subprocessor-register update (resolves the remaining part of SR-17).** Unleash is **self-hosted** (`docker compose` locally, a Kubernetes-native Deployment in production — [architecture-overview §6](../20-architecture/architecture-overview.md)), not a SaaS subprocessor, so it does not carry a DPA/SCC entry — no customer data leaves Dux's own environment through it. It is licensed **AGPLv3**: Dux calls it over the network without modifying its source, which does not trigger AGPL's network-copyleft obligation — flag this for re-review if the Unleash server is ever forked or patched. **Unleash Edge (the proxy/cache layer) is EOL 2026-12-31; Dux does not use it** ([kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md) confirms KS-L1 has no Edge dependency), so no migration is required before that date.

**Logging and monitoring.** The audit trail is retained 7 years, per the canonical retention policy ([observability-slo §1](../60-operations/observability-slo.md)). Loki (self-hosted) application logs: 30 days. Archives live in MinIO Object Locking, with a hash manifest.

**Coordinated vulnerability disclosure (interim, pre-seed).** Contact `security@dux.io` (PGP optional). **Acknowledge within 3 business days; triage and assign severity within 10.** No bug bounty at pre-seed. The counsel-owned policy lands at `legal/coordinated-disclosure-policy.md`, target 2026-09-30.

## 5. NHI lifecycle

**Creation** (request ticket + security approval + least-privilege scope) → **rotation** (API keys 90 days, agent credentials 90 days, CI/CD 180 days) → **revocation** (immediate on termination or compromise; otherwise 30 days post-project) → **review** (quarterly inventory; disable anything unused).

**OWASP NHI Top 10 (2025) assessment appendix.** Triggered when the NHI inventory exceeds 500. Annual attestation; owner: Security.

| NHI control | Lifecycle mapping | Evidence |
|-------------|-------------------|----------|
| NHI1 Improper Offboarding | revocation within 24 h of termination | IdP + K8s audit logs / Falco alerts / Vault audit log |
| NHI2 Weak Credentials | 90-day rotation; SSM/Vault only | rotation calendar |
| NHI3 Excessive Permissions | least-privilege scope at creation | ticket + approval |
| NHI4 Lack of Inventory | quarterly inventory review | inventory report in S3 |
| NHI5 No Rotation | keys 90 days; CI/CD 180 days | rotation attestations |
| NHI6 Shared Credentials | **max 2 active per agent during rotation** | agent credential table |
| NHI7 Unmonitored NHI | weekly shadow-AI reconcile | `undeclared_count: 0` |
| NHI8 Insecure Storage | SSM/Vault; Gitleaks on every PR | CI scan results |
| NHI9 Overprivileged Agents | `supervised` requires a delegation token | agent config audit |
| NHI10 Human Use of NHI | **a service account cannot impersonate a user** | access-review export |

`pnpm admin:nhi-inventory --env production` exports active NHIs from ECS/SSM, Vault, GitHub, and the MCP registry, quarterly. **Page Security when `undeclared_count > 0`.**

## 6. Operational runbook bridge

Seed runbooks ship §1–9 at Stage 1. Series A adds the mandatory §7–12:

| Section | Requirement |
|---------|-------------|
| §7 Pre-conditions | must carry a verification command and its expected output |
| §8 Automation gate | must link the rollback/halt last-test URL, no older than 30 days |
| §9 Execution steps | **every step names a human fallback** |
| §10 | a full OWASP triple cross-link |
| §11 | RED + USE copy-paste queries, plus the error-budget delta |
| §12 | a SEV1/SEV2 scheduled review within 48 h |

**Formal incident response:** NIST 800-61 phase mapping, with S3 evidence paths. Regulatory clocks: **GDPR 72 h; EU DORA roughly 4 h initial; customer notification 24–72 h.** Annual tabletop, rotating the scenario.

**AI Safety Lead appointment:** a primary and a deputy, holding halt authority. eBPF roadmap: draft M3, staging M6, **production mandatory M9.**

**Board pack minimum:** risk, certifications, AI governance, DORA.

## 7. Evidence framework

MinIO Object Locking, 7 years, with a deny-delete break-glass role.

GitHub Actions and in-cluster job collectors cover: CC6.1 access, CC6.2 joiners/movers/leavers, CC7.1 PRs, CC7.2 PagerDuty, CC8.1 deploys, vulnerability scans (Dependabot, Trivy), CloudNativePG restore, the DR game day, policy tags, and AI samples.

**Automation metric:** `compliance.evidence_automation_pct = automated ÷ total mapped`. **Target ≥80% by M3.** CC7.2 and CC8.1 must be automated by M6.

**Trust portal minimum page set** — SOC 2 summary, subprocessors, pentest executive summary, security overview, security FAQ, status link — **content-complete before the first $100 K ACV.**

**Tooling and auditor (architecture-panel-review-2026-07 §5.3 M4).** A compliance-automation platform (Vanta- or Drata-class) runs the continuous-evidence collection this section describes — control monitoring, the GitHub Actions/Lambda collector integrations, and the `compliance.evidence_automation_pct` metric feed into it rather than a bespoke spreadsheet. **Selection in progress; target decision by M3**, on the same clock as the ≥80% automation target above. **SOC 2 Type II audit: a qualified CPA firm (AICPA-member, PCAOB-registered where applicable), selection in progress** — engagement must be confirmed before the Type I readiness letter (M6) so the Type II observation window (§2) can start immediately after the Type I report issues, with no auditor-selection delay in between.

## 8. AI governance

**NIST AI RMF crosswalk** — Govern, Map, Measure, Manage. Published M4–6; a federal RFP gate.

**NIST CSF 2.0 crosswalk — a distinct framework from the AI RMF crosswalk above, not a duplicate.** AI RMF scopes AI-system risk; CSF 2.0 scopes organizational cybersecurity risk, and adds **Govern** as its sixth function. Each function maps to Dux mechanisms already documented elsewhere in this corpus:

| CSF 2.0 function | Dux control / mechanism |
|-------------------|--------------------------|
| Identify | asset/connector catalog ([catalogs.md §1](../10-product/catalogs.md)); the World Model asset inventory |
| Protect | RLS tenant isolation ([data-model.md §1](../20-architecture/data-model.md)); the kill switch ([kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md)); NHI lifecycle (§5 above) |
| Detect | governance-kernel anomaly triggers — confidence-abstention band, sandbox `TIMEOUT`/`OOM`, T4 outlier ([governance-kernel.md](../40-ai-safety/governance-kernel.md)); AIBOM drift monitoring (below) |
| Respond | incident runbooks ([incident-runbooks.md](../40-ai-safety/incident-runbooks.md)); the operational runbook bridge (§6 above) |
| Recover | DR RTO/RPO — 4 h RTO / 1 h RPO platform baseline ([dr-bcp.md §1](../60-operations/dr-bcp.md)) |
| Govern | this document — triggers (§1), policies (§4), evidence framework (§7) |

Published on the same M4–6 cadence as the AI RMF crosswalk, and the same federal-RFP gate.

**CIS Controls v8 crosswalk.** Dux ingests CIS-aligned CSPM findings (the Wiz connector, `scanner` role — [catalogs.md §1](../10-product/catalogs.md)), so ingested data needs a stated correspondence to Dux's own findings taxonomy:

| Dux findings category | CIS Controls v8 (top-level) |
|------------------------|------------------------------|
| Asset/connector inventory (`asset_discovery` role) | CIS Control 1 — Inventory and Control of Enterprise Assets |
| CSPM misconfiguration findings (Wiz `scanner` role) | CIS Control 4 — Secure Configuration of Enterprise Assets and Software |
| Vulnerability findings (NVD/CISA-KEV/EPSS ingestion) | CIS Control 7 — Continuous Vulnerability Management |
| The hash-chained audit trail (§4 Logging and monitoring above) | CIS Control 8 — Audit Log Management |

**AIBOM governance:**

- **Monthly attestation** — a signed GPG JSON carrying `agent_registry_hash`, stored in S3.
- **Component tracking:** `models.json`, `mcp-registry.json`, `prompts/`, and the agent configs.
- **Drift monitoring:** weekly output sampling; 10% stratified structural drift above 2σ carries a **24 h Security SLA**; plus latency burn-rate, cost anomaly, and safety rates.
- **Rollback procedure:** disable the flag → revert the agent config → roll back the MCP pin → customer comms.

**TenantHealthScore:** reliability 40%, cost 30%, safety 30%. Below 50 routes to Security and FinOps.

**AI red team:** quarterly, plus continuous CI — diff-scoped per config change, nightly, with a 5% flake quarantine.

**Model cards:** one per ADR-008 pin; refreshed quarterly and on any change.

> **Model-card note (BS-27).** Use the ADR-008 canonical IDs — `gpt-5.4`, `claude-sonnet-4-6`. **Not** the drifted `claude-sonnet-4` from the original model-card example.

**Enterprise pricing:** Professional at $50–150 K ACV ($8 K/month list); Enterprise at $150 K+ — the full 3-stage offering post-Gate 3, with an outcome-based option.

**Platform and API strategy:** `/v2` with a 90-day overlap; partner webhooks at Gate 3; rate tiers per [api-overview §4](../30-api/api-overview.md).
