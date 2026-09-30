<!-- INVARIANT: examples are single-line HTML comments so a fresh template parses to total=0 (MEMORY_EMPTY). Do NOT un-comment or split across lines. t100 guards this. -->
> This file is kept up to date automatically while the stage runs. Add observations at the review step, not by editing here directly.

## Interpretations
<!-- example: 2026-05-29T10:14:32Z — chose REST over GraphQL; the consuming team only needs CRUD, revisit if subscriptions land -->

## Deviations
<!-- example: 2026-05-29T10:14:32Z — skipped the optional caching layer the stage prose suggested; the dataset is small enough that it adds risk -->
- 2026-09-29T13:30:00Z — Moved the CV feasibility spike (US0.11) from the engine unit to the photos unit after mapping dependencies; keeping it in the engine unit created a unit-level cycle (photos needs the adapter first, engine measurements register pipeline steps).

## Tradeoffs
<!-- example: 2026-05-29T10:14:32Z — picked TDD over BDD this run; the team is unit-first and the domain is well-understood -->
- 2026-09-29T13:30:00Z — Pipeline steps are registered by the units that implement them (U8-U11 depend on U7), inverting the component-level orchestrator-to-report call so the unit graph stays acyclic; the DAG is fairly linear as a result, with real parallelism only late (U12/U13) and in fixture-driven early work.

## Open questions
<!-- example: 2026-05-29T10:14:32Z — confirm the retention window with compliance before the next stage hardens the schema -->
- 2026-09-29T13:30:00Z — U1 needs a seeded verified user before the full seeding CLI (US0.9, U2) exists; a minimal seed script in U1 is assumed.
