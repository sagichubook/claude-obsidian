---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-3, D-34, H1, H10, H11]
---

# CI/CD & Testing

**Purpose:** the merge-to-staging pipeline and the mandatory security suites.

**Parents:** BR-001, BR-002, BR-003.

Production release management — SemVer, blue-green — is a seed-stage concern: see [operations runbooks](../60-operations/runbooks.md).

## 1. Pipeline

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

## 2. Merge gates

| Gate | Trigger | Blocks merge |
|------|---------|--------------|
| Lint + typecheck | every PR | Yes |
| Unit / integration | every PR | Yes |
| Cross-tenant isolation | PR touching `api/`, `database/`, `core/` | Yes — ISO-001–010 |
| Golden set (with stratification) | PR touching `core/`, `python-eval/` | Yes — **P0 on any aggregate or per-stratum regression above 2%** |
| Connector sync | PR touching `connectors/` | Yes |
| Kill switch | PR touching admin routes or workflows | Yes — `test:kill-switch`, `test:kill-switch-idempotency` |
| Auth | PR touching auth middleware | Yes — AUTH-001–005 |
| MCP | PR touching `mcp/` or the agent registry | Yes — MCP-001–005 |
| Prompt injection | PR touching `core/`, `prompts/`, `mcp/` | Yes — custom corpus + Promptfoo critical/high |
| AIBOM validation | every PR | Yes — manifest drift |
| **Cost benchmark** | PR touching assessment logic | Yes — **a staging assessment averaging above $0.55 blocks the merge** (D-3). Cold-cache runs (0% hit) are tracked as a P1 burn-down against a $0.60 hard cap (AI-75) |
| OTel GenAI lint | PR touching `observability/` or LLM call sites | Yes — `no-direct-llm-sdk` |
| RLS policy gate | PR touching `database/` | Yes — `check-rls.sh`, plus its negative fixture |
| Governance kernel | PR touching agent action paths, `core/` | Yes — `pnpm test:governance-kernel`, GOV-001–013 failure modes ([governance-kernel](../40-ai-safety/governance-kernel.md)) |
| CaMeL benchmark | PR touching the S-LLM/P-LLM boundary, `prompts/` | Yes — `camel-benchmark`; a defended-task-completion regression blocks ([camel-plane](../40-ai-safety/camel-plane.md)) |
| `no-aws-sdk-outside-adapters` / npm audit on the Bedrock SDK tree | every PR | Yes — supply chain |
| Full-tree SBOM + CI-runner credential hygiene | every PR, Gate 1 | Yes — CycloneDX SBOM, Sigstore/SLSA provenance, scoped CI-runner tokens across the full npm/pip tree (H11, promoted from fast-follow; panel review Q4, §5) |
| Model version check | deploy | blocks if the staging model ≠ the `models.json` pin |
| Shadow-mode evaluation | candidate agent/model version, pre-promotion | Yes — blocks promotion below the pinned version's golden-set accuracy (DA-10, see §5) |
| Canary rollout | promotion of a shadow-cleared candidate | Yes — progressive 5%→25%→100% traffic ramp, auto-rollback on regression (DA-10, see §5) |
| E2E smoke | merge to `main` | Yes |
| Coverage drop on critical paths | every PR | warning above 2%; **blocks at −5%** |
| Docs referential integrity | PR touching `docs/` | Yes — `python3 scripts/validate-playbooks.py` exit 0: dead links and anchors, duplicate canonical IDs, task/epic/portfolio hour reconciliation, the front-matter contract, and no change history in spec prose (D-12) |

## 3. Golden set — (CVE × environment) pairs (H1)

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

**Accuracy floors — merge-blocked:**

| Milestone | Floor |
|-----------|-------|
| Month 1 | ≥65% |
| Month 2 | ≥75% |
| Month 3 / Gate 1 | ≥80% (held-out set) |

**A regression above 2% against the held-out set is a P0 block** (NFR-008 → TR-NFR-007).

**Per-stratum block:** no single decile, KEV, or maturity slice may regress more than 2%, **even when the aggregate is within 2%** (`test:golden --stratified`). **Gate 1 uses the held-out set only.**

ECE gate: ≤0.15 at n ≥ 50. DeepEval metrics run nightly as advisory: tool correctness >90%, task completion >95%, hallucination <2%, faithfulness >85%. **False-positive rate <5% on the golden set is the exception — it blocks merge if exceeded** (DA-09), matching the per-stratum regression-gate treatment above rather than the advisory-only cadence of the other four metrics.

> **Schema parity alone is a known blind spot.** "Drift or Dice" (Zenodo, 2026) documents schema checks reporting **zero regressions** even as output silently shrank 15.7% and a tier failed 25% of hard tasks. **Model migrations must additionally measure output-length, latency, and tier-failure deltas** — see the [model-EOL runbook](../40-ai-safety/incident-runbooks.md).

## 4. Mandatory suites

**Isolation — ISO-001–010.** 404 masking (**not 403**), fail-closed behavior, SQL injection cannot bypass RLS, cache/search/quota scoping, and a body-versus-path `tenant_id` contradiction → 403. ISO-013 covers the `tenant_cve_findings` materialized-view refresh.

**Tenant-ID fuzz — ISO-FUZZ-001–005** (fast-check, at the HTTP IDOR layer): swapped, missing, and malformed `tenant_id`; body/path contradiction; admin impersonation without scope and MFA. **Any cross-tenant read or write is a merge block.**

**Auth — AUTH-001–005.** Invalid and expired JWT; missing tenant claim; role enforcement; API-key 401.

**Kill switch.**

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

**MCP — MCP-001–005.** Unregistered server denied; schema drift blocked; PII redacted; egress allowlist enforced; cross-tenant JWT rejected.

**Prompt injection — three layers.**

| Layer | Cadence | Pass condition |
|-------|---------|----------------|
| Custom corpus | every PR | blocks |
| Promptfoo | PR touching agent or LLM code | blocks on critical/high. Cost cap $5/PR |
| Garak | nightly | probe subset — jailbreak, indirect injection, prompt extraction, encoding bypass. **0 critical, ≤2 new high** |

**Also:** error handling ERR-001–004; script security (`test:script-security`); re-assessment triggers.

**`ExploitabilityAssessmentWorkflow` test-scenario checklist (2026-07-20, resolves part of [OI-43](../00-meta/open-items.md)).** No `pnpm test:*` script or test file exists for this yet — this is a checklist for whoever writes it, not the suite itself. Each scenario cites the spec line it's derived from:

| Scenario | Assertion | Spec source |
|----------|-----------|--------------|
| Happy path | plan → retrieve → reason → decide → synthesize completes, verdict persisted | [workflows.md](../20-architecture/workflows.md) §3a |
| Max-turns exit | workflow aborts cleanly at the 10-turn cap, no partial-write leak | [architecture-overview.md](../20-architecture/architecture-overview.md) — "a bounded `while` loop, max 10 turns" |
| Budget-exceeded abort | `BUDGET_EXCEEDED` fires and aborts at the $0.75/assessment ceiling | [architecture-overview.md](../20-architecture/architecture-overview.md) — budget-check activity, $0.75 ceiling |
| Tool failure + retry | a failed tool call retries as its own Temporal activity, doesn't fail the whole workflow | [architecture-overview.md](../20-architecture/architecture-overview.md) — "each tool call is its own retryable Temporal activity" |
| Bedrock timeout | Converse API timeout is caught and retried/surfaced, not silently swallowed | Bedrock Converse API direct integration (ADR-021) |
| Malformed tool response | schema-validated tool-use rejects a malformed response without corrupting `assessment_messages` state | [workflows.md](../20-architecture/workflows.md) §3a; ADR-020 R2 constrained decoding |
| HITL signal timeout | **30 min**, not 24 h — the only HITL signal timeout actually documented anywhere in this corpus is the T4 escalation timeout ([kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md): "a T4 request times out at 30 min to an automatic L2"). The "24 h" figure in earlier open-item text was never corroborated by any spec and is corrected here, not carried forward. | [kill-switch-hitl.md](../40-ai-safety/kill-switch-hitl.md) |

## 5. Additional CI

| Check | When | Target |
|-------|------|--------|
| k6 load | Week 6 | 500 assessments/day; API p95 <300 ms by Week 8 |
| Chromatic visual regression | Week 8 | US-010, US-011, US-012 exposure states |
| llguidance constrained decoding | Week 8 | S-LLM prerequisites; retry rate <2%; Zod + retry fallback |

**Test traceability:** BR-001 / NFR-001 → ISO + fuzz. BR-003 / NFR-005 → KS-001–005. NFR-008 → golden set + DeepEval. AUTH / ADR-001 → AUTH-001–005.

### Accessibility on a streaming UI (H10, fast-follow)

**axe-core reports 0 violations and still misses the real problem.** It covers static contrast, but **it cannot see an ARIA live-region announcement storm** — and a screen reader announcing every streaming token and status change is unusable.

The streaming-heavy surfaces (`REASONING` / `TOOL_CALLING` / `EVALUATING`, plus live exposure states) therefore require:

- `aria-live="polite"` with **debounced** status announcements.
- A **"reasoning complete" summary**, instead of token-by-token narration.
- **Manual screen-reader testing** in the Gate-1 accessibility check.

**axe-core alone does not validate the WCAG 2.2 AA claim for a streaming UI.**

### Shadow mode and canary rollout for candidate agent/model versions (DA-10)

The model-version-check gate in §2 is a binary staging-vs-pin comparison — it blocks an unpinned version from reaching staging, but it says nothing about how a *new* pinned version earns promotion. Two additional stages close that gap:

- **Shadow mode.** The candidate version runs in parallel on sampled production traffic, alongside the currently pinned version. Its outputs are scored against the golden set but **never delivered to the tenant** — the pinned version's output remains the one returned. Promotion requires the candidate's golden-set accuracy to meet or exceed the pinned version's accuracy from §3, at the same stratification granularity (no per-stratum regression above 2%).
- **Canary / progressive rollout.** Once shadow-cleared, the candidate is promoted behind a **5% → 25% → 100% traffic ramp over a 7-day window**, with the cost, accuracy, and kill-switch-rate dashboards from [observability-slo §3](../60-operations/observability-slo.md) monitored at each step. A regression at any step halts the ramp and rolls back to the prior pin; it does not proceed to the next percentage automatically.

### Cache-hit-rate validation (A1, CI-07 closure spec)

**What this validates.** ADR-008 R2's cost envelope (≤$0.75/assessment hard, ≤$0.55 design) is derived **on a 45% cache-hit assumption** — prompt caching plus the ADR-010 R5 semantic cache. That assumption has never been measured against the golden set; it has only been assumed. This is the validation methodology for `US-001-T14` (CI-07 A1 closure), not the implementation itself.

**Method.** Run the full 250-CVE golden set (§3) with cache instrumentation enabled (`InstrumentedLLMClient` already emits per-call cache-hit/miss — [observability-slo §2](../60-operations/observability-slo.md)), and compute the **held-out hit rate per CaMeL tier** (S-LLM, P-LLM — ADR-008 R2), not a single blended figure — the two tiers have different cache-ability profiles (S-LLM's untrusted-CVE-text calls are the tenant-invariant semantic-cache candidates; P-LLM's customer-context calls use exact-key prompt caching only, per ADR-010 R5 §"restricted to tenant-invariant calls").

**Pass criteria:**

| Check | Threshold | On failure |
|-------|-----------|------------|
| S-LLM tier hit rate | ≥40% (within 5 pts of the 45% design assumption) | Re-derive the cost gates (D-3) against the measured rate, not the assumed one — this is a cost-model correction, not a code bug |
| P-LLM tier hit rate | ≥40% | Same |
| Blended cost per assessment | within the existing $0.55 design / $0.75 hard gates (§2) | Existing merge-gate failure path applies — this task doesn't add a new gate, it validates the input to the existing one |

**This is a one-time validation plus a recurring check**, not a new permanent CI gate: run once against the golden set to confirm or correct the 45% assumption, then re-run on any material change to prompt structure, model pin, or the semantic-cache threshold (ADR-010 R5's 0.95 similarity cutoff).

### Full-tree supply chain (H11, Gate 1)

**2026-07-19, D-34: LiteLLM is retired** ([ADR-010 R5](../20-architecture/adr-index.md#adr-010-r5--llm-routing-layer)) — Claude inference now goes through the direct Bedrock SDK (`@aws-sdk/client-bedrock-runtime`) behind `LLMProviderPort`, an ordinary first-party npm dependency subject to the same full-tree scrutiny as everything else in this section, not a special-cased proxy with its own signed-GHCR-digest/cosign pipeline. The AST scanner pulls in `@babel/parser` and tree-sitter, and the entire npm and pip tree is attack surface regardless of which single dependency prompted the original hardening.

**Promoted from fast-follow to Gate-1 blocking** (panel review Q4). The "Mini Shai-Hulud" npm/PyPI supply-chain worm (April 29 – May 2026) compromised 170+ packages — **including TanStack, Dux's own frontend framework** — via stolen CI-runner credentials, and it did so while **defeating SLSA Build L3 provenance**. Dux's stack was in the blast radius, and the attack beat the exact class of control this section relies on; a fast-follow timeline is no longer defensible.

Gate-1 requires:

- **A CycloneDX SBOM for the full application dependency tree.** (The AIBOM already uses CycloneDX 1.6 for AI components.)
- **Sigstore / SLSA provenance verification in CI.**
- **CI-runner credential hygiene** — short-lived, scoped tokens for `@tanstack/*`, `@aws-sdk/*`, and the rest of the npm/pip tree.
- **Renovate and Dependabot under the same signed-digest discipline.**

**A security vendor compromised through its own npm tree is the worst available headline.**

## 6. Release gate

`release-gate.yml` automates or prompts each check, and requires Engineering Lead sign-off before tagging.

Images are cosign-signed with an attached SBOM. **The deploy gate verifies the digest and the signature before staging or production.**
