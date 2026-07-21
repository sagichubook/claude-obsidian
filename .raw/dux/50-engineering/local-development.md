---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-3, D-34]
---

# Local Development

**Purpose:** run the full stack locally — NestJS API, Temporal workflow workers, PostgreSQL with RLS, connectors, and the TanStack dashboard — including cross-tenant isolation testing.

**Parents:** all epics.

## 1. Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | 22 LTS | API, web, workflow workers (`.nvmrc` + `engines`) |
| pnpm | 9+ | `corepack enable && corepack prepare pnpm@9.15.0 --activate` |
| Docker Desktop | latest | the full infrastructure stack |
| Python | 3.11+ | optional before Week 2. **After Week 2, use the `python-eval` container only** |
| Git | 2.40+ | source control |

## 2. Clone and install

```bash
git clone git@github.com:duxsecurity/dux.git dux
cd dux && pnpm install
cp .env.example .env.local
```

**(Resolves [OI-23](../00-meta/open-items.md).)** `github.com/duxsecurity/dux.git` is the canonical private monorepo — `github.com/duxsec` is a different, unrelated account and must never be used. Engineers without access should request it through the standard IAM onboarding path, not by searching for alternate org names.

**Environment loading.** The root `.env.local` is loaded by NestJS (`packages/api`), Vite (`packages/web`), and the workflow workers (`packages/core`), all through `dotenv/config`.

**`DATABASE_URL` lives in the root only** — workers inherit it through the turbo env.

## 3. Environment variables

Local defaults:

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

## 4. Start infrastructure and services

```bash
cd infra
docker compose up -d          # Postgres, PgBouncer, Valkey, MinIO, Vault, Unleash
docker compose --profile aws up -d localstack   # optional: AWS connector testing
pnpm infra:wait
pnpm --filter database migration:run            # applies RLS + FORCE
pnpm --filter database db:seed
docker compose up -d python-eval                # Week 2+ — canonical for the golden set
```

```bash
pnpm dev:all                  # everything at once
# or individually:
#   pnpm --filter api dev          → :3001
#   pnpm --filter web dev          → :3000
#   pnpm --filter core dev         → Temporal worker
#   pnpm --filter connectors dev   → :3002
```

**Verify:** web on `:3000`; API on `:3001/health`; connectors on `:3002/health`.

The Temporal worker connects to the self-hosted Temporal cluster's dev namespace (K8s dev environment, ADR-007 R3), or to a local Temporalite. **Local development must never expose a production workflow UI.**

### 4a. First-run setup notes (D-57)

Confirmed (Founder, 2026-07-21) as matching the team's actual local-dev workflow.

- **Temporal.** If not running the full K8s dev namespace, the standard OSS bootstrap is the Temporal CLI's embedded dev server: `temporal server start-dev` — starts a local Temporal server plus Web UI (default `localhost:8233`) with no external dependencies, sufficient for `pnpm --filter core dev` to connect against. This is what "a local Temporalite" above refers to; `temporalite` itself is deprecated upstream in favor of `temporal server start-dev`.
- **Vault.** `docker compose up -d vault` alone does not make `VAULT_TOKEN` usable — Vault starts sealed by default. For local dev only, run Vault in dev mode (`vault server -dev`, or the container's dev-mode equivalent), which auto-unseals and prints a root token to use as `VAULT_TOKEN`. **Dev mode stores everything in memory and is never a template for the production init/unseal flow** (which uses Shamir key shares and persistent storage) — production unseal steps are out of scope for this file.
- **Bedrock local auth (`LLM_ROUTING_PROFILE=local`).** The direct Bedrock SDK path (ADR-010 R5, D-34) authenticates via the standard AWS credential chain — typically `aws configure sso` against the org's AWS SSO, or a named profile (`AWS_PROFILE=<profile>`) with Bedrock invoke permissions in the target region. No Bedrock-specific env var beyond the standard AWS ones is required.
- **`NVD_API_KEY` and AWS connector credentials.** These are per-developer secrets with no fixed source in this repo. `NVD_API_KEY` is requested from NVD directly (api.nvd.nih.gov key request form); AWS connector credentials for local connector testing come from the same AWS SSO/profile used for Bedrock above, scoped to the connector's IAM role.

## 5. Tests

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

**The cost benchmark has no local equivalent yet.** It runs in CI only: a staging assessment averaging above **$0.55** blocks the merge ([ci-cd-testing §2](ci-cd-testing.md), D-3).

## 6. Common failure modes

| Symptom | Fix |
|---------|-----|
| `RLS policy violation`, or empty results | the middleware is not setting `app.tenant_id` — check the JWT tenant claim |
| Cross-tenant test failures | `pnpm db:reset --confirm && pnpm --filter database db:seed` |
| The workflow worker picks up no tasks | verify CloudNativePG is reachable; check `DATABASE_URL`; confirm the worker process is running in `packages/core` |
| NVD sync is stale | check the API key; verify the S3/MinIO cache holds the raw JSON |
| The AWS connector fails locally | `docker compose --profile aws up -d localstack`; set `AWS_ENDPOINT_URL`; mock IAM in tenant settings |
| A cache key appears cross-tenant | keys must use the `tenant:{HMAC-SHA256(tenant_id)[:16]}:` prefix |
| A long assessment is cancelled after 24 h | `WorldModelVersionPurgeJob` ran on a superseded version — inspect with `pnpm admin:world-model-version` |

## 7. Python eval

Containerized from Week 2: `docker compose up -d python-eval`.

Extraction into a gRPC microservice triggers when engineering headcount exceeds 8 — a seed ops delta.
