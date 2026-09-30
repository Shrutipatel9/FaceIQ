---
name: requirements-to-plan
depth: Standard
keywords: []
description: "Planning-only: derive project rules, requirements, stories, architecture, units, API contract and delivery plan from an authoritative client requirements document"
guard_policy: strict
---

# requirements-to-plan scope

Composed scope (greenfield). Treats `client_requirements.md` as the single
source of truth and produces the planning artifacts needed before
implementation: project rules (practices-discovery), a traceable
engineering spec (requirements-analysis), user stories and personas
(user-stories), architecture and ADRs (domain-design), implementation units
per story (units-generation), the frontend/backend API contract
(contract-design), and a sequenced delivery plan (delivery-planning).

Guard Policy is strict: no fences are lowered, and a change to an approved input (for example a revision of the client requirements document) reopens that approval.

## Why these stages, why skip those

Ideation is skipped because the client document already fixes context,
stakeholders, scope and benchmarks. Refined mockups wait on the client's
separate Report & UI Specification. Construction and Operation are skipped:
the requested outcome is the plan, not the code, and hosting is deliberately
undecided. Un-SKIP functional-design / nfr-requirements / nfr-design for
per-unit DB, rule and security design, and code-generation / build-and-test
to move into implementation.

## Membership

10 of 33 stages EXECUTE: the three Initialization stages plus
practices-discovery, requirements-analysis, user-stories, domain-design,
units-generation, contract-design and delivery-planning. The scope ships
with no keywords; name it explicitly with `--scope requirements-to-plan`.
