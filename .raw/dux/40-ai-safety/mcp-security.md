---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-4, D-17, H5]
---

# MCP Security Policy (PS-001–016)

**Purpose:** the requirements for MCP server registration, tool invocation, and resource access, across six defense layers. **Parents:** BR-003, BR-007. Sources: NSA MCP design (May 2026), CIS MCP Companion v8.1, OWASP MCP Cheat Sheet.

**Protocol context.** MCP has documented protocol-level holes that neither NIST AI RMF nor ISO 42001 covers: absent capability attestation, bidirectional sampling without origin authentication, and implicit multi-server trust (arXiv:2601.17549). PS-001–011 answers the big three. Cross-walk the policy against the fuller **MCP-38 threat taxonomy** (arXiv:2603.18063, 38 enumerated threats) at seed.

## 1. Phase-1 tool catalog

Every tool is registered in the AIBOM (`security/aibom/manifest.json`) and the MCP gateway, with SHA-256 schema pins (PS-006). `tenant_id` is injected from the agent JWT and is **never** accepted as a tool parameter (MCP-005).

**Read-only tools.** These are the only tools with defined PS rows today.

| Tool | `integration_id` | Rate (tenant / session) | Attack story |
|------|------------------|------------------------|--------------|
| `search_nvd(cve_id)` | `nvd` | 200 / 50 rpm | AIBOM-NS-001 — hijack via malicious CVE text |
| `search_github(cve_id)` | `github-research` | 30 / 15 rpm | AIBOM-GH-001 — SSRF via repo URL |
| `search_exploitdb(cve_id)` | `metasploit-index` | 30 / 15 rpm | AIBOM-EDB-001 — poisoned module metadata |
| `search_threat_intel(cve_id)` | `medium-rss` | 20 / 10 rpm | AIBOM-TI-001 — malicious blog redirect |
| `search_msrc(cve_id)` | `microsoft-msrc` | 30 / 15 rpm | AIBOM-MSRC-001 — spoofed advisory content |
| `query_assets(filters)` | `aws` | 100 / 50 rpm (max 50 rows/call; overflow → `summarizeContext`) | AIBOM-QA-001 — filter injection |
| `query_controls(asset_id)` | `aws` | 100 / 50 rpm | AIBOM-QC-001 — cross-asset enumeration |

The gateway enforces `min(tenant_limit, session_remaining)`. All tools are hash-pinned; the hash check runs <2 ms p99 from an in-memory cache, with a full `admin:mcp-schema-diff` on cache miss.

**Write tools.** Five canonical mitigation actions route through `VendorActionGate` (ADR-012 R3). Three execute at Gate 1 unattended by default, with human review reserved for anomaly escalation: `network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`. Two — `endpoint.isolate`, `patch.deploy_special_devices` — require mandatory HITL on every call at Gate 1 (D-17) until each earns unattended execution via a field-proven Gate-3 safety record. Connectors themselves **must not** call vendor mutation APIs directly.

| PS ID | Tool | `integration_id` | Rate (tenant / session) | Attack story |
|-------|------|-------------------|--------------------------|--------------|
| PS-012 | `endpoint.isolate(asset_id, mutation_key)` | `crowdstrike` | 10 / 5 rpm | AIBOM-EDR-001 — spoofed high-confidence exploitability verdict engineered to force a false-positive isolation of a production host |
| PS-013 | `network.blocklist_add(rule)` | `crowdstrike` (IOC/indicator blocklist, [catalogs §6](../10-product/catalogs.md), D-50) | 20 / 10 rpm | AIBOM-FW-001 — blocklist-rule injection to self-DoS a legitimate service or IP range |
| PS-014 | `policy.deploy_device_config(device_id, config)` | `intune` (Gate 3, W2) | 20 / 10 rpm | AIBOM-MDM-001 — poisoned config payload delivered through a compromised device-config template |
| PS-015 | `patch.deploy_special_devices(device_id, patch_ref)` | — (no connector pinned; [catalogs §6](../10-product/catalogs.md)) | 10 / 5 rpm | AIBOM-PATCH-001 — crafted patch target forces a firmware downgrade with no API-level rollback |
| PS-016 | `ticket.create_remediation(finding_id, assignee)` | `servicenow` | 60 / 30 rpm | AIBOM-ITSM-001 — prompt-injected ticket content used for social engineering against the assignee |

Rows carry `integration_id: —` where the AIBOM has no connector pinned yet — matching [catalogs §6](../10-product/catalogs.md), not a gap unique to this table. Each row's confidence floor, rollback dependency, and HITL posture is the full PS statement below (§3), not just the table cell.

## 2. Six defense layers

| Layer | Focus | Policy IDs |
|-------|-------|-----------|
| L1 Auth & identity | per-agent short-lived JWT (`agent_id` + `tenant_id` per request) | PS-001, PS-002, PS-005, PS-011 |
| L2 Schema integrity | SHA-256 pins; ECDSA signing (hash-only interim); re-validate before invocation | PS-006, PS-009 (seed) |
| L3 I/O sanitization | JSON Schema; regex DLP <5 ms; SSRF allowlist | PS-003, PS-008 |
| L4 Network egress | gateway-only egress; Host validation; sandbox allowlist | PS-004 |
| L5 Observability | hashed I/O; SIEM Loki `mcp.audit`, 730-day retention | PS-007 |
| L6 Multi-server isolation | per-server security domain; no cross-server shadowing | PS-002, PS-006 |

## 3. Policy statements

**PS-001 Registration.** Explicit tenant-admin registration; no implicit discovery. DCR is disabled pre-seed. The gateway exposes `/.well-known/oauth-protected-resource` (RFC 9728). Agents must not invoke unregistered servers.

**PS-002 Tenant-scoped allowlists.** Per-agent allowlist; no cross-tenant credential sharing; changes require admin approval and are audited.

**PS-003 Tool invocation (L3).** Research tools are egress-DLP'd — no tenant asset identifiers in outbound queries. `query_assets` returns ≤50 rows. JSON Schema is strict (`additionalProperties: false`). Output is secret- and PII-scanned at <5 ms p99. The URL allowlist blocks RFC1918, link-local, and metadata addresses. Tool timeout is 30 s. The kill switch is checked before each invocation. Instruction-like output patterns are redacted and raise `mcp.tool_output_injection_suspect`, optionally L1.

**PS-004 Egress (L4).** Gateway-only, allowlisted URLs; no direct outbound path bypassing the gateway. TLS 1.3 preferred, 1.2 minimum. The egress proxy blocks private IPs — verified by `test:mcp-security --case MCP-SSR-001`.

**PS-005 Credentials (L1).** Per-agent session JWT. The token's `aud` / `resource` must match `server_id`; cross-server reuse is rejected with `mcp.token_audience_mismatch`. Credentials live in SSM or Vault via `SecretsPort` — never in a plaintext database column — scoped to tenant plus connection, rotated every 90 days, and never present in prompts, logs, or telemetry. **Credential isolation is row-scoped by design, not path-scoped:** all tenants' connector credentials sit behind one shared Vault KEK rather than per-tenant Vault paths, a deliberate cost/ops tradeoff at this stage — tenant separation is enforced by the `tenant_id`-scoped row and `SecretsPort` access control, not by per-tenant key material.

**PS-006 Supply chain (L2).** ASI04 checklist before allowlisting. SHA-256 pin over canonical JSON (`name` + `description` + `input_schema`). `admin:mcp-tool-lint` blocks instruction-like descriptions. `admin:mcp-scan` (Invariant / Snyk mcp-scan) runs on every registration change to detect tool poisoning and rug-pulls, and is merge-blocking. The hash is re-validated before every invocation; on drift the tool is auto-disabled and re-consent is required. Pins rotate quarterly. Per-tool circuit breaker: 5 failures in 60 s.

**PS-007 Audit (L5).** Log `agent_id`, `tenant_id`, `tool_name`, `server_id`, `session_id`, `request_id`, timestamp, outcome, and `latency_ms`. Store `input_hash` and `output_hash`. Correlate through the `InstrumentedLLMClient` span. SIEM: Loki `mcp.audit`, 730-day retention. Alert above 50 denials/hour/tenant; L1 after 5 consecutive denials in a session.

**PS-008 PII (opt-in overlay).** PII in tool output requires tenant-admin opt-in (`tenant_settings.pii_opt_in_at`) before it may enter LLM context. Otherwise it is redacted per PS-003.

**PS-009 Message signing (seed).** ECDSA P-256 over request and response, with a 128-bit `nonce` and `iat`. Reject duplicates and anything older than 5 minutes. Signing is mutual; keys live in Vault with 90-day rotation. Pre-seed interim: PS-006 hash-only satisfies exit.

**PS-010 OS sandboxing (seed).** Pre-seed container hardening — read-only root filesystem, non-root user. Seccomp profiles land at Gate 2.

**PS-011 Remote OAuth (L1).** OAuth 2.1 with PKCE (S256). Token endpoint, redirect allowlist, and `resource` parameter (RFC 8707) are documented at registration.

**PS-012 `endpoint.isolate` (GOV-TOOL-01).** Mandatory HITL T3 on every call regardless of confidence (D-17) — no unattended path exists yet. `VendorActionGate` will not authorize execution without rollback [R-01](../10-product/features/mitigation-write-path.md) (`endpoint.restore_network`, keyed by `mutation_key`) on file. Promotes to unattended execution only on a field-proven Gate-3 safety record for the action class.

**PS-013 `network.blocklist_add` (GOV-TOOL-02).** Unattended by default at Gate 1; HITL T2 fires only below the 0.75 confidence floor. Rollback [R-02](../10-product/features/mitigation-write-path.md) (`network.blocklist_remove`) reverts the specific vendor-native rule ID only — never a broader flush.

**PS-014 `policy.deploy_device_config` (GOV-TOOL-03).** Unattended by default once the `intune` connector ships (Gate 3, W2); HITL T2 below the 0.75 confidence floor. Rollback [R-03](../10-product/features/mitigation-write-path.md) (`policy.restore_previous_config`) redeploys the pre-deploy config snapshot.

**PS-015 `patch.deploy_special_devices` (GOV-TOOL-04).** Mandatory HITL T3 on every call — firmware-only devices have no API-level rollback, so `GOV-TOOL-04` holds this action to mandatory HITL rather than letting it execute without an undo path. Rollback [R-04](../10-product/features/mitigation-write-path.md) (`patch.rollback_to_prior_version`) applies only where the vendor patch-management API supports it; otherwise the procedure is a manual runbook.

**PS-016 `ticket.create_remediation` (GOV-TOOL-05).** Unattended at Gate 1 — the lowest blast tier (T1), no confidence floor gate. Rollback [R-05](../10-product/features/mitigation-write-path.md) (`ticket.cancel`, reason `superseded_by_rollback`) fires on HITL rejection or duplicate-ticket detection.

## 4. Gateway circuit breaker

| Property | Value |
|----------|-------|
| Trip condition | >50% tool errors in 5 min, **or** p99 latency >2 s |
| Cooldown | 60 s (gateway returns 503) |
| Automatic close | 5 min of stable metrics |

Operator escalation: a per-tool 503 (PS-006) → retry after 60 s. An aggregate 503 → check upstream MCP health. Sustained failure → L2 kill switch.

## 5. Enforcement

Deny-by-default policy engine: PS-001–008 and PS-012–016 pre-seed (write path ships at Gate 1), PS-009/010 at seed.

| Control | Mechanism |
|---------|-----------|
| CI on MCP PRs | `security-review` label + allowlist + risk classification |
| Hardcoded MCP URLs | blocked by ESLint |
| Secrets | Gitleaks |
| Integration tests | MCP-001–005 — unregistered server denied; schema drift blocked; PII redacted; egress allowlist enforced; cross-tenant JWT rejected |
| Compliance evidence | `security/owasp/agentic-assessment-*.json`; gateway audit logs retained 2 years |

## 6. Server inventory

| Server | Credential |
|--------|-----------|
| `mcp-nvd-feed` | session JWT + NVD key in Vault |
| `mcp-aws-connector` | session JWT + IAM assume-role |

## 7. Tool risk classes

| Class | Phase-1 status |
|-------|----------------|
| Read-only | shipped — catalog in §1 |
| Write | shipped — the five ADR-012 R3 canonical actions only; 3 unattended by default with HITL on anomaly escalation, 2 (`endpoint.isolate`, `patch.deploy_special_devices`) mandatory HITL on every call (D-17). Catalog: PS-012–016 (§1, §3) |
| External | prohibited in Phase 1 |
| Code | prohibited in Phase 1 |
| Financial | prohibited in Phase 1 |

The OWASP MCP Top 10 crosswalk lives in [owasp-assessments](owasp-assessments.md).
