---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [H5]
---

# Application API — Phase-1 DTO Contracts

**Purpose:** the canonical request and response shapes, so Gate-1 work can proceed in parallel. NestJS DTOs implement these. The public OpenAPI ships at seed.

**Plane:** Application — Bearer JWT, `aud=api.dux.io` · **Parents:** BR-002, BR-006.

**Shape, by design (resolves the remaining half of [OI-31](../00-meta/open-items.md)).** This API is CVE-lookup-and-assessment-centric — `GET /cves/{id}/detail`, `GET /assessments/{id}` — not a generic "submit a scan, poll for a report" shape some exposure-management frameworks assume. That's a deliberate scoping decision, not a doc gap: Dux's unit of work is a CVE against a live World Model, continuously re-assessed (US-021), not a point-in-time scan job. A framework audit expecting the latter shape will not find it here.

## 1. Endpoints

| Endpoint | Story |
|----------|-------|
| `GET /dashboard/home`, `GET /dashboard/home/stream` | US-012 |
| `GET /research/dashboard`, `GET /research/dashboard/stream` | US-010 |
| `POST /research/queue` | US-008, US-010 |
| `GET /assessments/{id}`, `GET /assessments/{id}/trace`, `GET /assessments/{id}/replay` | US-001, US-017 |
| `GET /cves/{id}/detail` | US-011 |
| `GET /assets/{id}/context` | US-002 (FR-020) |
| `GET /controls/refinements` | US-005 (FR-019) |
| `POST /mitigations` — **unattended by default** | US-004 |
| `POST /fast-actions` — **unattended by default** | US-016 (FR-011) |
| `POST /remediation-tickets` — **unattended by default (create + route)** | US-018 (FR-013) |
| `POST /research/schedule`, `GET /research/schedule` | US-021 (FR-025) |
| `POST /connectors/aws/sync`, `POST /connectors/csv/upload` | US-013 |
| `POST /vulnerability-instances/{id}/acknowledgments` | US-023 |
| `POST /webhooks/configure` | US-014 |
| `POST /tenants/{id}/export`, `DELETE /tenants/{id}` | US-014 |
| `POST /v1/admin/kill-switch` | US-014 (management plane) |
| Chat SSE + POST | US-008 |

## 2. Key DTOs

**`GET /dashboard/home` → `DashboardHomeDto`**

| Field | Shape |
|-------|-------|
| `exposure_summary` | `ExposureDonutDto` — the four states, plus the delta against the prior period |
| `vulnerability_reduction` | `VulnerabilityReductionDto` |
| `queue_summary` | `completed` / `in_research` / `backlog` |
| `needs_attention` | `CveSummaryDto[]` → links through to US-011 |
| `connector_health` | `ConnectorHealthDto[]` — AWS `last_sync_at`, `asset_count`, and `stale_warning` when >24 h |
| `as_of` | ISO 8601 |

SSE: `queue_update` carries `QueueSummaryDto` + `as_of`. Target <5 s.

**`GET /research/dashboard` → `ResearchDashboardDto`**

| Field | Shape |
|-------|-------|
| `vulnerability_reduction` | as above |
| `calendar` | `ResearchCalendarDayDto[7]` — per-day completed / research / backlog, plus tooltip user lines |
| `cve_rows` | `CveQueueRowDto[]` — severity, status icon, tags. **The sort elevates Mitigation Required** |
| `view_mode` | `by_cve` / `by_asset` / `by_instance` |

SSE: `queue_row_update` patch. Target <1 s.

**`POST /research/queue`** — body is `{cve_id: "^CVE-\d{4}-\d{4,}$"}` **or** `{natural_language: string}` → `{assessment_id: uuid, status: queued | deduplicated, queue_position: int}`. **Idempotent**, via `AssessmentDeduplicationService`. `openapi.yaml` additionally declares a `409 AssessmentDedupConflict` response for this same endpoint; the rule distinguishing when a duplicate submission returns the soft `202`/`deduplicated` body versus the hard `409` is not yet stated — Engineering's to specify, not invented here. **This is also the investigation-trigger endpoint for the Agentic RAG loop (`agenticRAGWorkflow`, [ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval)) — a queued assessment is what the workflow investigates; there is no separate investigation-trigger endpoint (D-56).**

**`GET /assessments/{id}` → `AssessmentDto`** — `id`, `tenant_id`, `finding_id`, `cve_id`, `status`, `confidence_score`, `completed_at`. **Cross-tenant returns 404.**

**`GET /assessments/{id}/trace` → `AssessmentTraceDto`**

| Field | Shape |
|-------|-------|
| `assessment_id` | from `EXPLOITABILITY_ASSESSMENT` |
| `steps[]` | `ASSESSMENT_REASONING_STEP` — `step_order`, `step_type`, `content`, `source_refs` |
| `code_artifact` | `{language, source_code}` |
| `execution_results` | `null` \| `ExecutionResultDto`. **Populated at Gate 1** via the microVM; `null` **only** when the sandbox is disabled through the kill path |
| `exported_at` | timestamp |

**`GET /assessments/{id}/replay` → `AssessmentReplayDto`** — the `replay_trace_id` capability ([observability-slo §2](../60-operations/observability-slo.md)), keyed by the assessment's `trace_id`. Bearer JWT, `aud=api.dux.io` — same as the rest of this section.

| Field | Shape |
|-------|-------|
| `assessment_id` | uuid |
| `trace_id` | the shared OTel/Langfuse `trace_id` |
| `reasoning_steps[]` | `{step_order, step_type, content, source_refs, timestamp}` — the replayed span tree, in original order |
| `tool_calls[]` | `{tool_name, tenant_scope, input, output, timestamp}` — redacted per the sanitization rules in observability-slo §2 |
| `memory_access[]` | `{operation: read \| write, key, timestamp}` |
| `reconstructed_at` | timestamp — read-time reconstruction, not a stored artifact |

**Cross-tenant returns 404**, consistent with `GET /assessments/{id}`.

**`GET /cves/{id}/detail` → `CveDetailDto`**, via `CVEDetailQuery` / `ExposureProjection`: header (CVSS, EPSS, KEV), `risk_groups`, `flow_bar`, `factor_cards[]`, `assets[]`, `attack_paths[]`.

**The three taxonomies stay in separate DTO subtrees** ([exposure-analysis US-011](../10-product/features/exposure-analysis.md#us-011-exposure-analysis--cve-detail)). Optional `?projection=exposure|protection|action_cards`, defaulting to `exposure`.

**`GET /assets/{id}/context` → `AssetContextDto`** (US-002, FR-020, Gate 1, resolves OI-16). Reuses the exposure `assets[]` asset fields ([§3](#3-cve-read-projections), `ExposureProjection`) as its base, with `context_type = runtime` and one block per evidence source from [security-stepper US-002](../10-product/features/security-stepper.md#us-002-asset-context-evidence):

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

**Chat SSE** — `GET /chat/sessions/{id}/stream`, plus `POST /chat/sessions/{id}/hitl-response`. Events: `query`, `response`, `citation`, `processing_step`, `prioritization_cards`, `request_research_ack`, `hitl_request`. Contract: [kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md). **The same `hitl_request`/`hitl_response` pair carries both HITL surfaces in that file — §7's vendor-write approvals and §7a's Agentic RAG investigation-confidence-gate approvals (D-56)** — they're the same `WorkflowPort` HITL-signal primitive on two different workflow types, not two endpoints; `canonicalActionId` is absent on a §7a request, distinguishing it from a §7 vendor-write request.

**Connectors** — `POST /connectors/aws/sync` → `{sync_id, status, assets_ingested, started_at}`. `POST /connectors/csv/upload` is multipart; errors are `missing_column`, `invalid_os_family`, `duplicate_hostname`; limits are 50 K rows, UTF-8, unique `(tenant_id, hostname)`.

**Vendor writes — `POST /mitigations`, `POST /fast-actions`, `POST /remediation-tickets`.** All three require a client-supplied `Idempotency-Key` header, deduplicated the same way `POST /research/queue`'s `AssessmentDeduplicationService` works: a replayed key returns the original `202`/execution record rather than re-executing — required because each triggers a real vendor-side action (`endpoint.isolate`, `network.blocklist_add`, `policy.deploy_device_config`, `patch.deploy_special_devices`, `ticket.create_remediation`) with no other client-facing retry-safety mechanism. `POST /remediation-tickets`: `{finding_id | cve_id, ...}` → `202 {ticket_id, status: routed, canonical_action_id: "ticket.create_remediation"}`, executed via the same `VendorActionGate` pattern as `POST /mitigations`/`POST /fast-actions`.

**Kill switch** — `POST /v1/admin/kill-switch` (management plane): `{level: L1 | L2 | L3 | L4, tenant_id?, session_id?, reason}`. **Propagates in under 5 s p99** (NFR-005). `DELETE /v1/admin/kill-switch/{id}` deactivates.

**Acknowledgment** — `POST /vulnerability-instances/{id}/acknowledgments`: `{reason, expires_at?}` → `{acknowledgment_id, is_acknowledged: true, expires_at?}`. `DELETE …/acknowledgments/{ack_id}` revokes — the plural collection name matches the `acknowledgment_id` field it returns, consistent with this API's other resource names. The public read of `is_acknowledged` is true only for an **active** acknowledgment — not revoked, and with `expires_at` either null or in the future.

## 3. CVE read projections

In `packages/api/projections/`:

| Projection | Story | Output | Budget |
|-----------|-------|--------|--------|
| `ExposureProjection` | US-011 | severity, groups, attack paths, AWS evidence | NFR-013: p95 <500 ms |
| `ProtectionProjection` | US-003 | the four-state summary, plus vendor control panels | Gate 1 |
| `ActionCardProjection` | US-004 | mitigation steps, vendor deep-links, `canonical_action_id` | Gate 1 |

**A shared `CVEDetailQuery` base fetches the CVE and the tenant scope once**; each projection adds its own joins.

## 4. Error handling

`DuxErrorCode` is shared across REST, SSE, and webhooks:

| Code | HTTP |
|------|------|
| `AGENT_TIMEOUT` | 504 |
| `CONTEXT_EXHAUSTED` | 422 |
| `BUDGET_EXCEEDED` | 429 |
| `GOVERNANCE_BLOCKED` | 403 |
| `INSUFFICIENT_DATA` | 422 — `asset_gap` / `intel_gap` / `context_limit` |
| `VALIDATION_FAILED` | 422 — `details: [{field, message}]`. The single request-validation-failure shape corpus-wide; the CSV-upload `missing_column`/`invalid_os_family`/`duplicate_hostname` reasons and the DQL `DQL_EXPRESSION_TOO_COMPLEX` code (public-data-api.md §7) are `VALIDATION_FAILED` instances, not a separate error-code namespace |

Application error classes: `TenantIsolationError` (403), `ConnectorSyncError` (502), `AgentBudgetExceeded` (429), `AssessmentDedupConflict` (409).

UI copy and edge cases: [taxonomy §7–8](../10-product/taxonomy.md).
