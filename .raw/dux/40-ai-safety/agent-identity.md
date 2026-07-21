---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: []
---

# Agent Identity

**Purpose:** how assessment agents — and the optional Gate-5 physical-residency agents — are identified, authenticated, authorized, and audited. **Parents:** BR-003, BR-007.

Phase 1 uses JWT with SPIFFE-format claims. Full SPIFFE/SPIRE X.509 SVIDs target a Month-3 proof of concept.

## 1. JWT claim schema

Canonical — ADR-001 links here.

| Claim | Required | Example |
|-------|----------|---------|
| `sub` | yes | `spiffe://dux.io/tenant/{tenant_id}/agent/{agent_id}` |
| `agent_id` | yes, for agent tokens | uuid |
| `session_id` | yes, for agent tokens | assessment or chat session uuid |
| `tenant_id` | yes | tenant scope |
| `allowed_tools` | yes, for MCP sessions | `["query_assets","query_controls"]` |
| `aud` | yes | `api.dux.io` / `app.dux.io` |
| `exp` | yes | agent tokens default to 15 min |

## 2. Identity model

**Attributes:** `agent_id` (UUID, tenant-scoped), `tenant_id`, `identity_ref` (slug), `agent_type` (`assessment` · `physical_resident` · `supervised`), `credential_id`, `status` (`active` · `suspended` · `revoked`).

**Credential types:**

| Type | Use |
|------|-----|
| Per-agent **session JWT** | MCP gateway authentication |
| **API key** `agt_<prefix>_<secret>` | SHA-256 hashed. Public Data API only (data scopes: `custom_metrics:read`, `vulnerability_instances:read`, `cve_research:write`, [api-overview.md §3](../30-api/api-overview.md#3-auth)) — **never** MCP tool calls, and never authenticates `POST /v1/agents` (Management plane, platform-admin JWT only) |
| **OAuth client credentials** | seed, enterprise |
| **mTLS** X.509 | post-seed, 90-day rotation |

**Migration ladder:** pre-seed session JWT + API key → seed Gate 2 session JWT + OAuth → post-seed mTLS + SPIRE SVID.

## 3. Lifecycle

**Creation.** A platform admin calls `POST /v1/agents` (Management plane, platform-admin JWT — [api-overview.md §1](../30-api/api-overview.md#1-three-rest-planes)) with `identity_ref`, `agent_type`, and `config`. The platform generates the `agent_id` and its per-agent session JWT issuer. Default `agent_type` is `supervised`. Audit: `agent.created`.

**Rotation.** At most 2 active credentials — one API key and one session-JWT issuer, **never two keys with different scopes**. The old credential is revoked after a 24-hour grace period. Audit: `agent.credential.rotated`.

**Suspension.** Fires an L2 kill switch: new session credentials are blocked and existing sessions terminated. Reversible without regenerating credentials. Suspending a `physical_resident` agent may escalate to L3.

**Revocation.** Permanent. `status = revoked`, all sessions terminated, and the `agent_id` is **never reused** (recorded in `agents_tombstone`). Decommissioning purges cached credentials from Valkey and Vault; anonymizes PII once retention expires; and opens an AIBOM removal ticket.

**Autonomous approval.** `max_autonomy: autonomous` requires tenant-admin approval: `POST /v1/agents/{id}/autonomy-request` → approve → AIBOM and agent record updated.

## 4. Physical-residency agents (Gate 5 only)

DaemonSet `dux-resident-agent`. Heartbeat: `POST /resident-agents/{id}/heartbeat`, authenticated by **mTLS** (preferred) or a signed JWT carrying `agent_id`, `tenant_id`, `nonce`, and `exp` ≤60 s — a stale nonce is rejected. The response may carry `halt: true` to trip the kill switch.

Not used by the default Unified Integration Layer.

## 5. Shadow AI and behavioral baselines

A daily job compares declared agents (the `agents` table) against observed MCP `agent_id` headers and runtime `agent.session.started` audit records. Undeclared drift triggers **P0-B containment** plus a registry update, with an SLA of 1 hour to investigate and 4 hours to contain.

Baselines per `agent_type`:

| `agent_type` | req/min | Tool distribution | Output tokens p50/p95 | Cost p50/p95 | Cache hit | Fallback rate |
|-------------|---------|-------------------|----------------------|--------------|-----------|---------------|
| `assessment` | 2–8 | `query_assets` 60%, `run_assessment` 30%, other 10% | 800 / 2,400 | $0.08 / $0.25 | 80–95% | <5% |
| `physical_resident` (Gate 5) | 0.5–2 | heartbeat 80%, sync 20% | 200 / 600 | $0.02 / $0.06 | 70–90% | <2% |

`pnpm admin:agent-baseline-diff --agent-type assessment --window 5m` raises `DuxAgentBehaviorAnomaly` on a 2σ breach.

## 6. Inter-agent calls (ASI07)

Where multi-agent chaining is enabled, **every hop requires a signed service JWT** carrying `caller_agent_id`, `callee_agent_id`, and `tenant_id`. The gateway verifies it before forwarding.

The design stub `security/design/inter-agent-jwt.md` is a pre-seed exit artifact, and becomes mandatory at seed multi-agent.

## 7. Authorization

Least-privilege subsets live in the agent's `config.permissions`: `allowed_tools`, `max_autonomy` (`supervised` / `autonomous`), and `rate_limits`. **An agent cannot self-escalate**, and cross-tenant access is forbidden at the credential-validation layer.

MCP write-tool permissions (`allowed_tools`) are **distinct from** vendor mutation permissions, which are the ADR-012 canonical actions routed through `VendorActionGate`.

Delegation is recorded in `agent_delegation_grants(user_id, agent_id, scope, granted_at, expires_at)`.

## 8. Audit

Required fields on every action: `agent_id`, `tenant_id`, `credential_id`, `session_id`, `user_id?`, `action`, `timestamp`, `request_id`. Retention is 2 years (GDPR RoPA).

`agent_audit_log` is WORM append-only with an HMAC-SHA256 hash chain. The `chain_key` lives in Vault at `audit/chain-key`, and `/audit/verify` validates the chain with the server-held key — **a database writer cannot recompute a valid chain**. The chain head is anchored hourly to MinIO Object Locking (`dux-audit-anchors/`, COMPLIANCE mode, 7-year retention).
