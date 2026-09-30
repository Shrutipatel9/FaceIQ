# User Stories Assessment — FaceIQ

## Decision

**Execute.** User stories are needed for this project.

## Rationale

- The user explicitly asked to "plan the requirements in stories and each story have phases for implementation" (initial request).
- The product is almost entirely user-facing. It is a consumer web app with a long, gated journey (`WF-001`): account → consent → questionnaire → photos → payment → analysis → report → post-analysis screens.
- It carries complex business logic that stories must make testable:
  - the photo identity-check vote (`BR-005`);
  - the payment gate (`FR-015`, `BR-001`);
  - OTP and token rotation rules (`AUTH-011`, `AUTH-012`);
  - AI tone and content rules (`FR-008`, `FR-019`).
- The client will keep extending the codebase with their own team (`BC-005`). Well-formed, independently testable stories with acceptance criteria give that team a durable backlog.

## Factors Considered

| Factor | Signal | Weight |
|--------|--------|--------|
| Project type | Greenfield consumer web app (`CON-001`) | High |
| User-facing scope | Nearly all 26 functional requirements (FR1–FR26 in `requirements.md`) have a UI surface | High |
| Personas | One functional role (End User) with distinct behavioural needs (goal-driven, distressed, returning); Admin deferred (`AUTH-009`) | Medium |
| Business-logic complexity | Identity vote, payment gate, token family revocation, AI content rules | High |
| Cross-team work | Two separate apps (`NFR-001`, `NFR-005`) plus a later client team | Medium |

## Where Stories Add the Most Value

- Splitting the long WF-001 journey into independently testable vertical slices.
- Making the gating rules explicit per story, so that no story leaks content before payment or skips validation.
- Giving each story concrete Given/When/Then acceptance criteria that map to the tagged-test practice in `team-practices.md` (every requirement mapped to at least one test).
- Carrying implementation phases per story, as the user requested, so later units and the delivery plan can sequence them.
- Recording the reviewers' journey edge cases (dead ends, loading/error states, unverified accounts) as testable criteria before design starts.
