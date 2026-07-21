---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-2, D-22, H5]
---

# Workflows & Agent Orchestration

**Purpose:** how an assessment actually runs — the orchestration loop, the Temporal contract, the action budget, and the vendor write flows.

**Parents:** BR-002, BR-003 · **Engine:** self-hosted Temporal (on Kubernetes) behind `WorkflowPort` (ADR-007 R3) · **Pattern:** orchestrator plus isolated subagents. A swarm was rejected.

## 1. Why a supervisor with isolated subagents, and not a monolith

Production teams running monolithic, many-tool ReAct agents hit **context bloat** (around 180 K tokens), **early-result eviction**, and **high factual-error rates from context confusion** — a documented 47-tool monolith had **39% of its outputs flagged for factual errors**.

The industry converged on supervisor plus isolated subagents (LangChain Deep Agents, Red Hat's multi-agent supervisor, the Agent Patterns Catalog's "Subagent Isolation"). **Each subagent runs in a clean context window, with a narrow tool allowlist and a fixed output schema — and that schema doubles as the CaMeL security boundary.**

Start-small discipline: **three Gate-1 subagents, mapped to three distinct evidence domains.** Multi-agent chaining (ASI07) is deferred, behind a signed inter-agent-JWT stub.

## 2. Orchestration loop

```
IDLE → REASONING → TOOL_CALLING → EVALUATING → { COMPLETE | BLOCKED (governance) | FAILED }
```

Agent status messages map to `REASONING` / `TOOL_CALLING` / `EVALUATING` / `COMPLETE` — see [taxonomy §6](../10-product/taxonomy.md).

**Loop guards. Each escalates to a human; none retries silently:**

| Guard | Condition |
|-------|-----------|
| `loop_detected` | the same tool with the same arguments, more than 3 times |
| `budget_exceeded` | the action or cost budget is exhausted |
| `convergence_failure` | 3 turns yielding zero new entities |
| `oscillation_detected` | A → B → A → B within the last 4 turns |

The `getLoopGuardStatus` query surfaces these in the US-017 trace-viewer API.

## 3. Orchestrator-worker layers

| Layer | Component | Responsibility |
|-------|-----------|----------------|
| Outer orchestration | Temporal workflow (`ExploitabilityAssessmentWorkflow`) | lifecycle, CVE trigger, state machine, one child workflow per tenant |
| Activity facade | `AssessmentActivity` | thin entrypoint; wires collaborators; **no inline business logic** |
| Prerequisite subagent | `PrerequisiteExtractor` | S-LLM only; Zod-validated prerequisite schema (US-001) |
| Asset-context subagent | `AssetContextWorker` | scoped asset and runtime evidence (US-002) |
| Control-mapping subagent | `ControlMappingWorker` | vendor control panels and attack-path evidence (US-003) |
| Reasoning loop | `ReasoningLoop` | direct Bedrock Converse API `toolConfig` calls (no framework, ADR-021) plus MCP tool activities; enforces the action budget |
| Trace recording | `TraceRecorder` | persists `ASSESSMENT_REASONING_STEP` and code artifacts; **makes no LLM calls** |

**Hard rules:**

- **A distinct system prompt per tier.** Never reuse the orchestrator's prompt for a subagent.
- A worker's first message is a **structured brief**: objective, allowed tools, limits, output schema.
- **One child workflow per tenant**, for blast-radius isolation.
- Saga compensation persists partial state — for example, "analysis incomplete: asset context loaded, exploitability pending".
- Continue-as-new before the event limit.

## 3a. Message history storage

Bedrock Converse resends the full `messages` array every turn — ten turns of tool results can exceed Temporal's ~50 KB default workflow-state limit. Message history therefore lives in Postgres, not workflow state:

| Concern | Detail |
|---------|--------|
| Table | `assessment_messages` — `id`, `assessment_id`, `turn`, `role`, `content` (jsonb), `tool_use_id`, `created_at` |
| Workflow state | carries only `history_id` (UUID), `turn_count` (int), `spent_usd` (decimal), `status` (enum) — never the message array itself |
| Read/write | `ReasoningLoop` fetches by `history_id` at the top of each turn, appends the model response and any tool results at the end |

## 4. Temporal execution contract

| Concern | Specification |
|---------|---------------|
| Child workflows | one per tenant — `taskQueue: assessment-{tenant_id}`; the orchestrator runs on `assessment-orchestrator` |
| CaMeL middleware | synchronous request/response **inside** each activity — not a separate runtime |
| Heartbeats | required on MCP and external activities. NVD 30 s / 10 s; AWS 120 s / 30 s; MCP 60 s / 15 s. `scheduleToCloseTimeout` 15 min; max 2 retries; backoff 1 s → 30 s |
| Continue-as-new | `continueAsNewSuggested` at ≥8 K events; hard at ≥10 K; **never exceed the 35 K safety cap** |
| Versioning | `patched()` and worker build IDs. Pin `ExploitabilityAssessmentWorkflow` v1 until the golden-set gate passes v2. **Gate-1 exit criterion — an unversioned worker is the single most common Temporal production failure** |
| Tracing | OTel spans on **every** workflow. **Gate-1 exit criterion.** Spans carry `tenant_id_hash`, never the raw ID |
| Saga compensation | partial failure → `status=incomplete`, `partial_context_loaded=true` |
| Action budget | `checkCostCap` is a per-iteration guard — it runs every iteration alongside `checkKillSwitch` (§5's "2 per iteration"). It additionally evaluates cumulative cost against the governance-kernel cost ladder ([governance-kernel §2](../40-ai-safety/governance-kernel.md)) at hard checkpoints on iterations 5, 10, and 15. Halt with `status=blocked` plus an L2 kill switch when `workflow_actions_per_assessment` reaches 200 |
| Step-effect idempotency | external effects carry a `mutation_key` and reconcile at resume time — exactly-once effects |

## 5. Child-workflow mapping and action budget

| Transition | Child workflow / activity | Estimated actions |
|------------|---------------------------|-------------------|
| IDLE → REASONING | `PrerequisiteExtractionWorkflow` (S-LLM + schema) | 2 |
| REASONING → TOOL_CALLING (loop) | `ReasoningWorkflow` + `MCPInvocationWorkflow`, per iteration | 4–6 per iteration |
| Per-iteration guards | `checkKillSwitch`, `checkCostCap` | 2 per iteration |
| EVALUATING → COMPLETE | `FinalizeAssessmentWorkflow` | 2 |
| **Total** | 10 iterations, typical | **40–80** (p50 ≈ 55, p95 ≈ 58) |

Instrument `workflow_actions_per_assessment` from day one. Weighting: LLM = 1, MCP read = 2, MCP write = 5.

**The 70–80 tail is rare by construction, not by luck.** It occurs in under 5% of assessments, driven by multi-hop attack-path traversal on large asset inventories — cases that run past the 10-iteration typical case, up to the GOV-010 50-iteration ceiling. That tail pulls the mean up without moving the p95, which lands at 58 — inside the [Phase-1 KPI](../80-gtm/pricing-packaging.md#5-phase-1-kpis) SLO of p95 <60.

**Complexity pre-downgrade:** more than 100 assets, **or** a CVE description over 2 K tokens, auto-routes to the pinned `gpt-5.4-mini` (Unleash `complexity_router`). Log `downgrade_reason`.

## 6. Vendor integration flows

| Flow | Stage | Gate | Backend | Writes to vendor? |
|------|-------|------|---------|-------------------|
| A — Analyze | Analysis | Gate 1 | `ExploitabilityAssessmentWorkflow` + MCP read | No |
| B — Connector ingest | context source | Gate 1 | `sync()` → World Model | No |
| C — Action cards | Mitigations | **Gate 1, unattended by default** | `ActionCardProjection` + `QuickMitigationWorkflow` → ADR-012 R3 adapter | **Yes** |
| D — Fast Actions | Mitigations | **Gate 1, unattended by default** | `POST /fast-actions` → `QuickMitigationWorkflow` → ADR-012 R3 adapter | **Yes** |
| F — Remediation ticket | Remediation | **Gate 1 create + route, unattended by default** | `RemediationWorkflow` → `ticket.create_remediation` | **Yes** |
| G — Closed-loop validate | Mitigations | Gate 3 | `ClosedLoopValidationWorkflow` | re-assessment only |

**The vendor mutation sequence:**

```
POST /fast-actions
  → ActionPolicyGate.isAllowed
    → (escalate to HITL only on an anomaly)
      → VendorActionAdapter.execute
        → vendor native API
          → ActionResult + audit
```

**Agents reason over World Model snapshots and external intel. They do not invoke vendor mutation APIs inline.**

## 7. Gate-3 workflows

`QuickMitigationWorkflow` runs four phases: policy check → execute via `VendorActionPort` → enqueue the post-action sync → hand off to `ClosedLoopValidationWorkflow`.

`ClosedLoopValidationWorkflow` is a saga. Forward: `reassess`, `update_ticket`, `notify`. Compensating: `mark_superseded`, `revert_ticket`, `notify_rollback`.

## 8. Application-service conventions

Services are named `{Verb}{Noun}Service`. **Controllers and SSE handlers delegate to the same service** — `POST /research/queue` and the chat `request_research` event both go through `AssessmentEnqueuePort`.

| Concern | Pattern |
|---------|---------|
| Governance kernel | Chain of Responsibility — `IntentGate → BudgetGate → EffectGate → VendorActionGate → HITLGate` |
| S-LLM fallback | Strategy |
| Control refinements (FR-019) | Specification |

**`AssessmentActivity` decomposition — each collaborator has one job, and is forbidden from the others:**

| Component | Must not |
|-----------|----------|
| `PrerequisiteExtractor` | call MCP or the P-LLM |
| `ReasoningLoop` | persist the trace |
| `TraceRecorder` | invoke an LLM or a tool |
| `AssessmentActivity` | contain vendor or SQL logic |

## 9. GDPR and lifecycle workflows

| Workflow | Contract |
|----------|----------|
| `TenantExportWorkflow` | GDPR Art. 15 and 20 — completes within 24 h |
| `GDPRDeletionWorkflow` | Art. 17 — soft-delete → 30-day export hold → hard-delete → crypto purge within 90 days |
| `WorldModelVersionPurgeJob` | cancels workflows older than 24 h on a superseded version, **scoped to the affected tenant only**. `world_model_versions` is tenant-scoped ([data-model](data-model.md)) — **one tenant's connector sync must never cancel another tenant's in-flight work** |
| `ReassessmentSchedulerWorkflow` | continuous re-assessment (ADR-016) — see [continuous-assessment](../10-product/features/continuous-assessment.md) |

**The canonical deletion timeline — day-0 soft-delete, 30-day export window, day-90 hard purge — is authoritative in [multi-tenancy §5](multi-tenancy.md#5-tenant-lifecycle) (D-18).**
