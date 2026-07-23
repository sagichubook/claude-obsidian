---
type: resource
title: "Dux API Reference"
topic: "dux/api"
created: 2026-07-22
updated: 2026-07-23
tags: [resource, dux, dux/api]
status: mature
sources: [".raw/dux/30-api/api-overview.md", ".raw/dux/30-api/openapi.yaml", ".raw/dux/30-api/application-api.md", ".raw/dux/30-api/public-data-api.md", ".raw/dux/30-api/events-webhooks.md", ".raw/dux/20-architecture/architecture-diagrams.md"]
related: ["[[Dux]]", "[[Dux Product Guide]]", "[[Dux Feature Reference]]", "[[Dux AI Safety Guide]]", "[[Dux Taxonomy & Catalogs]]"]
---

# Three APIs, One Product: Inside the Dux Wire Contract

### Why a security-assessment platform needs three separate REST planes, a query language of its own, and a webhook queue it had to defend in writing against a stale spec

Navigation: [[Dux]] | [[Dux Feature Reference]] | [[Dux Taxonomy & Catalogs]]

Most products get away with one API. Dux needed three — and keeping them straight turned out to be one of the more consequential documentation decisions in the whole corpus, because a credential valid on one plane is explicitly rejected on the others, regardless of scope. Get that wrong in an integration and you don't get a slow failure; you get a hard 401 in a customer's CI pipeline. This reference exists to make the boundary impossible to miss.

Before the endpoint tables, one authority note that governs everything below: **`openapi.yaml` in the source repository is a draft skeleton — it inventories paths, auth, and limits but is not yet the wire authority.** It was added 2026-07-09 specifically to resolve blind spot BS-17a, "OpenAPI declared canonical but does not exist in the repo." Best practice here is contract-first: the file **moves to the API service repo at build start**, gains Spectral lint and contract tests in CI, and only then becomes the pinned `v1.0.0` wire authority for `/v1/*`. Until that happens, the prose contracts in this reference are canonical, and the skeleton must not be consumed by integrators. It has also already been corrected once in its short life: as of 2026-07-12 it stopped describing "two planes" and gained the management plane's paths and auth scheme, which it had omitted entirely.

> **`openapi.yaml` is a draft skeleton, not the wire authority.** It inventories the paths, auth, and limits of all three planes. It moves to the API service repo at build start — contract-first, Spectral-linted, contract-tested. Until then, the prose contracts in this folder are canonical.

---

## 1. The Three REST Planes

Dux exposes three separate REST planes. **Do not conflate them.** A credential valid on one plane is explicitly rejected on the others, regardless of scope.

| Plane | Path prefix | Auth | Ship gate | Public OpenAPI |
|---|---|---|---|---|
| **Application API** | `/dashboard/*`, `/research/*`, `/assessments/*`, `/cves/*`, `/connectors/*`, `/chat/*`, `/webhooks/configure`, `/vulnerability-instances/*` (acknowledgment) | Bearer JWT, `aud=api.dux.io` | Gate 1 | No — internal Redoc from Week 8 |
| **Public Data API** | `/v1/custom-metrics*`, `/v1/vulnerability-instances/{cve_id}`, `/v1/cve-research` | Bearer API key (`agt_…`, data scopes) | Seed trigger | Yes — `api.dux.io/docs` |
| **Management API** | `/v1/admin/*`, `/v1/agents/*` | Platform-admin JWT | Gate 1, internal | No |

**Path-alias note.** The application DTO tables use the shorthand `/admin/kill-switch`. **The runtime route is `POST /v1/admin/kill-switch`** — it belongs to the management plane, and the gateway rewrites the prefix per environment.

The rejection is not a soft warning either: a Public Data API key is hard-rejected on any Application-plane JWT route, and `POST /v1/agents` (Management plane) authenticates with platform-admin JWT only — it cannot be called with a Public Data API key at all, regardless of scopes.

---

## 2. Versioning

Three planes, three different relationships to the word "version" — and they are deliberately not unified, because unifying them would make one plane's stability promise a lie about another's.

| Surface | Version | Breaking-change policy |
|---|---|---|
| Public Data API | `v1.0.0`, pinned 2026-06-26 | **Additive only within v1.** A breaking change means `/v2`, with a 90-day overlap |
| Application API | unversioned | Breaking changes ship behind feature flags, with a changelog. Not published on `api.dux.io/docs` |
| Management API | `/v1/admin/*`, `/v1/agents/*` — `/v1` is a fixed path token, not a semantic version | Evolves independently of the Public Data API's `v1.0.0`/`/v2` policy, despite the shared `/v1` segment — the two are not to be conflated |
| SSE event schemas | `2026.06` | New event types are additive. **A field removal requires a version bump** |
| Webhook payloads | `v1`, HMAC-signed | `Idempotency-Key` required. Deprecated fields honored for 90 days |
| `WorkflowPort` definitions | `ExploitabilityAssessmentWorkflow` v1 | Gated by `patched()` |

---

## 3. Authentication

### Application and Management Planes

Bearer JWT from Better Auth, with `aud` enforced. 60-minute access token; 7-day refresh, with rotation (ADR-001).

**Security schemes (from `openapi.yaml`):**

```yaml
bearerJwt:
  type: http
  scheme: bearer
  bearerFormat: JWT
  description: "Application plane. aud=api.dux.io; wrong aud → 401. Access 60 min."

platformAdminJwt:
  type: http
  scheme: bearer
  bearerFormat: JWT
  description: "Management plane (/v1/admin/*, /v1/agents/*). Distinct, higher-privilege scheme from bearerJwt."
```

The `platformAdminJwt` scheme is itself a fix, not an original design: it was added 2026-07-12 because the kill-switch and agent-registry endpoints had previously inherited the default application `bearerJwt` with no explicit scope — a gap that meant a compromised application-plane JWT could, in principle, reach a management-plane action never intended for it.

### Public Data API

Bearer API key, shaped `agt_<8-char-prefix>_<32-char-secret>`. **Only the SHA-256 hash is stored; the plaintext is shown once.**

```yaml
apiKey:
  type: http
  scheme: bearer
  description: "Public data plane. API key agt_<prefix>_<secret> (SHA-256 stored), data scopes."
```

**Scopes:** `custom_metrics:read`, `vulnerability_instances:read`, `cve_research:write`.

**Errors.** A `422` returns the same `DuxError` envelope used everywhere else: `VALIDATION_FAILED`, with `details: [{field, message}]`.

---

## 4. Authorization Role Matrix

Authentication alone does not authorize every Application-plane endpoint — the tenant `admin` / `member` / `viewer` role (`USER.role`) gates each one (D-45). `platform_admin` is a distinct concept — Dux internal staff, not a tenant role — and always satisfies Management-plane auth separately.

| Endpoint | admin | member | viewer |
|----------|:-----:|:------:|:------:|
| `GET /dashboard/home` (`/stream`), `GET /research/dashboard` (`/stream`) | ✓ | ✓ | ✓ |
| `GET /assessments/{id}` (`/trace`, `/replay`), `GET /cves/{id}/detail` | ✓ | ✓ | ✓ |
| `GET /assets/{id}/context`, `GET /controls/refinements` | ✓ | ✓ | ✓ |
| `GET /research/schedule`, `GET /webhooks/deliveries`, `GET /chat/sessions/{id}/stream` | ✓ | ✓ | ✓ |
| `POST /research/queue`, `POST /research/schedule` | ✓ | ✓ | — |
| `POST /vulnerability-instances/{id}/acknowledgments` | ✓ | ✓ | — |
| `POST /chat/sessions/{id}/hitl-response` | ✓ | ✓ | — |
| `POST /remediation-tickets` — unattended by default (create + route), but lower blast-radius than the two below | ✓ | ✓ | — |
| `POST /webhooks/deliveries/{id}/replay` | ✓ | ✓ | — |
| `POST /mitigations`, `POST /fast-actions` — unattended by default, live security-posture writes | ✓ | — | — |
| `POST /connectors/aws/sync`, `POST /connectors/csv/upload` | ✓ | — | — |
| `POST /webhooks/configure` | ✓ | — | — |
| `POST /tenants/{id}/export`, `DELETE /tenants/{id}` | ✓ | — | — |

`member` covers read plus day-to-day analyst operation (triggering research, acknowledging findings, HITL approve/deny, remediation tickets); tenant configuration and destructive/high-blast-radius actions (connector setup, webhook configuration, export, tenant deletion, the two highest-blast-radius unattended writes) stay `admin`-only. `viewer` is read-only everywhere. Management-plane and Public Data API endpoints are unaffected — they authenticate on platform-admin JWT and API-key scopes respectively, not `USER.role`.

---

## 5. Rate Limits

Two distinct rate-limit tables, enforced per plane — the original corpus stated both without reconciling them, which is exactly the trap conflating the planes creates. They govern genuinely different traffic shapes: one is interactive dashboard and agent traffic, the other is programmatic batch access.

### Application API — Interactive Dashboard and Agent Traffic

| Tier | req/min |
|------|---------|
| Starter | 1,000 |
| Professional | 5,000 |
| Enterprise | 10,000 |

### Public Data API — Programmatic `/v1/*`

| Tier | req/min |
|------|---------|
| Starter | 60 |
| Professional | 300 |
| Enterprise | negotiated |

### Response Headers

Every response on a rate-limited plane carries `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` (IETF `RateLimit-*` draft), so a well-behaved client can throttle itself proactively rather than only reacting to a `429`.

A 429 (RFC 6585) additionally carries `Retry-After` (RFC 9110 §10.2.3, integer seconds). The tenant admin is notified if the queue stalls for more than 5 minutes.

**`RateLimited` response component (from `openapi.yaml`):**

```yaml
responses:
  RateLimited:
    description: "429 — plane-specific limits (D-6): app 1,000/5,000/10,000 req/min; public 60/300/negotiated req/min. DuxErrorCode BUDGET_EXCEEDED for agent budgets."
```

Flood control sits at the Cloudflare edge. Identity-aware plan and tenant limits are enforced **post-auth**, by `@nestjs/throttler` + Valkey. **Concurrent SSE streams are capped at 5 per `user_id`.**

---

## 6. Domains

| Domain | Purpose |
|--------|---------|
| `dux.io` | Marketing |
| `app.dux.io` | Customer application |
| `api.dux.io` | API (and `/docs` for public documentation) |
| `staging.dux.io` | Staging environment |
| `status.dux.io` | System status — **launch blocker** (GCIS §E) |
| `trust.dux.io` | Trust center — **launch blocker** (GCIS §E) |
| `docs.dux.io` | Documentation |
| `security@dux.io` | Security contact |

**`trust.dux.io` and `status.dux.io` are launch blockers.** They must not be linked from marketing until they return HTTP 200.

---

## 7. Error Handling

Every error on every plane — REST, SSE, or webhook — funnels through one shared envelope. That's a deliberate constraint: an integrator who has learned to parse `DuxError` once never has to relearn error handling for a different transport.

### DuxError Envelope

Shared across REST, SSE, and webhooks:

```yaml
schemas:
  DuxError:
    type: object
    description: "Shared error envelope — DuxErrorCode: AGENT_TIMEOUT 504, CONTEXT_EXHAUSTED 422, BUDGET_EXCEEDED 429, GOVERNANCE_BLOCKED 403, INSUFFICIENT_DATA 422 (asset_gap|intel_gap|context_limit), VALIDATION_FAILED 422 (details: [{field, message}]). Full copy rules: taxonomy §7."
    properties:
      code: { type: string }
      message: { type: string }
      reason: { type: string }
```

The `VALIDATION_FAILED` code itself replaced an orphaned name: `openapi.yaml` notes the shared 422 shape retired an earlier `HTTPValidationError` name that had drifted out of use, standardizing on `VALIDATION_FAILED` with a `details: [{field, message}]` payload as the one request-validation-failure shape used corpus-wide.

### DuxError Codes

| Code | HTTP | Notes |
|------|------|-------|
| `AGENT_TIMEOUT` | 504 | |
| `CONTEXT_EXHAUSTED` | 422 | |
| `BUDGET_EXCEEDED` | 429 | Agent budgets |
| `GOVERNANCE_BLOCKED` | 403 | |
| `INSUFFICIENT_DATA` | 422 | `asset_gap` / `intel_gap` / `context_limit` |
| `VALIDATION_FAILED` | 422 | `details: [{field, message}]` — the single request-validation-failure shape corpus-wide |

`VALIDATION_FAILED` is the single 422 shape used everywhere. The CSV-upload `missing_column` / `invalid_os_family` / `duplicate_hostname` reasons and the DQL `DQL_EXPRESSION_TOO_COMPLEX` code are `VALIDATION_FAILED` instances, not a separate error-code namespace.

### Application Error Classes

| Class | HTTP |
|-------|------|
| `TenantIsolationError` | 403 |
| `ConnectorSyncError` | 502 |
| `AgentBudgetExceeded` | 429 |
| `AssessmentDedupConflict` | 409 |

---

## 8. Application API — Endpoints

**Plane:** Application — Bearer JWT, `aud=api.dux.io`.

This shape resolves the remaining half of OI-31: this API is CVE-lookup-and-assessment-centric (`GET /cves/{id}/detail`, `GET /assessments/{id}`) — not a generic "submit a scan, poll for a report" shape some exposure-management frameworks assume. That's a deliberate scoping decision, not a documentation gap: Dux's unit of work is a CVE against a live World Model, continuously re-assessed (US-021), not a point-in-time scan job. A framework audit expecting the scan-and-poll shape will not find it here.

| Endpoint | Story | Notes |
|----------|-------|-------|
| `GET /dashboard/home`, `GET /dashboard/home/stream` | US-012 | |
| `GET /research/dashboard`, `GET /research/dashboard/stream` | US-010 | |
| `POST /research/queue` | US-008, US-010 | |
| `GET /assessments/{id}`, `GET /assessments/{id}/trace`, `GET /assessments/{id}/replay` | US-001, US-017 | |
| `GET /cves/{id}/detail` | US-011 | |
| `GET /assets/{id}/context` | US-002 (FR-020) | |
| `GET /controls/refinements` | US-005 (FR-019) | |
| `POST /mitigations` — **unattended by default** | US-004 | |
| `POST /fast-actions` — **unattended by default** | US-016 (FR-011) | |
| `POST /remediation-tickets` — **unattended by default (create + route)** | US-018 (FR-013) | |
| `POST /research/schedule`, `GET /research/schedule` | US-021 (FR-025) | |
| `POST /connectors/aws/sync`, `POST /connectors/csv/upload` | US-013 | |
| `POST /vulnerability-instances/{id}/acknowledgments` | US-023 | |
| `POST /webhooks/configure` | US-014 | |
| `POST /tenants/{id}/export`, `DELETE /tenants/{id}` | US-014 | |
| `POST /v1/admin/kill-switch` | US-014 (management plane) | |
| Chat SSE + POST | US-008 | |

The `openapi.yaml` skeleton fleshes out a handful of these with response-level detail worth carrying forward: `GET /assessments/{id}` documents a 404 on cross-tenant access — **never a 403**, so a tenant probing another tenant's assessment ID learns nothing about whether it exists. `POST /research/schedule` responds `201 created` on success, alongside the `GET` variant that simply returns the schedule. And `POST /v1/cve-research` on the Public Data plane changed its success code from `201` to `202` on 2026-07-21, aligning it with the async-enqueue semantics `POST /research/queue` already used — a fix worth knowing about if an older integration still checks for `201`.

---

## 9. Application API — Key DTOs

### `GET /dashboard/home` → `DashboardHomeDto`

| Field | Shape |
|-------|-------|
| `exposure_summary` | `ExposureDonutDto` — the four states, plus the delta against the prior period |
| `vulnerability_reduction` | `VulnerabilityReductionDto` |
| `queue_summary` | `completed` / `in_research` / `backlog` |
| `needs_attention` | `CveSummaryDto[]` → links through to US-011 |
| `connector_health` | `ConnectorHealthDto[]` — AWS `last_sync_at`, `asset_count`, and `stale_warning` when >24 h |
| `as_of` | ISO 8601 |

**SSE:** `queue_update` carries `QueueSummaryDto` + `as_of`. Target latency: <5 s.

### `GET /research/dashboard` → `ResearchDashboardDto`

| Field | Shape |
|-------|-------|
| `vulnerability_reduction` | as above |
| `calendar` | `ResearchCalendarDayDto[7]` — per-day completed / research / backlog, plus tooltip user lines |
| `cve_rows` | `CveQueueRowDto[]` — severity, status icon, tags. **The sort elevates Mitigation Required** |
| `view_mode` | `by_cve` / `by_asset` / `by_instance` |

**SSE:** `queue_row_update` patch. Target latency: <1 s.

### `POST /research/queue`

This is the front door for research, and it is worth understanding in some depth, because the same underlying capability is exposed a second way on the Public Data plane (see §11 below), and the differences between the two are deliberate rather than accidental drift.

**Request body (`ResearchQueueRequest`):**

```yaml
schemas:
  ResearchQueueRequest:
    type: object
    description: "Either cve_id (^CVE-\\d{4}-\\d{4,}$) or natural_language."
    properties:
      cve_id: { type: string, pattern: "^CVE-\\d{4}-\\d{4,}$" }
      natural_language: { type: string }
```

**Response:** `{assessment_id: uuid, status: queued | deduplicated, queue_position: int}`

**Idempotent**, via `AssessmentDeduplicationService`. `openapi.yaml` additionally declares a `409 AssessmentDedupConflict` response; the rule distinguishing when a duplicate submission returns the soft `202`/`deduplicated` body versus the hard `409` is not yet stated — Engineering's to specify.

This is also the investigation-trigger endpoint for the Agentic RAG loop (`agenticRAGWorkflow`, ADR-020 R2) — a queued assessment is what the workflow investigates; there is no separate investigation-trigger endpoint (D-56).

### `GET /assessments/{id}` → `AssessmentDto`

Fields: `id`, `tenant_id`, `finding_id`, `cve_id`, `status`, `confidence_score`, `completed_at`. Cross-tenant returns 404.

### `GET /assessments/{id}/trace` → `AssessmentTraceDto`

| Field | Shape |
|-------|-------|
| `assessment_id` | from `EXPLOITABILITY_ASSESSMENT` |
| `steps[]` | `ASSESSMENT_REASONING_STEP` — `step_order`, `step_type`, `content`, `source_refs` |
| `code_artifact` | `{language, source_code}` |
| `execution_results` | `null` \| `ExecutionResultDto`. **Populated at Gate 1** via the microVM; `null` **only** when the sandbox is disabled through the kill path |
| `exported_at` | timestamp |

### `GET /assessments/{id}/replay` → `AssessmentReplayDto`

The `replay_trace_id` capability (observability-slo §2), keyed by the assessment's `trace_id`. Bearer JWT, `aud=api.dux.io` — same as the rest of this section. This endpoint was the last of the three assessment reads to make it into the OpenAPI skeleton, added 2026-07-21 in a path-inventory sync — the capability itself predates that addition.

| Field | Shape |
|-------|-------|
| `assessment_id` | uuid |
| `trace_id` | the shared OTel/Langfuse `trace_id` |
| `reasoning_steps[]` | `{step_order, step_type, content, source_refs, timestamp}` — the replayed span tree, in original order |
| `tool_calls[]` | `{tool_name, tenant_scope, input, output, timestamp}` — redacted per the sanitization rules in observability-slo §2 |
| `memory_access[]` | `{operation: read \| write, key, timestamp}` |
| `reconstructed_at` | timestamp — read-time reconstruction, not a stored artifact |

Cross-tenant returns 404, consistent with `GET /assessments/{id}`.

### `GET /cves/{id}/detail` → `CveDetailDto`

Via `CVEDetailQuery` / `ExposureProjection`: header (CVSS, EPSS, KEV), `risk_groups`, `flow_bar`, `factor_cards[]`, `assets[]`, `attack_paths[]`.

The three taxonomies stay in separate DTO subtrees (exposure-analysis US-011). Optional `?projection=exposure|protection|action_cards`, defaulting to `exposure`.

### CVE Read Projections

In `packages/api/projections/`:

| Projection | Story | Output | Budget |
|-----------|-------|--------|--------|
| `ExposureProjection` | US-011 | severity, groups, attack paths, AWS evidence | NFR-013: p95 <500 ms |
| `ProtectionProjection` | US-003 | the four-state summary, plus vendor control panels | Gate 1 |
| `ActionCardProjection` | US-004 | mitigation steps, vendor deep-links, `canonical_action_id` | Gate 1 |

A shared `CVEDetailQuery` base fetches the CVE and the tenant scope once; each projection adds its own joins.

### `GET /assets/{id}/context` → `AssetContextDto`

(US-002, FR-020, Gate 1, resolves OI-16). Reuses the exposure `assets[]` asset fields (§3 `ExposureProjection`) as its base, with `context_type = runtime` and one block per evidence source from security-stepper US-002:

| Field | Shape |
|-------|-------|
| `asset_id`, `hostname`, `asset_type`, `os_family` | denormalized `ASSET` fields (§2 data-model) |
| `context_type` | `"runtime"` — the only value in Phase 1 |
| `endpoint_state` | nullable `{connector: "crowdstrike", status, prevention_policy, last_seen_at}` — `status` is `live` \| `insufficient_data` \| `connector_degraded` |
| `cloud_context` | nullable `{connector: "aws", vpc_id, subnet_id, security_group_reachability}` — `security_group_reachability` reuses the public-API `network_exposure` verdict enum (`internet` \| `external` \| `internal` \| `unreachable`) |
| `runtime_telemetry` | nullable `{connector: "splunk", listening_processes: string[], last_seen_at}` — the process-not-listening evidence feeding US-011 |
| `identity_context` | nullable `{connector: "servicenow" \| "entra-id", last_login_at, owner_team, certainty}` — shares shape with `OwnershipInferenceDto` (US-007) |
| `policy_context` | nullable `{connector: "intune", status: "connector_degraded", gate: "Gate 3 / W2"}` — renders the connector-degraded empty state until Intune ships |
| `insufficient_data_reason` | nullable — `asset_gap` \| `intel_gap` \| `context_limit`, set when every block above is null or stale |
| `as_of` | ISO 8601 |

Any block whose source connector is stale or absent is `null`, never fabricated — same rule as every other vendor-sourced field in this API (§1 `connector_health`).

### Chat SSE

`GET /chat/sessions/{id}/stream`, plus `POST /chat/sessions/{id}/hitl-response`.

**Events:** `query`, `response`, `citation`, `processing_step`, `prioritization_cards`, `request_research_ack`, `hitl_request`.

Contract: the Kill Switch. The same `hitl_request`/`hitl_response` pair carries both HITL surfaces — §7's vendor-write approvals and §7a's Agentic RAG investigation-confidence-gate approvals (D-56) — they're the same `WorkflowPort` HITL-signal primitive on two different workflow types, not two endpoints; `canonicalActionId` is absent on a §7a request, distinguishing it from a §7 vendor-write request.

**SSE schema version: `2026.06`.** New event types are additive. A field removal bumps the version.

**Reconnection** replays from `Last-Event-ID` against `chat_session_events`, over a 1-hour window. Beyond that window, the client falls back to a state snapshot.

### Connectors

`POST /connectors/aws/sync` → `{sync_id, status, assets_ingested, started_at}`.

`POST /connectors/csv/upload` is multipart; errors are `missing_column`, `invalid_os_family`, `duplicate_hostname`; limits are 50 K rows, UTF-8, unique `(tenant_id, hostname)`.

### Vendor Writes

`POST /mitigations`, `POST /fast-actions`, `POST /remediation-tickets`. All three require a client-supplied `Idempotency-Key` header, deduplicated the same way `POST /research/queue`'s `AssessmentDeduplicationService` works: a replayed key returns the original `202`/execution record rather than re-executing — required because each triggers a real vendor-side action (`endpoint.isolate`, `network.blocklist_add`, `policy.deploy_device_config`, `patch.deploy_special_devices`, `ticket.create_remediation`) with no other client-facing retry-safety mechanism.

**`POST /remediation-tickets`:** `{finding_id | cve_id, ...}` → `202 {ticket_id, status: routed, canonical_action_id: "ticket.create_remediation"}`, executed via the same `VendorActionGate` pattern as `POST /mitigations`/`POST /fast-actions`.

### Kill Switch Propagation

`POST /v1/admin/kill-switch` (management plane): `{level: L1 | L2 | L3 | L4, tenant_id?, session_id?, reason}`. **Propagates in under 5 s p99** (NFR-005). `DELETE /v1/admin/kill-switch/{id}` deactivates.

### Acknowledgment Endpoints

`POST /vulnerability-instances/{id}/acknowledgments`: `{reason, expires_at?}` → `{acknowledgment_id, is_acknowledged: true, expires_at?}`.

`DELETE …/acknowledgments/{ack_id}` revokes — the plural collection name matches the `acknowledgment_id` field it returns, consistent with this API's other resource names. That plural naming is itself a 2026-07-21 rename, made specifically so the collection name would match the field it returns.

The public read of `is_acknowledged` is true only for an **active** acknowledgment — not revoked, and with `expires_at` either null or in the future.

---

## 10. Public Data API

**Plane:** Public Data — Bearer API key (`agt_…`, data scopes). Ships at the Seed public-API trigger, **not** in Phase 1.

**Authority:** OpenAPI 3.1 is the source of truth. The contract is pinned at `v1.0.0` (2026-06-26). Auth mechanics, rate limits, and the versioning policy shared across all three planes are covered once in [api-overview.md §3–4](#3-authentication).

**Every endpoint is a GET, except `POST /v1/cve-research`.** There is no create, update, or delete for metrics or instances in v1.

### `GET /v1/custom-metrics` (FR-021, US-022)

A paginated list.

**Query parameters:**

| Parameter | Constraint |
|-----------|-----------|
| `sort` | column list; a `-` prefix means descending. Validated against `CustomMetricItem`'s own field list — an unresolvable column returns the same 422 shape as an unresolvable DQL field |
| `page` | ≥1, default 1 |
| `size` | 1–200, default 10 |
| `entity_type` | an `EntityType` (see §12) |
| `is_active` | boolean |
| `dashboard_id` | — |
| `search` | max length 255 |

**200** → `PaginatedResponse[CustomMetricItem]` — `items`, `total`, `page`, `size`, `pages`.

**`CustomMetricItem` required fields:** `id`, `display_name`, `description`, `entity_type`, `dql_filter`, `group_by`, `is_active`, `dashboard_id`, `ordinal`, `created_by`, `created_at`, `updated_at`. Optional: `dashboard_ids[]`.

### `GET /v1/custom-metrics/{id}`

→ `CustomMetricItem`.

### `GET /v1/custom-metrics/{id}/data`

→ `CustomMetricDataResponse`: `metric_id`; `data` (a `CustomMetricDataPoint[]` of `timestamp`, `group_by_values` — a nullable map — and a numeric `value`); `sources` (nullable).

**Time-range bounded**, since a metric's history can grow unbounded: `from`/`to` timestamps, plus the same cursor shape as §4. The exact page-size ceiling here is Engineering's to size against expected data-point volume — deliberately not invented in this reference.

### `GET /v1/vulnerability-instances/{cve_id}` (FR-022, US-024)

Path pattern: `^CVE-\d{4}-\d{4,}$`.

**Cursor pagination:** `cursor` (opaque), `limit` (1–5000, default 3000), `expand` (only `asset` is accepted).

**200** → `CursorPaginatedResponse[VulnerabilityInstanceV1Response]` — `items`, `has_more`, `next_cursor`.

### `VulnerabilityInstanceV1Response`

| Field | Required? | Notes |
|-------|-----------|-------|
| `id`, `cve_id`, `sources`, `last_seen_at`, `is_acknowledged`, `asset_id`, `external_uids` | required | — |
| `exploitability_status` | optional | `potentially_exploitable` \| `not_exploitable` \| `partially_mitigated` \| `null`. **Null means unresearched** |
| `remediations` | optional | nullable `string[]` |
| `network_exposure` | optional | Verdict — `internet` \| `external` \| `internal` \| `unreachable`; nullable |
| `asset` | optional | the full `AssetV1Response`, **only** when `expand=asset`. Its `type` is `device` \| `cloud_compute` |

### `POST /v1/cve-research` (FR-023, US-024)

**Request body (`CveResearchV1Request`):** `cve_ids`, 1–50 entries, each matching `^CVE-\d{4}-\d{4,}$`.

**202** → an array of `CveResearchV1Item`, each carrying `status: backlog | completed` — matching `POST /research/queue`'s async-enqueue semantics. As noted in §8, this success code moved from `201` to `202` on 2026-07-21 to make that alignment explicit.

**Dedup.** Shares the same per-`cve_id` dedup mechanism as `POST /research/queue` (not per-batch, since batches can overlap) — a resubmitted `cve_id` inside a batch does not enqueue a duplicate research run.

### Pagination Limits

| Parameter | Min | Max | Default |
|-----------|-----|-----|---------|
| `cve_ids` (batch) | 1 | 50 | — |
| `size` (custom-metrics) | 1 | 200 | 10 |
| `page` | 1 | — | 1 |
| `limit` (vulnerability-instances) | 1 | 5000 | 3000 |

---

## 11. The Application ↔ Public Bridge (Research Queue)

The same capability is exposed differently on each plane, and the differences are deliberate:

| Concern | Application (Phase 1, JWT) | Public Data (Seed, API key) |
|---------|---------------------------|------------------------------|
| Enqueue | `POST /research/queue` — a single `cve_id`, **or** `natural_language` | `POST /v1/cve-research` — a batch of `cve_ids` (1–50) **only** |
| Response | `{assessment_id, status: queued \| deduplicated, queue_position}` | an array of `CveResearchV1Item` (`status: backlog \| completed`) |
| In-progress state | `QueueSummaryDto.in_research` | **no `in_research`** — poll `backlog` until `completed` |

**Natural-language enqueue has no public-v1 equivalent.** API consumers supply CVE IDs matching `^CVE-\d{4}-\d{4,}$`.

---

## 12. DQL (Dux Query Language)

Custom metrics needed a way for tenants to define their own filtered views over World Model entities, without opening a door to arbitrary SQL. The answer is DQL: a tenant-scoped filter expression, stored in `CustomMetricItem.dql_filter` and evaluated against World Model entities.

### `EntityType` Enum

Closed set (taxonomy §2):

```yaml
EntityType:
  type: string
  enum: [device, user, label, finding, cloud_compute, vulnerability_instance, cve, mitigation]
```

`group_by` defines the aggregation dimensions for `CustomMetricDataPoint.group_by_values`.

Custom metrics are read-only over public v1. Configuration happens in the tenant-admin UI (US-022), at Seed.

### Grammar (Resolves OI-15)

DQL is a flat, single-entity boolean filter expression — **no joins, no subqueries.** `dql_filter` is stored as the raw expression string; `entity_type` fixes which World Model entity's columns are addressable.

```
expression   := clause (( "AND" | "OR" ) clause)*
clause       := "(" expression ")" | comparison
comparison   := field operator value
field        := identifier ("." identifier)*        // dotted path into jsonb columns
operator     := "=" | "!=" | ">" | ">=" | "<" | "<=" | "IN" | "NOT IN" | "CONTAINS" | "IS NULL" | "IS NOT NULL"
value        := string | number | boolean | "(" value ("," value)* ")"   // parenthesized list, for IN/NOT IN
```

- **Precedence:** `AND` binds tighter than `OR`; explicit parentheses override. No operator overloading — `=` is never a fuzzy match.
- **`CONTAINS`** is the only substring/array-membership operator, valid only on `string[]` and `text` columns (e.g. `sources CONTAINS "wiz"`); it does not accept a field on the right-hand side.
- **`field`** must resolve to a real column or a `jsonb` path on the `entity_type`'s backing table — an unknown field is a 422 at save time, not a silent empty result.
- **Grammar limits:** max expression length 2,000 characters; max nesting depth 5; max 20 clauses. Exceeding any limit is a 422 `DQL_EXPRESSION_TOO_COMPLEX`.
- **No arbitrary code execution** — the parser produces an AST that compiles to a parameterized SQL `WHERE` clause scoped by RLS; it never interpolates the raw string.

**Example:** `severity >= 7 AND (state = "exploitable" OR state = "under_research") AND asset.has_public_ip = true`

**Validation.** `POST`/`PUT` on a `CustomMetricItem` runs the parser synchronously; a parse or field-resolution failure returns 422 with the offending token's character offset. Valid grammar is cached as a pre-compiled AST alongside the raw string, so `GET /v1/custom-metrics/{id}/data` never re-parses on read.

This grammar is normative; the OpenAPI 3.1 `description` field on `dql_filter` links here rather than restating it.

---

## 13. Mapping and Exposure Rules

- **`confidence_score` is not exposed on v1.** Consumers get `exploitability_status` only.
- This surface maps to US-010's `view_mode=by_instance`, and to the Exposure Analysis asset rows.
- The public read of `is_acknowledged` reflects the application-plane acknowledgment (US-023).

**Publishing checklist (Seed, AI-142):** OpenAPI `securitySchemes` (Bearer API key); rate-limit `429` schemas; the scope enum; and a Security review **before** the trigger fires.

---

## 14. Events and Webhooks

### Event Catalog

The source of truth is the event catalog. Do not add an event type outside it.

#### Event Semantics

| Event family | Fires when |
|--------------|-----------|
| `finding.*` | a scanner row is created, updated, or deleted on World Model ingest |
| `vulnerability_instance.*` | the per-asset CVE projection changes — `exploitability_status`, `network_exposure`, `is_acknowledged`, or `last_seen_at` |
| `assessment.completed` | an application-layer assessment finishes |

**Public API consumers should subscribe to `cve_research.completed` and `vulnerability_instance.*`** rather than polling.

#### Gate-1 Event Set

| Event | Gate |
|-------|------|
| `assessment.requeued` (ADR-016) | Gate 1 |
| `attack_path.validated` | Gate 1 |
| `ownership.inferred` | Gate 1 |
| `control_asset_mapping.updated` | Gate 1 |
| `mitigation.executed` / `.blocked` | **Gate 1, unattended by default** |
| `remediation.ticket_created`, `ticket.*` | **Gate 1 — create + route, unattended by default** |
| `hitl_request` (SSE) | Gate 1 — **anomaly escalation only** |
| `cve_research.*`, `custom_metric.updated`, `vulnerability_instance.updated` | Seed |

### Outbound Delivery

Per ADR-005 R2: a **NATS JetStream-backed durable queue**. **Not BullMQ** — the legacy TRD's "Webhooks (outbound)" table specified BullMQ on Redis DB1, with `prefix: 'bull:webhook'`. That contradicts ADR-005 (which explicitly rejects BullMQ) and FR-010.

| Component | Specification |
|-----------|---------------|
| Queue | durable `webhook_queue` stream on NATS JetStream. **Not BullMQ** |
| Worker | `WebhookDeliveryWorker`, in `packages/notifications/` |
| Concurrency | max 3 concurrent deliveries per tenant |
| Retry | `delay = base × 2^n + random(0, 1000 ms)` for n = 0..4 (base 1 s → 16 s). **Max 5 attempts** |
| Dead letter | the `webhook_dead_letter` JetStream stream. Alert above 10 failures per tenant per hour. Retained 7 days |
| Replay | `pnpm ops:replay-webhooks --tenant {id}`, or `pnpm admin:webhook replay --tenant {id} --since 1h` (internal ops tooling). Tenant-facing: `GET /webhooks/deliveries` (filterable by status/event type) and `POST /webhooks/deliveries/{id}/replay`, scoped to the tenant's own dead-letter records |
| Signing | `X-Dux-Signature: sha256=HMAC_SHA256(payload, webhook_secret)` (header name consistent with the `X-Impersonate-Tenant` custom-header convention, multi-tenancy); the secret lives in Vault |
| Idempotency | the `Idempotency-Key` header, plus a dedup table. **Consumers must accept it** |
| Circuit breaker | delivery pauses above 10 failures per tenant per hour |

### Webhook Payload Versioning

`v1`, HMAC-signed. **`Idempotency-Key` is required.** Deprecated fields are honored for 90 days. `v1` and the SSE schema's `2026.06` are intentionally decoupled — each transport's version evolves on its own cadence, since the two channels have different consumers and different change tolerances, even where they carry the same underlying event catalog data.

**Payload envelope.** Every webhook body shares a common envelope:

```json
{
  "event_id": "uuid",
  "event_type": "mitigation.executed",
  "occurred_at": "ISO 8601",
  "tenant_id": "uuid",
  "api_version": "v1",
  "data": { "..." }
}
```

Where `data` is the event-specific DTO already defined elsewhere in this corpus for that event's producing entity (e.g. `mitigation.executed`'s `data` is the `VendorActionExecution` record).

Phase-1 assessment webhooks authenticate the same way as their triggering Application-plane request (Bearer JWT), not an API key. The public data-API webhooks — `cve_research.*` and `custom_metric.updated` — use `agt_…` Public Data API keys, from Seed.

### SSE Streams

| Stream | Events | Target latency |
|--------|--------|----------------|
| Chat | `query`, `response`, `citation`, `processing_step`, `prioritization_cards`, `request_research_ack`, `hitl_request` | — |
| Dashboard | `queue_update` | <5 s |
| Research | `queue_row_update` | <1 s |

SSE schema version: `2026.06`. New event types are additive. A field removal bumps the version.

---

## 15. Tenant Export and GDPR Deletion

### `POST /tenants/{id}/export`

Exports a bundle (Parquet default / JSON). Returns `202` — export started.

### `DELETE /tenants/{id}`

GDPR deletion (`GDPRDeletionWorkflow`): 30-day read-only window, 90-day purge. Returns `202` — deletion scheduled.

---

## 16. Management Plane Endpoints

**Plane:** Management — platform-admin JWT. Gate-1, internal.

### Kill Switch

`POST /v1/admin/kill-switch` — `{level: L1 | L2 | L3 | L4, tenant_id?, session_id?, reason}`. Propagation <5 s p99.

`DELETE /v1/admin/kill-switch/{id}` — deactivates.

### Agent Registry

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/agents` | GET | List registered Dux Agent definitions — `AgentDefinition[]` (`agent_id`, `tenant_id`, `identity_ref`, `agent_type`, `credential_id`, `status`). No API key issued by this call — agents authenticate via per-agent session JWT, D-46 |
| `/v1/agents` | POST | Provision agent definition — request `{identity_ref, agent_type, config}`, response `AgentDefinition` |
| `/v1/agents/{id}/autonomy-request` | POST | Request `max_autonomy: autonomous` for an agent — requires tenant-admin approval. Response: approved — AIBOM and agent record updated |
| `/resident-agents/{id}/heartbeat` | POST | Resident agent heartbeat — mTLS preferred, or a signed JWT with `agent_id`/`tenant_id`/`nonce`/`exp<=60s` (Gate 5). May carry `halt: true` to trip the kill switch |

---

## 17. HitL Response Contract

`POST /chat/sessions/{id}/hitl-response` — HITL approve/deny, anomaly-escalation path only since 2026-07-13. The rollbackProcedure URL is required in the payload.

The same `hitl_request`/`hitl_response` pair carries both HITL surfaces: §7's vendor-write approvals and §7a's Agentic RAG investigation-confidence-gate approvals (D-56). They are the same `WorkflowPort` HITL-signal primitive on two different workflow types, not two endpoints; `canonicalActionId` is absent on a §7a request, distinguishing it from a §7 vendor-write request.

---

## 18. CSV Upload Responses

`POST /connectors/csv/upload` — multipart, limits: 50 K rows, UTF-8, unique `(tenant_id, hostname)`.

| Response | Meaning |
|----------|---------|
| `201` | ingested |
| `422` | `missing_column` \| `invalid_os_family` \| `duplicate_hostname` |
| `429` | Rate limited |

---

## 19. Research Queue Comparison

| Concern | Application (JWT) | Public Data (API key) |
|---------|-------------------|----------------------|
| Enqueue | `POST /research/queue`: single CVE ID or natural-language string | `POST /v1/cve-research`: batch of 1–50 CVE IDs only |
| Response | Assessment ID, status (`queued`/`deduplicated`), queue position | Array of per-item result objects (`backlog`/`completed`) |
| In-progress visibility | Live `QueueSummaryDto.in_research` field | None: poll until the item reports `completed` |
| Dedup | Per `cve_id` via `AssessmentDeduplicationService` | Same per-`cve_id` dedup (not per-batch) |
| Natural-language | Supported | Not available |

---

## Sources

- `.raw/dux/30-api/api-overview.md`
- `.raw/dux/30-api/openapi.yaml`
- `.raw/dux/30-api/application-api.md`
- `.raw/dux/30-api/public-data-api.md`
- `.raw/dux/30-api/events-webhooks.md`
- `.raw/dux/20-architecture/architecture-diagrams.md`
