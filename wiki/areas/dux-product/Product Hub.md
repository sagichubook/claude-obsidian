---
type: area
title: "Product Hub"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub, cross-cutting]
related_projects: ["[[Dux Portfolio]]"]
related: ["[[Dux Overview]]", "[[Dux Product Area]]"]
sources: []
---

# Product Hub

Cross-cutting entry point for anyone approaching this vault by function ("I want product context") rather than by domain folder. For the full domain scope and standards, see **[[Dux Product Area]]** — this note is a role-based index into it and its closest neighbors.

## Core product

- [[Dux Product Overview]] — thesis, pillars, capabilities, personas, gate model
- [[Dux Product Area]] — the full 11-feature product domain index
- [[Dux Roadmap]] — quarterly/gate-based roadmap, decision log

## Registries this area depends on

- [[Dux Taxonomy and Controlled Vocabulary]]
- [[Dux Catalogs — Registries of Record]]
- [[Dux Traceability Matrix]]

## Execution

- [[Dux Portfolio]] — the 10-epic execution backlog behind this product

## Diagram

```mermaid
flowchart LR
    Hub["Product Hub"] --> Overview["Dux Product Overview"]
    Hub --> Area["Dux Product Area\n11 feature specs"]
    Hub --> Roadmap["Dux Roadmap"]
    Area --> Portfolio["Dux Portfolio\nexecution backlog"]

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class Area success
```

## Related

- [[Engineering Hub]]
- [[Growth Hub]]
