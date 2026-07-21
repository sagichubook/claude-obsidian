---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-34]
---

# Feature — Connector Hub (US-013, US-020)

**Purpose:** how a tenant connects its environment to Dux — AWS first, CSV as a fallback, vendor integrations alongside — and how sync health is surfaced.

**Nav:** Apps · **Epic:** EP-02 · **BR:** BR-004.

## US-013 Connector Hub

**Gate 1**, with AWS as P0.

**Job.** A tenant admin connects AWS (Gate 1), CSV as a fallback (P1), and vendor integrations (live at Gate 1 per ADR-011 R2). Success: the sync completes, the asset count is visible, and any error banner is actionable.

**Journey.** This is the prerequisite for live US-011 AWS evidence, and the deep-link target from the degraded empty states in US-002–007 (`?error=asset_gap`). US-020 extends it at Gate 5 with the physical-residency agent card.

**Orchestration.** Connector sync workflows — **not** Dux Agent reasoning. AWS sync → asset ingestion → World Model update, which triggers re-assessment (US-021, Gate 1). Sync runs as the **isolated `dux-connector-sync` K8s Deployment** ([ADR-006 R4](../../20-architecture/adr-index.md#adr-006-r4--deployment-topology)), never in the API container.

**API.**

| Endpoint | Contract |
|----------|----------|
| `POST /connectors/aws/sync` | → `{sync_id, status, assets_ingested, started_at}` |
| `POST /connectors/csv/upload` | multipart. Errors: `missing_column`, `invalid_os_family`, `duplicate_hostname`. Limits: 50 K rows, UTF-8, unique `(tenant_id, hostname)` |
| Webhook | `connector.sync_failed` |

**Auth.** AWS cross-account IAM role plus External ID (ADR-004). **Credentials never appear in agent traces.** STS validation runs at setup, with the error banner mapped from the failure:

| STS error | Banner |
|-----------|--------|
| `AccessDenied` | trust-policy fix message |
| `ExternalIdMismatch` | external-ID mismatch |
| `InvalidClientTokenId` | invalid credentials |

All three persist to `aws_role_status`.

**Throttling.** Backoff with jitter, max 5 retries. Per-service limits: EC2 100, IAM 20, S3 100 req/s. CloudWatch `AWSThrottledRequests` feeds the sync-health strip.

**Phase-1 constraint.** **One AWS account per tenant.** Multi-account delegated admin is deferred to Gate 2, enforced in `AwsDiscoveryService`.

**Vendor catalog.** Rows come from the integration catalog. **Never show a false "Connected"** — a vendor reads "Coming soon" until `connector_configs.status = active`, which requires `validateCredentials()` plus a first successful sync.

**MVP connector set (v4.0 source doc, D-34).** Ten named NestJS connectors, on top of ADR-011 R2's "≥3 live at Gate 1" floor — this table is the fuller target set, not a replacement criterion:

| Connector | Auth | Rate limit | Sync interval |
|-----------|------|------------|----------------|
| Tenable.io | API key | 10 rps | 60 min |
| Qualys | API key | 5 rps | 60 min |
| CrowdStrike | OAuth2 | 6 rps | 30 min |
| AWS Security Hub | IAM SigV4 | 20 rps | 15 min |
| Rapid7 InsightVM | API key | 10 rps | 60 min |
| Splunk | REST token | 10 rps | 5 min |
| Jira | Basic auth | 10 rps | 5 min |
| ServiceNow | OAuth2 | 10 rps | 5 min |
| Okta | API token | 10 rps | 15 min |
| Azure AD | Graph API OAuth2 | 10 rps | 15 min |

Each implements the shared `BaseConnector`/`AbstractVendorConnector` contract ([ADR-011 R2](../../20-architecture/adr-index.md#adr-011-r2--vendor-connector-framework)) — vendors override only `fetchPage` and `mapRecord`. Output publishes to NATS subjects (`vulnerabilities.raw`, `assets.raw`, `tickets.raw`, `identities.raw`), consumed by the same World Model ingestion path as AWS/CSV above.

**This table's per-connector rate limit and sync interval (D-34/v4.0 source doc, vendor-API-rate-limit-derived) is the single authoritative cadence source (D-47)** — it supersedes an older, coarser "sync cadence defaults" table (6 h/12 h/24 h, grouped by wave-taxonomy rather than vendor, with no rate-limit backing) that conflicted with it by up to 288x on the four overlapping connectors (CrowdStrike, Qualys, Splunk, ServiceNow) and has been removed. **Wiz and Intune are not in this table and have no rate-limit-derived cadence yet** — [OI-58](../../00-meta/open-items.md) tracks deriving one the same way, rather than carrying over the superseded table's unresearched 6 h figure for either. Manual "Sync now" remains available on demand for every connector.

**Safety.** A sync failure raises the webhook and the US-012 / US-014 health strip. An invalid CSV produces typed errors — **there is no partial, poisoned ingest**.

**Metrics.** Sync success rate; asset-count growth; time to first sync; connector error MTTR; CSV-fallback adoption.

**Marketing map.** "Ties together all data sources", capability #7. Residency here is *logical*, via the Unified Integration Layer. The word "all" must be qualified — the 42-source taxonomy spans several waves.

## US-020 Optional Physical Residency Admin

**Gate 5** · status: draft.

**Job.** A tenant admin monitors the `dux-resident-agent` — heartbeat, version, evidence sync — for air-gapped and on-prem listening checks.

**Orchestration.** A DaemonSet (FR-014) deployed **only inside customer infrastructure**. It reports to the ingestion API via `POST /resident-agents/{id}/heartbeat`, authenticated by mTLS (preferred) or a signed JWT carrying `nonce` and `exp` ≤60 s. Its evidence surfaces in the US-011 `process_not_listening` factor cards.

**Contract firewall.** **There is no in-VPC agent before Gate 5.** For Phases 1–4, "lives inside your environment" means *logical* residency via the [Unified Integration Layer](../../20-architecture/architecture-overview.md). Sales copy must not imply otherwise.

**Safety.** Agent certificates rotate. A compromised resident triggers a tenant-scoped KS-L3 and isolates the agent.

## Vendor-token revocation mid-assessment (resolves OI-14)

**Trigger.** A vendor call — connector sync, or an in-flight `AssetContextWorker`/`ExploitabilityAssessmentWorkflow` MCP read — returns an auth failure (`401`/`403`, or a vendor-specific revoked-token code) for a connector that was `active` at assessment start.

**Detection.** The MCP tool wrapper classifies the failure as `CONNECTOR_CREDENTIAL_REVOKED` (distinct from `ConnectorSyncError`, which covers transient/5xx failures) and emits it on the existing `connector.sync_failed` webhook with `reason=credential_revoked`.

**In-flight-assessment behavior.** The workflow does not fail the assessment outright:

1. In-progress evidence gathering from the revoked connector stops; steps already completed keep their evidence.
2. Remaining prerequisite/context checks that depend on that connector's evidence yield `INSUFFICIENT_DATA` (`intel_gap` or `asset_gap`, per [confidence-calibration §2](../../40-ai-safety/confidence-calibration.md)) rather than fabricating or reusing stale evidence — same rule as any other stale/missing connector.
3. `connector_configs.status` flips from `active` to `credential_revoked` (not `error`) — the vendor card shows "Reconnect required," not "Coming soon."
4. The US-014 health strip and `connector.sync_failed` webhook fire immediately; no wait for the next scheduled sync.

**Recovery.** A tenant admin re-authenticates through the Connector Hub (US-013) — the same OAuth/IAM-role flow as initial setup. On a successful `validateCredentials()`, `status` returns to `active` and a full (not delta) sync runs once, since the revocation window may have missed changes. Any assessment left with `INSUFFICIENT_DATA` from step 2 above is eligible for the tenant's next scheduled sweep (US-021) to pick up the restored evidence — it is not auto-retried, to avoid a reconnect storm re-triggering every affected assessment at once.

**Safety.** No partial-credential state is ever passed to a vendor mutation API — a revoked connector cannot execute a write action; `VendorActionGate` checks connector status before every write, the same gate that already blocks execution on a missing rollback entry.
