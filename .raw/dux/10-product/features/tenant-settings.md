---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: []
---

# Feature — Tenant Settings, Help, Custom Metrics (US-014, US-015, US-022)

**Purpose:** tenant administration — users, keys, webhooks, agent policy, kill-switch visibility, data export — plus the help hub and custom metrics.

**Nav:** Settings, Help · **Epics:** EP-01, EP-07, EP-08 · **BRs:** BR-001, BR-003, BR-005, BR-006, BR-011.

## US-014 Tenant Settings

**Gate 1.**

**Job.** A tenant admin or API consumer manages users, API keys, webhooks, agent policy, kill-switch visibility, and data export.

**Orchestration.** No agent runs on this page. `POST /v1/admin/kill-switch` propagates in under 5 s (NFR-005). Agent policy caps LLM spend. The Assessment-quality widget reports the 10% drift cohort.

**API.**

| Endpoint | Contract |
|----------|----------|
| `POST /v1/admin/kill-switch` | `{level: L1–L4, tenant_id?, session_id?, reason}` |
| `DELETE /v1/admin/kill-switch/{id}` | deactivate |
| `POST /webhooks/configure` | webhook registration |
| `POST /tenants/{id}/export` | data export |
| `DELETE /tenants/{id}` | tenant deletion |

**Data API keys (Seed).** `agt_…` Bearer tokens, carrying the scopes `custom_metrics:read`, `vulnerability_instances:read`, and `cve_research:write`.

**These are distinct from agent-provisioning keys** (`POST /v1/agents`), and the two families are not interchangeable:

- An admin can create and revoke scoped data keys.
- **Data keys authenticate the public REST data API only** — they are rejected on dashboard JWT routes.
- **Agent-provisioning keys cannot call public data endpoints** without the scopes above.
- The rate-limit tier is surfaced per key.
- Webhooks carry an HMAC signature and an `Idempotency-Key`.

**Kill-switch UI.** L2 and L3 raise an in-app banner:

> *Agent activity paused for your organization — {reason}. Activated {timestamp} UTC. Contact your administrator.*

L1 is a single session and shows no banner. L4 is platform-wide.

**Gate split.** Users and agent policy are P0. API keys and webhooks are P1. SSO and SCIM are a seed trigger — Phase 1 shows a deferral note.

**Data.** `TENANT`, `API_KEY`, `WEBHOOK_CONFIG`. Kill-switch state lives in NATS (with a CloudNativePG fallback mirror).

**Metrics.** Kill-switch activation count; webhook delivery success; export volume; API-key usage by tier and scope; LLM spend against cap.

## US-015 Help & Support

**Phase-1 exit (P1). Not a Gate-1 blocker.**

A static link hub: `docs.dux.io`, `status.dux.io`, `trust.dux.io`, and tier-appropriate support. Contextual `?` links resolve to `docs.dux.io/phase1#us-*`.

**`trust.dux.io` was not publicly reachable at the June-2026 scrape — that is a launch blocker.** A broken-link synthetic check covers it. No agent or safety impact.

## US-022 Custom Metrics Configuration

**Seed.**

**Job.** A tenant admin or API consumer defines tenant custom metrics — display name, DQL filter, `group_by`, dashboard binding. The metric appears in the public `GET /v1/custom-metrics` once the Seed API trigger fires.

**Orchestration.** None. It is a query engine over the World Model. Data: the `CUSTOM_METRIC` ERD entity, with EntityType as a filter dimension.

**API.** The admin UI writes the config; reads go through the [public data API](../../30-api/public-data-api.md). Webhook: `custom_metric.updated` (Seed).

**Safety.** Invalid DQL is rejected at save with a 422. **There is no arbitrary code execution.** Grammar: [public-data-api §7](../../30-api/public-data-api.md#7-dql-dux-query-language).
