<!-- INVARIANT: examples are single-line HTML comments so a fresh template parses to total=0 (MEMORY_EMPTY). Do NOT un-comment or split across lines. t100 guards this. -->
> This file is kept up to date automatically while the stage runs. Add observations at the review step, not by editing here directly.

## Interpretations
<!-- example: 2026-05-29T10:14:32Z — chose REST over GraphQL; the consuming team only needs CRUD, revisit if subscriptions land -->
- 2026-09-29T14:20:00Z — Treated the signed media download (C11) as the one documented exception to 'only the API client calls the backend', because img tags cannot carry the in-memory Bearer token; the URL itself still comes through the API client.

## Deviations
<!-- example: 2026-05-29T10:14:32Z — skipped the optional caching layer the stage prose suggested; the dataset is small enough that it adds risk -->
- 2026-09-29T14:20:00Z — Recorded PDF generation mode (sync vs 202 + retry) as an open contract question rather than deciding it, since the domain review left its owner unresolved (R-01).

## Tradeoffs
<!-- example: 2026-05-29T10:14:32Z — picked TDD over BDD this run; the team is unit-first and the domain is well-understood -->
- 2026-09-29T14:20:00Z — Contracts for content pending the Report & UI Specification use extensible objects so filling them later is additive, at the cost of weaker typing until then.

## Open questions
<!-- example: 2026-05-29T10:14:32Z — confirm the retention window with compliance before the next stage hardens the schema -->
