---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-20
decisions: [D-18, D-34, D-43]
---

# Multi-Tenancy Implementation

**Purpose:** how tenant isolation is enforced, from the JWT claim down to the connection pool — and how it is tested.

**Parents:** BR-001 · **Authority:** complements [ADR-002](adr-index.md). The canonical RLS SQL lives in [data-model §6](data-model.md).

## 1. Model

| Aspect | Decision |
|--------|----------|
| Isolation | shared database, shared schema |
| Tenant key | `tenant_id` — a cryptographically secure UUID |
| Database enforcement | PostgreSQL RLS, with FORCE |
| Application enforcement | middleware sets tenant context on **every** request |
| Auth source of truth | the NestJS JWT `tenant_id` claim (ADR-001) |
| Resource lookup | composite key `(tenant_id, id)` |
| Exceptions | **none, without a new ADR** |
| Overhead budget | RLS costs roughly **5–15%** (2026 field consensus). Owned explicitly, and kept low by `tenant_id`-leading composite indexes |

## 2. Context propagation rules

OWASP-validated. All seven are mandatory.

1. **Tenant context comes from the authenticated JWT claim only.** Never from a client header, query parameter, or path segment without validation.
2. Every tenant-scoped database operation runs inside a transaction with `SET LOCAL app.tenant_id = $1`.
3. **Every tenant-scoped query runs through PgBouncer in transaction mode (`pool_mode=transaction`), with `SET LOCAL app.tenant_id` inside that transaction.** `SET LOCAL` is transaction-scoped by construction — the GUC cannot leak to the next borrower, because it does not survive past the transaction that set it. **CloudNativePG-fronted PgBouncer (2026-07-19, D-33) can run session-mode pooling if ever needed** — unlike Neon's pooler, this is now self-hosted and unconstrained — but transaction mode remains the default with no stated reason to change it.

   > **The pool-reset trap this guards against is real, and transaction-mode `SET LOCAL` closes it without `DISCARD ALL` or session-mode pooling.** A prior version of this document required PgBouncer session mode — unimplementable on Neon's pooled endpoint, which was transaction-mode only. That mandate is retired; the self-hosted CloudNativePG migration makes session mode available again, but there is still no reason stated to use it.

4. **Admin impersonation**, in order: validate the admin JWT → verify MFA → read `X-Impersonate-Tenant` → replace the effective `tenant_id` → write the `admin.impersonate` audit record → `SET LOCAL` → query. **RLS still applies to the impersonated tenant.**
5. **Service to service:** an internal JWT with `iss=dux-internal`, `aud=worker`, TTL 5 min. It **must** carry `tenant_id`, and the worker rejects a missing claim *before* `SET LOCAL`.
6. **The kill-switch `LISTEN/NOTIFY` fallback (KS-007) uses CloudNativePG's direct, unpooled endpoint** — `LISTEN`/`NOTIFY` require a persistent session connection, which the pooled endpoint cannot provide in either mode. Local development may use the same direct connection with `SET LOCAL`, for parity.
7. **Every OTel span carries `dux.tenant_id_hash`** — `HMAC-SHA256[:8]`. **The raw `tenant_id` never appears in a log.**

## 3. Application-layer enforcement

| Layer | Enforcement | Verification |
|-------|-------------|--------------|
| API gateway | reject any request without valid tenant context; JWT-derived only | AUTH-003 |
| Service layer | assert `resource.tenant_id == request.tenant_id` | unit tests |
| Cache (Valkey) | key is `tenant:{HMAC-SHA256(tenant_id, secret)[:16]}:`. On read: deserialize, then **assert `payload.tenant_id === request.tenant_id`** — otherwise treat it as a miss and emit `cache_tenant_mismatch` | key-naming lint |
| File storage (R2) | per-tenant envelope encryption — a DEK wrapped by the platform KEK in Vault/SSM. **The path `tenants/{HMAC[:12]}/` is obfuscation only, not a control.** `StoragePort` mints pre-signed URLs valid ≤15 min, after a JWT check; a GET verifies the JWT tenant **before** unwrapping the DEK | `test:storage-envelope` |
| Message queue | `tenant_id` travels in the payload; `WorkerTenantGuard` validates it before `SET LOCAL` | consumer tests |
| Search (Postgres FTS) | `tenant_search_index (tenant_id, doc_type, entity_id)`; every query includes `tenant_id` | ISO-006 |
| LLM calls | `llm_usage_events` tagged with `tenant_id` | metering tests |

`TenantScopedRepository<T>` enforces composite-key lookups. **ESLint bans `findOne({ where: { id } })` without a `tenant_id`.** The GUC is re-asserted at runtime before every query.

## 4. Graph latency — two numbers, deliberately not unified

| Number | Value | Purpose |
|--------|-------|---------|
| **NFR-004 SLO ceiling** | 3-hop CTE p95 **<200 ms above 2 K assets** (TR-NFR-006) | the commitment |
| **Migration trigger** | p95 **>150 ms above 1 K assets, for 7 consecutive days** | the early warning |

**The trigger fires *before* the SLO breaches**, giving lead time to act. **Apache AGE (D-34) is the first-line response (2026-07-20, D-43):** the graph layer already lives on the same CloudNativePG Postgres, so the trigger is answered with AGE-native scaling levers — index tuning, read-replica routing, partitioning — not a database migration. **Neo4j is kept on the books only as a further-future escape valve**, for the case where AGE-native tuning itself can't hold the SLO; it is not an active migration in progress. **The 50 ms and 1 K-asset gap between the trigger and the SLO ceiling is deliberate lead-time budget either way. Do not "simplify" it into one number.**

## 5. Tenant lifecycle

**Provisioning.** Create the tenant and default roles (the first user is admin) → validate the AWS role ARN and external ID → run `pnpm test:isolation --filter ISO-007` and `check-rls.sh` (**activation is blocked on failure**) → connector sync → audit `tenant.provisioned`.

**Suspension.** Block new assessments and writes (403); set `agent_sessions.status = blocked`; send the Temporal cancel signal; open a read-only 30-day export window; activate KS-L3; audit `tenant.suspended`.

**Deletion.** Soft-delete → days 0–30, export available (24 h SLA) → days 31–90, legal-hold retention → **day 90, hard purge** across MinIO, the database, and backups. Audit logs are anonymized on purge. Audit `tenant.deleted`, then `tenant.purged`.

**A `legal_hold` flag blocks the day-90 purge, and notifies Legal.**

## 6. Noisy neighbor

**Primary detector:**

```promql
sum by (tenant_id)(rate(pg_stat_statements_calls[5m]))
  / sum(rate(pg_stat_statements_calls[5m])) > 0.10
```

Sustained for 5 min → throttle that tenant's assessment queue. Auto-resume below 5%.

**Secondary (TEN-05):** above 8% for 30 min — this catches the slow bleed the primary detector misses.

**Pool cap:** 5 connections per tenant, plus 20% headroom, with a `pgbouncer_pool_exhaustion` alert.

**Cost cap:** $25 per hour per tenant, enforced through the `LLM_USAGE_EVENT` index.

## 7. Isolation testing

**Mandatory:** ISO-001–010, plus API-layer tenant-ID fuzzing (`pnpm test:fuzz-tenant-id`, ISO-FUZZ-001–005).

These run on **every** PR touching `packages/api/`, `packages/database/`, or `packages/core/`.

**Any cross-tenant read is a merge block.** Full suite tables: [ci-cd-testing](../50-engineering/ci-cd-testing.md).
