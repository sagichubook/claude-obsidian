---
type: area
title: "Dux"
created: 2026-07-22
updated: 2026-07-22
review_cadence: weekly
tags: [area, dux, dux/hub]
status: evergreen
related: ["[[index]]", "[[Dux Product Guide]]", "[[Dux Architecture Guide]]", "[[Dux AI Safety Guide]]"]
sources: [".raw/dux/README.md", ".raw/dux/00-meta/vision-reference.md", ".raw/dux/00-meta/decisions-log.md"]
---

# Dux

Navigation: [[index]] | [[hot]] | [[log]]

## What Dux is

Every security team drowning in scanner output asks the same question: of the thousands of findings on the board, which ones can an attacker actually use right now? **Dux** is a multi-tenant SaaS platform built to answer that question directly: an AI agent takes a CVE plus a customer's live environment evidence and works out what's genuinely exploitable and the fastest way to shut the door on it, across an Analyze → Mitigate → Remediate pipeline that runs unattended by default for three of its five possible write actions. It's defensive only, full stop: never penetration-testing-as-a-service, never a scanner replacement.

This page is the single front door into the full knowledge base below: what the product does, how it's built, how it stays safe while acting with real autonomy, and how the business around it runs.

## About the company

Dux is built by **Dux, Inc.**, dual-headquartered in Tel Aviv (R&D) and New York (go-to-market), founded in 2024. The company closed a $9M seed round in December 2025 with a three-way co-lead, and sells specifically into finance and healthcare: a buyer profile that drove several of the platform's more expensive architecture decisions (self-hosted infrastructure, multi-cloud portability) over a faster, cheaper, single-vendor path. Headcount sits at 24 as of the most recent corpus review.

The founder, **Sagi**, is the sole named decision-maker across the entire body of documentation this knowledge base is built from: every non-mechanical judgment call traces to a dated, attributed decision in [[Dux Decisions & Traceability Reference]], never to silent prose edits. That single-threaded decision authority is also why the documentation itself follows one hard rule throughout: specs describe current truth in the present tense, and all change history lives in exactly one place.

## The Dux Agent

**Dux Agent** is the only name that ever appears in front of a customer for the AI system doing the work: internal names for the runtime services actually doing the reasoning never leak into customer-facing surfaces. See [[Dux Product Guide]] for the full persona and the runtime architecture behind it.

## How this knowledge base is organized

Ten source domains (product, architecture, API, AI safety, engineering, operations, governance, GTM, and the execution backlog) have been consolidated into a small set of dense, purpose-built guides rather than left as dozens of narrow pages. Each domain gets an **explanation-and-how-to guide** written to be read start to finish, and where a domain also needs fast lookup (API contracts, decision history, the controlled vocabulary), it gets a companion **reference** page built for scanning, not reading.

| You are | Start here | Then |
|---|---|---|
| An engineer building a feature | [[Dux Product Guide]] | [[Dux Feature Reference]] → [[Dux API Reference]] |
| An architect | [[Dux Architecture Guide]] | [[Dux AI Safety Guide]] for the safety spine built on top of it |
| A security reviewer | [[Dux AI Safety Guide]] | [[Dux AI Safety Operations Reference]] |
| On-call / SRE | [[Dux Operations Guide]] | [[Dux AI Safety Operations Reference]] for the incident runbooks |
| Founder, PM, or GTM | [[Dux GTM Guide]] | [[Dux Product Guide]] for the roadmap it has to stay honest against |
| Customer success | [[Dux Customer Success Guide]] | [[Dux Operations Guide]] for the incident-communication mechanics |
| Auditor or compliance reviewer | [[Dux Governance & Compliance Guide]] | [[Dux Decisions & Traceability Reference]] |
| Anyone, before starting real work | [[Dux Governance & Compliance Guide]] for open items | whatever guide the open item touches |

## The domain map

**Product**
- [[Dux Product Guide]]: thesis, the five delivery pillars, write-action autonomy, personas, the gate model, the roadmap
- [[Dux Feature Reference]]: every screen and API surface behind the eight-icon sidebar and chat
- [[Dux Taxonomy & Catalogs]]: the controlled vocabulary, the confidence model, and the nine extension registries

**Architecture**
- [[Dux Architecture Guide]]: system context, the technology stack, how an assessment actually runs, the data model, tenant isolation, and the key architecture decisions behind all of it

**AI safety**
- [[Dux AI Safety Guide]]: the governance kernel, the kill switch, the CaMeL dual-LLM boundary, sandbox execution, MCP security
- [[Dux AI Safety Operations Reference]]: agent identity, confidence calibration, OWASP maturity ratings, the twelve incident runbooks

**Engineering**
- [[Dux Engineering Guide]]: local setup, coding standards, code review, and the full CI/CD pipeline including the golden set

**Operations**
- [[Dux Operations Guide]]: seed-stage activation criteria, observability and SLOs, deploy/rollback runbooks, disaster recovery
- [[Dux Customer Success Guide]]: onboarding, support tiers, incident communication, tenant health

**Governance**
- [[Dux Governance & Compliance Guide]]: the open-items discipline, SOC 2/ISO/EU AI Act, and the Series B governance roadmap

**Go-to-market**
- [[Dux GTM Guide]]: the claims firewall, the business model, pricing, competitive positioning, and live external corrections

**API**
- [[Dux API Reference]]: the three REST planes, every DTO contract, DQL, events and webhooks

**Reference**
- [[Dux Decisions & Traceability Reference]]: the full decision history and the business-requirement-to-verification-command join table

**Execution**
- [[Dux Portfolio]]: the ten-epic Phase-1 backlog, hour rollup, and the rules governing how it's allowed to change

## A note on how this knowledge base itself is maintained

This corpus was ingested with a genuinely unusual amount of rigor: four independent verification passes cross-checked citation coverage, dense ID ranges, dollar figures and diagram coverage, and inline-code spans against the source material, catching and fixing nineteen real content gaps along the way. The full account of that process lives in `wiki/migration-audit.md` and `wiki/validation-checklist.md`, preserved as a historical record from before this knowledge base was consolidated into the guides above. The source material itself remains mirrored in full at `.raw/dux/` and is never modified: every fact in every guide above traces back to it.

## Sources

- `.raw/dux/README.md`
- `.raw/dux/00-meta/vision-reference.md`
- `.raw/dux/00-meta/decisions-log.md`
