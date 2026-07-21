---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-1, D-2, D-3, D-27, D-28, D-29, D-30, D-31, D-33, D-34, D-35, H1, H5]
---

# Architecture Decision Records

**Purpose:** the canonical architecture decisions, with their revisions.

**Status convention.** `Accepted` unless noted. **R2** is the ideal-state revision (2026-07-05, Gap Closure v2.2); **R3** is the 2026-07-13 write-path re-gating. Where a revision exists it is canonical, and the original decision is retained beneath it as `Superseded`.

**An ADR is a decision record.** Unlike a spec, it is *meant* to carry its own history — the `Superseded` lines are the format working as intended.

**Legacy numbering (AI-22).** Blueprint ADR-034–037 map to corpus ADR-003–006 — legacy "ADR-034" is ADR-003 here. **CI blocks any out-of-corpus ADR reference.**

**Staleness rule (AI-21).** Every ADR carries a `last_reviewed` / `next_review` pair. **CI lint fails any ADR more than 180 days stale.**

## Summary

| ADR | Decision | Status |
|-----|----------|--------|
| ADR-001 | Better Auth via `AuthPort`; JWT + refresh rotation; SPIFFE-format agent claims | Accepted |
| ADR-002 | **R2:** Shared-schema RLS FORCE on CloudNativePG; NATS/Valkey cache and pub/sub | Accepted (R2) |
| ADR-003 | **R2:** Drizzle ORM on CloudNativePG; expand-contract migrations; ephemeral cluster-per-PR | Accepted (R2) |
| ADR-004 | AWS SDK v3 asset discovery; cross-account IAM + external ID | Accepted |
| ADR-005 | **R2:** Notification queues on NATS JetStream + SES/Slack/webhooks — **not** BullMQ | Accepted (R2) |
| ADR-006 | **R4:** Kubernetes (EKS) from Gate 1, Pulumi IaC; Vault secrets; Cloudflare-only WAF; MinIO audit anchor | Accepted (R4) |
| ADR-007 | **R3:** Self-hosted Temporal on K8s is the canonical `WorkflowPort` from Gate 1; DBOS demoted | Accepted (R3) |
| ADR-008 | **R2:** CaMeL-tiered LLM routing; ≤$0.75 SLO at 45% cache; **Gate-2 triage via Bedrock's cheapest model (D-35), vLLM+Phi-4 retired** | Accepted (R2) |
| ADR-009 | Agent-as-directory registry — filesystem SSoT, AIBOM-native | Accepted |
| ADR-010 | **R5:** LiteLLM removed — direct Bedrock SDK behind `LLMProviderPort`, NestJS-level provider fallback/retry; Bifrost demoted to a Gate-2 spike, evaluated only if routing complexity outgrows NestJS fallback logic | Accepted (R5) |
| ADR-011 | **R2:** Vendor connector framework in Phase 1; ≥3 connectors live at Gate 1 | Accepted (R2) |
| ADR-012 | **R3:** Vendor-action write path in Phase 1, **unattended by default at Gate 1**; HITL is anomaly escalation only; closed-loop validation at Gate 3 | Accepted (R3) |
| ADR-013 | Provider ports + ESLint boundary — only `packages/adapters/*` may import vendor SDKs | Accepted |
| ADR-014 | **R2:** React + Vite SPA (TanStack Router/Query), MinIO + Cloudflare CDN; SSE + POST, not WebSocket | Accepted (R2) |
| ADR-015 | **R4:** Self-hosted Firecracker on K8s is the **Gate-1 default**; E2B retired; AST pre-scan mandatory | Accepted (R4) |
| ADR-016 | **R2:** Continuous Assessment Engine — event-driven (NATS pub/sub) + scheduled + dirty-check | Accepted (R2) |
| ADR-017 | **R3:** Multi-provider Claude inference — Bedrock primary → direct Anthropic fallback → local vLLM emergency, via direct SDK + NestJS fallback (not LiteLLM); OpenAI stays direct | Accepted (R3) |
| ADR-018 | Frontend design system — headless (React Aria + Radix) on a design-token source of truth; WCAG 2.2 AA enforced | Accepted |
| ADR-019 | Data visualization — headless charts (Visx) + SVG; dedicated graph lib when multi-hop ships | Accepted |
| ADR-020 | **R2:** Agentic RAG enabled — Postgres/pgvector + Apache AGE graph extension (both on CloudNativePG); hybrid + self-hosted reranker; constrained decoding, GraphRAG as a security component | Accepted (R2) |
| ADR-021 | Remove Mastra and LangGraph.js; the agent reasoning loop is a Temporal TypeScript workflow calling AWS Bedrock Converse API directly | Accepted |

---

## ADR-001 — Authentication

**Better Auth** (a library, $0 per MAU) behind `AuthPort`. Users live in CloudNativePG under RLS.

| Concern | Decision |
|---------|----------|
| Access JWT | claims `sub`, `email`, `tenant_id`, `role`, `jti`, `aud`, `exp`. `aud` is `api.dux.io` or `app.dux.io`; **a wrong `aud` is a 401** |
| Refresh | token families with rotation and reuse detection — `REFRESH_TOKEN_REUSE_DETECTED` invalidates the entire family |
| Lifetimes | access 60 min; refresh 7 days; **agent tokens 15 min** |
| Transport | HTTP-only cookies for the dashboard; Bearer JWT for the API. **No browser Supabase client** |
| MFA | optional TOTP in Phase 1; **mandatory before any pilot ≥$50 K**; enforced for `admin` |
| Passwords | Argon2id (m=65536, t=3, p=4), 12-character minimum, HIBP check |
| Route guards | `UserAuthGuard` / `AgentAuthGuard` / dual-scope matrix |
| Impersonation | session GUCs (`app.tenant_id`, `impersonator_sub`, `impersonation_reason`); nightly IdP reconciliation; manual grants expire after 24 h |
| Agent claims | SPIFFE format — `spiffe://dux.io/tenant/{t}/agent/{a}` |

**Adapters:** `BetterAuthAdapter` (default) ↔ Supabase ↔ WorkOS (enterprise SAML).

**Security review findings (resolves SR-02, 2026-07-16):**

| # | Gap | Fix |
|---|-----|-----|
| SEC-AUTH-01 | Per-tenant rate limiting is applied **post-auth** ([architecture-overview §1](architecture-overview.md)) — `login`, `signup`, `password-reset`, and `mfa-verify` are pre-auth and have no `tenant_id` yet, so they were undocumented against brute-force/credential-stuffing. | IP-based **and** account-based throttling on all four endpoints, ahead of and independent from the per-tenant throttler: max 5 failed logins per account per 15 min (progressive backoff), max 20 per IP per 15 min. Cloudflare edge (coarse limits, already in the trust-boundary diagram) is the first layer; this is the application-layer backstop. |
| SEC-AUTH-02 | No documented path to force-invalidate a **live access JWT** before its 60-minute natural expiry. Refresh-token-family invalidation (`REFRESH_TOKEN_REUSE_DETECTED`) stops a new access token from being minted, but an already-issued one stays valid until it expires — on a suspected account compromise, forced admin logout, or SCIM deprovisioning, that's up to a 60-minute exposure window. | A short-TTL Upstash denylist (`revoked_jti:{jti}`), checked by `UserAuthGuard`/`AgentAuthGuard` on every request, TTL'd to the token's remaining `exp`. Populated on: password change, admin-forced logout, SCIM deprovision, and KS-L2/L3/L4 activation for the affected tenant or session. This is a targeted denylist, not a full session store — it only ever holds tokens actively being revoked early. |
| SEC-AUTH-03 | JWT signing algorithm and cookie security attributes were unspecified. | **RS256** (asymmetric — a compromised API instance can verify but not mint tokens), with quarterly key rotation via AWS SSM (D-5 convention). Dashboard session cookies: `Secure`, `HttpOnly` (already stated), `SameSite=Strict`, `__Host-` prefix. |

**Not flagged as a gap:** Better Auth's own supply-chain posture is already covered by the Gate-1 full-tree SBOM + CI-runner credential-hygiene control ([ci-cd-testing §5](../50-engineering/ci-cd-testing.md)) — it's an npm-tree dependency like any other, not a special case.

**LOCK-01: Better Auth 1.5 (2026-02-28) shipped self-service SAML, with SCIM reaching GA on 2026-04-16 — SAML/SCIM is no longer a triggering gap.** **Two-rung exit ladder:** rung 1 is the Better Auth SSO plugin (days, $0 marginal cost); rung 2 is WorkOS ($125/connection/month), reserved for capabilities the plugin doesn't cover. **Trigger:** the first enterprise security questionnaire demanding SSO — pre-RFP, not the first SAML RFP.

**Rejected:** Clerk and Auth0 (per-MAU cost); Keycloak (operational burden). **Series A:** hardware keys.

**OI-36 closed (2026-07-19, D-33).** The live `demo`/`api-demo.dux.io` deployment observed running Descope was environment drift, not an adopted alternative — Better Auth is confirmed as canon. See [decisions-log](../00-meta/decisions-log.md).

## ADR-002 R2 — Multi-tenancy

Shared database, shared schema, with a `tenant_id` UUID on every tenant-scoped table.

- **RLS FORCE.** No `BYPASSRLS`, and no superuser for the application or migration roles.
- `SET LOCAL app.tenant_id` per transaction. **Fail closed if the GUC is unset.**
- `TenantContextMiddleware` validates the UUID → 401 `tenant_context_invalid`.
- **Composite foreign keys prevent IDOR.**
- `cves` is global and read-only — ESLint `no-direct-cves-query`; join through `findings` only.
- **Transaction-mode pooling via PgBouncer, with `SET LOCAL` per transaction.** `SET LOCAL` is transaction-scoped by construction, so RLS is correct under transaction-mode pooling. Session-mode pooling is now an available option (self-hosted PgBouncer, no Neon-imposed constraint) but there is no stated reason to switch off the transaction-mode default.
- `TenantScopedRepository<T>` plus a `no-raw-findone` lint. The GUC is re-asserted at runtime before every query.

**Rejected:** schema-per-tenant and database-per-tenant — cost and operational burden at pre-seed. EDBT 2026 validates shared-schema RLS with `tenant_id`-leading indexes.

**R2 (2026-07-19, D-33): Postgres host moves from Neon to CloudNativePG** (self-hosted operator on Kubernetes) — see [ADR-006 R4](#adr-006-r4--deployment-topology). RLS FORCE, `SET LOCAL`, `TenantContextMiddleware`, composite-FK, and `TenantScopedRepository` mechanics are all host-agnostic and unchanged. Backups move from Neon's managed PITR to CloudNativePG's native backup/restore (WAL archiving to MinIO). **Superseded:** Neon branch-per-PR ephemeral test databases — see [ADR-003 R2](#adr-003-r2--database-migrations) for the replacement mechanism.

Full spec: [multi-tenancy](multi-tenancy.md).

## ADR-003 R2 — Database migrations

**Drizzle ORM** on CloudNativePG. **Expand-contract only**; forward-fix preferred; a pre-migration base-backup snapshot runs before any destructive change (CloudNativePG's native backup, replacing Neon's PITR pre-hook).

- **Ephemeral CloudNativePG cluster per PR**, provisioned via the K8s operator (`Cluster` CR from a base-backup template) — replaces Neon's branch-per-PR. 7-day stale cleanup, alert above 20 live ephemeral clusters, same lifecycle intent as the original.
- `pnpm ops:migrate` runs before deploy.
- **`check-rls.sh` is a merge gate**, and ships with a negative fixture that *must* fail.
- `SchemaSyncService` detects drift → fail-closed P1.
- `world_model_versions` gets a 24 h in-flight compatibility window plus a purge job.
- `CREATE INDEX CONCURRENTLY` from seed.

**Rejected:** TypeORM, Prisma, Flyway (JVM). **Reversal trigger:** more than 3 rollbacks in a quarter.

**R2 (2026-07-19, D-33).** Host moves Neon → CloudNativePG ([ADR-002 R2](#adr-002-r2--multi-tenancy)/[ADR-006 R4](#adr-006-r4--deployment-topology)). The per-PR ephemeral-database mechanism above is new engineering surface this migration creates — no equivalent existed for a self-hosted operator before; sized in the backlog re-cost ([decisions-log](../00-meta/decisions-log.md)).

## ADR-004 — Asset discovery

**AWS SDK v3**, modular clients: EC2, IAM, ELBv2, ECS (list → describe), EKS (cluster only), S3, RDS, Lambda, CloudFront.

Cross-account IAM assume-role with an external ID (`tenants.settings.aws_role_arn` + `external_id`); STS validation errors map to UI banners. Delta sync per tenant and service; throttle backoff, max 5 retries; assets soft-delete via `deleted_at` and carry a historical badge.

**One AWS account per tenant in Phase 1.** AWS Organizations delegated admin lands at Gate 2; AWS Config at Phase 2+.

**Rejected:** CloudQuery (premature), Prowler (a scanner, not discovery). **Reversal trigger:** if the SDK approach proves inadequate above 10 K assets, revisit CloudQuery.

## ADR-005 R2 — Notification engine

**Durable queues on NATS JetStream** — `email_queue` (SES), `slack_queue`, `pdf_queue` (Gotenberg), `webhook_queue` (durable retries). **Not BullMQ.**

- MJML + Handlebars templates, with snapshot CI.
- SES with DKIM, SPF, DMARC. Bounce <5% and complaint <0.1% are P2; **internal halt at 0.05%**.
- Webhooks: 5 attempts with exponential backoff → `webhook_dead_letter` (a JetStream dead-letter stream); alert above 10 per tenant per hour; replay CLI; HMAC-SHA256 plus `Idempotency-Key`.

**Rejected:** SNS/SQS (AWS-only, breaks portability), a custom queue.

> **Correction (BS-10, retained).** The legacy TRD's "Webhooks (outbound)" table specified BullMQ on Redis DB1. **That contradicts this ADR and FR-010.** Canonical delivery uses the JetStream-backed durable queue, with the same retry, HMAC, and idempotency parameters. See [events-webhooks](../30-api/events-webhooks.md).

**R2 (2026-07-19, D-33): Neon-backed table queues → NATS JetStream.** Same four queue roles, same retry/HMAC/idempotency contract — transport only. Closes a gap this ADR never carried a reversal trigger for, by resolving the choice outright rather than adding a trigger to the old one. **Superseded:** the Neon-table transport is retained only as prior-state history.

## ADR-006 R4 — Deployment topology

**Kubernetes (Amazon EKS, managed control plane) from Gate 1.** `dux-api`, `dux-connector-sync`, and `dux-sandbox` are separate K8s Deployments, same blast-radius isolation intent as the prior ECS services (`dux-litellm` is retired — see [ADR-010 R5](#adr-010-r5--llm-routing-layer)). **Pulumi (TypeScript)** provisions the EKS cluster and workloads — same monorepo language as `packages/api`/`packages/core`, keeping the OI-17 one-language rationale intact. Managed node group with auto-scaling: CPU >70% for 2 min or memory >80% for 2 min scales the node group up; agent queue depth >50 scales Temporal workers.

**R4 (2026-07-19, D-34): DigitalOcean/Linode LKE → EKS.** The self-hosted-Kubernetes decision itself (R3, D-33) is unchanged — this narrows only the CSP/cluster flavor. EKS is a FedRAMP-authorized CSP with GovCloud availability, reopening [OI-38](../00-meta/open-items.md) favorably; the manifests, Pulumi IaC language, Vault/NATS/Valkey/MinIO/CloudNativePG/self-hosted-Temporal/self-hosted-Firecracker workloads, and the Cloudflare-edge-WAF-plus-Falco defense-in-depth posture all continue to run unchanged, now on EKS. The multi-cloud/on-prem-portability rationale from R3 still holds — Kubernetes manifests run identically on EKS, GCP, Azure, DigitalOcean, Linode, and on-prem — EKS is a specific, FedRAMP-capable point on that same portability spectrum, not a reversal of it.

**Rationale (2026-07-19, D-33, retained).** Dux sells to finance and healthcare. AWS-only (ECS Fargate) is a rewrite the moment an ICP demands air-gapped or on-prem deployment — the exact failure mode already observed with E2B in [ADR-015 R4](#adr-015-r4--sandbox).

**Secrets:** **HashiCorp Vault** (self-hosted on K8s) replaces AWS SSM Parameter Store — Vault was already carried as "optional later" in the original D-5; this migration makes it the default. Secrets-rotation cadence table below updated accordingly.

**WAF:** AWS WAF is retired with ECS. **Cloudflare edge WAF** (already retained for DNS/CDN) becomes the sole WAF layer, backstopped by **Falco** for in-cluster runtime detection (sandbox escapes, anomalous syscalls) — defense-in-depth shifts from network-layer-plus-app-layer to edge-layer-plus-runtime-layer.

**Audit archive:** **MinIO Object Locking (Compliance mode)** replaces the AWS S3 + Object Lock anchor — same S3-API object-lock semantics, self-hosted.

Detail: [architecture-overview §2](architecture-overview.md).

> **Superseded (R4):** DigitalOcean Kubernetes / Linode LKE as the cluster CSP — see the D-34 rationale above; self-hosted Kubernetes on EKS is now canonical. **Superseded (R3):** ECS Fargate, AWS CDK, AWS SSM Parameter Store, AWS WAF, and the AWS S3 audit anchor — all retired in favor of the Kubernetes/Pulumi/Vault/Cloudflare-WAF/MinIO stack above. **Superseded (R2, retained from history):** a single Railway container, an inline Railway → AWS migration RFC, VPS deploy scripts, and the resident-agent chart in Phase-1 infrastructure.

## ADR-007 R3 — Durable execution engine

**Self-hosted Temporal (on Kubernetes) is the canonical `WorkflowPort` engine from Gate 1** — battle-tested durability, native retries, visibility, and HITL signals, now run in-boundary rather than as a SaaS dependency.

**R3 (2026-07-19, D-33): Temporal Cloud → self-hosted Temporal on K8s, persistence on CloudNativePG.** This closes a gap where the $500/mo Temporal Cloud cost trigger was fireable but unmonitored — no `dux_cost_temporal_cents` panel existed — by removing the SaaS cost variable entirely rather than instrumenting it. Self-hosted Temporal gives **event-sourced audit trails** — every approval, rejection, and escalation is an immutable workflow-history event, in-boundary — which is now a sales-relevant property for finance/healthcare ICPs, not just infrastructure. mTLS between workers and the Temporal frontend is now issued via **cert-manager or Vault** (replacing D-16's AWS-SSM-stored-certificate mechanism); the per-tenant-DEK `PayloadCodec` wrapping scheme is unchanged, now wrapped by a Vault-held KEK instead of an SSM-held one.

**DBOS is demoted to a future cost-optimization spike behind the same port** (R2, retained) — this removes the single largest unverified bet — the un-passed DBOS safety spike — from the critical path.

**Namespace-per-tenant trigger (flagged for re-derivation, D-33).** The original ~20K-poller-per-namespace ceiling and the 15,000-active-task-queue trigger ([OI-22](../00-meta/open-items.md)/D-2) were both keyed to Temporal Cloud's SaaS multi-tenancy limits. Self-hosted Temporal's actual scaling ceiling depends on the cluster's own resource allocation, not a vendor-imposed poller cap — the trigger needs re-deriving against self-hosted capacity planning, not silently carried forward. Tracked as a new open item pending that re-derivation.

**Near-ceiling scale path (SR-15, retained pending re-derivation).** **Task Queue Priority & Fairness** (fairness key = `tenant_id`) remains the mechanism for capping one tenant's queue depth without starving others, inside the single-namespace-per-environment model (D-2) — its trigger threshold is part of the namespace re-derivation flagged above, not the mechanism itself.

**Orchestration pattern.** Supervisor plus isolated subagents; one child workflow per tenant (`assessment-{tenant_id}`); a distinct system prompt per tier; saga compensation; continue-as-new (suggest ≥8 K, hard 10 K, cap 35 K); heartbeat timeouts NVD 30 s/10 s, AWS 120 s/30 s, MCP 60 s/15 s; max 2 retries.

**Inner reasoning loop.** No agent framework runs inside workflow steps — the Week-6 Mastra vs. LangGraph.js bake-off (C1–C5) is closed without being run: the reasoning loop calls the Bedrock Converse API directly, as ordinary Temporal activities. See [ADR-021](#adr-021--remove-mastra-and-langgraphjs-use-temporal--bedrock-converse-api-direct).

**App UI** is hand-rolled SSE via `RealtimePort`, with a CopilotKit/AG-UI spike in Week 10.

**The golden set is the security regression gate.** DeepEval, over **(CVE × synthetic-environment) pairs with per-environment ground-truth verdicts** — exploitability is a property of the pair, not of the CVE. **A CVE-only set validates triage, not environmental reasoning** (H1).

**Reference pattern.** This design aligns with the Temporal-community **Sandbox Orchestration Harness** reference (temporal.io blog, May 2026; a Code-Exchange community pattern, not an official Temporal product) — a durable workflow dispatching untrusted agent-generated code to ephemeral isolated sandboxes. The community reference itself uses pause/resume + snapshot-fork; Dux deliberately tightens to fresh-microVM-per-invocation, no snapshot reuse (see ADR-015).

**Gate-1 exit additions.** **Worker Deployment Versioning** (Temporal's current GA mechanism — the pre-2025 experimental `patched()` + build-ID scheme was removed from Temporal Server in March 2026; `patched()` itself remains a valid in-workflow fallback), pinned to v1 until the golden set passes on v2 — **an unversioned worker is the most common Temporal production failure** — and OTel tracing on all workflows.

**State-ownership rules.** Workflow history is durable engine state, and must stay replay-safe. Message history is **never** carried in workflow state — it lives in Postgres (`assessment_messages`, RLS-scoped), and workflow state carries only `history_id`/`turn_count`/`spent_usd`/`status` (ADR-021). **Never dual-write reasoning state alongside workflow history (anti-criterion A4).** CaMeL split state is ephemeral per activity; the tool audit lives in `agent_audit_log` (durable, classification SECRET).

**Rejected:** an unconstrained swarm (AutoGen, Strands-swarm); CrewAI; eve-as-runtime (Vercel lock-in — its conventions were adopted into ADR-009 instead).

**Transport and payload security (D-16 R2, D-33).** The self-hosted Temporal frontend uses mTLS client certificates, issued per environment via **cert-manager or Vault** (replacing the AWS-SSM-stored-certificate mechanism) — rotated every 90 days; the SDK loads them via `TEMPORAL_TLS_CERT` / `TEMPORAL_TLS_KEY` references, never inline values. A custom `PayloadCodec` (data converter) encrypts every workflow and activity payload with a per-tenant DEK, wrapped by the platform KEK using Vault's transit engine (replacing the SSM half of the prior Vault/SSM wrapping scheme; see [multi-tenancy §5](multi-tenancy.md)), before the payload leaves the worker process. Self-hosted Temporal never holds plaintext payloads — same guarantee as the SaaS deployment, now enforced in-boundary rather than trusted to a vendor.

> **Superseded (R3, 2026-07-19, D-33):** Temporal Cloud as the deployment target — see [ADR-006 R4](#adr-006-r4--deployment-topology) for the self-hosted-K8s rationale. **Superseded (R2):** ADR-007 was "Provisional (DBOS gated by safety spike)", with Temporal as the fallback. R2 inverts that. Retained from the DBOS era: **(a)** the DBOS-vs-Inngest appendix — DBOS as an in-process library with state in its own database, $0 SaaS, and a credible exit, versus Inngest as managed, with state in a vendor's US cloud and per-step metering from $75/month; the verdict was DBOS for the audited multi-tenant core, with Inngest optional for non-critical async only. **(b)** The AI-44 database-split RFC (Week 8, a Gate-2a blocker): DBOS table inventory, RLS migration plan, connection-string swap via `DatabasePort`, checkpoint replay validation, and COST-03 reconciliation. Moot for Temporal state — but it applies again if the DBOS spike revives.

## ADR-008 R2 — Multi-provider LLM routing

**CaMeL-tiered routing.**

| Tier | Input | Providers | Model |
|------|-------|-----------|-------|
| **S-LLM** | untrusted public CVE text; no tools | US/EU-domiciled only | `gpt-5.4-mini`; optional Groq Llama fallback (US-domiciled) |
| **P-LLM** | customer context | trusted US/EU only | `gpt-5.4`; escalation to `gpt-5.5` via `reasoning_model_tier` (enterprise + critical CVE) |

Fallback model: `claude-sonnet-4-6`. S-LLM chain: `gpt-5.4-mini → claude-haiku-4-5 → rule-based extractor`, after 3 failures. Router step taxonomy: triage, extract, reason, summarize. **`claude-*` models route via the direct Bedrock SDK's multi-provider fallback chain — see [ADR-017 R3](#adr-017-r3--multi-provider-claude-inference-path).**

**Gate-2 triage path (D-35, retired the vLLM + Phi-4 S-LLM option below).** Classification, severity triage, and duplicate detection route to the **Bedrock Converse API's cheapest available model** (e.g. `amazon.titan-text-lite-v2`), alongside — not replacing — the `gpt-5.4-mini → claude-haiku-4-5 → rule-based` chain above. No self-hosted inference; see [ADR-021](#adr-021--remove-mastra-and-langgraphjs-use-temporal--bedrock-converse-api-direct). Gate-2 scoped: no Phase-1 change to the routing chain.

> **Superseded (D-35):** the self-hosted **vLLM + Phi-4 14B** Gate-2 S-LLM path (D-33) — retired in favor of Bedrock's cheapest model, removing the Kubernetes GPU-node operational surface for no material accuracy loss on this class of call.

**Failover.** Triggered by a PromQL error rate above 50% with more than 10 requests in 5 min, **or** a status-page poll, **or** p95 above 2× for 5 min. Detection within 60 s.

**Low-traffic guard: force HITL T3 on fallback verdicts until a golden-set spot check passes — at least 5 assessments per 15 min.** Auto-rollback if fallback accuracy drops more than 5% for 15 min. An emergency model pin lasts at most 24 h. Per-tenant daily caps raise `LLM_TENANT_BUDGET_EXCEEDED` and freeze at L2.

**R2 cost envelope.** SLO **≤$0.75 per assessment hard, ≤$0.55 design, on a 45% cache-hit assumption**. **$0.28–0.32 is a stretch figure only, and is never a pricing input.**

**Cost gates (D-3):** soft circuit breaker **$0.675**; the CI cost gate blocks a staging average above **$0.55**; the Gate-1 criterion is **<$0.75 per workflow**.

Prompt caching is engineered around a stable prefix — roughly 90% cache-read discounts at both providers. Semantic cache from Phase 1, now a NestJS interceptor responsibility on the direct-SDK path rather than a LiteLLM proxy feature (ADR-010 R5). The Batch API gives 50% off offline jobs: golden set, calibration, enrichment.

**SEC-03 side-channel residual:** a fixed 3-sample count, with token padding at 512/1024/2048. Accepted risk, with CISO sign-off.

**EU routing:** an EU tenant goes Azure OpenAI EU → Bedrock EU (`eu-central-1`, Claude) → OpenAI US (Art. 49 consent). **Flagged, not resolved this pass (D-34 judgment call b):** the Azure OpenAI EU leg predates ADR-010 R5's LiteLLM removal, and the v4.0 source document that drove this pass's other changes does not mention Azure OpenAI anywhere — it may be an orphaned routing leg now that LiteLLM's multi-provider routing is gone, or an intentionally-retained regional detail the v4.0 doc was simply silent on. Left as-is pending explicit Founder confirmation; see [decisions-log D-34](../00-meta/decisions-log.md).

> **Superseded:** the $0.28–0.32 target, the $0.50 hard ceiling, and the $0.45 soft breaker — all keyed to the old ceiling and an optimistic 85% cache rate.

## ADR-009 — Agent-as-directory registry

Agent configuration lives in Git-versioned `packages/agents/{type}/` — `agent.ts`, `instructions.md`, `tools/`, `skills/`, `connections/`, `sandbox/`.

The AIBOM (CycloneDX 1.6) is generated per deploy at `security/aibom/manifest.json`, with a CI drift check and `test:agent-registry-parity`. **CODEOWNERS puts `@dux-security` on `tools/` and `instructions.md`.** A deploy-time checksum is verified against the signed release artifact.

This adopts eve's best pattern without the Vercel coupling.

**Rejected:** DB-stored prompts (weak audit trail); eve platform configs (lock-in).

## ADR-010 R5 — LLM routing layer

**R5 (2026-07-19, D-34): LiteLLM is removed entirely.** Direct Bedrock SDK (TypeScript, `@aws-sdk/client-bedrock-runtime`) behind `LLMProviderPort`, with **NestJS-level fallback/retry orchestration** (Bedrock primary → direct Anthropic API fallback → local vLLM emergency — see [ADR-017 R3](#adr-017-r3--multi-provider-claude-inference-path)) replaces the proxy. This resolves the R3/R4 tenant-isolation and supply-chain concerns outright rather than mitigating them: the cross-tenant `redis-semantic` cache-hit defect (LiteLLM issue #19575, closed upstream "not planned") and the March-2026 PyPI supply-chain backdoor are moot once there is no LiteLLM dependency to carry the risk.

**What moves where.** LiteLLM's per-tenant virtual keys, budget enforcement, and semantic caching become NestJS interceptor/middleware responsibilities on `LLMProviderPort`, not a sidecar proxy's job:

| LiteLLM responsibility (retired) | NestJS-native replacement |
|---|---|
| Per-tenant virtual keys | `LLMProviderPort` resolves the provider credential per-tenant from Vault; no shared proxy-level key |
| Budget enforcement (`LLM_TENANT_BUDGET_EXCEEDED`) | `LLMBudgetInterceptor` on the same port, same per-tenant-daily-cap semantics and L2 freeze behavior (ADR-008 R2) |
| Semantic caching (`redis-semantic`) | `LLMSemanticCacheInterceptor` on Valkey, same tenant-invariant-only restriction (ADR-008 R2) — the cross-namespace defect this restriction was mitigating no longer applies once the cache lives in application code, not a third-party proxy's shared cache namespace |
| Provider fallback chain | NestJS `LLMFallbackService` (Bedrock → Anthropic direct → vLLM), weighted-retry logic replacing LiteLLM's router |

**Bifrost, reframed.** No longer a primary-slot bake-off. **Evaluated at Gate 2 only if multi-provider routing complexity outgrows what the NestJS fallback logic can cleanly own** — a possible future gateway layer, not a required migration target. Portkey and TensorZero are no longer tracked as bake-off candidates; the tenant-isolation and supply-chain posture that motivated the R3/R4 bake-off no longer applies once the LiteLLM dependency itself is gone.

**Supply chain.** The `no-litellm-pypi` ESLint rule and `.pip-audit` LiteLLM gate are retired (see [engineering-standards](../50-engineering/engineering-standards.md)) — replaced by ordinary npm-dependency auditing on `@aws-sdk/client-bedrock-runtime` and the rest of the Bedrock SDK tree, same posture as any other first-party dependency (ADR-001's "not flagged as a gap" precedent for Better Auth's supply-chain treatment).

**Rejected:** keeping LiteLLM as a degraded-mode fallback behind the direct SDK — carries the same #19575/PyPI-backdoor exposure for no operational benefit once the direct SDK is the primary path; Vercel AI Gateway (lock-in).

**Note (D-35).** With Mastra and LangGraph.js also removed ([ADR-021](#adr-021--remove-mastra-and-langgraphjs-use-temporal--bedrock-converse-api-direct)), `LLMProviderPort` and `LLMFallbackService` construct the Bedrock `ConverseCommand` directly — no proxy and no agent framework sits between NestJS and Bedrock.

> **Superseded (R5):** the LiteLLM proxy (R2–R4) and its Gate-2/P0-spike Bifrost-as-primary-slot bake-off framing — retired in favor of the direct-SDK + NestJS-fallback architecture above. **Superseded (R4, retained from history):** the Bifrost bake-off pulled forward to a parallel P0 spike. **Superseded (R3, retained from history):** LiteLLM stays the Phase-1 proxy with Bifrost/Portkey promoted to a required primary-slot bake-off. **Superseded (R2, retained from history):** LiteLLM proxy in Phase 1 for per-tenant virtual keys, budget enforcement, and semantic caching, restricted to tenant-invariant calls; `dux-litellm` K8s Deployment, signed-GHCR-digest-only supply chain.

## ADR-011 R2 — Vendor connector framework

The framework ships in **Phase 1**, with **≥3 live connectors at Gate 1**: CrowdStrike (runtime and controls), Wiz (cloud findings), and ServiceNow **or** Entra ID (ownership).

Shared contract in `packages/connectors/vendor-contract.ts`: a base `VendorConnector` (`validateCredentials`, `sync(SyncCursor)`, `mapToWorldModel`, `healthCheck`, `connectorRole`) plus role marker interfaces — `AssetDiscoveryConnector`, `ScannerConnector`, `IdentityConnector`, and the R2 additions `ThreatIntelConnector`, `NetworkContextConnector`, `ValidationConnector`, `TicketingConnector`.

`AbstractVendorConnector.sync()` is a Template Method — **vendors override only `fetchPage` and `mapRecord`**.

Credentials: AES-256 in JSONB, with Vault/SSM transit for OAuth; RLS-scoped by `tenant_id` and `connector_id`. `world_model_versions` bumps on a material change.

**NestJS role-token injection means a `ScannerConnector` cannot be injected into a US-002 runtime path.** Sync runs as an isolated `dux-connector-sync` K8s Deployment ([ADR-006 R4](#adr-006-r4--deployment-topology)).

The full source × ConnectorRole × wave taxonomy is preserved, and **CI asserts no orphan OpenAPI `Sources` value**. The enumerated wire values live in [catalogs §1](../10-product/catalogs.md); the count discrepancy against the previously-cited figure is [OI-37](../00-meta/open-items.md).

Per-vendor field-mapping annexes: CrowdStrike (`device_id`, `prevention_policy`); Intune (`complianceState`); Qualys (`QID`); Wiz (`issue_id`); ServiceNow (`cmdb_ci`, `assignment_group`); Entra (`objectId`, `department`, `manager`).

**Rejected:** per-vendor ADRs (sprawl); iPaaS middleware (residency).

> **Superseded:** connectors gated at Gate 2c.

## ADR-012 R3 — Vendor action framework

**The vendor-action write path ships in Phase 1, unattended by default.** `endpoint.isolate`, `network.blocklist_add`, `patch.deploy_special_devices`, and `ticket.create_remediation` execute at Gate 1 **without waiting for human approval**. `policy.deploy_device_config` is the exception: it is gated to **Gate 3**, pending the Intune connector (W2/Gate 3 in [catalogs](../10-product/catalogs.md)) — the action has no vendor to execute against until that connector ships, and goes unattended-by-default once it does.

HITL (T1–T3) is retained as the **anomaly-escalation path** — confidence abstention, sandbox `TIMEOUT`/`OOM`, or a T4 outlier. See [kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md).

Canonical execution lives in `packages/actions/`, behind `VendorActionPort` and `ActionPolicyPort`. **Connectors must not call vendor mutation APIs — every write goes through `VendorActionGate`.** Adapters map canonical IDs to vendor-native names, and persist both in the audit record.

**Post-action refresh:** `VendorActionExecution` → targeted delta sync (`sync_reason=post_mitigation`) → closed-loop re-assessment (Gate 3, US-019). **Gate 3 still requires a field-proven safety record before any finding is auto-closed on validation.**

**Gate-3 remediation orchestration (resolves [OI-29](../00-meta/open-items.md)).** **Strands A2A v0.2 is the default candidate**, superseding the LiteLLM-native-A2A-client candidate ([ADR-010 R5](#adr-010-r5--llm-routing-layer) retired the LiteLLM dependency that candidate reused). It is adopted for Gate-3 remediation-agent orchestration only if it passes:

| # | Criterion |
|---|-----------|
| C1 | Supports the supervisor/subagent topology already in use for the assessment workflow (§ Orchestration pattern) — no rewrite of the Temporal child-workflow-per-tenant shape |
| C2 | Carries `tenant_id`-scoped auth end to end (JWT claims, SPIFFE format per ADR-001), not a shared service credential between remediation agents |
| C3 | Preserves the governance-kernel gate chain (`IntentGate → … → HITLGate`) as a synchronous pre-check on every A2A-dispatched action — no action bypasses `VendorActionGate` because it originated from an agent-to-agent call instead of a direct tool call |
| C4 | Adds no un-auditable hop — every A2A message is a span in the existing OTel trace tree ([observability-slo §2](../60-operations/observability-slo.md)), replayable via `replay_trace_id` |
| C5 | Latency overhead versus a direct MCP tool call stays within the existing p95 <60 actions/assessment budget ([observability-slo §3](../60-operations/observability-slo.md)) |

**If Strands A2A v0.2 fails any of C1–C5**, the fallback is a custom JWT-scoped handler. **No bespoke transport is built before the Strands candidate has actually been spiked against C1–C5.** This is an evaluation framework, not yet a completed evaluation: the spike itself is Gate-3 scoped work, tracked as a backlog task under EP-06.

**Rejected:** an unconstrained Strands swarm.

> **Superseded (R1):** the entire write path gated to Gate 3. **Superseded (R2):** mandatory HITL on every write at Gate 1.

## ADR-013 — Provider ports

Only `packages/adapters/*` may import a vendor SDK (`import/no-restricted-paths`). **CI merge-blocks any net-new `eslint-disable` in `api/` or `core/`.** The boundary extends to `@neondatabase/*` and `dbos` outside `adapters/` and `core/workflow`.

Detail: [architecture-overview §5](architecture-overview.md).

## ADR-014 R2 — Frontend

**React + Vite (SPA), with TanStack Router and TanStack Query.** Static build served from **MinIO, behind Cloudflare CDN**. **The browser talks only to NestJS — no browser BaaS client.** Realtime is **SSE + POST via `RealtimePort`, not WebSocket**. Session cookies come from Better Auth.

**R2 (2026-07-19, D-33): TanStack Start on Cloudflare Pages → React + Vite SPA.** TanStack Start's own R1 text already flagged it as "the highest API-churn-risk component until Gate 2" and beta-maturity risk; a security dashboard cannot break because a server-function framework changes its API mid-Gate. React + Vite is boring and proven; **TanStack Router + TanStack Query are retained** for type-safe routing and server state — only the SSR/server-function framework layer is dropped, not the whole TanStack ecosystem. If SSR is ever needed for auth, **React Router v7 framework mode** is the named fallback (more mature than TanStack Start), not adopted now.

Static hosting moves from Cloudflare Pages to **MinIO** (self-hosted, S3-compatible) fronted by **Cloudflare CDN** — Cloudflare is retained for DNS/CDN/edge WAF, not as the deployment target; this also resolves the Pages-vs-Workers question the prior R1 text raised (moot once Pages itself is retired).

Lockfile pinning, with a bump policy: changelog review plus a golden-path smoke test over US-012, US-011, and US-008.

**Rejected:** Next.js (heavy, Vercel-optimized); browser BaaS clients; Vercel-coupled hosting; keeping TanStack Start (beta-maturity risk on a compliance-sold dashboard outweighs its DX advantages).

## ADR-015 R4 — Sandbox

**R4 (2026-07-19, D-33): self-hosted Firecracker on Kubernetes is the Gate-1 default, not a Gate-2/3 pull-forward.** R3 already established that finance/healthcare ICPs refuse a managed microVM vendor (E2B) probing their environment regardless of a signed DPA — that's a live procurement fact, not a future risk. Under the 2026-07-19 portability/compliance-first stack decision, waiting until Gate-2/3 to ship the thing enterprise buyers actually require is no longer consistent with the rest of the stack's priorities. `firecracker-containerd` or Kata Containers are an allowed K8s-integration bridge if the direct-Firecracker path isn't ready Day 1 — **Firecracker is the target architecture; Kata is an interim implementation detail, not a fallback decision.** E2B is retired, not kept as a fallback.

**R3 (2026-07-18, retained reasoning).** E2B (managed) sees the customer's environment being probed by executed investigation code, which is a heavier subprocessor than a generic sandbox for the finance/healthcare design-partner ICP — several such buyers will refuse it in procurement regardless of a signed DPA.

The `ScriptSecurityScanner` AST pre-scan runs before **every** execution, unchanged. `NoOpSandboxAdapter` is retained **only** as the emergency kill path.

This is what makes "agents write and run code" true at Gate 1: `execution_results` is populated at Gate 1.

**Why a microVM.** **Shared-kernel containers are rejected for LLM-generated code** — CVE-2024-21626 (runc, "Leaky Vessels"), CVE-2024-0132, and SandboxEscapeBench 2026. The 2026 consensus is that Docker and runc isolation **is not a security boundary for AI-generated code**. gVisor is read-only defense in depth only, and never runs LLM code. **Now that the deployment target is Kubernetes rather than a managed PaaS (ADR-006 R3), Firecracker runs directly, self-hosted, in-boundary** — the prior constraint ("managed PaaS cannot run Firecracker, so a managed microVM vendor sits behind `SandboxPort`") no longer applies.

**A fresh ephemeral microVM per invocation, never reused across runs** — VM reuse is a data-leak vector.

This durable-workflow → ephemeral-microVM shape is Temporal's published Sandbox Orchestration Harness reference (May 2026, see ADR-007). Emerging SmolVM-class sub-200 ms cold-start vendors are `SandboxPort` swap candidates at scale — no Phase-1 change.

Detail: [sandbox-execution](../40-ai-safety/sandbox-execution.md).

> **Superseded (R4):** E2B/Modal as the managed-microVM default — retired, not kept as a fallback; self-hosted Firecracker/Kata is now the Gate-1 default rather than residency-only. **Superseded (R1):** artifact-only Phase 1, with execution deferred to Gate 2. The AI-17 runtime assertion that `execution_results` be null now inverts — it is null **only** during the kill path.

## ADR-016 R2 — Continuous Assessment Engine

Event-driven re-assessment (a KEV/NVD/EPSS delta, or a connector bumping `world_model_versions`) plus a scheduled sweep (24 h by default), through `ReassessmentSchedulerWorkflow` (Temporal) and a **NATS core pub/sub** event bus.

**R2 (2026-07-19, D-33): Upstash pub/sub → NATS core pub/sub**, the same bus now used for kill-switch propagation ([ADR-006 R4](#adr-006-r4--deployment-topology)) — one event-bus technology across the platform instead of two. Debounce/dirty-check mechanics below are unchanged.

`ReassessmentDebouncer` coalesces per `(tenant, cve, asset)` within a 15-minute window. **An evidence-hash dirty-check gates the P-LLM re-run, so most triggers resolve as "no material change" without any LLM call.**

This is what makes the continuous-assessment claim true at Gate 1.

Detail: [continuous-assessment](../10-product/features/continuous-assessment.md).

## ADR-017 R3 — Multi-provider Claude inference path

**Multi-provider via the direct Bedrock SDK + NestJS fallback logic: Bedrock primary → direct Anthropic API fallback → local vLLM emergency path.** OpenAI models (`gpt-5.4-mini`, `gpt-5.4`, `gpt-5.5`) stay on the direct OpenAI API, unaffected.

**R3 (2026-07-19, D-34): the transport mechanism changes, not the provider order or rationale.** [ADR-010 R5](#adr-010-r5--llm-routing-layer) retires LiteLLM entirely; the Bedrock→Anthropic→vLLM fallback chain below is now carried by `LLMProviderPort`'s direct Bedrock SDK client plus a NestJS `LLMFallbackService` (weighted retry, provider health tracking) rather than a LiteLLM proxy hop. The R2 rationale for multi-provider resilience itself — no single point of failure, AWS not the sole/primary cloud since [ADR-006 R4](#adr-006-r4--deployment-topology) reaffirms EKS as one CSP among several the manifests run on — is unchanged.

**R2 (2026-07-19, D-33, retained reasoning).** Supersedes ADR-017 R1/D-13's Bedrock-only decision. R1's core rationale — routing Claude through Bedrock keeps inference inside the same AWS IAM boundary as the rest of the platform — no longer holds once AWS is not the primary/only cloud. R1's own reversal trigger (Bedrock p95 latency >2× the direct-Anthropic baseline for 7 days) was flagged as unenforceable (no Bedrock-specific latency panel existed, no pre-cutover baseline preserved). A single-provider strategy is a single point of failure: a Bedrock regional outage previously meant no Claude inference at all.

**Mechanics.** `LLMProviderPort` (direct Bedrock SDK, `@aws-sdk/client-bedrock-runtime`) carries three provider entries for `claude-sonnet-4-6`/`claude-haiku-4-5`: Bedrock (primary, IAM/SigV4 via the workload's K8s service-account-bound IAM role, native on EKS), direct Anthropic API (fallback, API key via Vault), and a local **vLLM** deployment (emergency path, same self-hosted footprint as the Gate-2 S-LLM path in [ADR-008 R2](#adr-008-r2--multi-provider-llm-routing)). NestJS `LLMFallbackService` tracks per-provider latency/error-rate and reweights on the fly (mirroring the failover triggers in ADR-008 R2). Prompt-caching and semantic-cache behavior (ADR-008 R2, ADR-010 R5) are unchanged across all three legs — now NestJS interceptor logic rather than proxy-level config.

**Compliance note.** EKS restores the FedRAMP-relevant AWS-hosting-boundary story R1 relied on (Appendix B of the v4.0 source doc: EKS ✅ FedRAMP Moderate, GovCloud-capable) — see [compliance-program.md](../70-governance/compliance-program.md) and [decisions-log D-34](../00-meta/decisions-log.md).

**Rejected:** Bedrock-only (status quo before R2) — reintroduces the single-point-of-failure that multi-provider resilience addresses; a load-balanced (not failover) dual-path — same operational-surface-doubling objection R1 raised against it, still valid; keeping LiteLLM as the transport (superseded by ADR-010 R5).

> **Superseded (R3):** LiteLLM as the transport mechanism for this fallback chain — see [ADR-010 R5](#adr-010-r5--llm-routing-layer). **Superseded (R2, retained from history):** the Bedrock-only decision and its AWS-IAM-boundary/FedRAMP rationale (R1, 2026-07-14, D-13).

## ADR-018 — Frontend design system

**Decision (D-29): adopt a headless component layer on a design-token source of truth, and own the styling.** The corpus already carries the taxonomy design tokens (stage-pill colors, risk-group and exposure-state icons, the accessibility rules); this ADR decides the component layer that consumes them.

| Concern | Decision |
|---------|----------|
| Data-dense surfaces | **React Aria Components** for the asset tables, the ≤5,000-row vulnerability-instance lists, and the research queue — its grid/table/listbox accessibility is the deepest available, and these grids are where **WCAG 2.2 AA at 0 axe-core violations (TR-NFR-010)** is at most risk |
| Everything else | Radix primitives (a shadcn-style setup is an acceptable single-system alternative — Radix-based, source-owned) for overlays, menus, dialogs |
| Tokens | A single design-token source of truth (Style Dictionary or CSS custom properties) drives the Tailwind theme. **The amber token is fixed at the token layer** — the taxonomy contrast audit fails it at 2.4:1, below the 3:1 non-text / 4.5:1 text AA floors — so it can never regress per-component |
| Enforcement | **"Color and shape, never color alone" and "SVG-with-ARIA, never emoji" become CI lints**, not conventions; axe-core runs in CI (already implied by TR-NFR-010) |
| Streaming | The SSE row-patch surfaces (`RealtimePort`, queue <1s / dashboard <5s) carry **`aria-live="polite"` regions with focus preservation** — the specific accessibility gap H10 names on streaming surfaces; headless primitives do not solve this for you |

Runs on the [ADR-014 R2](#adr-014-r2--frontend) frontend (React + Vite, browser-talks-only-to-NestJS, SSE + POST).

**Rejected:** a full component library (MUI / Mantine / Chakra) — faster to ship but imposes a house style that fights the bespoke taxonomy design system; hand-rolling every component from tokens — heaviest cost against the Gate-1 buffer for a 5-engineer team.

## ADR-019 — Data visualization

**Decision (D-30): headless charts plus SVG, on the same token system and accessibility discipline as the rest of the UI.** The product is chart-dense — the exposure donut, the vulnerability-reduction trend, the confidence distribution, factor cards, and the reachability/attack-path graph.

| Surface | Decision |
|---------|----------|
| Standard charts (donut, trend, distributions) | **Visx** (D3 scales/shapes as React primitives) — fully themeable to the design tokens, no vendor house-style. Recharts is an acceptable faster path for the plain donut/trend if the buffer bites |
| Attack-path / relationship graph | **Custom SVG now** (single-hop vuln → asset → control). When multi-hop attack-path traversal ships (ADR-020), adopt a dedicated graph library — **Cytoscape.js or Sigma + graphology** — rather than hand-rolling layout |
| Color | A contrast-validated categorical palette encoding by **color and a second channel** — already the corpus rule, and already visible in the eye/umbrella/tree risk-group icons |
| Accessibility | **Every chart carries a table or ARIA-description fallback** — charts are the most common silent WCAG failure; the fallback is budgeted, not bolted on |

**Rejected:** a high-level charting library (Nivo / ECharts / Chart.js) — fast defaults, but a recognizable house style that fights the bespoke design system and the icon/shape accessibility rules.

## ADR-020 R2 — Agentic RAG and graph retrieval

**R2 (2026-07-19, D-34): `rag_enabled = true`.** This reverses D-31's own decision, which itself reversed nothing before it — D-31 explicitly endorsed `rag_enabled = false` ("RAG hallucinates; security cannot"), and D-33 explicitly reaffirmed that endorsement. The reversal is deliberate, not silent, and answers the original objection directly rather than overwriting it:

**Why now, and not before.** D-31's objection was that unconstrained RAG output can't be trusted as an input to security decisions. The Agentic RAG design specified here does not remove that objection by assumption — it removes it by **constrained decoding**: every retrieve/reason/decide step in the retrieval loop is forced through schema-validated tool-use (Bedrock Converse API `toolConfig`, `toolChoice` pinned to a specific tool), the same token-by-token structural enforcement mechanism already mandatory for every other LLM output in this corpus. There is no free-text LLM output anywhere in the loop for a hallucination to hide in — the model can only ever emit values that satisfy the declared JSON schema. This mechanism did not exist as a stated mitigation when D-31/D-33 rejected RAG; its availability is the entire basis for the reversal. Full decision record: [decisions-log D-34](../00-meta/decisions-log.md).

**The loop.** A Temporal workflow (`agenticRAGWorkflow`) runs plan → retrieve (parallel: graph, episodic memory, threat-intel APIs, semantic knowledge) → reason → decide (confidence < 0.85 loops back with new queries, max 5 iterations) → synthesize. A confidence band of 0.85–0.95 routes to a human-approval gate (signal-based, 2-hour timeout, escalates to on-call) before the workflow proceeds — this sits on the same `WorkflowPort`/HITL-signal mechanism as every other Gate-3 human-approval path in this corpus ([ADR-007 R3](#adr-007-r3--durable-execution-engine), [kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md)), not a new mechanism.

| Concern | Decision |
|---------|----------|
| Vector store | **Postgres (pgvector + pgvectorscale)** on CloudNativePG until ~100M vectors force a dedicated store — one RLS-enforced store keeps the tenancy model unified. Keep the camel-plane requirement: `tenant_id`-leading HNSW, adversarial-neighbor test satisfied before the flag flips ([camel-plane](../40-ai-safety/camel-plane.md)) |
| Graph store | **Apache AGE** (Postgres extension), same CloudNativePG instance as pgvector — no second database, no second isolation model to secure. This is the concrete implementation of D-31's "the graph is built as a security-reviewed component" line: **per-edge provenance + integrity hashing**, and the "connector-asserted controls are untrusted for negative verdicts" rule extended to **every edge**. A poisoned edge gates an unattended write (GraphRAG poisoning: >93% success at <0.05% corpus edit) |
| Retrieval | **Hybrid search (dense + BM25) + a cross-encoder reranker** on top-20 → 5 — the reranker buys more precision (5–15 NDCG@10) than aggressive ANN tuning |
| Reranker hosting | **Self-hosted `bge-reranker-v2-m3` as the residency-clean default** (in-boundary, no per-call fee); Cohere/Voyage API as the speed-first option for non-residency tenants — mirroring the sandbox/inference residency logic |
| Constrained decoding | Every plan/analyze/synthesize step uses Bedrock Converse API tool-use with `toolChoice` pinned to a single named tool and a required-fields JSON schema — no `JSON.parse` retry loop anywhere in the pipeline |

**Rejected:** free-text LLM output anywhere in the retrieval/reasoning loop (the exact pattern D-31 rejected, still rejected — constrained decoding is additive, not a relaxation); a dedicated vector or graph store at first-need (premature — a second isolation model to secure, deferred until scale forces it); managed reranker as the default (data leaves the boundary — friction for the EU/residency posture).

> **Superseded (R2):** `rag_enabled = false` and the structured-retrieval-only posture (R1/D-31, 2026-07-18) — reversed per the constrained-decoding rationale above, not silently dropped. R1's non-graph mechanics (Postgres-first vector store, hybrid+reranker retrieval, self-hosted reranker default) carry forward unchanged into R2's table.

**Note (D-35).** The plan/retrieve/reason/decide loop above is a Temporal workflow calling the Bedrock Converse API directly — retrieval is a Temporal activity against pgvector + Apache AGE, not a call mediated by an agent framework ([ADR-021](#adr-021--remove-mastra-and-langgraphjs-use-temporal--bedrock-converse-api-direct)).

## ADR-021 — Remove Mastra and LangGraph.js; use Temporal + Bedrock Converse API direct

**Status:** Accepted

**Context.** [ADR-007 R3](#adr-007-r3--durable-execution-engine) carried Mastra (primary) and LangGraph.js (alternative) as a conditional inner-reasoning-graph bake-off, gated on criteria C1–C5 and a Week-6 decision sprint (US-008-T02). [OI-39](../00-meta/open-items.md) already shows the Gate-1 backlog running over its 2,080 h envelope, with the Gate-2 vLLM+Phi-4 S-LLM path (ADR-008 R2) as unestimated net-new scope. Both Mastra and LangGraph.js are abstraction layers over capabilities the stack already owns outright: Temporal (durable execution, retries, signals, per-tenant queues) and the Bedrock Converse API (multi-turn conversation, native tool use via `toolConfig`, structured output via `outputConfig.textFormat`, and streaming via `ConverseStream`) — the same constrained-decoding mechanism already load-bearing in [ADR-020 R2](#adr-020-r2--agentic-rag-and-graph-retrieval).

**Decision.**
- **Remove** Mastra and LangGraph.js from the architecture entirely — the Week-6 bake-off (C1–C5) is closed without being run; engine-only was always the fallback, and now ships as canon.
- **Implement** the agent reasoning loop, `ExploitabilityAssessmentWorkflow`, as a Temporal TypeScript workflow calling the Bedrock Converse API directly as ordinary activities — no agent framework in the loop.
- **S-LLM activity** uses `outputConfig.textFormat` with a JSON schema for structured CVE fact extraction — no `toolConfig`, and raw CVE text never reaches the P-LLM.
- **P-LLM activity** uses `toolConfig` with the discovered MCP tool schemas, receiving only structured facts and asset context.
- **Message history** lives in Postgres (`assessment_messages`), not Temporal workflow state — see [workflows §3a](workflows.md#3a-message-history-storage).
- **Retire** the Gate-2 self-hosted vLLM + Phi-4 S-LLM triage path ([ADR-008 R2](#adr-008-r2--multi-provider-llm-routing)); Gate-2 triage/classification/dedup routes to Bedrock's cheapest available model instead.
- The multi-provider Claude inference chain ([ADR-017 R3](#adr-017-r3--multi-provider-claude-inference-path)) is unaffected — Bedrock → direct Anthropic → local vLLM emergency fallback stays as-is; only the Gate-2 triage path and the inner reasoning graph are retired.

**Consequences.**
- (+) Reduced supply-chain attack surface — no framework dependency tree to audit alongside `@aws-sdk/client-bedrock-runtime`.
- (+) Single debug surface: Temporal plus first-party code, not framework indirection on top of it.
- (+) Direct access to Bedrock Converse features (constrained decoding, `ConverseStream`) without waiting on a framework's support for them.
- (+) Closes the Week-6 bake-off task (backlog-ep05 US-008-T02, 16 h) without running it, and removes the vLLM+Phi-4 Gate-2 path from [OI-39](../00-meta/open-items.md)'s unestimated scope.
- (-) The platform owns the ReAct loop directly (S-LLM extract → P-LLM reason → tool execute → budget check, a few hundred lines) rather than delegating it to a framework.
- (-) No framework-provided agent-graph visualization — mitigated by Temporal's own workflow-history UI plus Langfuse tracing (already in the stack for OTel GenAI).

**Rejected:** any replacement agent framework (LangChain, the OpenAI Agents SDK, CrewAI, AutoGen) — the point of this ADR is that Temporal + Bedrock Converse already cover the required capabilities; adding a different framework would reintroduce the same abstraction cost under a new name.
