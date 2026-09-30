# Units Generation — Questions

This stage groups the 67 stories (`user-stories/stories.md`) and 16 components (`domain-design/components.md`) into **units of work**: independently buildable and testable pieces for Construction. It records what depends on what. Deciding what to build first is left to Delivery Planning.

Draft plan, 13 vertical units (a feature's backend and its screens together):

| Unit | Stories | Components |
|------|---------|------------|
| U1 walking-skeleton | US0.1, US0.3–US0.6 | Platform (base), Identity (login step), WebUI shell, AuthStore, ApiClient |
| U2 platform-services | US0.2, US0.7–US0.9 | Platform (gates, adapters, rate limits, seeding) |
| U3 identity | US1.1–US1.11, US9.3 | Identity, AuthStore, ApiClient |
| U4 onboarding | US2.1–US2.4 | Onboarding |
| U5 face-analysis-engine | US0.11, US5.4, US5.5, US5.8, US5.9 | FaceAnalysisEngine |
| U6 photos | US3.1–US3.9 | Photos, MediaStore, JourneyStatus (onboarding part) |
| U7 payments | US4.1–US4.3 | Payments |
| U8 analysis-pipeline | US0.10, US5.1–US5.3 | AnalysisOrchestrator |
| U9 insights | US5.6, US5.7 | InsightGeneration |
| U10 image-generation | US5.10, US5.11, US7.2 | ImageGeneration |
| U11 report | US6.1–US6.5, US7.1 | Report |
| U12 beauty-assistant | US7.3–US7.5 | BeautyAssistant |
| U13 account-and-app-wide | US8.1–US8.3, US9.1, US9.2, US9.4, US9.5 | WebUI settings and navigation, JourneyStatus (post-payment part) |

Fill each `[Answer]:` with a letter, or `X` plus your own text.

## Q1 — How to cut units

A. Vertical feature units, as in the draft (13 units): each unit delivers a feature's backend and its screens together, with the CV engine, AI text and AI images as separate units because they are the highest-risk parts (recommended)
B. One unit per component (16 units), backend and frontend split
C. Four coarse units: foundation, onboarding and photos, payment and analysis, report and post-analysis
X. Other (please specify)

[Answer]: A

## Q2 — How the software is deployed

The client requires a separate backend and frontend (`NFR-001`, `NFR-005`). Hosting is undecided.

A. Two deployables: one FastAPI backend codebase, run as an API process plus a background-worker process (modular monolith: components are modules with enforced boundaries), and one Next.js frontend. Units are modules inside these, not separate services (recommended: simplest for a small team, and host-neutral)
B. Several backend services (for example auth, payments and analysis each deployed separately)
X. Other (please specify)

[Answer]: A

## Q3 — The first unit

Your affirmed practice is to build a thin end-to-end slice first: the email + password login step through the full auth layering.

A. U1 is that walking skeleton: repo and CI, backend and frontend foundations, the one-command local setup, and the login step through UI → Auth Store → API Client → auth API → auth service. It runs end to end on its own (recommended)
B. No skeleton unit; foundations and identity are ordinary units
X. Other (please specify)

[Answer]: A

## Q4 — Building units in parallel

A. Units with no dependency between them may be built in parallel. For example, the CV engine, AI text and payments can proceed side by side once their dependencies exist (recommended)
B. Strictly one unit at a time
X. Other (please specify)

[Answer]: A

## Q5 — Unit kinds

Each unit gets a kind, which decides which design documents it needs later:
- `service`: deployed executable behaviour;
- `library`: reusable code with no endpoint of its own;
- `ui`: frontend surface.

A. U2, U5 and U9 are `library` (no endpoints of their own). U13 is `ui` (mainly screens and cross-cutting frontend work). All others are `service` (recommended)
B. All units are `service`
X. Other (please specify)

[Answer]: A

## Consolidated Summary Confirmation

Summary of answers and the resulting plan:
- 13 vertical feature units; each delivers a feature's backend and screens together (Q1 A)
- Modular monolith: one FastAPI codebase run as API and background-worker processes, plus one Next.js frontend; units are modules, not services (Q2 A)
- U1 is the walking skeleton: repo/CI, foundations, one-command local setup, and the login step through the full auth layering (Q3 A)
- Independent units may be built in parallel (Q4 A)
- Kinds: platform-services, face-analysis-engine and insights are library; account-and-app-wide is ui; all others are service (Q5 A)
- Adjustment found while mapping dependencies: the CV feasibility spike (US0.11) moves from the engine unit to the photos unit, because photo validation needs it first while engine measurements run inside the pipeline; this keeps the dependency graph free of cycles. Units are numbered in dependency order: U1 walking-skeleton, U2 platform-services, U3 identity, U4 onboarding, U5 photos, U6 payments, U7 analysis-pipeline, U8 face-analysis-engine, U9 insights, U10 image-generation, U11 report, U12 beauty-assistant, U13 account-and-app-wide

Does this all look correct before I generate the artifact?

- Looks correct
- Request changes

[Answer]: Looks correct
