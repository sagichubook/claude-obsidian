---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-1, D-9, D-33, D-34]
---

# Sandbox Execution (ADR-015 R4)

**Purpose:** the execution boundary for LLM-generated investigation scripts. **Parents:** BR-002.

**Self-hosted Firecracker on Kubernetes ships at Gate 1** (2026-07-19, D-33 — accelerated from R3's Gate-2/3-ICP-driven timeline). The managed-microVM era (E2B default, Modal fallback) is retired, not kept as a fallback.

## 1. Decision matrix

| Case | Adapter | Behavior |
|------|---------|----------|
| **Gate 1** | **`SelfHostedFirecrackerAdapter`** via `SandboxPort` (`firecracker-containerd` or Kata as the K8s-integration bridge) | investigation-script execution behind the port; AST pre-scan mandatory; `execution_results` populated |
| Emergency kill path | `NoOpSandboxAdapter` | artifact-only; `execution_results = null`. Only during a kill-path event |
| Gate 2+ read-only containers | gVisor | defense in depth for non-script MCP tool containers **only** — never for LLM-generated script execution |
| Gate 5 physical residency | customer-side DaemonSet + `SandboxPort` remote adapter | optional physical-residency agents |

**Why a microVM and not a container.** Shared-kernel containers are **not** a security boundary for AI-generated code — the named escape CVEs are **CVE-2024-21626** (runc, "Leaky Vessels") and CVE-2024-0132, and container sandboxes demonstrate escape under adversarial LLM code (SandboxEscapeBench, 2026). gVisor is therefore acceptable only as read-only defense in depth. **Now that the deployment target is Kubernetes on EKS (ADR-006 R4) rather than a managed PaaS, Firecracker runs directly, self-hosted, in-boundary** — the prior constraint that "managed PaaS cannot run Firecracker microVMs, so a managed microVM vendor is required" no longer applies.

**The microVM boundary itself is not CVE-free (SR-07, still applies self-hosted).** Two 2026 Firecracker escape-class CVEs are on record: **CVE-2026-5747** (virtio-pci out-of-bounds, opt-in PCI path only; fixed 1.14.4/1.15.1) and **CVE-2026-1386** (jailer symlink handling). Firecracker patch cadence — not just the container-vs-microVM boundary decision above — is an active, ongoing control, not a one-time architectural choice, and self-hosting makes Dux directly responsible for patch currency rather than trusting a vendor's SLA. **A newly disclosed VMM-class CVE is a kill-path rehearsal trigger**: on disclosure, the on-call AI Safety Lead confirms the in-cluster Firecracker version against the fix, and if patching lags, exercises the emergency `NoOpSandboxAdapter` kill path rather than waiting.

**A fresh ephemeral microVM per invocation, never reused across runs.** VM reuse is a data-leak vector. This durable-workflow → ephemeral-sandbox shape **aligns with** the Temporal-community **Sandbox Orchestration Harness** reference (temporal.io blog, May 2026) — a Code-Exchange community pattern, not an official Temporal product, whose own model is pause/resume plus snapshot-fork. Dux deliberately tightens beyond that reference to fresh-microVM-per-invocation with no snapshot reuse (SR-08; see [ADR-007 §Reference pattern](../20-architecture/adr-index.md#adr-007-r3--durable-execution-engine) for the same correction applied to the workflow-engine ADR).

**Self-hosted operational checklist (supersedes the Week-2 vendor checklist, 2026-07-19, D-33):** `firecracker-containerd`/Kata runtimeclass integration on the K8s node pool (EP-01-F02-T11); **Firecracker version and patch SLA now an internal ops responsibility, not a vendor-evidenced claim**; a subscription to Firecracker's own CVE/security-advisory feed; kill-path fallback verified. No subprocessor code-residency review is needed — investigation code never leaves Dux's own cluster.

## 2. Path selector

```
LLM-generated investigation script?
  YES → SelfHostedFirecrackerAdapter via SandboxPort only
        (firecracker-containerd/Kata as the K8s-integration bridge)
  NO  → read-only MCP API call?
          YES → gVisor read-only container (defense in depth), or no sandbox
          NO  → optional physical-resident agent on customer infra (Gate 5)
                → customer-side sandbox + eBPF, Month 9
```

## 3. AST scan pipeline (`ScriptSecurityScanner`)

Runs before every execution:

1. The agent submits a script.
2. It is parsed — `@babel/parser` for TS/JS, tree-sitter for Python.
3. The AST is walked against `packages/security/script-rules/`.
4. **Pass** → enqueue the microVM. **Fail** → `SCRIPT_BLOCKED` plus an audit record, and no execution.

| Component | Specification |
|-----------|---------------|
| Blocked calls | `exec()`, `eval()`, `Function()`, `child_process`, `subprocess`, `os.system`, `spawn` |
| Blocked imports | `fs` (except a read-only allowlist), `net`, `http`, `https`, `dgram`, `child_process`, `cluster` |
| Network egress | default DROP; allowlist is NVD and GitHub only; no AWS metadata endpoints unless the tenant opts in, with IMDSv2 hop limit = 1 |
| Sandbox limits | read-only filesystem (tmpfs for output); 512 MB / 1 CPU; 60 s wall-clock |
| Audit | SHA-256 script hash, output, `scanner_version`, `ruleset_version` — written to `AUDIT_EVENT`. **Execution is blocked if `ruleset_version` ≠ the deployed scanner** |

| Failure code | Event | Cause |
|--------------|-------|-------|
| `SCRIPT_BLOCKED` | `script.security.blocked` | AST rule violation |
| `SCRIPT_SANDBOX_TIMEOUT` | `script.sandbox.timeout` | 60 s wall-clock exceeded |
| `SCRIPT_SANDBOX_OOM` | `script.sandbox.oom` | 512 MB exceeded |

CI gate: `pnpm test:script-security` — merge-blocking on any new blocked-pattern bypass.

## 4. eBPF timeline

| Phase | Timeline | Scope |
|-------|----------|-------|
| Gate 2 pilot | Seed Month 2+ | read-only syscall audit on investigation scripts |
| Series A Months 1–8 | interim | self-hosted Firecracker + MCP allowlist as compensating controls; quarterly board disclosure |
| **Series A Month 9** | mandatory | per-agent eBPF syscall filtering for transaction-touching agents (kill switch + purple-team sign-off) |
| Series B | hardening | policy hardening, escape detection, quarterly purple team |

## 5. Partial-failure semantics and sandbox budget (D-9)

| Failure | Retry | Verdict effect |
|---------|-------|----------------|
| `SCRIPT_BLOCKED` | none | the blocked script earns no exploitable-verdict credit |
| `SCRIPT_SANDBOX_TIMEOUT` | 1 retry | still failing, with a strong verdict (`exploitable` / `likely`) → HITL T3 |
| `SCRIPT_SANDBOX_OOM` | none | strong verdict → HITL T3 |

**An assessment never silently completes as `exploitable` on missing execution.**

Per-tenant sandbox budget: **300 sandbox-seconds per hour and 5 concurrent microVMs**, enforced in the governance kernel (`BudgetGate`). A breach raises `budget_exceeded` → L2. The enforcement metric is `dux_cost_sandbox_seconds_per_tenant`.
