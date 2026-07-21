---
owner: Engineering
status: canonical
gate: 2
last_reviewed: 2026-07-21
decisions: []
---

# Public Data API (`/v1/*`)

**Purpose:** the programmatic read surface — custom metrics, vulnerability instances, and batch CVE research.

**Plane:** Public Data — Bearer API key (`agt_…`, data scopes) · **Ships:** on the Seed public-API trigger, **not** in Phase 1 · **Parents:** BR-011, BR-012.

**Authority:** OpenAPI 3.1 is the source of truth. The contract is pinned at `v1.0.0` (2026-06-26). Auth mechanics, rate limits, and the versioning policy shared across all three planes are covered once in [api-overview.md §3–4](api-overview.md#3-auth) — this file covers only what's specific to the Public Data plane's endpoints.

**Every endpoint is a GET, except `POST /v1/cve-research`.** There is no create, update, or delete for metrics or instances in v1.

## 1. `GET /v1/custom-metrics` (FR-021, US-022)

A paginated list.

**Query parameters:**

| Parameter | Constraint |
|-----------|-----------|
| `sort` | column list; a `-` prefix means descending. Validated against `CustomMetricItem`'s own field list (below) — an unresolvable column returns the same 422 shape as an unresolvable DQL field (§7) |
| `page` | ≥1, default 1 |
| `size` | 1–200, default 10 |
| `entity_type` | an `EntityType` |
| `is_active` | boolean |
| `dashboard_id` | — |
| `search` | max length 255 |

**200** → `PaginatedResponse[CustomMetricItem]` — `items`, `total`, `page`, `size`, `pages`.

**`CustomMetricItem` required fields:** `id`, `display_name`, `description`, `entity_type`, `dql_filter`, `group_by`, `is_active`, `dashboard_id`, `ordinal`, `created_by`, `created_at`, `updated_at`. Optional: `dashboard_ids[]`.

## 2. `GET /v1/custom-metrics/{id}`

→ `CustomMetricItem`.

## 3. `GET /v1/custom-metrics/{id}/data`

→ `CustomMetricDataResponse`: `metric_id`; `data` (a `CustomMetricDataPoint[]` of `timestamp`, `group_by_values` — a nullable map — and a numeric `value`); `sources` (nullable).

**Time-range bounded**, since a metric's history can grow unbounded: `from`/`to` timestamps, plus the same cursor shape as §4. The exact page-size ceiling is Engineering's to size against expected data-point volume — not invented here.

## 4. `GET /v1/vulnerability-instances/{cve_id}` (FR-022, US-024)

Path pattern: `^CVE-\d{4}-\d{4,}$`.

**Cursor pagination:** `cursor` (opaque), `limit` (1–5000, default 3000), `expand` (only `asset` is accepted).

**200** → `CursorPaginatedResponse[VulnerabilityInstanceV1Response]` — `items`, `has_more`, `next_cursor`.

| Field | Required? | Notes |
|-------|-----------|-------|
| `id`, `cve_id`, `sources`, `last_seen_at`, `is_acknowledged`, `asset_id`, `external_uids` | required | — |
| `exploitability_status` | optional | `potentially_exploitable` \| `not_exploitable` \| `partially_mitigated` \| `null`. **Null means unresearched** |
| `remediations` | optional | nullable `string[]` |
| `network_exposure` | optional | Verdict — `internet` \| `external` \| `internal` \| `unreachable`; nullable |
| `asset` | optional | the full `AssetV1Response`, **only** when `expand=asset`. Its `type` is `device` \| `cloud_compute` |

## 5. `POST /v1/cve-research` (FR-023, US-024)

Body — `CveResearchV1Request`: `cve_ids`, 1–50 entries, each matching `^CVE-\d{4}-\d{4,}$`.

**202** → an array of `CveResearchV1Item`, each carrying `status: backlog | completed` — matching `POST /research/queue`'s async-enqueue semantics for the same underlying capability ([api-overview §6](api-overview.md#6-application--public-bridge-the-research-queue)).

**Dedup.** Shares the same per-`cve_id` dedup mechanism as `POST /research/queue` (not per-batch, since batches can overlap) — a resubmitted `cve_id` inside a batch does not enqueue a duplicate research run.

## 6. Pagination limits

| Parameter | Min | Max | Default |
|-----------|-----|-----|---------|
| `cve_ids` (batch) | 1 | 50 | — |
| `size` (custom-metrics) | 1 | 200 | 10 |
| `page` | 1 | — | 1 |
| `limit` (vulnerability-instances) | 1 | 5000 | 3000 |

## 7. DQL (Dux Query Language)

A tenant-scoped filter expression, stored in `CustomMetricItem.dql_filter` and evaluated against World Model entities.

**`EntityType` is a closed set:** `device`, `user`, `label`, `finding`, `cloud_compute`, `vulnerability_instance`, `cve`, `mitigation`.

`group_by` defines the aggregation dimensions for `CustomMetricDataPoint.group_by_values`.

**Custom metrics are read-only over public v1.** Configuration happens in the tenant-admin UI (US-022), at Seed.

### DQL grammar (resolves OI-15)

DQL is a flat, single-entity boolean filter expression — **no joins, no subqueries.** `dql_filter` is stored as the raw expression string; `entity_type` (§7 above) fixes which World Model entity's columns are addressable.

```
expression   := clause (( "AND" | "OR" ) clause)*
clause       := "(" expression ")" | comparison
comparison   := field operator value
field        := identifier ("." identifier)*        // dotted path into jsonb columns, e.g. metadata.owner_team
operator     := "=" | "!=" | ">" | ">=" | "<" | "<=" | "IN" | "NOT IN" | "CONTAINS" | "IS NULL" | "IS NOT NULL"
value        := string | number | boolean | "(" value ("," value)* ")"   // parenthesized list, for IN/NOT IN
```

- **Precedence:** `AND` binds tighter than `OR`; explicit parentheses override. No operator overloading — `=` is never a fuzzy match.
- **`CONTAINS`** is the only substring/array-membership operator, valid only on `string[]` and `text` columns (e.g. `sources CONTAINS "wiz"`); it does not accept a field on the right-hand side.
- **`field`** must resolve to a real column or a `jsonb` path on the `entity_type`'s backing table (§2 [data-model](../20-architecture/data-model.md)) — an unknown field is a 422 at save time, not a silent empty result.
- **Grammar limits:** max expression length 2,000 characters; max nesting depth 5; max 20 clauses. Exceeding any limit is a 422 `DQL_EXPRESSION_TOO_COMPLEX`.
- **No arbitrary code execution** (unchanged from [tenant-settings](../10-product/features/tenant-settings.md#us-022-custom-metrics-configuration)) — the parser produces an AST that compiles to a parameterized SQL `WHERE` clause scoped by RLS; it never interpolates the raw string.

**Example:** `severity >= 7 AND (state = "exploitable" OR state = "under_research") AND asset.has_public_ip = true`

**Validation.** `POST`/`PUT` on a `CustomMetricItem` runs the parser synchronously; a parse or field-resolution failure returns 422 with the offending token's character offset. Valid grammar is cached as a pre-compiled AST alongside the raw string, so `GET /v1/custom-metrics/{id}/data` never re-parses on read.

This grammar is normative; the OpenAPI 3.1 `description` field on `dql_filter` links here rather than restating it.

## 8. Mapping and exposure rules

**`confidence_score` is not exposed on v1.** Consumers get `exploitability_status` only.

This surface maps to US-010's `view_mode=by_instance`, and to the Exposure Analysis asset rows. The public read of `is_acknowledged` reflects the application-plane acknowledgment (US-023).

**Publishing checklist (Seed, AI-142):** OpenAPI `securitySchemes` (Bearer API key); rate-limit `429` schemas; the scope enum; and a Security review **before** the trigger fires.
