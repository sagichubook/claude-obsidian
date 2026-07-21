---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: []
---

# Feature — Exposure Analysis (US-011)

**Purpose:** the per-CVE drill-down that produces a defensible exploitability verdict, backed by AWS security-group or assessment-logic evidence.

**Nav:** Exposure · **Epic:** EP-05 · **BRs:** BR-002, BR-004 · **Gate:** 1. This is the primary Analyze drill-down surface.

## US-011 Exposure Analysis / CVE Detail

**Job.** A security engineer drills into a single CVE: severity badges, risk groups, flow bar, mitigation factor cards, asset table, and attack paths — ending in a verdict they can defend.

**Journey.** Entry from a US-010 row click, the US-012 needs-attention table, or `GET /cves/{id}/detail`. Exit to the US-017 trace panel, US-008 chat, or the US-004 action cards.

**Orchestration.** The full assessment pipeline — Prerequisite, AssetContext, and ControlMapping subagents — per the [agent orchestration loop](../../20-architecture/workflows.md). Factor cards come from the `factor_type` catalog. Sources: MCP NVD, AWS APIs, and assessment logic.

**API.** `GET /cves/{id}/detail` → `CveDetailDto` (`CVEDetailQuery` / `ExposureProjection`):

| Field | Contents |
|-------|----------|
| header | CVSS, EPSS, KEV |
| `risk_groups` | group breakdown |
| `flow_bar` | pipeline state |
| `factor_cards[]` | mitigation factors |
| `assets[]` | affected assets |
| `attack_paths[]` | reachability paths |

**The three taxonomies live in separate DTO subtrees and must not be merged: a risk group is not an exposure state, and neither is a factor card.**

Also available: `GET /attack-paths`, `GET /assessments/{id}`. Export and Audit Log actions are exposed on the view. **NFR-013: p95 <500 ms at 1 K assets.**

**Factor cards at Gate 1.**

| Card | Status |
|------|--------|
| `aws_sg_blocks_port` | live |
| `product_not_affected` | live |
| `network_reachability` | partial |
| `firewall_blocks_exploitation` | live at Gate 1 (CrowdStrike) |
| `process_not_listening` | Gate 5 |

See [taxonomy §4](../taxonomy.md).

**Safety.** Cross-tenant `GET /cves/{id}` returns 404. With AWS absent, factor cards show assessment logic only. KS-L1 halts the in-flight assessment for the session.

**Metrics.** Assessment confidence distribution; factor-card coverage; attack-path query latency; drill-down → trace open rate; golden-set exploitability accuracy.

**Marketing map.** "Determines what's viable for an attacker" (BusinessWire); "maps how every vulnerability, asset, and control connects" — capability #2. Vendor cards are backed by live connectors at Gate 1 (ADR-011 R2).

## Risk-group icons

Distinct from the four exposure-state instance icons:

| Icon | Meaning |
|------|---------|
| Crossed eye | high-risk group |
| Umbrella | medium |
| Tree | low or mitigated |

These appear on the group breakdown rows in US-011.

## UI mode

Full-width structured drill-down, confirmed in the June-2026 Figma. The flow bar and asset table are both required.

**The reference UI demo numbers are illustrative, not measured** — 8,341 researched; a 74.3 / 15.6 / 10.1% split; 78% certainty. See [vision-reference](../../00-meta/vision-reference.md). They are not product metrics and must not be cited as such.
