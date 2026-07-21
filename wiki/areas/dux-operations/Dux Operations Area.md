---
type: area
title: "Dux Operations Area"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub]
related_projects: []
related: ["[[Dux Overview]]", "[[Operations Overview]]"]
sources: []
---

# Dux Operations Area

## Scope

Everything under `60-operations/` in the Dux corpus: seed-stage operational activation, observability/SLO, deploy/rollback/migration runbooks, DR/BCP, and customer lifecycle. **In scope:** operations-overview.md, observability-slo.md, runbooks.md, dr-bcp.md, customer-lifecycle.md. **Out of scope, reused by reference:** the 12 canonical AI safety incident procedures live in [[AI Safety Incident Runbooks]] — this area's runbooks are stage deltas only, never a duplicate step table.

## Reference material

- [[Operations Overview]] — Gate 2 activation criteria, founder checklist, service catalog
- [[Observability & SLO]] — MELT stack, LLM instrumentation, MTTP, SLA ladder
- [[Seed Operational Runbooks]] — deploy, rollback, SSO, migrations, tenant lifecycle
- [[DR-BCP]] — RTO/RPO ladder, chaos scenarios, game days
- [[Customer Lifecycle & Comms]] — onboarding, offboarding, status page, health monitoring

## Diagram

```mermaid
flowchart TB
    Area["Dux Operations Area\n(this index)"] --> Overview["Operations Overview\nGate 2 activation"]
    Overview --> Obs["Observability & SLO\nMELT, MTTP, SLA ladder"]
    Overview --> Runbooks["Seed Operational Runbooks\ndeploy, rollback, migrations"]
    Overview --> DR["DR-BCP\nRTO/RPO, game days"]
    Overview --> Lifecycle["Customer Lifecycle & Comms\nonboarding, status page"]
    Runbooks -.deltas of.-> AISafety["AI Safety Incident Runbooks\n(dux-ai-safety area)"]

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class Overview success
```

## Related

- [[Dux AI Safety Area]]
- [[Dux Overview]]

## Review cadence

Weekly.
