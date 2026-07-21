---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-4, D-10, D-17, H5]
---

# AI Safety Overview

**Purpose:** the safety spine every Dux write and execution surface inherits, and the index to the documents that specify each control. **Parents:** BR-003 · Pillar B — safety scales with autonomy.

At Gate 1, executed investigation code ships unattended, and three of five vendor write actions (`network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`) ship **unattended by default**. Each still inherits the full spine: tenant isolation, governance gates, kill switch, and hash-chained audit. Human review is an anomaly-escalation path for those three, not a gate on every write — which means the controls below, not a human, are the primary defense for them. The two fleet-impacting actions (`endpoint.isolate`, `patch.deploy_special_devices`) are the exception: mandatory HITL on every call until each earns unattended execution via a field-proven safety record (D-17).

## 1. Safety spine (invariant)

Every agent surface inherits all six, without exception:

| Control | Guarantee |
|---------|-----------|
| RLS FORCE tenant isolation | no cross-tenant read or write is reachable, including by a compromised agent |
| CaMeL dual-LLM boundary | untrusted content never reaches a tool-calling context |
| MCP gateway | deny-by-default tool access |
| Kill switch KS-L1–L4 | halt propagates in <5 s, tenant-scoped |
| Hash-chained audit | every action is recorded and tamper-evident |
| AIBOM CI | supply chain is pinned and drift-blocked |

## 2. The lethal trifecta

Dux agents hold all three of the properties that make an agent dangerous:

1. **They read private customer data** — assets, runtime, identity, network, existing controls.
2. **They ingest untrusted content** — CVE text, exploit code, threat intel.
3. **They can act externally** — mitigation deployment, vendor API calls.

An agent holding all three is a lateral-movement vector inside the customer's environment: prompt-inject it through malicious content and the attacker inherits every credential and network path the agent holds. This is the 2026 supervisor risk (Red Hat) — an agent with access to everything is a single point of compromise.

The mesh answers each leg directly:

| Leg | Control |
|-----|---------|
| (1) private data | RLS FORCE + tenant isolation contain what any single agent can reach |
| (2) untrusted content | CaMeL neutralizes the injection vector; role-typed tool allowlists bound the blast radius |
| (3) external action | governance-kernel gates + kill switch + hash-chained audit on every write; human review escalates on anomaly |

**The CISO-facing statement of this posture:** Dux takes governed, audited, kill-switch-backed action in your environment at machine speed. Human review is available on request per asset class, and automatic on anomalies — it is not required on every write.

## 3. Dux as a target (credential honeypot)

Every marketed write — isolate endpoint, deploy policy, create ticket — executes against the **customer's** CrowdStrike, Intune, or ServiceNow, using **the customer's credentials, held by Dux**. A platform compromise would therefore mean write access to every customer's EDR, cloud, and ITSM. This is the highest-value target the architecture creates, and it is the threat an enterprise security reviewer will raise. Answer it with this section, not ad hoc.

Because human review no longer gates the default write path, these controls — not a human check — carry the weight against both agent misjudgment and platform-compromise replay:

- **Least-privilege scoped action credentials, per connector.** Write scope is limited to the 5 canonical actions ([catalogs](../10-product/catalogs.md)); never broad admin.
- **Per-action credential minting** — short-lived, not long-lived stored keys, wherever the vendor supports it. Where it does not: AES-256 at rest with SSM/Vault transit (ADR-011).
- **Bounded blast radius.** Worst case per tenant is the canonical action set on connected assets, bounded by governance-kernel budget and effect gates, the kill switch, and hash-chained audit. Replay of an approved action is countered by `mutation_key` idempotency plus audit anchoring.

## 4. Defense in depth (L1–L8)

| Layer | Control | Addresses |
|-------|---------|-----------|
| L1 Input containment | CaMeL dual-LLM | ASI01, ASI06 |
| L1b Output classifier | inline classifier on S-LLM output, pre-schema (Month 2) | ASI01 |
| L2 Structured output | constrained decoding (FSM / JSON schema) | ASI01, ASI05 |
| L3 Tool contracts | MCP hash pinning + schema validation | ASI02, ASI04 |
| L4 Identity | JWT with SPIFFE-format claims (SPIRE target Month 3) | ASI03 |
| L4b Inter-agent auth | signed service JWT per hop | ASI07 |
| L5 Execution isolation | Self-hosted Firecracker microVM on K8s (Gate 1) + AST pre-scan; gVisor read-only as defense in depth | ASI05, ASI10 |
| L6 Governance gates | Intent + Budget + Effect + DLP kernel | ASI01–ASI10 |
| L7 Audit | HMAC-SHA256 hash chain + hourly MinIO Object Locking anchoring | ASI09 |
| L8 Kill switch | <5 s propagation, tenant-scoped | ASI10 |

## 5. STRIDE threat model

| Threat | Vector | Mitigation |
|--------|--------|-----------|
| Spoofing | agent-to-platform auth | JWT / SPIFFE workload identity |
| Tampering | data in transit | TLS 1.3; mTLS service-to-service (seed) |
| Goal hijack / injection | untrusted CVE and intel | CaMeL dual-LLM + Intent gate |
| Supply chain (ASI04) | MCP tool schema drift | hash pinning + CI drift block + rotation |
| Repudiation | audit logs | HMAC chain (Vault key) + MinIO Object Locking anchoring |
| Information disclosure | customer data | RLS; AES-256; field-level PII encryption |
| Denial of service | API / assessment flood | `@nestjs/throttler` + Valkey; circuit breakers |
| Elevation of privilege | agent tool escalation | least-privilege RBAC; MCP allowlist; governance gates |

## 6. Component index

| Concern | Specification |
|---------|---------------|
| Dual-LLM boundary, prerequisite schema, CDA, retrieval | [camel-plane](camel-plane.md) |
| GOV-001–013 synchronous gates | [governance-kernel](governance-kernel.md) |
| KS-L1–L4 and the HITL contract | [kill-switch-hitl](kill-switch-hitl.md) |
| Agent identity, lifecycle, shadow AI | [agent-identity](agent-identity.md) |
| MCP policy PS-001–011, tool catalog | [mcp-security](mcp-security.md) |
| Self-hosted Firecracker microVM, AST scan, decision matrix | [sandbox-execution](sandbox-execution.md) |
| Confidence ensemble, Platt scaling, abstention | [confidence-calibration](confidence-calibration.md) |
| OWASP ASI01–10 / LLM01–10 / MCP crosswalk | [owasp-assessments](owasp-assessments.md) |
| The 12 canonical agentic incident runbooks | [incident-runbooks](incident-runbooks.md) |

## 7. Gate-1 exit criteria (safety)

- OWASP LLM and Agentic assessments at **Partial or better**.
- **ASI01 and ASI02 Implemented.** ASI10 Partial or better — kill switch and cost cap Implemented; eBPF deferred to Series A Month 9.
- **LLM01, LLM06, LLM10 Implemented.**
- **LLM09 is the only Gate-1 blocker** — the EXP-CIT-001 citation test.

## 8. Write-path controls

`VendorActionGate` (GOV-014) authorizes every canonical write action against the `GOV-TOOL-*` risk matrix — consequence scope, reversibility, and an unattended-execution confidence floor per action — and falls back to a HITL tier below that floor. Each action's compensating rollback procedure is on file in the [mitigation-write-path rollback catalog](../10-product/features/mitigation-write-path.md). See [governance-kernel §4](governance-kernel.md).
