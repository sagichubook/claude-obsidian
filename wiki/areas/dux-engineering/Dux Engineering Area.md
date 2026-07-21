---
type: area
title: "Dux Engineering Area"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub]
related_projects: []
related: ["[[Dux Overview]]", "[[Dux Architecture Area]]"]
sources: []
---

# Dux Engineering Area

## Scope

Everything under `50-engineering/` in the Dux corpus: coding standards, CI/CD pipeline, and local development. **In scope:** engineering-standards.md, ci-cd-testing.md, local-development.md. **Adjacent, not duplicated here:** system architecture and tech stack live in [[Dux Architecture Area]]; production monitoring and SLOs live in [[Dux Operations Area]] — this area is team practice and pipeline mechanics only.

## Reference material

- [[Engineering Standards]] — coding standards, branching, code review, contribution guide
- [[CI-CD & Testing|CI/CD & Testing]] — merge pipeline, golden set, mandatory security suites
- [[Local Development]] — full local stack, first-run setup, common failure modes

## Diagram

```mermaid
flowchart TB
    Area["Dux Engineering Area\n(this index)"] --> Standards["Engineering Standards\ncoding, branching, review"]
    Area --> CICD["CI/CD & Testing\nmerge pipeline, golden set"]
    Area --> Local["Local Development\nfull stack setup"]
    Local -.feeds.-> CICD
    Standards -.gates via.-> CICD

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class CICD success
```

## Related

- [[Dux Architecture Area]]
- [[Dux Overview]]

## Review cadence

Weekly.
