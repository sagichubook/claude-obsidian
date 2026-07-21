---
type: area
title: "Growth Hub"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub, cross-cutting]
related_projects: []
related: ["[[Dux Overview]]", "[[Dux GTM Area]]"]
sources: []
---

# Growth Hub

Cross-cutting entry point for GTM, pricing, and positioning work. For the full domain scope, see **[[Dux GTM Area]]**.

## Positioning and pricing

- [[Competitive Positioning & POC]] — analyst anchor, feature matrix, competitor counters
- [[Pricing & Packaging]] — tiers, outcome pricing, Phase-1 KPIs
- [[Lean Canvas]] — one-page business model, validated vs. hypothesis

## Claims discipline

- [[GTM Guardrails]] — the claims firewall every public statement must clear
- [[External Corrections 2026-07]] — third-party listing correction tracker

## Diagram

```mermaid
flowchart LR
    Hub["Growth Hub"] --> Guardrails["GTM Guardrails\nclaims firewall"]
    Guardrails --> Pricing["Pricing & Packaging"]
    Guardrails --> Competitive["Competitive Positioning & POC"]
    Guardrails --> Canvas["Lean Canvas"]

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class Guardrails success
```

## Related

- [[Customer Success Hub]]
- [[Product Hub]]
