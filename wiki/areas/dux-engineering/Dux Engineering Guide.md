---
type: area
title: "Dux Engineering Guide"
topic: "dux/engineering"
created: 2026-07-22
updated: 2026-07-22
tags: [area, dux, dux/engineering]
status: mature
sources: [".raw/dux/50-engineering/engineering-standards.md", ".raw/dux/50-engineering/ci-cd-testing.md", ".raw/dux/50-engineering/local-development.md"]
related: ["[[Dux]]", "[[Dux Architecture Guide]]", "[[Dux AI Safety Guide]]", "[[Dux Portfolio]]"]
---

# Dux Engineering Guide

Navigation: [[Dux]] | [[Dux Architecture Guide]] | [[Dux AI Safety Guide]]

This guide is scoped deliberately narrow: team practice and pipeline mechanics only. System architecture and the technology stack live in [[Dux Architecture Guide]]; production monitoring and SLOs live in [[Dux Operations Guide]]. If you're looking for *why* a system is built a certain way, this isn't that page: it's *how the team ships it safely*.

---

## 1. Local Development

### 1.1 Repository and Prerequisites

The canonical repository is `github.com/duxsecurity/dux.git` — a private monorepo. One naming trap worth flagging before anything else: `github.com/duxsec` is a different, unrelated GitHub account, and using it by mistake is an easy, embarrassing error the team has explicitly called out to avoid **(resolves [OI-23](../00-meta/open-items.md))**. Engineers without access should request it through the standard IAM onboarding path, not by searching for alternate org names.

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | 22 LTS | API, web, workflow workers (`.nvmrc` + `engines`) |
| pnpm | 9+ | `corepack enable && corepack prepare pnpm@9.15.0 --activate` |
| Docker Desktop | latest | the full infrastructure stack |
| Python | 3.11+ | optional before Week 2. **After Week 2, use the `python-eval` container only** |
| Git | 2.40+ | source control |

### 1.2 Clone and Install

```bash
git clone git@github.com:duxsecurity/dux.git dux
cd dux && pnpm install
cp .env.example .env.local
```

**Environment loading.** The root `.env.local` is loaded by NestJS (`packages/api`), Vite (`packages/web`), and the workflow workers (`packages/core`), all through `dotenv/config`. `DATABASE_URL` lives in the root only — workers inherit it through the turbo env.

### 1.3 Environment Variables

| Variable | Local value | Notes |
|----------|-------------|-------|
| `DATABASE_URL` | `postgresql://postgres:postgres@localhost:5432/dux_dev` | root `.env.local` only |
| `VALKEY_URL` | `redis://localhost:6379` | Valkey speaks the Redis wire protocol, so local tooling is unchanged |
| `JWT_SECRET` | generate it: `openssl rand -base64 32` | never commit a value |
| `OPENAI_API_KEY` | — | |
| `ANTHROPIC_API_KEY` | — | |
| `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY` | empty locally | |
| `LLM_ROUTING_PROFILE` | `local` | direct Bedrock SDK behind `LLMProviderPort` — **no LiteLLM proxy, locally or anywhere else** (ADR-010 R5, D-34) |
| `NVD_API_KEY` | — | |
| `VAULT_ADDR` | `http://localhost:8200` | |
| `VAULT_TOKEN` | — | |
| `UNLEASH_URL`, `UNLEASH_API_TOKEN` | — | |
| `S3_ENDPOINT` | `http://localhost:9000` | MinIO |
| `S3_BUCKET` | `dux-artifacts` | |
| `AWS_ENDPOINT_URL` | `http://localhost:4566` | LocalStack — requires `--profile aws` |
| `APP_URL` | `http://localhost:3000` | |
| `API_URL` | `http://localhost:3001` | |

> **AI-66 (P0). No plaintext development credentials in the docs or the repository.** Generate them at seed time. `git-secrets` and a pre-commit hook block credential commits.

### 1.4 Start Infrastructure and Services

```bash
cd infra
docker compose up -d          # Postgres, PgBouncer, Valkey, MinIO, Vault, Unleash
docker compose --profile aws up -d localstack   # optional: AWS connector testing
pnpm infra:wait
pnpm --filter database migration:run            # applies RLS + FORCE
pnpm --filter database db:seed
docker compose up -d python-eval                # Week 2+ — canonical for the golden set
```

### 1.5 Docker Compose Profiles

| Profile | Services | Purpose |
|---------|----------|---------|
| *(default)* | Postgres, PgBouncer, Valkey, MinIO, Vault, Unleash | Core infrastructure |
| `aws` | LocalStack | AWS connector testing (`AWS_ENDPOINT_URL=http://localhost:4566`) |
| `python-eval` | python-eval container | Golden-set evaluation (Week 2+) |

### 1.6 Start Application Services

```bash
pnpm dev:all                  # everything at once
# or individually:
#   pnpm --filter api dev          → :3001
#   pnpm --filter web dev          → :3000
#   pnpm --filter core dev         → Temporal worker
#   pnpm --filter connectors dev   → :3002
```

**Verify:** web on `:3000`; API on `:3001/health`; connectors on `:3002/health`.

### 1.7 Temporal Worker Setup

The Temporal worker connects to the self-hosted Temporal cluster's dev namespace (K8s dev environment, ADR-007 R3), or to a local Temporalite. **Local development must never expose a production workflow UI.**

If not running the full K8s dev namespace, use the official OSS bootstrap: `temporal server start-dev` — starts a local Temporal server plus Web UI (default `localhost:8233`) with no external dependencies, sufficient for `pnpm --filter core dev` to connect against. The older `temporalite` tool is deprecated upstream in favor of `temporal server start-dev`.

### 1.8 First-Run Setup Notes (D-57)

Confirmed (Founder, 2026-07-21) as matching the team's actual local-dev workflow.

- **Temporal.** Use `temporal server start-dev` — no external dependencies needed.
- **Vault.** `docker compose up -d vault` alone does not make `VAULT_TOKEN` usable — Vault starts sealed by default. For local dev only, run Vault in dev mode (`vault server -dev`, or the container's dev-mode equivalent), which auto-unseals and prints a root token to use as `VAULT_TOKEN`. **Dev mode stores everything in memory and is never a template for the production init/unseal flow** (which uses Shamir key shares and persistent storage) — production unseal steps are out of scope for this file.
- **Bedrock local auth (`LLM_ROUTING_PROFILE=local`).** The direct Bedrock SDK path (ADR-010 R5, D-34) authenticates via the standard AWS credential chain — typically `aws configure sso` against the org's AWS SSO, or a named profile (`AWS_PROFILE=<profile>`) with Bedrock invoke permissions in the target region. No Bedrock-specific env var beyond the standard AWS ones is required.
- **`NVD_API_KEY` and AWS connector credentials.** These are per-developer secrets with no fixed source in this repo. `NVD_API_KEY` is requested from NVD directly (api.nvd.nih.gov key request form); AWS connector credentials for local connector testing come from the same AWS SSO/profile used for Bedrock above, scoped to the connector's IAM role.

---

## 2. Test Suite

### 2.1 Full Test Command Listing

```bash
pnpm test                    # unit (Vitest, all TS packages)
pnpm test:integration        # TestContainers
pnpm test:isolation          # ISO-001–010 — required when touching api/database/core
pnpm test:fuzz-tenant-id     # API-layer IDOR fuzz (ISO-FUZZ-001–005)
pnpm test:golden             # python-eval container — not a host venv
pnpm test:fuzz               # property-based tenant isolation
pnpm test:e2e                # Playwright (override STAGING_URL for remote)
pnpm test:kill-switch
pnpm test:kill-switch-idempotency   # KS-001–007
pnpm test:calibration        # the ECE gate
pnpm test:script-security    # sandbox AST pre-scan
pnpm test:governance-kernel  # GOV-001–013 — merge-blocking before Gate 1
pnpm camel-benchmark         # CaMeL defended-task-completion regression
./check-rls.sh               # RLS policy gate, with its negative fixture
pnpm test:prompt-injection
pnpm aibom:validate
pnpm scan:deps
pnpm scan:image
pnpm db:reset --confirm
pnpm test:ci
```

**The cost benchmark has no local equivalent yet.** It runs in CI only: a staging assessment averaging above **$0.55** blocks the merge ([§3.2](#32-merge-gates), D-3).

### 2.2 Python Eval Extraction Trigger

Containerized from Week 2: `docker compose up -d python-eval`. Extraction into a gRPC microservice triggers when engineering headcount exceeds 8 — a seed ops delta.

---

## 3. CI/CD Pipeline

### 3.1 Pipeline Flow

```
push
  → lint + typecheck
  → unit
  → integration
  → fuzz + isolation
  → security gates (Snyk, AIBOM, prompt injection, auth, MCP,
                    kill switch, visual, schema-decode)
  → golden set
  → build images (cosign-signed GHCR + signed SBOM)
  → deploy staging
  → smoke
```

### 3.2 Merge Gates

| # | Gate | Trigger | Blocks merge |
|---|------|---------|--------------|
| 1 | Lint + typecheck | every PR | Yes |
| 2 | Unit / integration | every PR | Yes |
| 3 | Cross-tenant isolation | PR touching `api/`, `database/`, `core/` | Yes — ISO-001–010 |
| 4 | Golden set (with stratification) | PR touching `core/`, `python-eval/` | Yes — **P0 on any aggregate or per-stratum regression above 2%** |
| 5 | Connector sync | PR touching `connectors/` | Yes |
| 6 | Kill switch | PR touching admin routes or workflows | Yes — `test:kill-switch`, `test:kill-switch-idempotency` |
| 7 | Auth | PR touching auth middleware | Yes — AUTH-001–005 |
| 8 | MCP | PR touching `mcp/` or the agent registry | Yes — MCP-001–005 |
| 9 | Prompt injection | PR touching `core/`, `prompts/`, `mcp/` | Yes — custom corpus + Promptfoo critical/high |
| 10 | AIBOM validation | every PR | Yes — manifest drift |
| 11 | **Cost benchmark** | PR touching assessment logic | Yes — **a staging assessment averaging above $0.55 blocks the merge** (D-3). Cold-cache runs (0% hit) are tracked as a P1 burn-down against a **$0.60 hard cap** (AI-75) |
| 12 | OTel GenAI lint | PR touching `observability/` or LLM call sites | Yes — `no-direct-llm-sdk` |
| 13 | RLS policy gate | PR touching `database/` | Yes — `check-rls.sh`, plus its negative fixture |
| 14 | Governance kernel | PR touching agent action paths, `core/` | Yes — `pnpm test:governance-kernel`, GOV-001–013 failure modes ([governance-kernel](../40-ai-safety/governance-kernel.md)) |
| 15 | CaMeL benchmark | PR touching the S-LLM/P-LLM boundary, `prompts/` | Yes — `camel-benchmark`; a defended-task-completion regression blocks ([camel-plane](../40-ai-safety/camel-plane.md)) |
| 16 | `no-aws-sdk-outside-adapters` / npm audit on the Bedrock SDK tree | every PR | Yes — supply chain |
| 17 | Full-tree SBOM + CI-runner credential hygiene | every PR, Gate 1 | Yes — CycloneDX SBOM, Sigstore/SLSA provenance, scoped CI-runner tokens across the full npm/pip tree (H11, promoted from fast-follow; panel review Q4, §5) |
| 18 | Model version check | deploy | blocks if the staging model ≠ the `models.json` pin |
| 19 | Shadow-mode evaluation | candidate agent/model version, pre-promotion | Yes — blocks promotion below the pinned version's golden-set accuracy (DA-10, see §3.9) |
| 20 | Canary rollout | promotion of a shadow-cleared candidate | Yes — progressive 5%→25%→100% traffic ramp, auto-rollback on regression (DA-10, see §3.9) |
| 21 | E2E smoke | merge to `main` | Yes |
| 22 | Coverage drop on critical paths | every PR | warning above 2%; **blocks at −5%** |
| 23 | Docs referential integrity | PR touching `docs/` | Yes — `python3 scripts/validate-playbooks.py` exit 0: dead links and anchors, duplicate canonical IDs, task/epic/portfolio hour reconciliation, the front-matter contract, and no change history in spec prose (D-12) |

### 3.3 Cost Benchmark Gate (D-3, AI-75)

A staging assessment averaging above **$0.55** blocks the merge. Cold-cache runs (0% hit) are tracked as a P1 burn-down against a **$0.60 hard cap**. The cost envelope ($0.75/assessment hard, $0.55 design) is derived on a 45% cache-hit assumption (see §3.10).

### 3.4 Golden Set — (CVE × Environment) Pairs (H1)

**The unit of evaluation is a (CVE × synthetic-environment) pair, with a per-environment ground-truth verdict** — exploitable *here*, or not. **It is not a CVE with a fixed label.**

Exploitability is a property of the pair. **A CVE-only set validates CVE triage — which is precisely what Dux says it is not.**

**Adversarial environment fixtures are first-class eval cases:** the same CVE with the control present and absent; the asset reachable and unreachable. **Without them, ">80% accuracy" does not back the marketing claim.**

**Composition.** Frozen in `tests/golden/`, with a `sha256` manifest, locked in Week 1 (`test:golden --verify-lock`). The CVE axis, across 250 CVEs:

| Dimension | Distribution |
|-----------|--------------|
| CVSS | ~25 per decile bin |
| KEV | 40% KEV / 60% non-KEV |
| Exploit maturity | 30% functional / 50% PoC / 20% theoretical |
| Asset count | 30% single / 50% multiple / 20% widespread |
| Control complexity | 40 / 40 / 20 |

Each is then crossed with the environment fixtures above.

### 3.5 Accuracy Floors — Merge-Blocked

| Milestone | Floor |
|-----------|-------|
| Month 1 | ≥65% |
| Month 2 | ≥75% |
| Month 3 / Gate 1 | ≥80% (held-out set) |

**A regression above 2% against the held-out set is a P0 block** (NFR-008 → TR-NFR-007).

**Per-stratum block:** no single decile, KEV, or maturity slice may regress more than 2%, **even when the aggregate is within 2%** (`test:golden --stratified`). **Gate 1 uses the held-out set only.**

### 3.6 ECE Gate and DeepEval Metrics

ECE gate: ≤0.15 at n ≥ 50. DeepEval metrics run nightly as advisory: tool correctness >90%, task completion >95%, hallucination <2%, faithfulness >85%. **False-positive rate <5% on the golden set is the exception — it blocks merge if exceeded** (DA-09), matching the per-stratum regression-gate treatment above rather than the advisory-only cadence of the other four metrics.

> **Schema parity alone is a known blind spot.** "Drift or Dice" (Zenodo, 2026) documents schema checks reporting **zero regressions** even as output silently shrank 15.7% and a tier failed 25% of hard tasks. **Model migrations must additionally measure output-length, latency, and tier-failure deltas** — see the [model-EOL runbook](../40-ai-safety/incident-runbooks.md).

---

## 4. Mandatory Security Suites

### 4.1 Isolation Suite (ISO-001–010)

404 masking (**not 403**), fail-closed behavior, SQL injection cannot bypass RLS, cache/search/quota scoping, and a body-versus-path `tenant_id` contradiction → 403. ISO-013 covers the `tenant_cve_findings` materialized-view refresh.

### 4.2 Tenant-ID Fuzz (ISO-FUZZ-001–005)

Fast-check, at the HTTP IDOR layer: swapped, missing, and malformed `tenant_id`; body/path contradiction; admin impersonation without scope and MFA. **Any cross-tenant read or write is a merge block.**

### 4.3 Auth (AUTH-001–005)

Invalid and expired JWT; missing tenant claim; role enforcement; API-key 401.

### 4.4 Kill Switch (KS-001–007)

| Test | Assertion |
|------|-----------|
| KS-001 | L2–L4 propagation <5 s p99 |
| KS-L1 | ≤30 s, via Unleash |
| KS-002 | L3 plus status copy |
| KS-003 | L2 |
| KS-004 | L4 — monthly chaos |
| KS-005 | audit |
| KS-006 | 429 |
| KS-007 | CloudNativePG `LISTEN/NOTIFY` <10 s |

### 4.5 MCP (MCP-001–005)

Unregistered server denied; schema drift blocked; PII redacted; egress allowlist enforced; cross-tenant JWT rejected.

### 4.6 Prompt Injection — Three Layers

| Layer | Cadence | Pass condition |
|-------|---------|----------------|
| Custom corpus | every PR | blocks |
| Promptfoo | PR touching agent or LLM code | blocks on critical/high. Cost cap $5/PR |
| Garak | nightly | probe subset — jailbreak, indirect injection, prompt extraction, encoding bypass. **0 critical, ≤2 new high** |

### 4.7 Additional Security Checks

- Error handling ERR-001–004
- Script security (`test:script-security`)
- Re-assessment triggers

### 4.8 Error Handling Standard (DuxErrorCode)

`DuxErrorCode` is the shared vocabulary across REST, SSE, and webhooks ([application-api §4](../30-api/application-api.md)) — new error paths extend that enum rather than inventing ad hoc codes. Never silently swallow a governance-kernel or connector failure; it either maps to a `DuxErrorCode` or raises the appropriate audit event.

### 4.9 `ExploitabilityAssessmentWorkflow` Test Scenarios

Checklist (2026-07-20, resolves part of [OI-43](../00-meta/open-items.md)). No `pnpm test:*` script or test file exists for this yet — this is a checklist for whoever writes it, not the suite itself. Each scenario cites the spec line it's derived from:

| Scenario | Assertion | Spec source |
|----------|-----------|--------------|
| Happy path | plan → retrieve → reason → decide → synthesize completes, verdict persisted | [workflows.md](../20-architecture/workflows.md) §3a |
| Max-turns exit | workflow aborts cleanly at the 10-turn cap, no partial-write leak | [architecture-overview.md](../20-architecture/architecture-overview.md) — "a bounded `while` loop, max 10 turns" |
| Budget-exceeded abort | `BUDGET_EXCEEDED` fires and aborts at the $0.75/assessment ceiling | [architecture-overview.md](../20-architecture/architecture-overview.md) — budget-check activity, $0.75 ceiling |
| Tool failure + retry | a failed tool call retries as its own Temporal activity, doesn't fail the whole workflow | [architecture-overview.md](../20-architecture/architecture-overview.md) — "each tool call is its own retryable Temporal activity" |
| Bedrock timeout | Converse API timeout is caught and retried/surfaced, not silently swallowed | Bedrock Converse API direct integration (ADR-021) |
| Malformed tool response | schema-validated tool-use rejects a malformed response without corrupting `assessment_messages` state | [workflows.md](../20-architecture/workflows.md) §3a; ADR-020 R2 constrained decoding |
| HITL signal timeout | **30 min**, not 24 h — the only HITL signal timeout actually documented anywhere in this corpus is the T4 escalation timeout ([kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md): "a T4 request times out at 30 min to an automatic L2"). The "24 h" figure in earlier open-item text was never corroborated by any spec and is corrected here, not carried forward. | [kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md) |

---

## 5. Additional CI Checks

| Check | When | Target |
|-------|------|--------|
| k6 load | Week 6 | 500 assessments/day; API p95 <300 ms by Week 8 |
| Chromatic visual regression | Week 8 | US-010, US-011, US-012 exposure states |
| llguidance constrained decoding | Week 8 | S-LLM prerequisites; retry rate <2%; Zod + retry fallback |

### 5.1 Test Traceability

| Requirement | Coverage |
|-------------|----------|
| BR-001 / NFR-001 | ISO + fuzz |
| BR-003 / NFR-005 | KS-001–005 |
| NFR-008 | Golden set + DeepEval |
| AUTH / ADR-001 | AUTH-001–005 |

### 5.2 Accessibility on a Streaming UI (H10, fast-follow)

**axe-core reports 0 violations and still misses the real problem.** It covers static contrast, but **it cannot see an ARIA live-region announcement storm** — and a screen reader announcing every streaming token and status change is unusable.

The streaming-heavy surfaces (`REASONING` / `TOOL_CALLING` / `EVALUATING`, plus live exposure states) therefore require:

- `aria-live="polite"` with **debounced** status announcements.
- A **"reasoning complete" summary**, instead of token-by-token narration.
- **Manual screen-reader testing** in the Gate-1 accessibility check.

**axe-core alone does not validate the WCAG 2.2 AA claim for a streaming UI.**

### 5.3 Shadow Mode Details (DA-10)

The model-version-check gate (§3.2 #18) is a binary staging-vs-pin comparison — it blocks an unpinned version from reaching staging, but it says nothing about how a *new* pinned version earns promotion. Shadow mode closes that gap:

- The candidate version runs in parallel on sampled production traffic, alongside the currently pinned version.
- Its outputs are scored against the golden set but **never delivered to the tenant** — the pinned version's output remains the one returned.
- Promotion requires the candidate's golden-set accuracy to meet or exceed the pinned version's accuracy from §3.4, at the same stratification granularity (no per-stratum regression above 2%).

### 5.4 Canary Rollout Details (DA-10)

Once shadow-cleared, the candidate is promoted behind a **5% → 25% → 100% traffic ramp over a 7-day window**, with the cost, accuracy, and kill-switch-rate dashboards from [observability-slo §3](../60-operations/observability-slo.md) monitored at each step. A regression at any step halts the ramp and rolls back to the prior pin; it does not proceed to the next percentage automatically.

### 5.5 Cache-Hit-Rate Validation (A1, CI-07 Closure Spec)

**What this validates.** ADR-008 R2's cost envelope (≤$0.75/assessment hard, ≤$0.55 design) is derived **on a 45% cache-hit assumption** — prompt caching plus the ADR-010 R5 semantic cache. That assumption has never been measured against the golden set; it has only been assumed. This is the validation methodology for `US-001-T14` (CI-07 A1 closure), not the implementation itself.

**Method.** Run the full 250-CVE golden set (§3.4) with cache instrumentation enabled (`InstrumentedLLMClient` already emits per-call cache-hit/miss — [observability-slo §2](../60-operations/observability-slo.md)), and compute the **held-out hit rate per CaMeL tier** (S-LLM, P-LLM — ADR-008 R2), not a single blended figure — the two tiers have different cache-ability profiles (S-LLM's untrusted-CVE-text calls are the tenant-invariant semantic-cache candidates; P-LLM's customer-context calls use exact-key prompt caching only, per ADR-010 R5 §"restricted to tenant-invariant calls").

**Pass criteria:**

| Check | Threshold | On failure |
|-------|-----------|------------|
| S-LLM tier hit rate | ≥40% (within 5 pts of the 45% design assumption) | Re-derive the cost gates (D-3) against the measured rate, not the assumed one — this is a cost-model correction, not a code bug |
| P-LLM tier hit rate | ≥40% | Same |
| Blended cost per assessment | within the existing $0.55 design / $0.75 hard gates (§3.2) | Existing merge-gate failure path applies — this task doesn't add a new gate, it validates the input to the existing one |

**This is a one-time validation plus a recurring check**, not a new permanent CI gate: run once against the golden set to confirm or correct the 45% assumption, then re-run on any material change to prompt structure, model pin, or the semantic-cache threshold (ADR-010 R5's 0.95 similarity cutoff).

### 5.6 Full-Tree Supply Chain (H11, Gate 1)

**2026-07-19, D-34: LiteLLM is retired** ([ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer)) — Claude inference now goes through the direct Bedrock SDK (`@aws-sdk/client-bedrock-runtime`) behind `LLMProviderPort`, an ordinary first-party npm dependency subject to the same full-tree scrutiny as everything else in this section, not a special-cased proxy with its own signed-GHCR-digest/cosign pipeline. The AST scanner pulls in `@babel/parser` and tree-sitter, and the entire npm and pip tree is attack surface regardless of which single dependency prompted the original hardening.

**Promoted from fast-follow to Gate-1 blocking** (panel review Q4). The "Mini Shai-Hulud" npm/PyPI supply-chain worm (April 29 – May 2026) compromised 170+ packages — **including TanStack, Dux's own frontend framework** — via stolen CI-runner credentials, and it did so while **defeating SLSA Build L3 provenance**. Dux's stack was in the blast radius, and the attack beat the exact class of control this section relies on; a fast-follow timeline is no longer defensible.

Gate-1 requires:

- **A CycloneDX SBOM for the full application dependency tree.** (The AIBOM already uses CycloneDX 1.6 for AI components.)
- **Sigstore / SLSA provenance verification in CI.**
- **CI-runner credential hygiene** — short-lived, scoped tokens for `@tanstack/*`, `@aws-sdk/*`, and the rest of the npm/pip tree.
- **Renovate and Dependabot under the same signed-digest discipline.**

**A security vendor compromised through its own npm tree is the worst available headline.**

---

## 6. Coding Standards

### 6.1 Language and Style

**Language.** TypeScript strict mode across `packages/*` (NestJS API, React + Vite web, Temporal workers, Pulumi infra). Python only in `packages/python-eval` (DeepEval, Evidently, calibration) — containerized from Week 2; no host venv (see [§1.2](#12-clone-and-install)).

**Naming.** `camelCase` for variables/functions, `PascalCase` for classes/types/components, `UPPER_SNAKE_CASE` for constants, `SCREAMING_SNAKE` singular for ERD entities against `snake_case` plural physical tables ([data-model §5](../20-architecture/data-model.md)).

**Immutability.** Prefer returning new objects over in-place mutation, particularly in `packages/core/world-model` and anywhere touching `AsyncLocalStorage`-scoped `TenantContext` — a mutated shared context is a cross-tenant-leak-shaped bug class, not just a style preference.

**File organization.** Feature/domain folders under `packages/*` ([architecture-overview §4](../20-architecture/architecture-overview.md)), not by type. Target 200–400 lines per file; extract before a file exceeds ~800.

### 6.2 Custom Lint Rules

Each rule is normative in its owning spec — this table is the index, not a second source of truth:

| Rule | Enforces | Defined in |
|------|----------|-----------|
| `import/no-restricted-paths` | only `packages/adapters/*` imports a vendor SDK | [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) |
| `no-direct-llm-sdk` | every LLM call routes through `InstrumentedLLMClient` | [observability-slo §2](../60-operations/observability-slo.md) |
| `no-aws-sdk-outside-adapters` | Bedrock SDK (`@aws-sdk/client-bedrock-runtime`) is imported only inside `packages/adapters/*`, same boundary as every other vendor SDK | [ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer), [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) |
| `no-direct-cves-query` | `cves` is global and read-only; join through `findings` | [ADR-002](../20-architecture/adr-index.md#adr-002-r2--multi-tenancy) |
| `no-raw-findone` | every lookup goes through `TenantScopedRepository<T>`, never a raw `findOne` | [ADR-002](../20-architecture/adr-index.md#adr-002-r2--multi-tenancy) |

**No net-new `eslint-disable` in `api/` or `core/`** — CI blocks the merge (ADR-013).

---

## 7. Branching, Review, and Release

### 7.1 Branching Model

**Trunk-based**, short-lived feature branches off `main`. An ephemeral CloudNativePG cluster is created per PR ([ADR-003 R2](../20-architecture/adr-index.md#adr-003-r2--database-migrations)) — 7-day stale cleanup, alert above 20 live ephemeral clusters.

**Merge strategy:** squash-merge to `main`. The squashed commit message follows Conventional Commits, matching the PR title.

**Conventional Commits list:**

| Prefix | Usage |
|--------|-------|
| `feat:` | new feature |
| `fix:` | bug fix |
| `refactor:` | code restructuring with no behavior change |
| `docs:` | documentation only |
| `test:` | adding or updating tests |
| `chore:` | tooling, config, maintenance |
| `perf:` | performance improvement |
| `ci:` | CI/CD changes |

Body explains *why*, not *what* — the diff already shows what changed.

### 7.2 CODEOWNERS Gates

| Path | Required reviewer(s) | Source |
|------|----------------------|--------|
| `packages/agents/tools/` and `instructions.md` | `@dux-security` | [ADR-009](../20-architecture/adr-index.md#adr-009--agent-as-directory-registry) |
| `packages/database/migrations/` (any `down` migration) | Engineering Lead + on-call | [runbooks §5](../60-operations/runbooks.md) |

### 7.3 Review SLA and Reviewer Checklist

**Every PR passes the full merge-gate pipeline** (§3.2) before human review is requested — lint, typecheck, unit/integration, isolation/fuzz, the security-gate suite, and the golden set where applicable. Human review is not a substitute for a red gate.

**Review SLA:** first response within 1 business day; re-review after changes within 4 business hours. A PR touching `core/`, `api/auth`, or `packages/agents/tools/` requires a second reviewer from outside the immediate feature team.

**What a reviewer checks, beyond the automated gates:**

- The change traces to a BR/Epic/US/FR ID ([traceability-matrix](../00-meta/traceability-matrix.md)), or is explicitly infra/chore with no story to trace to.
- No new `eslint-disable`, no new TBD/placeholder left in shipped code.
- Tests assert behavior, not implementation — table-driven for connector/adapter logic, matching the Repository-pattern seams already in use.
- A tenant-scoped query uses `TenantScopedRepository<T>` / the composite `(tenant_id, id)` key — never a raw lookup by `id` alone.

**Release gate.** `release-gate.yml` requires Engineering Lead sign-off before tagging, on top of the automated checks (§8.1).

### 7.4 Hotfix Path

A P0 hotfix lands within 30 minutes with 1 approval; P0 may bypass the normal review SLA, never the merge gates in [ci-cd-testing](ci-cd-testing.md).

### 7.5 Contribution Guide

**Setup:** clone, `pnpm install`, `docker compose up`, seed data, and the local test suite (see §1).

**Before opening a PR:**

1. Run `pnpm test:ci` locally — the same suite CI runs, minus staging-only checks (cost benchmark, k6 load, Chromatic).
2. Confirm `pnpm test:isolation` passes if the change touches `api/`, `database/`, or `core/`.
3. Write the PR description against the template: what changed, the BR/FR/US it closes (if any), and a test plan.
4. Keep the PR scoped to one logical change — a bundled refactor-plus-feature PR is harder to review and to revert.

**Commit messages** follow Conventional Commits (§7.1). Body explains *why*, not *what*.

### 7.6 First PR Checklist

For a new engineer: local dev stack running (`pnpm dev:all`), `pnpm test:isolation` green, read [architecture-overview §1–5](../20-architecture/architecture-overview.md) and [ADR-013](../20-architecture/adr-index.md#adr-013--provider-ports) before touching a provider port.

---

## 8. Release

### 8.1 Release Gate

`release-gate.yml` automates or prompts each check, and requires Engineering Lead sign-off before tagging.

Images are cosign-signed with an attached SBOM. **The deploy gate verifies the digest and the signature before staging or production.**

---

## 9. Common Failure Modes

| Symptom | Fix |
|---------|-----|
| `RLS policy violation`, or empty results | the middleware is not setting `app.tenant_id` — check the JWT tenant claim |
| Cross-tenant test failures | `pnpm db:reset --confirm && pnpm --filter database db:seed` |
| The workflow worker picks up no tasks | verify CloudNativePG is reachable; check `DATABASE_URL`; confirm the worker process is running in `packages/core` |
| NVD sync is stale | check the API key; verify the S3/MinIO cache holds the raw JSON |
| The AWS connector fails locally | `docker compose --profile aws up -d localstack`; set `AWS_ENDPOINT_URL`; mock IAM in tenant settings |
| A cache key appears cross-tenant | keys must use the `tenant:{HMAC-SHA256(tenant_id)[:16]}:` prefix |
| A long assessment is cancelled after 24 h | `WorldModelVersionPurgeJob` ran on a superseded version — inspect with `pnpm admin:world-model-version` |

---

## Sources

- `.raw/dux/50-engineering/engineering-standards.md`
- `.raw/dux/50-engineering/ci-cd-testing.md`
- `.raw/dux/50-engineering/local-development.md`
