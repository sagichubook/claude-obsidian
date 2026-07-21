---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-34]
---

# Data Model (ERD)

**Purpose:** the entities, their tenancy classification, referential integrity, retention, indexes, and the canonical RLS DDL.

**Parents:** BR-001, BR-005.

**Authority:** the canonical index list and the RLS SQL patterns live here. Pooling, lifecycle, and quotas live in [multi-tenancy](multi-tenancy.md).

## 1. Tenancy model

Shared database, shared schema, with RLS (ADR-002).

**Global entities — no `tenant_id`:** `CVE`, `EPSS_SCORE`, `CALIBRATION_RECORD`, `AGENT_DEFINITION`.

**Everything else carries a `tenant_id` foreign key — including `WORLD_MODEL_VERSION`.**

> **Why `WORLD_MODEL_VERSION` is tenant-scoped, not global.** ADR-011 R2 bumps it whenever a connector reports a material change. **A global version row would let one tenant's connector sync cancel every other tenant's in-flight assessments**, through `WorldModelVersionPurgeJob` — see [workflows §9](workflows.md). That is a cross-tenant blast-radius bug, not a modelling preference.

**Composite key rule.** Every resource lookup uses `(tenant_id, id)` — **never `id` alone**. This is what prevents IDOR.

**Index policy.** Every tenant-scoped index **leads with `tenant_id`**. The RLS query rewrite depends on it.

## 2. Core entities

Columns abbreviated; the full column tables are preserved in the ERD.

| Entity | Key columns and notes |
|--------|----------------------|
| `TENANT` | `id`, `name`, `slug` (UK), `settings` jsonb (requires `aws_role_arn`, `external_id`), `status` enum — `provisioning` → `active` ↔ `suspended` → `deleted` → `purged`. **`purged` is terminal; there is no transition out of it** |
| `USER` | composite UK `(tenant_id, email)`; `role` enum `admin` / `member` / `viewer`; `auth_provider`; `external_auth_id` |
| `CVE` *(global)* | `id`, `description`, `cvss_score`, `kev_status`, `last_modified` |
| `EPSS_SCORE` *(global)* | `cve_id` PK, `epss_score`, `percentile`, `synced_at` |
| `EPSS_SCORE_HISTORY` *(global)* | `cve_id`, `epss_score`, `percentile`, `snapshot_date`; composite PK `(cve_id, snapshot_date)`; 90-day rolling retention. Appended daily alongside the `EPSS_SCORE` upsert — feeds [predictive-risk-forecasting](../10-product/features/predictive-risk-forecasting.md) |
| `ASSET` | `hostname`, `asset_type` enum, `vpc_id`, `subnet_id`, `os_family`, `has_public_ip`, `metadata` jsonb, `deleted_at` (soft delete) |
| `ASSET_RELATIONSHIP` | `source_asset_id`, `target_asset_id`, `relationship_type`; unique on `(tenant_id, source, target, type)` |
| `FINDING` | `asset_id`, `cve_id`, `state` enum — `open` / `under_research` / `exploitable` / `mitigated` / `accepted` / `false_positive` |
| `VULNERABILITY_INSTANCE` | `asset_id`, `cve_id`, `sources[]`, `exploitability_status`, `network_exposure`, `last_seen_at`, `external_uids` |
| `VULNERABILITY_INSTANCE_ACKNOWLEDGMENT` | `reason`, `expires_at`, `revoked_at`; auto-expire job |
| `CUSTOM_METRIC` | `display_name`, `entity_type`, `dql_filter`, `group_by[]`, `dashboard_id`, `ordinal` |
| `EXPLOITABILITY_ASSESSMENT` | `finding_id`, `agent_session_id` (the KS-L1 target), `status` enum — `queued` / `researching` / `evaluating` / `complete` / `failed`; `reasoning_chain` (legacy, dropped at Gate 3); `confidence_score` (Platt); `calibration_record_id`. Partial unique on `(tenant_id, finding_id) WHERE status IN (active states)` |
| `ASSESSMENT_REASONING_STEP` | `step_order`, `step_type` (`reasoning` / `tool_result` / `conclusion`), `content`, `source_refs` |
| `ATTACK_PATH` | `path_nodes` jsonb — normalize to `ATTACK_PATH_NODE` at Gate 2 if the CTE degrades; `validated` |
| `CONTROL`, `CONTROL_ASSET_MAPPING` | vendor; `control_type` and subtype; `settings`; `mapping_type` |
| `AGENT_SESSION` | `session_type` (`assessment` / `chat`), `status`. **This is the KS-L1 scope** |
| `MCP_TOOL_INVOCATION` | `tool_name`, `server_id`, `outcome`, `latency_ms` — the PS-007 audit record |
| `CHAT_SESSION` / `CHAT_MESSAGE` / `CHAT_ACTION` | messages partitioned (1 year); `token_count` for billing; `hitl_status` |
| `USER_PREFERENCE` + `PREFERENCE_SCOPE` + `PREFERENCE_APPLICATION` | natural-language query, parsed scope, action, confidence, expiry |
| `ASSESSMENT_STATE_TRANSITION` | `from_status`, `to_status`, `actor_id` |
| `WEBHOOK_CONFIG` + `WEBHOOK_DEAD_LETTER` | `secret_ref` (Vault/SSM); DLQ payload and `attempt_count` |
| `AUDIT_EVENT` | `action`; `hash_chain` = `HMAC-SHA256(chain_key, prev_hash ‖ tenant_id ‖ action ‖ payload_hash ‖ created_at)`; `chain_seq` (monotonic per tenant, TEN-08); the genesis row has `prev_hash = 'GENESIS'`. **`chain_key` lives in Vault at `audit/chain-key`, rotated quarterly** |
| `MITIGATION_STEP`, `OWNERSHIP_EVIDENCE` | Gate-2 entities (dotted in the diagram) |
| `WORLD_MODEL_VERSION` (`world_model_versions`) | `tenant_id` FK; composite PK `(tenant_id, version)`; `active` |
| `AGENT_DEFINITION` *(global)* | `name`, `version`, `permission_scope`, `active` |
| `LLM_USAGE_EVENT` | `model`, `input_tokens`, `output_tokens`, `cost_usd`. **Enforces the $25/hour cap** |
| `CALIBRATION_RECORD` *(global)* | `model_version`, `prompt_version`, `platt_params`, `brier_score`, `ece`, `active` |

## 3. Referential integrity

| Parent | Child | ON DELETE |
|--------|-------|-----------|
| `tenants` | every `tenant_id` FK table | CASCADE |
| `findings` | `exploitability_assessments` | RESTRICT |
| `assets` | `findings` | RESTRICT — composite FK `(tenant_id, asset_id)` |
| `exploitability_assessments` | `assessment_state_transitions`, `attack_paths` | CASCADE |
| `user_preferences` | `preference_scopes`, `preference_applications` | CASCADE |
| `webhook_configs` | `webhook_dead_letters` | SET NULL — **preserve the DLQ** |

**Tenant purge order. Do not reorder these:**

1. Halt workflows, and trip the kill switch.
2. Delete the MinIO prefix `tenants/{hash}/`.
3. `DELETE FROM tenants` — the cascade does the rest.
4. Revoke Vault secrets.
5. Write the `tenant.purged` audit record.

## 4. Retention matrix

| Data | Hot (Postgres) | Cold | Notes |
|------|----------------|------|-------|
| `audit_events` | 90 days | 7 years, Parquet in MinIO | actor IDs hashed 2 years post-purge; chain head anchored hourly to MinIO Object Locking (`dux-audit-anchors/`, COMPLIANCE mode, 7 years) |
| `mcp_tool_invocations` | 90 days | tenant-prefix archive | purged on hard delete |
| `chat_messages` | 1 year | export bundle | **PII lives in `content`**; purged on hard delete |
| Assessment state | per partition | — | state transitions retained |
| LLM traces | Langfuse retention | — | `tenant_id` in metadata; the sanitizer runs before export |
| API traces | 7 days | — | 10% head sampling, plus 100% of errors |

## 5. Indexing strategy

Physical tables are `snake_case` plural; ERD entities are `SCREAMING_SNAKE` singular. Global tables lead their index with the natural primary key.

**Every tenant-scoped index leads with `tenant_id`.** Representative set:

| Table | Index |
|-------|-------|
| `ASSET` | `(tenant_id, hostname)`, `(tenant_id, subnet_id)`, `(tenant_id, last_synced_at DESC)` |
| `FINDING` | `(tenant_id, cve_id, asset_id)` |
| `EXPLOITABILITY_ASSESSMENT` | `(tenant_id, status, completed_at)`, `(tenant_id, finding_id)` |
| `ASSESSMENT_REASONING_STEP` | `(tenant_id, assessment_id, step_order)` |
| `AUDIT_EVENT` | `(tenant_id, created_at DESC)` |
| `EPSS_SCORE` *(global)* | `(cve_id)` |

**Monitoring.** Alert if any tenant-scoped table exceeds **more than 0 sequential scans per 5 min in staging**, or **more than 10 per hour in production**. A sequential scan on a tenant-scoped table means the RLS rewrite has lost its index.

## 6. Row-level security (canonical DDL)

```sql
ALTER TABLE assets ENABLE ROW LEVEL SECURITY;
ALTER TABLE assets FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_select ON assets FOR SELECT USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
CREATE POLICY tenant_insert ON assets FOR INSERT WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
CREATE POLICY tenant_update ON assets FOR UPDATE USING (tenant_id = current_setting('app.tenant_id', true)::uuid) WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
CREATE POLICY tenant_delete ON assets FOR DELETE USING (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

**Apply all four policies to every tenant-scoped table, including `world_model_versions`** (see §1).

**Global tables:** `cves` gets `FOR SELECT USING (true)`, and ingestion runs under a separate connector role. Likewise `epss_scores`, `calibration_records`, `agent_definitions`.

**Migration CI.** `check-rls.sh` verifies `ENABLE` **and** `FORCE` on every `tenant_id` table, within a single transaction: `CREATE TABLE` → `ENABLE` → `FORCE` → `CREATE POLICY` → `CREATE INDEX`. ISO-013 covers the `tenant_cve_findings` materialized-view refresh.

## 7. RAG schema

**Enabled (D-34, ADR-020 R2): `rag_enabled = true`.** Agentic RAG runs hybrid vector + BM25 retrieval over `tenant_embeddings`, extended with an Apache AGE graph layer (below).

`tenant_embeddings` — `embedding vector(1536)`, RLS FORCE. **A `tenant_id`-leading HNSW index is not implementable in pgvector** — HNSW is a single-column ANN index type and cannot compose with a leading scalar column the way a btree does. The actual approach: `tenant_embeddings` is declaratively partitioned by `tenant_id` (`PARTITION BY LIST (tenant_id)`, matching the composite-key pattern in §1), with a local HNSW index built per tenant partition — search never crosses a partition boundary, so ANN recall stays tenant-scoped by construction, not by query-time filtering.

**The ISO-012 ANN adversarial-neighbor test against this live, tenant-scoped HNSW index is not yet confirmed executed** — the architectural flip to `rag_enabled = true` is current canon, but this doc-only pass cannot itself produce that execution artifact. Tracked as [OI-41](../00-meta/open-items.md).

**Graph layer (D-34, ADR-020 R2): Apache AGE**, a Postgres extension on the same CloudNativePG instance as `tenant_embeddings` above — no second database, no second isolation model to secure. Per-edge provenance and integrity hashing apply the same "connector-asserted data is untrusted for negative verdicts" rule to every graph edge ([camel-plane §7](../40-ai-safety/camel-plane.md)). **Concrete node/edge/provenance column-level schema is not yet specified anywhere in this corpus** — the architectural decision exists, the ERD entities for it don't yet. Tracked as [OI-46](../00-meta/open-items.md).

See [camel-plane §7](../40-ai-safety/camel-plane.md).
