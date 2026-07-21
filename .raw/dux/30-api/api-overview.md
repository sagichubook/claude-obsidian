---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-6]
---

# API Overview

**Purpose:** the three REST planes, their auth, versioning, and rate limits.

**Parents:** BR-006, BR-011.

**Authority:** OpenAPI 3.1 is canonical for the public `/v1/*` wire contract — endpoints, enums, field constraints, pagination. The application and management planes are governed by the BR → FR → US chain.

> **[`openapi.yaml`](openapi.yaml) is a draft skeleton, not the wire authority.** It inventories the paths, auth, and limits of all three planes. **It moves to the API service repo at build start** — contract-first, Spectral-linted, contract-tested. Until then, **the prose contracts in this folder are canonical.**

## 1. Three REST planes

**Do not conflate them.**

| Plane | Path prefix | Auth | Ship gate | Public OpenAPI |
|-------|-------------|------|-----------|----------------|
| **Application API** | `/dashboard/*`, `/research/*`, `/assessments/*`, `/cves/*`, `/connectors/*`, `/chat/*`, `/webhooks/configure`, `/vulnerability-instances/*` (acknowledgment) | Bearer JWT, `aud=api.dux.io` | Gate 1 | No — internal Redoc from Week 8 |
| **Public Data API** | `/v1/custom-metrics*`, `/v1/vulnerability-instances/{cve_id}`, `/v1/cve-research` | Bearer API key (`agt_…`, data scopes) | Seed trigger | Yes — `api.dux.io/docs` |
| **Management API** | `/v1/admin/*`, `/v1/agents/*` | platform-admin JWT | Gate 1, internal | No |

**Path-alias note.** The application DTO tables use the shorthand `/admin/kill-switch`. **The runtime route is `POST /v1/admin/kill-switch`** — it belongs to the management plane, and the gateway rewrites the prefix per environment.

## 2. Versioning

| Surface | Version | Breaking-change policy |
|---------|---------|------------------------|
| Public Data API | `v1.0.0`, pinned 2026-06-26 | **additive only within v1.** A breaking change means `/v2`, with a 90-day overlap |
| Application API | unversioned | breaking changes ship behind feature flags, with a changelog. Not published on `api.dux.io/docs` |
| Management API | `/v1/admin/*`, `/v1/agents/*` — `/v1` is a fixed path token, not a semantic version | evolves independently of the Public Data API's `v1.0.0`/`/v2` policy, despite the shared `/v1` segment — the two are not to be conflated (§1) |
| SSE event schemas | `2026.06` | new event types are additive. **A field removal requires a version bump** |
| Webhook payloads | `v1`, HMAC-signed | `Idempotency-Key` required. Deprecated fields honored for 90 days |
| `WorkflowPort` definitions | `ExploitabilityAssessmentWorkflow` v1 | gated by `patched()` |

## 3. Auth

**Application and Management planes.** Bearer JWT from Better Auth, with `aud` enforced. 60-minute access token; 7-day refresh, with rotation (ADR-001).

**Public Data API.** Bearer API key, shaped `agt_<8-char-prefix>_<32-char-secret>`. **Only the SHA-256 hash is stored; the plaintext is shown once.**

Scopes: `custom_metrics:read`, `vulnerability_instances:read`, `cve_research:write`.

**Public Data API keys are a distinct credential, not interchangeable with either JWT:**

- **A Public Data API key is rejected on dashboard (Application-plane) JWT routes.**
- **`POST /v1/agents` (Management plane) authenticates with platform-admin JWT, not an API key — it cannot be called with a Public Data API key at all, regardless of scopes.**

**Errors.** A `422` returns the same `DuxError` envelope used everywhere else (application-api.md §4): `VALIDATION_FAILED`, with `details: [{field, message}]`.

**Authorization.** Authentication alone does not authorize every Application-plane endpoint — the tenant `admin` / `member` / `viewer` role (`USER.role`, [data-model.md §2](../20-architecture/data-model.md#2-core-entities)) gates each one as follows (D-45). `platform_admin` is a distinct concept — Dux internal staff, not a tenant role — and always satisfies Management-plane auth separately.

| Endpoint | admin | member | viewer |
|----------|:-----:|:------:|:------:|
| `GET /dashboard/home`(`/stream`), `GET /research/dashboard`(`/stream`) | ✓ | ✓ | ✓ |
| `GET /assessments/{id}`(`/trace`, `/replay`), `GET /cves/{id}/detail` | ✓ | ✓ | ✓ |
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

`member` covers read plus day-to-day analyst operation (triggering research, acknowledging findings, HITL approve/deny, remediation tickets); tenant configuration and destructive/high-blast-radius actions (connector setup, webhook configuration, export, tenant deletion, the two highest-blast-radius unattended writes) stay `admin`-only. `viewer` is read-only everywhere. Management-plane and Public Data API endpoints are unaffected — they authenticate on platform-admin JWT and API-key scopes respectively, not `USER.role` (§1, §3 above).

## 4. Rate limits — two planes (D-6)

**Two distinct tables.** The original corpus stated both without reconciling them; they are different planes, with different traffic shapes.

**Application API** — interactive dashboard and agent traffic:

| Tier | req/min |
|------|---------|
| Starter | 1,000 |
| Professional | 5,000 |
| Enterprise | 10,000 |

**Public Data API** — programmatic `/v1/*`:

| Tier | req/min |
|------|---------|
| Starter | 60 |
| Professional | 300 |
| Enterprise | negotiated |

Every response on a rate-limited plane carries `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` (IETF `RateLimit-*` draft), so a well-behaved client can throttle itself proactively rather than only reacting to a `429`. A 429 (RFC 6585) additionally carries `Retry-After` (RFC 9110 §10.2.3, integer seconds). The tenant admin is notified if the queue stalls for more than 5 min.

Flood control sits at the Cloudflare edge. Identity-aware plan and tenant limits are enforced **post-auth**, by `@nestjs/throttler` + Valkey. **Concurrent SSE streams are capped at 5 per `user_id`.**

## 5. Domains

`dux.io` (marketing) · `app.dux.io` (customer app) · `api.dux.io` (and `/docs`) · `staging.dux.io` · `status.dux.io` · `trust.dux.io` · `docs.dux.io` · `security@dux.io`

**`trust.dux.io` and `status.dux.io` are launch blockers** (GCIS §E). **They must not be linked from marketing until they return HTTP 200.**

## 6. Application ↔ public bridge (the research queue)

The same capability is exposed differently on each plane, and the differences are deliberate:

| Concern | Application (Phase 1, JWT) | Public Data (Seed, API key) |
|---------|---------------------------|------------------------------|
| Enqueue | `POST /research/queue` — a single `cve_id`, **or** `natural_language` | `POST /v1/cve-research` — a batch of `cve_ids` (1–50) **only** |
| Response | `{assessment_id, status: queued \| deduplicated, queue_position}` | an array of `CveResearchV1Item` (`status: backlog \| completed`) |
| In-progress state | `QueueSummaryDto.in_research` | **no `in_research`** — poll `backlog` until `completed` |

**Natural-language enqueue has no public-v1 equivalent.** API consumers supply CVE IDs matching `^CVE-\d{4}-\d{4,}$`.
