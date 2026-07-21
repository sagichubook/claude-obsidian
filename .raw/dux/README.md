---
owner: Founder
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-10, D-11, D-12]
---

# Dux — Developer Documentation

**Product:** Dux — an agentic exposure management platform (multi-tenant SaaS) · **Company:** Dux, Inc. · **Canon:** ideal state, Gap Closure v2.2.

This tree is the developer-ready corpus. The four stage playbooks live in git history as the historical record (moved to `archive/` in `ed146a3`, removed from the working tree in `84fec31`). **Where this tree and the playbooks disagree, this tree is current.** Wire-format authority for the public `/v1/*` API remains the OpenAPI 3.1 spec.

## What Dux is

AI agents — "Dux Agent" — take a CVE plus a customer's live environment evidence (assets, runtime, identity, network, existing controls), and determine **what is *actually exploitable here*, and the *fastest path to protection*.**

The pipeline is three stages — **Analyze → Mitigate → Remediate** — live end to end at Gate 1, **unattended by default**. Human review is an anomaly-escalation path, not a gate on every write.

Agents **write and execute** investigation code in self-hosted Firecracker microVM sandboxes, and re-assess continuously as threat intel and the environment change.

**Defensive only. Never PTaaS.**

## Reading paths

| You are | Start | Then |
|---------|-------|------|
| Engineer building a feature | [Product overview](10-product/product-overview.md) | [Traceability matrix](00-meta/traceability-matrix.md) → your feature spec in `10-product/features/` → [API contracts](30-api/api-overview.md) |
| Engineer setting up | [Local development](50-engineering/local-development.md) | [CI/CD & testing](50-engineering/ci-cd-testing.md) → [Engineering standards](50-engineering/engineering-standards.md) |
| Architect | [Architecture overview](20-architecture/architecture-overview.md) | [ADR index](20-architecture/adr-index.md) → [Workflows](20-architecture/workflows.md) → [Data model](20-architecture/data-model.md) |
| Security reviewer | [AI safety overview](40-ai-safety/safety-overview.md) | [MCP security](40-ai-safety/mcp-security.md) → [OWASP assessments](40-ai-safety/owasp-assessments.md) |
| On-call | [Operations overview](60-operations/operations-overview.md) | [Incident runbooks](40-ai-safety/incident-runbooks.md) → [Seed runbooks](60-operations/runbooks.md) |
| Founder / PM / GTM | [GTM guardrails](80-gtm/gtm-guardrails.md) | [Pricing & packaging](80-gtm/pricing-packaging.md) → [Competitive](80-gtm/competitive.md) |
| Auditor / compliance | [Compliance program](70-governance/compliance-program.md) | [Traceability matrix](00-meta/traceability-matrix.md) |
| Anyone, before starting work | [Open items](00-meta/open-items.md) — what is still undecided, and what it blocks | |

Several process-record audit files that once lived in `00-meta/` and `20-architecture/` (stack-review, claims-implementation, marketing-claims, architecture-panel, security-domain, doc-framework, stack-design-review) have been removed once their findings were closed or registered — see [decisions-log](00-meta/decisions-log.md) for the disposition history, [open-items](00-meta/open-items.md) OI-25/26/27/30/31 for what's still open, and git history for the original files. `00-meta/quick-reference.md` is a pure compilation of already-documented facts — start there for a fast fact-check.

## Tree

```
docs/
├── README.md                        ← you are here
├── 00-meta/                         provenance, open questions, cross-cutting indices
│   ├── open-items.md                OI-## register: every open question, owner, blocking gate
│   ├── decisions-log.md             locked rewrite + stack decisions, numbered and dated
│   ├── traceability-matrix.md       BR → Epic → US → FR/NFR chain
│   ├── vision-reference.md          all marketing/scraped language + gap list  [process-record]
│   └── quick-reference.md           gate criteria, kill-switch levels, action budgets, confidence bands, control IDs
├── 10-product/
│   ├── product-overview.md          vision→scope, capabilities, personas, gates, plan
│   ├── taxonomy.md                  exploitability model, enums, naming, design system
│   ├── catalogs.md                  registries: integrations, agents, events, flags, actions
│   └── features/                    one spec per surface (US-referenced)
├── 20-architecture/
│   ├── architecture-overview.md     system context, containers, monorepo, ports
│   ├── architecture-diagrams.md     rendered Mermaid diagram set, end to end
│   ├── adr-index.md                 ADR-001…021, revisions applied as adopted
│   ├── workflows.md                 orchestration loop, Temporal contract, budgets
│   ├── data-model.md                ERD, retention, referential integrity, RLS
│   └── multi-tenancy.md             isolation spec, lifecycle, noisy neighbor
├── 30-api/
│   ├── api-overview.md              3 planes, auth, versioning, rate limits
│   ├── openapi.yaml                 DRAFT spec skeleton (moves to service repo at build; BS-17a)
│   ├── application-api.md           Phase-1 DTO contracts, SSE, errors
│   ├── public-data-api.md           /v1 endpoints, DQL, pagination
│   └── events-webhooks.md           event catalog + outbound delivery
├── 40-ai-safety/
│   ├── safety-overview.md           spine, trifecta, defense layers, STRIDE
│   ├── camel-plane.md               dual-LLM boundary, schemas, CDA
│   ├── governance-kernel.md         GOV-001–013 gates
│   ├── kill-switch-hitl.md          KS-L1–L4 + HITL tiers/contract
│   ├── agent-identity.md            JWT/SPIFFE, lifecycle, shadow AI
│   ├── mcp-security.md              PS-001–011, tool catalog, gateway
│   ├── sandbox-execution.md         self-hosted Firecracker microVM, AST scan, decision matrix
│   ├── confidence-calibration.md    ensemble, Platt, abstention
│   ├── owasp-assessments.md         ASI01–10, LLM01–10, MCP crosswalk
│   └── incident-runbooks.md         12 canonical agentic failure modes
├── 50-engineering/
│   ├── local-development.md
│   ├── ci-cd-testing.md             merge gates, golden set, suites
│   └── engineering-standards.md     coding standards, branching, review, contribution
├── 60-operations/
│   ├── operations-overview.md       stage triggers, roles, service catalog
│   ├── runbooks.md                  deploy/rollback/SSO/tenant/etc. (seed deltas)
│   ├── observability-slo.md         MELT, SLOs, burn-rate alerting
│   ├── dr-bcp.md                    RTO/RPO, chaos, game days
│   └── customer-lifecycle.md        onboarding, billing, status page, health
├── 70-governance/
│   ├── compliance-program.md        SOC 2, ISO 27001/42001, policies, evidence
│   └── series-b-scale.md            Series B programs (explicit backlog shell)
├── 80-gtm/
│   ├── gtm-guardrails.md            claims firewall (GCIS-true), qualification
│   ├── pricing-packaging.md         tiers, SLA ladder, outcome pricing
│   ├── competitive.md               battle cards, POC framework, ROI
│   ├── lean-canvas.md               one-page business model (hypothesis-tagged)
│   └── external-corrections-2026-07.md  ready-to-paste site/listing correction text (OI-26)
└── 90-execution/
    ├── README.md                    backlog framework rules (IDs, DoD, deviations)
    ├── traceability.md              portfolio matrix + hour rollups vs capacity
    └── backlog-ep01…ep10.md         Epic → Feature → Story → Task decomposition
```

## Authority & conventions

1. **Authority order:** decisions-log → GCIS v2.2 → OpenAPI 3.1 (`/v1/*` wire contract only) → BR→FR→US → agent registry (manifests). **Claims-alignment rule (D-10, 2026-07-14, Sagi):** the Marketing Claims map binds **GTM copy, product naming, and UI strings**. It does **not** bind safety posture, control design, gate criteria, or SLOs — an engineering doc states what the system does. Where a live public claim and engineering reality diverge, the divergence is raised in [open-items.md](00-meta/open-items.md) as a claim-integrity item; it does not bend the spec. The Gate/build-status record in [traceability-matrix.md](00-meta/traceability-matrix.md) tracks actual delivery.
2. **Stage model:** playbook names = operational maturity, not funding rounds. Pre-seed = Phase 1 build (Days 0–90); seed = post-Gate-2 ops (Days 90–365); Series A = certifications/governance; Series B = global-scale programs (backlog shell). The company's $9M seed round (Dec 2025) is unrelated to "seed playbook" activation. When reading the archived playbooks: frontmatter `status: accepted` means its triggers/gates were binding; `draft` means directional only.
3. **Terminology:** **Dux Agent** = only customer-facing agent name; *assessment agent* internal role only; *Mitigation nav* = research queue (Analyze), distinct from the *Mitigate* pipeline stage; kill switch (noun) / kill-switch (adjective); World Model is a proper noun; API/code say `tenant_id`, UI may say organization.
4. **Open questions:** every unresolved question lives in [open-items.md](00-meta/open-items.md) as `OI-##`, with an owner, a severity, and what it blocks (D-11). A spec links to the item rather than restating it, and never carries a `> **TBD**` box of its own.
5. **Current truth only:** specs are written in the present tense (D-12). **Change history belongs in [decisions-log.md](00-meta/decisions-log.md) — never in body prose.** No "re-gated 2026-07-13", no "superseded — was: …", no dated parentheticals. A reader should learn the current state by reading, not by mentally applying a chain of diffs.
6. **Validation:** after editing docs, run `python3 scripts/validate-playbooks.py`. **Exit 0 is required to merge.** It checks:
   - dead links **and anchors** (renaming a heading changes its anchor — the validator catches the resulting dead links, but keep anchors stable where you can);
   - duplicate canonical H2 IDs (`## US-`, `## EP-`, `## ADR-`, …) — each may be defined in exactly one file;
   - task → epic → portfolio hour reconciliation, including the grand total;
   - the front-matter contract below;
   - **change history in spec prose** — a reintroduced "re-gated …" or "superseded — was: …" fails the gate.

## Document contract

Every file in this tree carries front-matter:

```yaml
---
owner: Engineering | Founder | GTM | Security
status: canonical | draft | backlog-shell | process-record
gate: 1 | 2 | 3 | n/a
last_reviewed: YYYY-MM-DD
decisions: [D-4, H5]     # optional: the decisions that govern this file
---
```

Then: an `# H1`, a **Purpose** line, and numbered `##` sections.

**What `status` means:**

| Status | Meaning |
|--------|---------|
| `canonical` | current truth. Build from it |
| `draft` | being written; do not build from it yet |
| `backlog-shell` | a planning placeholder. **Inherit the earlier stage's artifacts** — do not execute from a stub |
| `process-record` | dated audit evidence. **Not a spec, and not on any reading path** |

**Three exceptions to the present-tense rule, all deliberate:**

- **[decisions-log.md](00-meta/decisions-log.md)** is the history home. It carries all of it.
- **[adr-index.md](20-architecture/adr-index.md)** keeps its `Superseded` lines. An ADR is a decision record — carrying its own supersession history is the format working as intended.
- **[traceability.md](90-execution/traceability.md)**'s hour-rollup section keeps its dated historical paragraphs (2026-07-20, D-42) — a running capacity ledger needs its history visible the same way an ADR does, to show which lever fired and why.
