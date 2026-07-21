---
owner: Engineering
status: canonical
gate: 2
last_reviewed: 2026-07-19
decisions: [D-34]
---

# DR / BCP

**Purpose:** the RTO/RPO ladder, the partition matrix, and the game-day gates.

**Parents:** BR-008.

The canonical AI and infrastructure procedures live in the [incident runbooks](../40-ai-safety/incident-runbooks.md). This document adds only the disaster-recovery layer.

## 1. RTO / RPO

| Target | Value | Implementation |
|--------|-------|----------------|
| RTO | **4 h** (platform baseline) | K8s multi-AZ node pools + CloudNativePG base-backup failover |
| RTO, Tier 1 (game-day stretch) | **<1 h** | auth, API, workflows. The DR game day measures the gap against the 4 h baseline |
| RPO | **1 h** | CloudNativePG WAL archiving to MinIO + hourly logical backups |
| Restore drill | quarterly | a full staging restore from a CloudNativePG base-backup or ephemeral-cluster snapshot |

**Per-component failover targets (v4.0 source doc, D-34) — narrower-scope than the platform RTO/RPO above, which covers full-disaster restore.** These are the individual-component failover numbers within a otherwise-healthy cluster: CloudNativePG <5 min RPO / <15 min RTO (streaming replication + S3-compatible WAL archives to MinIO); NATS JetStream <1 min RPO / <5 min RTO (multi-replica streams); Valkey <1 h RPO / <30 min RTO (RDB + AOF snapshots); MinIO <1 h RPO / <30 min RTO (erasure coding + site replication). Not a contradiction of the 4 h/1 h platform baseline above — that number assumes a harder failure (node-pool or region loss, §2) than a single component's own HA mechanism absorbing.

## 2. Chaos and partition scenarios

| Scenario | Expected behavior | RTO | Cadence |
|----------|-------------------|-----|---------|
| NATS outage, 30 min | the kill switch falls back to CloudNativePG `LISTEN/NOTIFY` (<10 s); rate limits degrade to in-memory per replica (**a documented over-admission risk**); SSE falls back to REST polling (the US-012 fallback) | <10 s kill propagation | quarterly chaos |
| CloudNativePG primary failover | base-backup restore, or the operator's automated failover | <1 h Tier 1 (Month-6 game day) | Month 6 |
| K8s node-pool or region loss | redeploy to an alternate node pool or region; DNS failover | <4 h | semi-annual |
| Bedrock provider outage | `LLMProviderPort`'s NestJS `LLMFallbackService` reweights away from Bedrock automatically on error-rate/latency signal → direct Anthropic API, then local vLLM (ADR-017 R3) | <60 s | quarterly |
| Kill-switch propagation | KS-L2/L3 at p99 <5 s (KS-001); recovery within 15 min | — | quarterly |
| Model provider outage | ADR-008 fallback within 60 s | — | quarterly |
| MCP gateway failure | the circuit breaker trips (§8 R4); the queue degrades gracefully; recovers within the RTO below | **<5 min circuit-open, <10 min full recovery** | quarterly |

**Trip and recovery criteria (resolves OI-18).** The circuit breaker trips at the R4 threshold ([incident-runbooks §R4](../40-ai-safety/incident-runbooks.md#r4--mcp-dependency-failure)) — MCP errors above 50% over 5 min, or a health-check failure. `ops:chaos-gate` now asserts three bounds:

| Bound | Value | Source |
|-------|-------|--------|
| Circuit-open detection | <5 min | existing R4 trigger window |
| Queue-depth ceiling while open | requests queue, capped at `GOV-004`'s per-tenant daily budget — no unbounded backlog | governance-kernel GOV-004 |
| Recovery (half-open → closed) | <10 min total from trip, once the error rate drops below 5% (R4 step 5) | derived: 5 min detection + up to 5 min half-open probe interval |

A drill that exceeds the 10-minute recovery bound is a scenario failure, same as any other RTO miss in this table.

## 3. Game days

**Chaos Friday.** The first Friday monthly, in staging. **Scenarios must meet their success criteria before that week's production deploy** (`ops:chaos-gate`). An SRE sign-off is required to bypass.

**DR game day (Seed Month 6).** A full, timed regional-failover exercise. Target: **RTO <1 h for Tier 1.** It documents the gap against the 4 h active-passive baseline, and feeds the Series B multi-region active-active design.

**DNS abort path (resolves OI-19; mechanism rewritten 2026-07-19, D-33).** The rollback runbook's MinIO static-frontend revert (§3 above, `runbooks.md`) uses:

| Step | Mechanism |
|------|-----------|
| Revert MinIO static-frontend deployment | restore the prior build's object version from MinIO versioning (`mc cp --version-id <id>` from the bucket's version history) |
| Revert DNS record (if a record, not just the frontend build, changed) | Cloudflare API `PATCH /zones/{zone_id}/dns_records/{id}` restoring the prior `content`/`proxied` values, captured before every DNS change per the standing pre-change-snapshot rule |
| Credentials | a scoped Cloudflare API token (`Zone:DNS:Edit`, single zone `dux.io`) plus MinIO admin credentials, both in Vault (D-5 R2 convention), rotated every 90 days |
| On-call owner | `@platform-oncall` (same PagerDuty service as the INFRA rollback runbook) — Cloudflare console / MinIO console access is a break-glass fallback, not the primary path |

**Exercise.** The DNS abort path is a required step in the first staging game day (§3 above) — a MinIO static-frontend deployment is rolled back live, and the Cloudflare API / MinIO admin credentials are confirmed to actually authenticate, not merely to be present in Vault.

## 4. Series B forward

Series B targets Tier 1 active-active RTO of **<1 h** (**<15 min** at scale), with a **required multi-region game day before any EU or APAC go-live**.

Governed by [series-b-scale](../70-governance/series-b-scale.md) — a backlog shell. **Inherit the Series A BCP/DR posture until that document is content-complete.**

Under ADR-006 R4 (Kubernetes/EKS from Gate 1), the former "Railway → AWS migration" prerequisite for multi-region is **already satisfied**. The trigger is now capacity and region topology, not a hosting migration.
