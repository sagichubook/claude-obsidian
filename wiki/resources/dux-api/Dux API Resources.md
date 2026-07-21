---
type: resource
title: "Dux API Resources"
topic: "dux/api"
created: 2026-07-21
updated: 2026-07-21
tags: [resource, dux, dux/hub]
sources: []
related: ["[[Dux Overview]]", "[[API Overview]]"]
---

# Dux API Resources

## Scope

Everything under `30-api/` in the Dux corpus: the three-plane REST contract, DTO shapes, DQL, and outbound event delivery. In scope: api-overview.md, application-api.md, public-data-api.md, events-webhooks.md, openapi.yaml (draft skeleton).

## Reference material

- [[API Overview]] — the three planes, auth, versioning, rate limits
- [[Application API]] — Phase-1 DTO contracts (Bearer JWT plane)
- [[Public Data API]] — programmatic read surface + DQL (Bearer API key plane, Gate 2/Seed)
- [[Events & Webhooks]] — outbound delivery, SSE, event semantics

## Diagram

```mermaid
flowchart TB
    Hub["Dux API Resources\n(this index)"] --> Overview["API Overview\n3 planes, auth, rate limits"]
    Overview --> App["Application API\nBearer JWT"]
    Overview --> Pub["Public Data API\nBearer agt_ key"]
    Overview --> Events["Events & Webhooks\nNATS JetStream delivery"]
    App -.shares error codes.-> Events
    Pub -.shares error codes.-> Events
```

## Related

- [[Dux Overview]]
- [[Dux Architecture Area]]
