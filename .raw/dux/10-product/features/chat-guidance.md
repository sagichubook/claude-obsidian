---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-4, D-7, D-17, H4, H5]
---

# Feature — Chat Guidance (US-008)

**Purpose:** the conversational surface onto the Dux Agent — request research, compare remediation strategies, and approve chat-initiated write actions.

**Nav:** global chat panel + in-context · **Epic:** EP-05 · **BRs:** BR-002, BR-006, BR-007 · **Gate:** 1, read-only path.

**Chat is not the only write surface with a live human-approval gate.** Product write actions triggered elsewhere in the UI (US-004, US-016, US-018) execute unattended by default for 3 of 5 canonical actions (`network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`); `endpoint.isolate` and `patch.deploy_special_devices` are mandatory-HITL there too (D-17). The chat-triggered write-tool path (`chat_write_tools`) is gated for every action regardless of which of the five it targets, because **arbitrary MCP writes issued from open-ended conversation carry a prompt-injection blast radius (LLM01) that the other, schema-constrained write surfaces do not have**.

This is a deliberate, scoped exception on top of D-17's per-action gating. Revisit it only when chat-specific injection defenses — the CaMeL S-LLM boundary plus the prompt-injection regression suite — earn the same confidence as the constrained surfaces.

## US-008 Chat Guidance

**Job.** A security engineer converses with the Dux Agent to request research, select prioritization paths, and — where write is enabled — approve supervised actions.

**This is not a general-purpose security chatbot.** Every turn and every action routes through governed agent workflows: audit, kill switch, tenant scope, HITL.

**Failure-domain isolation.** Chat gets a **separate SSE connection pool** and its **own LLM quota bucket** (`InstrumentedLLMClient` budget, NFR-011). Chat degradation or a chat rate limit **must not** block `ExploitabilityAssessmentWorkflow` queue processing.

**Orchestration.** The same agent stack as assessment; MCP is read-only in Phase 1.

Prioritization cards are a **control-plane write**, exempt from `chat_write_tools`, persisted to `session_routing_preferences` (ephemeral, 24 h TTL, with a threat model and hash-chained audit per AI-13).

The chat path is blocked until the Week-4 DBOS / inner-loop chat spike passes. Go/no-go: pass, or fall back to a DBOS-only inner loop with hand-rolled SSE.

**API.**

| Surface | Contract |
|---------|----------|
| SSE | `GET /chat/sessions/{id}/stream` — events: `query`, `response`, `citation`, `processing_step`, `prioritization_cards`, `request_research_ack`, `hitl_request` |
| POST | `POST /chat/sessions/{id}/hitl-response` |
| Alias | `POST /research/queue` — Request Research |

Flag: `chat_interface`, on at Gate 1.

**Remediation-option comparison.** For a broad ask — "what should I do about exploitable vulns on my Linux servers?" — `prioritization_cards` can carry **several competing remediation strategies side by side**. For example, "patch a single shared component" against "target the specific devices responsible for N% of your unique exploited-in-the-wild CVEs".

Each card is quantified by **the share of exploited or actively-targeted CVEs it addresses**, not by raw instance count. This reuses the existing `prioritization_cards` mechanism — no new API surface, just a defined card shape: strategy label, affected-asset list, and `exploited_cve_coverage_pct`.

**HITL contract.** See [kill-switch-hitl](../../40-ai-safety/kill-switch-hitl.md).

Reconnection replays `Last-Event-ID` from `chat_session_events` over a 1-hour window; beyond that it falls back to a `GET /chat/sessions/{id}/state` snapshot. **Workflow state is unaffected — reconnection is transport-only.** Limits: 5 concurrent SSE streams per `user_id`, over HTTP/2.

**Data.** World Model read-replica, <5 s lag. Citations carry AWS and NVD context.

**Safety.** KS-L1 per session. Prompt-injection regression suite (LLM01). **Write tools require HITL.**

**Metrics.** Chat-spike C1–C5; SSE latency p95; Request Research conversion from chat; prioritization-card selection rate; chat LLM cost per session against budget.

## Phased delivery

| Milestone | Delivery |
|-----------|----------|
| Week 4 | DBOS / inner-loop chat spike (go/no-go). Blocks US-008 production routing until it passes |
| Week 6 | **Closed without a bake-off (D-35, ADR-021).** No inner agent framework — the reasoning loop calls the Bedrock Converse API directly from Temporal activities. The CopilotKit / AG-UI chat-UI spike (W10–12) still runs on its own merits, independent of this decision |
| Week 8 | API-level HITL approve/deny gate active for write-tool paths |
| **Gate 1 (Week 12)** | Minimal approve/deny UI live, with impact preview (H4) |
| Week 14 | Full chat HITL UI. `chat_write_tools` external MCP write tools enabled — **no chat write before this UI exists** (AI-133) |

**Two write surfaces, two risk profiles.** Product write actions (US-004/016/018) are unattended at Gate 1 for 3 of 5 canonical actions, with the approve/deny surface serving only anomaly escalation; `endpoint.isolate`/`patch.deploy_special_devices` use that same approve/deny surface as a mandatory gate (D-17). Chat write tools stay gated to the Week-14 full chat HITL UI regardless of action. That difference is deliberate, and the reason is at the top of this file.
