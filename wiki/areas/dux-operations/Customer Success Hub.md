---
type: area
title: "Customer Success Hub"
created: 2026-07-21
review_cadence: weekly
tags: [area, dux, dux/hub, cross-cutting]
related_projects: []
related: ["[[Dux Overview]]", "[[Dux Operations Area]]"]
sources: []
---

# Customer Success Hub

Cross-cutting entry point for onboarding, support, and customer health. For the full operations domain, see **[[Dux Operations Area]]**.

## Onboarding and activation

- [[Dux Onboarding & Activation]] — the 9-step procedure, 7-day activation gate, funnel caveats

## Support

- [[Dux Support Playbook]] — support tiers, incident-routing decision tree, escalation ladder
- [[Customer Lifecycle & Comms]] — full onboarding/offboarding/status-page/health-monitoring source

## Diagram

```mermaid
flowchart LR
    Hub["Customer Success Hub"] --> Onboard["Dux Onboarding & Activation"]
    Hub --> Support["Dux Support Playbook"]
    Onboard --> Lifecycle["Customer Lifecycle & Comms"]
    Support --> Lifecycle

    classDef success fill:#e8f5e9,stroke:#2e7d32;
    class Lifecycle success
```

## Related

- [[Growth Hub]]
- [[Engineering Hub]]
