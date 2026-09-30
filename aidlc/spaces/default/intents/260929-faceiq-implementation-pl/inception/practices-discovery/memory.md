<!-- INVARIANT: examples are single-line HTML comments so a fresh template parses to total=0 (MEMORY_EMPTY). Do NOT un-comment or split across lines. t100 guards this. -->
> This file is kept up to date automatically while the stage runs. Add observations at the review step, not by editing here directly.

## Interpretations
<!-- example: 2026-05-29T10:14:32Z — chose REST over GraphQL; the consuming team only needs CRUD, revisit if subscriptions land -->
- 2026-09-29T07:11:33Z — Walking skeleton framed as guidance for a later build workflow; this planning-only scope sets no skeleton value and skips Construction, so no skeleton ceremony runs in this workflow.
- 2026-09-29T07:11:33Z — client_requirements.md used as evidence alongside org.md defaults; the user asked for project rules derived from it as the single source of truth.
- 2026-09-29T07:40:00Z — Treated the free-text deployment answer ('no need to deploy anywhere, just build and push to github') as no deployment jobs, confirmed by follow-up Q7a: GitHub Actions still runs blocking checks on every push and PR.
- 2026-09-29T07:40:00Z — A non-secret default inside the typed settings object, overridable by configuration, is not hardcoding (NFR-008); secrets never get in-code defaults.

## Deviations
<!-- example: 2026-05-29T10:14:32Z — skipped the optional caching layer the stage prose suggested; the dataset is small enough that it adds risk -->
- 2026-09-29T07:11:33Z — Product/business rules (FR-008(f), FR-019, BR-008..BR-012) kept out of discovered-rules.md; they belong in requirements-analysis, not project-wide engineering rules.
- 2026-09-29T07:40:00Z — Org default deployment rule (staging on merge, manual prod approval) not affirmed for this project; replaced by build-and-push only until a host is chosen.

## Tradeoffs
<!-- example: 2026-05-29T10:14:32Z — picked TDD over BDD this run; the team is unit-first and the domain is well-understood -->
- 2026-09-29T07:11:33Z — Proposed an explicit 80% coverage floor as a recommendation, because the org default applies it only to stock scopes and this custom scope is not on that list.
- 2026-09-29T07:11:33Z — Draft recommends one repository with backend/ and frontend/ folders over two repositories, for easier handover to the client team (BC-005, NFR-011); left open for the interview.
- 2026-09-29T07:40:00Z — Walking skeleton set to the auth slice (full login layering) over the lighter infra slice; the developer reviewer preferred infra, the human chose auth to de-risk the most constrained path.
- 2026-09-29T07:40:00Z — E2E tests read the OTP from a local SMTP catcher rather than a test-only inbox endpoint, avoiding a production-exposure risk.

## Open questions
<!-- example: 2026-05-29T10:14:32Z — confirm the retention window with compliance before the next stage hardens the schema -->
- 2026-09-29T07:40:00Z — Target market and privacy law (GDPR Art. 9, BIPA, CCPA) for face photos, biometric signature and health answers sent to OpenAI are unstated; formal privacy policy is out of scope (section 2.2).
- 2026-09-29T07:40:00Z — Semgrep vs CodeQL left open; depends on whether the repository is public or has GitHub Advanced Security.
