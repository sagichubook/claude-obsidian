---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-7, D-8, D-34]
---

# EP-01 — Multi-Tenant Platform & Auth

**Objective:** BR-001 zero cross-tenant leakage (KPI: isolation 100% in CI) · **Metrics:** `test:fuzz-tenant-id` 100%, ISO-001–010 + ISO-FUZZ-001–005 green, AUTH-001–005 green · **Target:** Gate 1 (Week 12, [D-7 R1](../00-meta/decisions-log.md)); isolation harness Week 4 (blocks Gate 1) · **Priority:** P0 · **Constraints:** ADR-001 (Better Auth via `AuthPort`), ADR-002 R2 (shared-schema RLS FORCE on CloudNativePG; PgBouncer transaction mode), composite FKs, fail-closed GUC.

## EP-01-F01 — Tenant platform: auth, isolation, settings, lifecycle

**Parent:** EP-01 · **FRs:** FR-001, FR-009 · **ACs:** [tenant-settings spec](../10-product/features/tenant-settings.md) success criteria; FR-001/009 verification rows · **Dependencies:** CloudNativePG provisioned; Better Auth 1.5 · **API boundary:** auth routes, `/tenants/*`, settings UI · **SLA:** API p95 <300 ms (NFR-002); export <24 h (NFR-006).

### US-014 — Tenant Settings (Gate 1) · 13 pts · persona: tenant admin
**ACs:** users+roles P0; agent policy P0; API keys/webhooks P1; kill-switch visibility (BR-003); audit export (BR-005). **DoD:** merge gates + `test:fuzz-tenant-id` + AUTH suite. **Spike:** PgBouncer pooling-mode fuzz (Week 4, AI-7).

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-014-T01 | Better Auth integration behind `AuthPort` (JWT claims, refresh rotation, reuse detection) | Backend | Code | 24 | 1–2 | TS-1 |
| US-014-T02 | RLS FORCE migrations + `TenantContextMiddleware` + fail-closed GUC + composite FKs | Backend | Code | 24 | 1–2 | TS-2 |
| US-014-T03 | PgBouncer session-mode config + pooling fuzz spike (AI-7; merge-blocking Week 4) | DevOps | Research | 12 | 3–4 | TS-2 |
| US-014-T04 | `test:fuzz-tenant-id` property suite (ISO-FUZZ-001–005, fast-check) | QA | Test | 16 | 3–4 | TS-3 |
| US-014-T05 | Tenant settings UI (users/roles, agent policy toggle, kill-switch banner) | Frontend | Code | 20 | 5–7 | TS-3 |
| US-014-T06 | API keys + webhook config surface (P1) | Backend | Code | 12 | 8–9 | TS-1 |
| US-014-T07 | `TenantExportWorkflow` + GDPR delete path (FR-009; 30-day window, 90-day purge) | Backend | Code | 16 | 8–9 | TS-1 |
| US-014-T08 | Impersonation guardrails (session GUCs, audit `admin.impersonate`, 24 h expiry) + review | Security | Review | 8 | 9 | SEC |

**Enabler tasks (platform CI, D-EX-3):**

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| EP-01-F01-T01 | `check-rls.sh` merge gate + negative fixture | DevOps | Config | 6 | 2 | TS-2 |
| EP-01-F01-T02 | ISO-001–010 integration suite | QA | Test | 16 | 3–4 | TS-3 |

## EP-01-F02 — Platform foundations & launch blockers (enabler — D-EX-3)

**Parent:** EP-01 · **FRs:** — (non-FR traceability rows: trust/status portals = GCIS §E launch blocker; AI-2 on-call; TR-NFR-008 flags) · **ACs:** CI pipeline green end-to-end; SLO alerts firing in staging (Week 8); `trust.dux.io`/`status.dux.io` HTTP 200 before first NDA partner; on-call named before Gate 1 · **Dependencies:** none (Week-1 start) · **API boundary:** infra, CI, observability, public shells · **SLA:** deploy <15 min; rollback <5 min.

**Re-costed 2026-07-19 (D-33) for the self-hosted-Kubernetes stack, hosting target narrowed 2026-07-19 (D-34).** `EP-01-F02-T02` (ECS Fargate baseline, 20 h) is retired and replaced by the K8s cluster bring-up task group T02a–g below — self-hosted install/ops of five services is strictly more work than pointing four ECS services at managed AWS tiers, not a 1:1 hour swap. Two new tasks (T10, T11) are added for work pulled forward from Gate-2/3 by the accelerated ADR-015 R4 (Firecracker) timeline. T02a's cluster CSP moves DO/Linode LKE → EKS (D-34); hours are unchanged — EKS cluster provisioning via Pulumi is not materially more or less work than the DO/Linode equivalent it replaces.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| EP-01-F02-T01 | Monorepo + CI pipeline skeleton (lint→unit→integration→security gates→golden→build→staging) | DevOps | Config | 20 | 1–2 | TS-2 |
| EP-01-F02-T02a | K8s cluster provisioning (Pulumi, Amazon EKS, node pools, nginx Ingress) | DevOps | Deploy | 16 | 1–2 | TS-2 |
| EP-01-F02-T02b | CloudNativePG operator install + backup/restore to MinIO + ephemeral-cluster-per-PR tooling (ADR-003 R2) | DevOps | Deploy | 16 | 2–3 | TS-2 |
| EP-01-F02-T02c | NATS + JetStream install (event bus + durable queues, ADR-005 R2/ADR-016 R2) | DevOps | Deploy | 8 | 2–3 | TS-2 |
| EP-01-F02-T02d | Valkey install (cache, rate limits, session state) | DevOps | Deploy | 4 | 2 | TS-2 |
| EP-01-F02-T02e | MinIO install + Object Locking config (audit anchor + static-frontend origin) | DevOps | Deploy | 8 | 2–3 | TS-2 |
| EP-01-F02-T02f | Self-hosted Temporal install (server, CloudNativePG persistence, cert-manager mTLS, ADR-007 R3) | DevOps | Deploy | 20 | 2–4 | TS-2 |
| EP-01-F02-T02g | Vault install + secret migration off design (replaces AWS SSM, D-5 R2) | DevOps | Deploy | 12 | 3–4 | TS-2 |
| EP-01-F02-T03 | Observability stack: OTel GenAI conventions, self-hosted Langfuse + Grafana LGTM (Loki/Tempo/Prometheus), Falco, SLO burn-rate alerts (staging Week 8) | DevOps | Config | 28 | 6–8 | PY-2 |
| EP-01-F02-T04 | `python-eval` containerization (Week 2; golden-set flake <2%) | DevOps | Config | 8 | 2 | PY-2 |
| EP-01-F02-T05 | Local dev: docker compose + CloudNativePG dev-cluster flow | DevOps | Config | 8 | 1–2 | TS-3 |
| EP-01-F02-T06 | Trust + status portal shells @ HTTP 200 (DNS runbook; launch blocker GCIS §E) | DevOps | Deploy | 12 | 2–4 | PM |
| EP-01-F02-T07 | On-call rotation + PagerDuty + incident dry-run (AI-2, before Gate 1) | DevOps | Config | 8 | 8–9 | SEC |
| EP-01-F02-T08 | `models.json` pins + Unleash flags baseline + cost-gate CI ($0.55 staging avg) | DevOps | Config | 10 | 2–3 | PY-2 |
| EP-01-F02-T09 | Frontend scaffolding: React + Vite (TanStack Router/Query) + MinIO/Cloudflare CDN baseline (unblocks every FE task — costed spine) | Frontend | Code | 14 | 2–3 | TS-3 |
| EP-01-F02-T10 | **Retired (D-34):** the P0 Bifrost bake-off spike is superseded — LiteLLM removal ([ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer)) means there's no primary-slot proxy to bake off against; Bifrost is now a Gate-2-only spike, evaluated only if NestJS fallback logic can't cleanly own routing complexity. Hours removed from the Gate-1 envelope; see [decisions-log D-34](../00-meta/decisions-log.md) capacity note. | — | — | 0 | — | — |
| EP-01-F02-T11 | **New (D-33):** Firecracker/Kata K8s integration, pulled forward from Gate-2/3 (`firecracker-containerd` runtimeclass, sandbox broker rewrite for in-cluster microVMs, ADR-015 R4) | DevOps | Deploy | 24 | 4–6 | TS-2 |

**Legal/PM lane (0 eng hours, two hard external deadlines — tracked here, owned LEGAL/PM):** EU AI Act counsel opinion Week 6 (blocks EU provisioning) · **ZDR with OpenAI + Anthropic (H2/D-8 — subprocessor-listing precondition)** · self-hosted Langfuse requires no DPA (D-33, supersedes the Langfuse-DPA line) · SLA counsel sign-off (AI-226a) · pre-publication claims checklist.

**EP-01 total: 370 h** (was 272 h; +98 h from the 2026-07-19 self-hosted-K8s stack replacement, D-33/D-34 — the original +114 h D-33 estimate included the P0 Bifrost bake-off spike (EP-01-F02-T10, 16 h), retired by D-34 — see [decisions-log](../00-meta/decisions-log.md) and the recomputed grand total in [traceability](traceability.md)).
