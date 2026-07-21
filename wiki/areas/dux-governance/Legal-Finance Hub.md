---
type: area
title: "Legal-Finance Hub"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub, cross-cutting]
related_projects: []
related: ["[[Dux Overview]]", "[[Dux Governance Area]]"]
sources: []
---

# Legal-Finance Hub

Cross-cutting entry point for compliance, governance, and financial/capacity planning. For the full governance domain, see **[[Dux Governance Area]]**.

## Compliance and governance

- [[Compliance Program]] — SOC 2, ISO 27001/42001, EU AI Act, NHI lifecycle
- [[Series B Scale Programs]] — ERM, TPRM, data sovereignty (backlog shell)
- [[Open Items Register]] — every unresolved question, with severity and owner

## Decisions and finance

- [[Dux Decisions Log]] — every decision, dated, with rationale — the capacity re-baseline history lives here
- [[Dux Portfolio]] — the 2,118h execution backlog against the 2,160h envelope
- [[Pricing & Packaging]] — outcome-pricing model, revenue lines

## Diagram

```mermaid
flowchart TB
    Hub["Legal-Finance Hub"] --> Compliance["Compliance Program\nSOC 2, ISO, EU AI Act"]
    Hub --> DLog["Dux Decisions Log\ncapacity + safety-posture history"]
    Hub --> Portfolio["Dux Portfolio\ncost/hour envelope"]
    Compliance -.inherited by.-> SeriesB["Series B Scale Programs"]

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class Compliance,DLog success
```

## Related

- [[Engineering Hub]]
- [[Growth Hub]]
