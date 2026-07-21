---
type: area
title: "Engineering Hub"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub, cross-cutting]
related_projects: []
related: ["[[Dux Overview]]", "[[Dux Architecture Area]]", "[[Dux Engineering Area]]", "[[Dux AI Safety Area]]"]
sources: []
---

# Engineering Hub

Cross-cutting entry point spanning three vault domains that together form "Engineering" in the generic PARA sense: architecture, AI safety, and engineering practice. For full domain scope, see the three area indexes linked below.

## Architecture

- [[Dux Architecture Area]] — system context, deployment, data model, orchestration, multi-tenancy
- [[Dux Architecture Decision Records]] — the 21 ADRs; authoritative over any prose or diagram

## AI safety

- [[Dux AI Safety Area]] — the six-control safety spine, MCP security, OWASP assessments, incident runbooks

## Engineering practice

- [[Dux Engineering Area]] — coding standards, CI/CD, local development

## API contracts

- [[Dux API Resources]] — the three-plane REST contract

## Diagram

```mermaid
flowchart TB
    Hub["Engineering Hub"] --> Arch["Dux Architecture Area"]
    Hub --> Safety["Dux AI Safety Area"]
    Hub --> Eng["Dux Engineering Area"]
    Hub --> API["Dux API Resources"]
    Eng -.merge gates.-> Safety
    Arch -.deploys.-> Eng

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class Arch,Safety,Eng success
```

## Related

- [[Product Hub]]
- [[Legal-Finance Hub]]
