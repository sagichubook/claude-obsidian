---
owner: Engineering
status: canonical
gate: 2
last_reviewed: 2026-07-19
decisions: [D-3, D-5, D-18, D-34]
---

# Seed Operational Runbooks

**Purpose:** the seed-stage operational procedures — deploy, rollback, SSO, migrations, tenant lifecycle, and the seed-only extensions.

**Parents:** BR-006.

**These are stage deltas only.** The canonical AI and infrastructure procedures live in the [pre-seed incident runbooks](../40-ai-safety/incident-runbooks.md). What follows adds PagerDuty IDs, admin CLI, and seed thresholds — **do not duplicate the canonical step tables here.**

Topology is Kubernetes, Amazon EKS (ADR-006 R4).

## 1. Gate 2 dry-run

Run before on-call go-live.

| Runbook | Staging command | Expected |
|---------|-----------------|----------|
| Deploy | `pnpm ops:test-deploy --env staging` | `deploy_ms < 900000` |
| Rollback | `pnpm ops:test-rollback --env staging` | `rollback_ms < 300000` |
| Cross-tenant leak | `ops:test-runbook --runbook cross-tenant` | containment within 60 s |
| Token cost runaway | `--runbook token-cost` | the cap is enforced |
| Model provider outage | `--runbook model-outage` | fallback within 60 s |
| MCP dependency failure | `--runbook mcp-failure` | circuit opens in under 5 min |
| Rate limit cascade | `--runbook rate-limit` | throttle active |
| Context window exhaustion | `--runbook context-exhaust` | checkpoint saved |
| Prompt cache invalidation | `--runbook cache-warm` | hit rate above 85% |
| Agent quota exhaustion | `--runbook quota-cap` | hard cap enforced |
| Shadow AI | `admin:shadow-ai-reconcile --env staging` | `undeclared_count: 0` |
| Kill switch — KS-001 / KS-L1 / KS-007 | `ops:test-kill-switch --level L2` / `--level L1` / `--fallback pg-notify --level L3 --iterations 50` | p99 <5 s / ≤30 s / <10 s |
| Bedrock SDK dependency audit | weekly GHA `npm audit` + Sigstore provenance check against the pinned `@aws-sdk/*` versions | no unaddressed high/critical CVE |

**Admin CLI status gate (OPS-08).** Every `pnpm admin:*` command referenced by these runbooks **must be status `implemented`, not `spec`, before Gate 2.** A `spec`-status command **blocks the enterprise pilot**. `ops:smoke` in CI covers each referenced command.

## 2. Deploy (INFRA)

GitHub Actions → **Pulumi-driven K8s rolling deploy**. `pnpm ops:migrate --env production` runs **before** the API image.

**preStop drain:** the replica stops accepting new workflows, waits `max(step_timeout, 120s)` for in-flight activities, and only then terminates. `dbos_workflows_orphaned_total` alerts if it rises above 0.

**Auto-rollback** on an error rate above 1%, or p95 above 2× baseline.

Versioning: SemVer for the API, CalVer for the UI. A hotfix lands within 30 min (1 approval; P0 bypasses).

**Steps:**

1. Confirm CI is green.
2. Run migrations.
3. Monitor the K8s rollout — `/health` must return 200.
4. Check post-deploy metrics: error rate <0.5%, p95 <300 ms.
5. Confirm every task is on the new image.
6. Deploy the static frontend to MinIO (behind Cloudflare CDN), if the frontend changed.
7. RED verification for 15 min.
8. Verify kill-switch propagation at p99 <5 s.
9. Update the status page.
10. Archive the evidence to S3 (SOC 2 CC8.1).

**Cost benchmark gate: a staging average at or below $0.55** (D-3).

**The AI safety check is omitted for INFRA deploys** — unless `packages/agents/` or `models.json` changed, in which case run `aibom:validate`, `shadow-ai-reconcile`, and `test:prompt-injection`.

## 3. Rollback (INFRA)

**Target: under 5 minutes.**

Revert the K8s rolling deploy to the previous Deployment revision (`kubectl rollout undo`). Revert the MinIO static-frontend deploy if the frontend is implicated.

> This depends on the Cloudflare DNS abort path — see [dr-bcp §3](dr-bcp.md#3-game-days) for the `wrangler`/API rollback mechanism, credentials, and on-call owner.

Poll `/health` and the error rate; monitor RED for 15 min. **Disabling the feature flag is the first option to try, before a rollback.**

A down-migration drill runs quarterly, requiring 2 approvals (Engineering lead + on-call). `ops:migrate` takes a CloudNativePG base-backup snapshot before any destructive migration.

**Forward-fix only when there is no safe down migration. Never drop a column in production without a deprecation cycle.**

## 4. SSO onboarding (seed trigger)

OIDC preferred, with **PKCE mandatory** — the implicit flow is disabled. SAML 2.0 is the fallback. Okta and Entra are primary.

JIT provisioning defaults to `viewer`, with group → role mapping. SCIM 2.0 is optional: tokens live in Vault, rotate every 90 days, and emit an `sso.scim.token.rotated` audit record. Session timeout is 8 h.

**Rollback:** `admin:sso-disable --tenant $ID` — under 5 min, invalidating sessions.

## 5. Database migration (INFRA)

Drizzle Kit. `check-rls.sh` verifies FORCE RLS on every `tenant_id` table. `test:isolation` runs post-migration.

A production `down` migration requires **2 approvals**; CODEOWNERS on `packages/database/migrations/` enforces it.

**Forbidden:**

- Dropping `tenant_id` without an ADR.
- Disabling RLS.
- Long migrations without `CONCURRENTLY`.

## 6. Secret rotation

A 30 / 7 / 1-day notification sequence, by email and in-app banner. Self-service rotation UI. Delivery is tracked in the audit log.

Secrets live in HashiCorp Vault (D-5 R2), including transit for OAuth refresh tokens.

## 7. Tenant provisioning and offboarding

**Provisioning.** Idempotent slug. Validate the AWS role with `admin:tenant-verify-aws-role` — on failure the tenant moves to `status=aws_role_failed`, and alerts if it stays stuck for more than 30 min. `test:isolation --tenant $NEW_ID` is a gate. Audit `tenant.provisioned`.

**Offboarding.** Soft-delete at day 0 → 24 h export SLA within the days 0–30 export window → revoke sessions, keys, and credentials → days 31–90 legal-hold retention → day-90 purge. The `legal_hold` flag blocks the day-90 purge and notifies Legal. Then the destruction certificate. Matches [multi-tenancy §5](../20-architecture/multi-tenancy.md), the lifecycle authority.

## 8. NVD sync stale (INFRA, P2)

**Trigger:** `NVD_SYNC_WARN` above 2 h; `NVD_SYNC_STALE` above 4 h.

1. Check the API key in the secrets store.
2. Read the ingestion logs.
3. Verify 429 backoff.
4. Verify the cache — 48 h TTL, sha256.
5. Enable the intel fallback.
6. `admin:nvd-resume --chunk-days 120`.
7. It clears once feed age drops below 7,200 s.

Notify the customer **only** if the outage exceeds 24 h.

**The AI safety check is omitted by design — no agent is involved.** Impact framing is platform-trust erosion; the MRR-at-risk formula is in the [incident runbooks](../40-ai-safety/incident-runbooks.md).

## 9. Incident response routing

COMPOSITE incidents route to the [canonical agentic runbooks](../40-ai-safety/incident-runbooks.md). INFRA-only incidents use general IR.

The `ai-safety` PagerDuty service pages `@ai-safety-oncall`, with a **60-second halt escalation drill before on-call go-live**.

**Crossing the MRR-at-risk threshold auto-pages the CEO and Founder.**

## 10. Shadow AI detection (AI-AGENT, P0-B)

**Trigger:** `DuxShadowAI` — `undeclared_count > 0`.

1. Page the CTO within 5 min.
2. The AI Safety Lead halts within 60 s.
3. L2 kill switch.
4. Export session evidence.
5. Enumerate the undeclared agents against the registry.
6. **Block the deploy pipeline until `undeclared_count: 0`.**
7. Register the agent, and baseline it afterwards.

L3 and L4 escalation criteria follow blast radius, tool scope, and any cross-tenant indicators.

## 11. Seed-only extensions

**Agent quota exhaustion.** Hard-cap enforcement 1 h after the 100% alert; L2 at 120%. **The governance-kernel cost cap fails closed, independently of Stripe meter reconciliation.**

**Rate limit cascade (depth).** Dynamic throttle at `max(1, floor(baseline × 0.5))`. Escalate provider quota after 15 min.

**Context exhaustion (depth).** Checkpoint at 80%, abandon at 100%, max 3 resumes. More than 10 sessions per hour triggers a prompt-template review.

**Neo4j reconciliation failure.** Divergence above 0.1% → set the `neo4j_graph` flag to 0%, so all reads fall back to the CTE (<30 s) → `admin:neo4j-replay --from-offset` → a 7-day shadow period before retrying.

**Chaos Friday.** The first Friday monthly, in staging. **It gates that week's production deploy** (`ops:chaos-gate`).

**Billing reconciliation drift.** A Stripe-versus-platform delta above 5% → `admin:quota-hold` → backfill or credit → require 3 clean daily runs.

**Assessment dedup failure.** When `AssessmentDeduplicationService` misses — a duplicate `assessment_id` within the CVE + asset window — alert `@platform-oncall` (legacy alert name: `DuxAssessmentDedupFailure`). **Queue segmentation keeps one tenant's dedup storm off the shared workers.**
