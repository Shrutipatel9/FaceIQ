# Practices Discovery — Questions

These questions settle how the team will build FaceIQ. The suggested answers come from our standard defaults, the client requirements (`client_requirements.md`), and the release engineer's draft as reviewed by the quality, developer and security engineers. Drafts: `team-practices.md`, `discovered-rules.md`, `contributions/`.

Fill each `[Answer]:` with a letter, or `X` plus your own text.

## Q1 — Repository layout

The client requires two separate applications: a Python/FastAPI backend and a Next.js/TypeScript frontend (NFR-001, NFR-005). Where should they live?

A. One repository with top-level `backend/` and `frontend/` folders, each with its own tooling, README and CI job (recommended: easier handover to the client's team, and an API change and its frontend use can ship in one pull request)
B. Two separate repositories, one per application
X. Other (please specify)

[Answer]: A

## Q2 — Branching and code review

How should changes reach `main`?

A. Short-lived feature branches merged to `main` by pull request: CI must pass and 1 reviewer must approve; squash-merge; titles cite the requirement IDs they implement, e.g. `AUTH-012` (recommended; our standard default)
B. Same as A, but 2 reviewer approvals are required
C. Same as A, but small changes may be committed straight to `main` without a pull request
X. Other (please specify)

[Answer]: A

## Q3 — Build a thin end-to-end slice first?

A walking skeleton is a minimal version that runs the whole way through, built first to prove the pieces connect before the real features go in. It applies when implementation starts (this workflow only plans).

A. Yes: a simple page → API client → one FastAPI endpoint → PostgreSQL via SQLAlchemy and one Alembic migration, runnable with one documented command
B. Yes: the email + password login step running through the full required auth layering (UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services); heavier, but de-risks the most constrained part (recommended)
C. No: start straight on features
X. Other (please specify)

[Answer]: B

## Q4 — When are tests written?

Every functional, auth and business rule must map to at least one test (client requirements §12). The question is the order of writing tests versus code.

A. Tests after code: build each layer, then write and run its tests (our standard default)
B. Mixed: write tests first for the rules the client specifies exactly (auth token rotation and lockout, the payment gate, the photo identity-check vote, recommendation tiering), and write tests after code everywhere else (recommended)
C. Tests first (TDD) for everything
D. Behaviour scenarios first (BDD, Given/When/Then feature files) driving all work
X. Other (please specify)

[Answer]: B

## Q5 — Test coverage floor

What minimum test coverage should block a merge?

A. 80% line coverage per application
B. 80% line and branch coverage per application, plus 90% for the auth, payment-gate and photo identity-check code; exclusions only through a reviewed list (recommended)
C. No enforced floor; coverage is reported only
X. Other (please specify)

[Answer]: B

## Q6 — Face photos as test data

Computer-vision tests need faces, but face photos are sensitive biometric data. What is allowed?

A. Never use real user photos. Test most CV logic on stored landmark coordinates instead of images, and use only licensed or consented images for the few tests that need real pictures, kept outside the public repository if required (recommended)
B. Team members' own photos may be committed, with their written consent
X. Other (please specify)

[Answer]: A

## Q7 — Deployment and CI

Hosting is deliberately undecided (NFR-010, CON-007). What should the build pipeline do for now?

A. Run CI on GitHub Actions for every pull request. Adopt our standard deployment rule for when a host is chosen: deploy to staging on merge, production only after manual approval. Keep everything host-neutral with all settings from environment variables (recommended)
B. Run CI only; decide the deployment approach later, once a host is chosen
X. Other (please specify)

[Answer]: X. Currently no need to deploy anywhere.just build and push to github

## Q8 — Code style and tooling

Which tooling set should the project standardise on?

A. Backend: Ruff for lint and format, mypy strict, pytest. Frontend: TypeScript strict, ESLint, Prettier, Vitest with React Testing Library, Playwright for end-to-end, pnpm. Both: pre-commit hooks, and frontend API types generated from the backend's OpenAPI schema with a CI check for drift (recommended)
B. Same as A, but Black + Ruff, Jest and npm
C. Same as A, but no pre-commit hooks and no generated API types
X. Other (please specify)

[Answer]: A

## Q9 — Security checks in CI

The draft had no security scanning. How strict should it be?

A. Blocking: secret scanning (Gitleaks), code scanning (Semgrep or CodeQL) and dependency scanning (pip-audit, npm audit, Dependabot) fail the build on High/Critical findings. Waivers are kept in the repository with an expiry date (recommended)
B. The same scanners, but advisory only until launch
C. Only secret scanning is blocking; the rest are advisory
X. Other (please specify)

[Answer]: A

## Q10 — Hard project rules

The draft and reviews propose hard rules, the ALWAYS and NEVER lists in `discovered-rules.md` plus reviewer additions. Examples: no third-party auth service; refresh token only in an httpOnly cookie; server-side 402/409 gates; no hardcoded secrets, price or model names; no hosting assumptions; no out-of-scope features; no live OpenAI, Stripe or email calls in per-PR tests; never log tokens, OTPs, photos or health answers. Which become enforced project rules?

A. All client-stated ones, plus the security and test-data rules the reviewers added (recommended)
B. Only the rules directly stated by the client; keep the rest as ordinary practices
X. Other (please specify)

[Answer]: A

## Q7a — Follow-up: automated checks on GitHub

You said there is no need to deploy anywhere for now: just build and push to GitHub. Should GitHub still run the automated checks (lint, type-check, tests, coverage floor, security scans) on every push and pull request?

A. Yes: GitHub Actions runs the checks on every push and pull request, and a failing check blocks the merge. No deployment jobs (recommended: the coverage floor and security checks you chose need somewhere to run)
B. No automated checks on GitHub for now; developers run the checks locally before pushing
X. Other (please specify)

[Answer]: A

## Consolidated Summary Confirmation

Summary of answers:
- Repository: one repository with `backend/` and `frontend/` folders (Q1 A)
- Branching: short-lived branches, pull request with CI green and 1 approval, squash-merge, titles cite requirement IDs (Q2 A)
- Thin end-to-end slice first: yes, the login step through the full auth layering (Q3 B)
- Test ordering: tests first for exactly-specified rules (auth, payment gate, identity vote, tiering), tests after elsewhere (Q4 B)
- Coverage: 80% line and branch per app, 90% for auth, payment-gate and identity-check code, reviewed exclusion list (Q5 B)
- Test photos: no real user photos; landmark-coordinate fixtures; only licensed or consented images (Q6 A)
- Deployment: none for now; just build and push to GitHub (Q7 X)
- GitHub checks: GitHub Actions runs lint, type-check, tests, coverage and security scans on every push and pull request, blocking merge; no deployment jobs (Q7a A)
- Tooling: Ruff, mypy strict, pytest; TypeScript strict, ESLint, Prettier, Vitest + React Testing Library, Playwright, pnpm; pre-commit hooks; OpenAPI-generated API types with drift check (Q8 A)
- Security scanning: blocking on High/Critical (Gitleaks, Semgrep/CodeQL, pip-audit, npm audit, Dependabot); waivers in-repo with expiry (Q9 A)
- Hard rules: all client-stated rules plus the reviewers' security and test-data rules (Q10 A)

Does this all look correct before I generate the artifact?

- Looks correct
- Request changes

[Answer]: Looks correct
