---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-34]
---

# Engineering Standards

**Purpose:** coding standards, the branching model, the code-review process, and the contribution guide — the layer OI-24 found missing from `50-engineering/`. **Parents:** BR-001, BR-002, BR-003.

**Scope note.** This file states team practice. It does not restate the merge gates already canonical in [ci-cd-testing](ci-cd-testing.md), the RLS/tenancy rules in [data-model](../20-architecture/data-model.md) / [multi-tenancy](../20-architecture/multi-tenancy.md), or the provider-port boundary in [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) — it links to them.

## 1. Coding standards

**Language.** TypeScript strict mode across `packages/*` (NestJS API, React + Vite web, Temporal workers, Pulumi infra). Python only in `packages/python-eval` (DeepEval, Evidently, calibration) — containerized from Week 2; no host venv (see [local-development §7](local-development.md)).

**Naming.** `camelCase` for variables/functions, `PascalCase` for classes/types/components, `UPPER_SNAKE_CASE` for constants, `SCREAMING_SNAKE` singular for ERD entities against `snake_case` plural physical tables ([data-model §5](../20-architecture/data-model.md)).

**Custom lint rules, consolidated** (each already normative in its owning spec — this table is the index, not a second source of truth):

| Rule | Enforces | Defined in |
|------|----------|-----------|
| `import/no-restricted-paths` | only `packages/adapters/*` imports a vendor SDK | [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) |
| `no-direct-llm-sdk` | every LLM call routes through `InstrumentedLLMClient` | [observability-slo §2](../60-operations/observability-slo.md) |
| `no-aws-sdk-outside-adapters` | Bedrock SDK (`@aws-sdk/client-bedrock-runtime`) is imported only inside `packages/adapters/*`, same boundary as every other vendor SDK | [ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer), [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) |
| `no-direct-cves-query` | `cves` is global and read-only; join through `findings` | [ADR-002](../20-architecture/adr-index.md#adr-002-r2--multi-tenancy) |
| `no-raw-findone` | every lookup goes through `TenantScopedRepository<T>`, never a raw `findOne` | [ADR-002](../20-architecture/adr-index.md#adr-002-r2--multi-tenancy) |

**No net-new `eslint-disable` in `api/` or `core/`** — CI blocks the merge (ADR-013).

**Immutability.** Prefer returning new objects over in-place mutation, particularly in `packages/core/world-model` and anywhere touching `AsyncLocalStorage`-scoped `TenantContext` — a mutated shared context is a cross-tenant-leak-shaped bug class, not just a style preference.

**File organization.** Feature/domain folders under `packages/*` ([architecture-overview §4](../20-architecture/architecture-overview.md)), not by type. Target 200–400 lines per file; extract before a file exceeds ~800.

**Error handling.** `DuxErrorCode` is the shared vocabulary across REST, SSE, and webhooks ([application-api §4](../30-api/application-api.md)) — new error paths extend that enum rather than inventing ad hoc codes. Never silently swallow a governance-kernel or connector failure; it either maps to a `DuxErrorCode` or raises the appropriate audit event.

## 2. Branching model

**Trunk-based**, short-lived feature branches off `main`. An ephemeral CloudNativePG cluster is created per PR ([ADR-003 R2](../20-architecture/adr-index.md#adr-003-r2--database-migrations)) — 7-day stale cleanup, alert above 20 live ephemeral clusters.

**Merge strategy:** squash-merge to `main`. The squashed commit message follows Conventional Commits (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`, `perf:`, `ci:`), matching the PR title.

**CODEOWNERS gates** (already normative elsewhere, indexed here): `@dux-security` on `packages/agents/tools/` and `instructions.md` ([ADR-009](../20-architecture/adr-index.md#adr-009--agent-as-directory-registry)); Engineering Lead + on-call on `packages/database/migrations/` for any `down` migration ([runbooks §5](../60-operations/runbooks.md)).

**Hotfix path.** A P0 hotfix lands within 30 minutes with 1 approval; P0 may bypass the normal review SLA in §3 below, never the merge gates in [ci-cd-testing](ci-cd-testing.md).

## 3. Code review process

**Every PR passes the full merge-gate pipeline** ([ci-cd-testing §1–2](ci-cd-testing.md)) before human review is requested — lint, typecheck, unit/integration, isolation/fuzz, the security-gate suite, and the golden set where applicable. Human review is not a substitute for a red gate.

**Review SLA:** first response within 1 business day; re-review after changes within 4 business hours. A PR touching `core/`, `api/auth`, or `packages/agents/tools/` requires a second reviewer from outside the immediate feature team.

**What a reviewer checks, beyond the automated gates:**

- The change traces to a BR/Epic/US/FR ID ([traceability-matrix](../00-meta/traceability-matrix.md)), or is explicitly infra/chore with no story to trace to.
- No new `eslint-disable`, no new TBD/placeholder left in shipped code.
- Tests assert behavior, not implementation — table-driven for connector/adapter logic, matching the Repository-pattern seams already in use.
- A tenant-scoped query uses `TenantScopedRepository<T>` / the composite `(tenant_id, id)` key — never a raw lookup by `id` alone.

**Release gate.** `release-gate.yml` requires Engineering Lead sign-off before tagging, on top of the automated checks ([ci-cd-testing §6](ci-cd-testing.md)).

## 4. Contribution guide

**Setup:** [local-development](local-development.md) — clone, `pnpm install`, `docker compose up`, seed data, and the local test suite.

**Before opening a PR:**

1. Run `pnpm test:ci` locally — the same suite CI runs, minus staging-only checks (cost benchmark, k6 load, Chromatic).
2. Confirm `pnpm test:isolation` passes if the change touches `api/`, `database/`, or `core/`.
3. Write the PR description against the template: what changed, the BR/FR/US it closes (if any), and a test plan.
4. Keep the PR scoped to one logical change — a bundled refactor-plus-feature PR is harder to review and to revert.

**Commit messages** follow Conventional Commits (§2). Body explains *why*, not *what* — the diff already shows what changed.

**First PR checklist** for a new engineer: local dev stack running (`pnpm dev:all`), `pnpm test:isolation` green, read [architecture-overview §1–5](../20-architecture/architecture-overview.md) and [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) before touching a provider port.
