# User Stories — FaceIQ

## Overview

- **Source:** `inception/requirements-analysis/requirements.md` (FR1–FR26, NFR1–NFR12), traced to `client_requirements.md` IDs.
- **Testing posture:** follows the affirmed `team-practices`:
  - tests first for exactly-specified rules; tests after elsewhere;
  - every requirement mapped to a tagged test;
  - 80% coverage per app, 90% for auth, payment-gate and identity-check code.
- **Answers applied:** the story plan and review follow-ups in `user-stories-questions.md` (Q1–Q19). Answers are **[Decided by delivery team]** unless a client ID is cited.
- **Personas:** P1 Alex (goal-driven), P2 Jordan (appearance-anxious), P3 Sam (returning). See `personas.md`. "All" means a cross-cutting condition any persona may have.
- **Structure:** ten epics in WF-001 journey order. E0 holds **Enabler** items (technical foundations with acceptance criteria and phases, not "As a user" value stories).
- **Size:** each story is about 1–3 developer-days (Q3, Q9). Stories estimated above 3 days in review were split.
- **Each story carries:**
  - MoSCoW priority;
  - trace IDs and dependencies;
  - Given/When/Then acceptance criteria (`AC{group}.{seq}.{n}`);
  - its own ordered **implementation phases**:
    - **P-Data:** schema and Alembic migration.
    - **P-API:** backend service, repository and route, with their tests.
    - **P-UI:** frontend screen, state and API-client wiring, with their tests.
    - **P-E2E:** Playwright test and docs.
- **"Tests first":** marks phases where the affirmed ordering puts tests before code:
  - auth token rotation, reuse and OTP lockout;
  - the 402 payment gate;
  - the identity vote and 409 re-check;
  - recommendation tiering.

  Deliberate extensions of the same idea (frontend refresh logic, webhook, cross-user access) are marked "tests first (extension)".
- **Conventions used in every acceptance criterion:**
  - A cross-user request for another user's resource returns **404**, so resources cannot be enumerated.
  - A rate-limited request returns **429** with code `rate_limited`.
  - An unpaid request to a gated route returns **402**.
  - Time-based rules are tested with an injectable clock at boundary ±1 s.
- **Not automated acceptance criteria:**
  - **Visual realism** and identity preservation of AI images: manual review (`ASM-011`).
  - **Live-model behaviour:** the opt-in evaluation suite, never a per-PR gate.
  - Content the client marked **unconfirmed** (Face Shape, Skin view, Protocol, key stats, label sets): the reviewed pending list until the Report & UI Specification arrives.
- **MVP boundary:** decided in Delivery Planning; priorities here only inform it.

## Definition of Done (every story)

Applied to every story instead of separate app-wide stories (Q10):

1. **Tests:** acceptance criteria automated as tagged tests (requirement IDs), with happy path plus at least two error or edge cases per test file. Coverage floors hold.
2. **Accessibility (NFR6):** every story with a P-UI phase passes automated axe checks with zero violations and a manual keyboard pass for its screens. Form errors are shown inline and linked to their field; a toast is never the only error channel.
3. **Feedback (FR25.1, `BR-009`):** every action with a success/failure outcome shows a top-right toast through the shared component. Toasts are polite for success, assertive for error, and pause on hover/focus. Anti-enumeration flows show only neutral wording. A network error or 5xx keeps form data and never signs the user out (`FE-006`).
4. **Confirmation (FR25.2, `BR-010`):** irreversible actions (Logout, Start Analysis) use the shared confirmation dialog. Focus moves into the dialog, starting on the non-destructive action, and returns to the trigger on close.
5. **Security:**
   - Owner-scoped endpoints include a cross-user 404 test.
   - Gated endpoints are registered with the payment gate.
   - No token, OTP, photo, biometric signature or health answer is logged.
   - AI text renders as text only.
6. **Docs:** README configuration keys and public-module docstrings/TSDoc are updated in the same pull request.
7. **Pending content:** UI phases that depend on the pending Report & UI, Onboarding Questionnaire, Photo Capture or Photo Validation Specification use format fixtures only. They never invent questions, thresholds, labels or layout.

## Story Map

| Epic | Stories | Must | Should |
|------|---------|------|--------|
| E0 Enablers | US0.1–US0.11 | 11 | 0 |
| E1 Account & Session | US1.1–US1.11 | 11 | 0 |
| E2 Consent & Onboarding | US2.1–US2.4 | 4 | 0 |
| E3 Photos & Validation | US3.1–US3.9 | 9 | 0 |
| E4 Payment | US4.1–US4.3 | 3 | 0 |
| E5 Analysis Pipeline | US5.1–US5.11 | 11 | 0 |
| E6 Report | US6.1–US6.5 | 5 | 0 |
| E7 Post-Analysis Experience | US7.1–US7.5 | 4 | 1 |
| E8 Settings | US8.1–US8.3 | 3 | 0 |
| E9 App-wide | US9.1–US9.5 | 3 | 2 |
| **Total** | **67** | **64** | **3** |

**Won't Have (this scope)** (`client_requirements.md` section 2.2):
- Admin panel and review workflow.
- SendGrid notifications, and any "we'll email you" promise.
- Meta Pixel/GTM.
- A formal retention policy.
- A working PayPal integration.
- User-triggered image regeneration (NFR8).
- Editing the full name (`ASM-009`).

---

## E0 — Enablers

### US0.1 — Repository scaffold and core CI (Enabler)
**Priority:** Must · **Traces:** NFR9, NFR10, NFR11 · **Depends on:** none

One repository with `backend/` and `frontend/`, their toolchains, and the core GitHub Actions checks.
- AC0.1.1 Given a pull request to `main`, when CI runs, then lint (Ruff, ESLint), format check (Ruff, Prettier), type-check (mypy strict, `tsc --noEmit`) and tests run for both apps, and any failure blocks merge.
- AC0.1.2 Given the repository root, when `.gitignore` is inspected, then it excludes `.env`, `.env.*`, `*.pem` and `*.key`. `.env.example` files list every configuration key.
- AC0.1.3 Given the CI workflow, when inspected, then it contains no deployment job (NFR11).
- AC0.1.4 Given each app, when its README is read, then it documents setup, run, test and configuration keys.

Implementation phases:
1. Scaffold `backend/` (pyproject, Ruff, mypy, pytest; pinned Python version with MediaPipe wheels for Windows, Linux x86_64 and macOS) and `frontend/` (Next.js App Router, TypeScript strict, ESLint, Prettier, Vitest, pnpm).
2. Pre-commit hooks, `.gitignore`, `.env.example`.
3. GitHub Actions: lint, format, types and tests jobs (SHA-pinned actions, `permissions: contents: read`).
4. Per-app READMEs.

### US0.2 — Security, coverage and traceability gates (Enabler)
**Priority:** Must · **Traces:** NFR10, NFR4 · **Depends on:** US0.1

The blocking security scans, coverage floors and the requirement-to-test traceability check.
- AC0.2.1 Given a commit containing a secret-like string, when CI runs, then the Gitleaks scan fails the build.
- AC0.2.2 Given a High/Critical finding from Semgrep or CodeQL, pip-audit or pnpm audit with no unexpired waiver, when CI runs, then the build fails.
- AC0.2.3 Given coverage below 80% line or branch in either app, or below 90% in the auth, payment-gate and identity-check backend modules or the frontend auth store and API client, when CI runs, then the build fails.
- AC0.2.4 Given a client `FR-*`, `AUTH-*` or `BR-*` ID with no tagged test and not on the reviewed pending list, when CI runs, then the build fails.
- AC0.2.5 Given a waiver without an expiry date, when CI runs, then the build fails.

Implementation phases:
1. Gitleaks, Semgrep/CodeQL (choice depends on repository visibility), pip-audit, pnpm audit, Dependabot.
2. Coverage configuration with per-module floors and the reviewed exclusion list.
3. Traceability script (test tags vs client IDs) plus a pending-list file (initial entries: `BR-006` and `BR-007` as commercial rules that cannot be automated; unconfirmed content IDs).

### US0.3 — Backend foundation (Enabler)
**Priority:** Must · **Traces:** NFR1, NFR2, NFR12, NFR4 · **Depends on:** US0.1

A FastAPI app with a typed settings object, PostgreSQL through SQLAlchemy and Alembic, a health endpoint, request-ID and timing logs, one error envelope, and a public-config endpoint.
- AC0.3.1 Given a valid environment, when `GET /health` is called, then it returns 200 with database connectivity status.
- AC0.3.2 Given a required secret is missing, when the app starts, then it fails fast with a clear configuration error. No secret has an in-code default.
- AC0.3.3 Given any handled error, when it reaches the client, then it uses the single JSON error envelope with a stable code.
- AC0.3.4 Given sentinel values for password, OTP, token, `reset_token`, Stripe signature and a health answer passed through a request, when captured logs are scanned, then no sentinel appears. Each request logs a request ID and duration.
- AC0.3.5 Given `GET /config/public`, when called, then it returns only non-secret values: the required photo-angle set, price and currency.

Implementation phases:
1. P-API: typed settings object, app factory, error envelope and handlers, log redaction, request-ID and timing middleware, health and public-config routes.
2. P-Data: SQLAlchemy base and Alembic environment; migration round-trip test (upgrade → downgrade → upgrade) in CI.

### US0.4 — One-command local environment (Enabler)
**Priority:** Must · **Traces:** NFR11, NFR10 · **Depends on:** US0.3

One documented command starts PostgreSQL, a local SMTP catcher (Mailpit), the backend API, the job worker and the frontend. CI uses the same services.
- AC0.4.1 Given a clean machine with the documented prerequisites, when the single start command runs, then all five services start and the health check passes.
- AC0.4.2 Given CI, when E2E jobs run, then PostgreSQL and Mailpit run as service containers with the same configuration keys.
- AC0.4.3 Given the setup, when inspected, then no hosting provider is assumed (`NFR-010`).

Implementation phases:
1. Compose file (or equivalent) and start script.
2. CI service containers.
3. README section.

### US0.5 — Frontend foundation (Enabler)
**Priority:** Must · **Traces:** NFR1, NFR6, NFR9, FR25, NFR4 · **Depends on:** US0.3

A Next.js app shell with the centralized API client, shared toast and confirmation components, OpenAPI type generation, a CSP, and accessibility tooling.
- AC0.5.1 Given any frontend module outside `lib/api`, when lint runs, then a direct `fetch` call is an error.
- AC0.5.2 Given the toast and confirm-dialog components, when tested, then they meet the Definition of Done items 3 and 4 and pass axe.
- AC0.5.3 Given the backend OpenAPI schema changes, when CI runs, then stale generated API types fail the drift check.
- AC0.5.4 Given any page response, when headers are inspected, then a Content-Security-Policy header is present.
- AC0.5.5 Given every `ApiError.code` in the envelope catalogue, when mapped, then it has toast copy. An unmapped code falls back to a generic message.

Implementation phases:
1. P-UI: app shell, API client (credentials included, envelope mapping), toast and confirm-dialog components.
2. P-UI: OpenAPI type generation and drift check; ESLint import restrictions; CSP.
3. P-E2E: Playwright with per-route axe harness; Mock Service Worker for unit tests.

### US0.6 — Auth layering walking skeleton (Enabler)
**Priority:** Must · **Traces:** FR4.8, FR3.1, NFR1 (`AUTH-001`, `AUTH-005`, `AUTH-009`, `FE-001`, `FE-003`, `FE-007`, `NFR-005`) · **Depends on:** US0.3, US0.4, US0.5

The thin end-to-end slice (affirmed Walking Skeleton): the email + password login step through UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services. It stops at "OTP required" and issues or sends no code.
- AC0.6.1 Given a seeded verified user, when the login form submits email and password, then the request passes through the store and API client to the backend auth service, and the response is "OTP required" with no token.
- AC0.6.2 Given the allow-listed frontend origin with `credentials: include`, when it calls the backend, then CORS succeeds. A non-listed origin is refused.
- AC0.6.3 Given the code, when lint runs, then a UI component importing auth API functions or token state directly fails (FE-007). A backend service importing FastAPI, or a route importing a repository, also fails (import-linter).
- AC0.6.4 Given the `users` table, when a user is created, then `role` exists and defaults to `"user"` (`AUTH-009`).
- AC0.6.5 Given the dependency manifests, when checked, then no third-party auth SDK (Supabase Auth, Auth0, Firebase Auth) is present (`AUTH-001`).

Implementation phases:
1. P-Data: `users` table (full name, email, password hash, verified flag, role, timestamps) (DATA-001).
2. P-API: auth service with Argon2id password verification behind the login route.
3. P-UI: single Zustand auth store (no `persist`) and login form wired only through the store.
4. P-E2E: the skeleton test, run by the one documented command (the Construction verification command candidate).

### US0.7 — Vendor adapter interfaces and fakes (Enabler)
**Priority:** Must · **Traces:** NFR2, NFR8, FR17.2 · **Depends on:** US0.3

Config-selected interfaces for email, AI text, image generation, payments and file storage. Each has a fake and a shared contract-test harness. Each real adapter ships with its first consuming story: email (US1.2), storage (US3.2), Stripe (US4.1), OpenAI text (US5.6), OpenAI image (US5.10).
- AC0.7.1 Given a vendor setting is changed, when the app starts, then the factory returns the matching implementation.
- AC0.7.2 Given the per-PR test run, when any code attempts outbound network access, then the socket-blocking plugin fails the test (no live OpenAI, Stripe live mode or real email).
- AC0.7.3 Given the image-adapter contract suite, when run against the fake, then it passes.

Implementation phases:
1. P-API: interfaces and config-driven factory.
2. P-API: fakes, contract-suite harness, network blocking in tests.

### US0.8 — Rate-limit primitive (Enabler)
**Priority:** Must · **Traces:** NFR4, NFR8 (`AUTH-011`) · **Depends on:** US0.3

PostgreSQL-backed counters keyed per account, per IP and per user, with configurable windows and limits (no cache server is decided).
- AC0.8.1 Given a limit N per window, when request N arrives, then it succeeds. Request N+1 inside the window gets 429 `rate_limited`.
- AC0.8.2 Given the window elapses (injectable clock), when a request arrives, then it succeeds again.

Implementation phases:
1. P-Data: counters table.
2. P-API: limiter dependency with tests.

### US0.9 — Test-data factories and E2E seeding (Enabler)
**Priority:** Must · **Traces:** NFR10 · **Depends on:** US0.3

Backend factories and a seeding CLI that write directly to the database (never through a test-only HTTP endpoint). They cover users, OTP records, sessions, payments, photo sets and analysis states.
- AC0.9.1 Given the seed command with a named scenario (for example `paid-consistent-set`), when run, then the database holds exactly that state and E2E tests can start from it.
- AC0.9.2 Given the application's HTTP routes, when listed, then no test-only seeding route exists.

Implementation phases:
1. P-API: factories.
2. Seeding CLI and scenario catalogue.

### US0.10 — Background job runner (Enabler)
**Priority:** Must · **Traces:** NFR7, FR13.3 · **Depends on:** US0.3

A separate worker process polls a PostgreSQL jobs table using `SKIP LOCKED` with a lease/heartbeat. Steps are idempotent and resumable, with bounded retries and per-step timeouts. The process model is to be confirmed by ADR in Domain Design [Recommendation].
- AC0.10.1 Given steps A→B→C where B fails once, when retries are enabled, then B is retried and C runs. A is not re-run.
- AC0.10.2 Given a step fails past the configured retry limit N, when attempt N fails, then the job is marked failed at that step and completed steps are kept.
- AC0.10.3 Given two workers, when both poll, then no job is claimed twice.
- AC0.10.4 Given a worker dies mid-step, when its lease expires, then another worker reclaims the job and continues from the first incomplete step.
- AC0.10.5 Given a step exceeds its configured timeout, when detected, then it counts as a failed attempt.

Implementation phases:
1. P-Data: job and step tables.
2. P-API: worker loop, claim/lease, retry/timeout, per-step duration logging (NFR12).
3. P-API tests with fake steps.

### US0.11 — Computer-vision feasibility spike (Enabler, time-boxed)
**Priority:** Must · **Traces:** FR10, FR11, FR14 (`ASM-002`, `ASM-010`) · **Depends on:** US0.1

A time-boxed check that the chosen approach works before the validation and measurement stories begin. It covers:
- MediaPipe Face Landmarker on the pinned Python and OS;
- multi-face detection;
- pose (yaw) from landmarks;
- an occlusion signal;
- stability of the 5-ratio identity signature across front and 3/4 views.
- AC0.11.1 Given licensed or consented sample images, when the spike runs, then a written finding records, for each technique, whether it works and what inputs it needs. Risks go to Domain Design.
- AC0.11.2 Given the model file, when installed, then it is fetched by pinned URL with a SHA-256 check (or committed with its licence).

Implementation phases:
1. Spike scripts.
2. Findings note (no production code merged).

---

## E1 — Account & Session

### US1.1 — Discover the product on the landing page
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR1 (`FR-001`, `CON-003`) · **Depends on:** US0.5

As a first-time visitor, I want a clear landing page that explains how it works and lets me start or sign in, so that I understand the product before committing.
- AC1.1.1 Given the landing page, when "Get started" is clicked, then the signup screen opens.
- AC1.1.2 Given the landing page, when the header "Sign in" link is clicked, then the login screen opens.
- AC1.1.3 Given "How it works" scrolls into view, when shown, then the four steps (questionnaire → photos → analysis → report) animate in order. With reduced motion preferred, they appear without animation.
- AC1.1.4 Given a signed-in user with a restored session, when they open the landing page or click "Get started", then they go to their next incomplete step (FR26).

Implementation phases:
1. P-UI: landing page, header, scroll animation with reduced-motion support (copy owned by the delivery team, FR1.3).
2. P-E2E: navigation tests.

### US1.2 — Sign up with name, email and password
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR2.1, FR2.4 (`FR-002`, `AUTH-003`, `WF-002`, Q6, Q8, Q19), NFR4 · **Depends on:** US0.6, US0.7, US0.8

As a new visitor, I want to create an account with my full name, email and password, so that I can start my analysis.
- AC1.2.1 Given a new email, when full name, email and a valid password are submitted, then an unverified user is created, a verification code is emailed and the OTP screen shows. No token is issued.
- AC1.2.2 Given an email belonging to a **verified** account, when signup is submitted, then the same OTP-screen outcome is shown and the account owner is emailed a "someone tried to sign up with your email" notice. No account is created or disclosed.
- AC1.2.3 Given an email belonging to an **unverified** account, when signup is submitted, then the pending signup is updated and a fresh verification code is sent (Q8).
- AC1.2.4 Given a password of 9 characters, or one on the common-password list, when submitted, then signup is refused with an inline field message. A password of 10 characters that is not on the list is accepted. The password rules are shown before submission, and paste is allowed (Q19).
- AC1.2.5 Given the signup rate limit is exceeded per IP or per email, when another request arrives, then it gets 429.
- AC1.2.6 Given a stored user, when the password column is read, then it holds an Argon2id hash, never the plaintext.

Implementation phases:
1. P-Data: OTP records table (hashed code, purpose, expiry, attempts, consumed) (DATA-002).
2. P-API: register service (password policy, Argon2id, unverified user, neutral and resume paths), real SMTP email adapter (provider-neutral).
3. P-UI: signup form through the auth store.
4. P-E2E: signup → OTP screen using Mailpit.

### US1.3 — One-time code rules (verify service)
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR2.2 (`AUTH-011`), FR5.1 · **Depends on:** US1.2

As a user, I want my emailed code to be short-lived and resistant to guessing, so that nobody can get into my account with it.
- AC1.3.1 Given an issued OTP, when the `otp_records` row is read, then no column equals the plaintext code.
- AC1.3.2 Given an OTP, when submitted at 9 min 59 s, then it is accepted. At 10 min 00 s it is rejected (the limit is exclusive).
- AC1.3.3 Given 5 failed attempts for the same account and purpose, when a 6th attempt is made within 15 minutes of the 5th failure, then it is rejected with a lockout message stating when to retry, even if the code is correct.
- AC1.3.4 Given a lockout started at T, when a correct fresh OTP is submitted at T + 15 min + 1 s, then it is accepted.
- AC1.3.5 Given an active lockout, when a resend is requested, then no code is issued and the lockout and attempt count are not reset.

Implementation phases (tests first):
1. P-API tests first: expiry, attempts and lockout for purposes `signup`, `login` and `password_reset` with an injectable clock.
2. P-API: OTP verify service.

### US1.4 — Complete signup: tokens and session
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR2.3, FR4.1 (`AUTH-002`, `AUTH-004`, `AUTH-012`, `DATA-003`) · **Depends on:** US1.3

As a new user, I want entering the correct code to verify my account and sign me in, so that I can continue straight away.
- AC1.4.1 Given a correct, unexpired OTP, when submitted, then the account is marked verified, an access token with `exp = iat + 15 min` and a token-type claim is returned, and a refresh cookie valid for 7 days is set.
- AC1.4.2 Given the `Set-Cookie` header, when inspected, then the refresh cookie has `HttpOnly`, `Secure`, an explicit `SameSite` and a `Path` scoped to the auth routes.
- AC1.4.3 Given the OTP screen, when shown, then it names the email the code was sent to, accepts pasting the full code, and uses numeric input with one-time-code autofill hints.

Implementation phases:
1. P-Data: refresh sessions table with lineage (`family_id`, parent/replaced-by, `revoked_at`) (DATA-003).
2. P-API: token service (fixed algorithm allow-list, token-type claim), session record, cookie.
3. P-UI: OTP screen; the access token is stored in memory only.
4. P-E2E: full signup with OTP via Mailpit.

### US1.5 — Resend a code without being spammed
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR2.2 (`AUTH-011`), NFR4 · **Depends on:** US1.3, US0.8

As a user waiting for a code, I want to request a new one, so that I can continue if the first email didn't arrive.
- AC1.5.1 Given the last code was sent 59 s ago, when resend is requested, then it is refused with the remaining wait time. At 60 s it is allowed.
- AC1.5.2 Given a new code is sent, when the previous code is submitted, then it is rejected.
- AC1.5.3 Given the per-IP resend limit N is reached across accounts, when request N+1 arrives, then it gets 429.
- AC1.5.4 Given the resend countdown, when running, then its remaining time is available to screen readers without announcing every second.

Implementation phases (tests first):
1. P-API tests first: cooldown and supersession.
2. P-API: resend endpoint with limits.
3. P-UI: resend control with countdown.

### US1.6 — Log in with password and code
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR3 (`AUTH-003`, `AUTH-006`, `WF-002`, Q6, Q8) · **Depends on:** US1.4

As a returning user, I want to log in with my email, password and an emailed code, so that my account stays protected.
- AC1.6.1 Given correct credentials for a verified account, when submitted, then a login OTP is emailed and the OTP screen shows. No token is issued yet.
- AC1.6.2 Given an unknown email, and separately a wrong password, when each is submitted, then both return the identical status code and body.
- AC1.6.3 Given correct credentials for an **unverified** account, when submitted and the emailed code is verified, then the account becomes verified and the user continues to consent (Q8).
- AC1.6.4 Given the login rate limit per IP or per account is exceeded, when another attempt arrives, then it gets 429.
- AC1.6.5 Given the correct login OTP, when submitted, then the user is signed in and routed to the next incomplete step (FR26).

Implementation phases (tests first for the login purpose's lockout):
1. P-API tests first: lockout and expiry tests parametrised for purpose `login`.
2. P-API: login service reusing the OTP and token services; generic failure responses.
3. P-UI: extend the US0.6 login form with the OTP step.
4. P-E2E: login journey via Mailpit.

### US1.7 — Protect my session if a token is stolen
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR4.2, FR4.7, FR4.1 (`AUTH-010`, `AUTH-012`, `AUTH-013`), NFR4 · **Depends on:** US1.4

As a user, I want a replayed or stolen session token to be useless, so that my account and photos stay private.
- AC1.7.1 Given a valid refresh, when completed, then a new refresh token replaces the cookie and the old token no longer works.
- AC1.7.2 Given an already-rotated refresh token, when presented again, then the entire token family is revoked and 401 is returned.
- AC1.7.3 Given a refresh request missing `X-Requested-With`, or with an Origin (or, when Origin is absent, a Referer) not on the allow-list, when received, then it returns 403 `csrf_rejected`.
- AC1.7.4 Given a refresh token at 7 days + 1 s, when presented, then 401 is returned.

Implementation phases (tests first):
1. P-API tests first: rotation, reuse and family revocation, expiry, CSRF.
2. P-API: `POST /auth/refresh` and CSRF middleware.

### US1.8 — Stay signed in across reloads and restarts
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR4.4, FR4.5 (`FE-002`, `AUTH-014`) · **Depends on:** US1.7

As a returning user, I want my session to survive page reloads and browser restarts, so that I don't have to log in every visit.
- AC1.8.1 Given a valid refresh cookie, when the app starts, then it calls refresh then `/auth/me`, and the user sees their screen without the login page.
- AC1.8.2 Given the session is being restored, when a protected route renders, then it shows a loading state and does not redirect.
- AC1.8.3 Given no cookie, or an expired or revoked one, when the app starts, then the login screen shows with no error toast and no redirect loop.
- AC1.8.4 Given sign-in has completed, when browser storage is inspected, then nothing is written to `localStorage` or `sessionStorage`, and lint fails if the auth store imports Zustand `persist`.

Implementation phases:
1. P-API: `GET /auth/me`.
2. P-UI: app-start restore, initializing flag, client-side protected-route guard.
3. P-E2E: reload and restart scenario.

### US1.9 — Seamless token refresh
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR4.3, FR4.5 (`FE-004`, `FE-005`, `FE-006`) · **Depends on:** US1.8

As a signed-in user, I want expired access tokens to renew silently, so that I'm never interrupted mid-task.
- AC1.9.1 Given an access token with a 15-minute lifetime, when fake time reaches expiry − 60 s, then exactly one refresh is sent before any request gets 401.
- AC1.9.2 Given three concurrent requests all receive 401, when handled, then exactly one refresh is sent and each request is retried once.
- AC1.9.3 Given refresh fails with a network error or 5xx, when handled, then the user stays signed in and no redirect happens.
- AC1.9.4 Given refresh returns 401, when handled, then the store is cleared, the user is sent to login, and a toast says the session has ended.

Implementation phases (tests first (extension)):
1. P-UI tests first: Mock Service Worker with fake timers for proactive, 401-retry, single-flight and no-clear cases.
2. P-UI: API client refresh logic.

### US1.10 — Log out safely
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR4.6 (`WF-002` step 5, `BR-010`), FR25.2 · **Depends on:** US1.8

As a user, I want to log out after confirming, so that nobody else can use my session on this device.
- AC1.10.1 Given the user confirms Logout, when processed, then the server session is revoked, the cookie cleared, the store emptied and the login screen shown.
- AC1.10.2 Given the confirmation dialog, when cancelled or dismissed with Escape, then the user stays signed in.
- AC1.10.3 Given the user navigates away, closes the tab or changes visibility, when this happens, then no request is sent to `/auth/logout`.

Implementation phases:
1. P-API: logout endpoint (CSRF-protected).
2. P-UI: logout via the store and the shared confirm dialog.
3. P-E2E: logout journey.

### US1.11 — Reset a forgotten password
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR5 (`AUTH-015`, `BR-009`, Q19) · **Depends on:** US1.3, US1.7

As a user who forgot my password, I want to reset it with an emailed code, so that I can regain access.
- AC1.11.1 Given any email (registered or not), when a reset is requested, then the same neutral message is shown.
- AC1.11.2 Given a correct 6-digit code, when verified, then a single-use `reset_token` with the configured lifetime (default 10 minutes) is issued. It is accepted at TTL − 1 s and rejected at TTL + 1 s.
- AC1.11.3 Given a used `reset_token`, when presented again, then it is rejected.
- AC1.11.4 Given a `reset_token` sent as a Bearer token to a protected endpoint, when received, then 401 is returned.
- AC1.11.5 Given a successful new password that meets the Q19 rules, when saved, then every session of that account on every device is revoked.
- AC1.11.6 Given the forgot-password rate limit is exceeded, when another request arrives, then it gets 429.

Implementation phases (tests first):
1. P-API tests first: `reset_token` single use, expiry and token type; OTP lockout for purpose `password_reset`.
2. P-API: request, verify and set-password endpoints.
3. P-UI: three-screen flow.
4. P-E2E: reset journey via Mailpit.

---

## E2 — Consent & Onboarding

### US2.1 — Consent to processing of my photos and health answers
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR7, NFR5.1 (Q1, Q18) · **Depends on:** US1.8

As a new user, I want to see and explicitly agree to how my face images and health answers are used, including by the AI vendor, so that I stay in control of sensitive data.
- AC2.1.1 Given a verified user without consent, when they try to reach the questionnaire or photo upload, then the consent step is shown first and cannot be skipped.
- AC2.1.2 Given the user accepts both consents, when saved, then exactly two records exist: (1) face images and biometric signature, including AI-vendor sharing; (2) health answers, including AI-vendor sharing. Each record has a timestamp and the consent-text version.
- AC2.1.3 Given the user declines, when they return from any entry point, then they land on the consent step and can review and accept it. No later step is reachable.
- AC2.1.4 Given a reusable `require_consent(type)` dependency, when applied to a route without the matching consent, then that route returns 403.

Implementation phases:
1. P-Data: consent records table.
2. P-API: consent endpoints and the `require_consent` dependency.
3. P-UI: consent screen before the questionnaire (wording configurable, pending OQ1).
4. P-E2E: consent gating.

### US2.2 — Questionnaire definition and answers
**Persona:** P1 Alex, P2 Jordan · **Priority:** Must · **Traces:** FR8.1, FR8.3, FR7.3 (`FR-003`, `DATA-004`, Q13) · **Depends on:** US2.1

As a new user, I want my answers saved as I go, with follow-up questions that fit my answers, so that my report is personalised and I can come back later.
- AC2.2.1 Given a branching answer, when submitted, then the server accepts only the next question allowed by the definition's branch.
- AC2.2.2 Given submitted answers, when stored, then each is saved with its question ID and the version of the question text.
- AC2.2.3 Given the user changes an earlier branching answer, when saved, then answers on the branch no longer taken are discarded.
- AC2.2.4 Given health-section answers without the health consent, when submitted directly to the API, then 403 is returned.
- AC2.2.5 Given user B's questionnaire, when requested by user A, then 404 is returned.

Implementation phases:
1. P-Data: versioned questionnaire definition and responses tables.
2. P-API: definition endpoint; answer endpoint with branch validation and incremental save. Uses a **format fixture only**; real questions come from the pending Onboarding Questionnaire Specification.

### US2.3 — Answer the questionnaire
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR8.1 (`FR-003`, Q13), NFR6 · **Depends on:** US2.2

As a new user, I want to answer the questions one by one and resume where I left off, so that the questionnaire is easy to finish.
- AC2.3.1 Given the user leaves mid-questionnaire, when they return, then they resume at the first unanswered question with earlier answers kept.
- AC2.3.2 Given the questionnaire is in progress, when shown, then a text progress indicator shows the position.
- AC2.3.3 Given keyboard-only use, when answering every question type, then all controls are operable.

Implementation phases:
1. P-UI: questionnaire runner rendering from the definition.
2. P-E2E: branch and resume paths with the format fixture (updated once the specification arrives).

### US2.4 — Confirm the mandatory disclaimer
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR8.2 (`FR-004`, `BR-003`) · **Depends on:** US2.3

As a user finishing the questionnaire, I want to confirm the disclaimer that recommendations are informational, not medical, so that I understand what the report is and isn't.
- AC2.4.1 Given the disclaimer is unchecked, when Submit is activated, then submission is blocked and an inline message linked to the checkbox explains why.
- AC2.4.2 Given a direct API submission without the disclaimer flag, when received, then it is refused.
- AC2.4.3 Given the disclaimer is checked, when submitted, then the questionnaire is complete and the user goes to the photo requirements screen.

Implementation phases:
1. P-API: server-side enforcement (wording pending the specification).
2. P-UI: checkbox-gated submit.
3. P-E2E: bypass test.

The experience for a user who cannot confirm the disclaimer is a client question (Open Questions, OQ-S2).

---

## E3 — Photos & Validation

### US3.1 — See the photo requirements before I start
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR9.1 (`FR-005`) · **Depends on:** US2.4

As a user about to take photos, I want a checklist of how to take them, so that my photos pass first time.
- AC3.1.1 Given the Photo Requirements screen, when shown, then all 7 items appear:
  - remove glasses/hat;
  - natural, even lighting;
  - plain white background;
  - tie back long hair;
  - remove makeup;
  - avoid neck-covering clothing;
  - no filters.
- AC3.1.2 Given a user who has not completed the questionnaire, when opening this screen's URL, then they are routed per FR26.

Implementation phases:
1. P-UI: requirements screen.

### US3.2 — Upload photos securely
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR9.2, FR9.4, FR10.3, FR7.3 (`FR-005`, `DATA-005`), NFR5.3, NFR8 · **Depends on:** US3.1, US0.7, US0.8

As a user, I want each photo stored safely and privately, so that my face images can't leak.
- AC3.2.1 Given a valid image for a configured angle, when uploaded, then it is decoded, re-encoded with EXIF/GPS removed, and stored encrypted as that angle's single photo.
- AC3.2.2 Given a non-image, polyglot, oversize or decompression-bomb file, when uploaded, then it is rejected as unreadable.
- AC3.2.3 Given an angle not in the configured set, when uploaded, then 422 is returned.
- AC3.2.4 Given an existing photo for an angle, when a retake is uploaded, then exactly one photo exists for that angle.
- AC3.2.5 Given the stored file, when its raw bytes are read from the backing store, then they neither equal nor contain the plaintext image.
- AC3.2.6 Given upload without the images consent, when called, then 403 is returned. When the per-user upload limit is exceeded, 429 is returned.

Implementation phases:
1. P-Data: photos table (one per angle per user, validation result, persisted landmarks) (DATA-005).
2. P-API: upload endpoint with hardening; real storage adapter (application-level encryption, key from settings); accepted formats JPEG, PNG and WebP (HEIC listed as an open question).

### US3.3 — Upload or capture each photo angle
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR9.2, FR9.3 (`ASM-005`), NFR6 · **Depends on:** US3.2

As a user, I want to upload or capture my front, left 3/4 and right 3/4 photos, so that the analysis has the views it needs.
- AC3.3.1 Given each configured angle (read from `/config/public`), when the user picks a file or captures with the camera, then the photo is uploaded for that angle.
- AC3.3.2 Given camera permission is denied or no camera exists, when capture is chosen, then an inline message offers file upload for the same angle.
- AC3.3.3 Given an upload in progress, when the angle tile renders, then a busy state is announced to screen readers and the same angle cannot be submitted twice.
- AC3.3.4 Given an upload fails with a network error or 5xx, when handled, then an error toast shows, the user stays signed in, and they can retry that angle.

Implementation phases:
1. P-UI: per-angle upload/capture component (layout pending the Photo Capture Specification).
2. P-E2E: upload journey with licensed fixture images.

### US3.4 — Private delivery of my photos and images
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** NFR5.3, NFR4 (Q7) · **Depends on:** US3.2

As a user, I want my photos and generated images visible only to me, so that no one else can see my face.
- AC3.4.1 Given an image, when requested, then it is served only through an HMAC-signed, owner-bound link with the configured TTL. It is accepted at TTL − 1 s and refused at TTL + 1 s.
- AC3.4.2 Given another user's image ID, when requested with my session, then 404 is returned.
- AC3.4.3 Given the storage location, when accessed without a signed link, then nothing is publicly readable.
- AC3.4.4 Given access logs, when inspected, then signed-link tokens are scrubbed.

Implementation phases:
1. P-API: signed-link issuance and verification, and a file-serving endpoint.
2. P-API tests: expiry and cross-user tests.

### US3.5 — Clear feedback when a photo isn't usable (basic checks)
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR10.1, FR10.2 (`FR-006`, `BR-005`, `ASM-002`, `BR-004`) · **Depends on:** US3.3, US0.11

As a user, I want to know right away why a photo failed, in words about the photo and not about me, so that I can retake it without feeling judged.
- AC3.5.1 Given a photo with no face or more than one face, when validated, then it is rejected with the stable reason code for face count.
- AC3.5.2 Given fixtures failing resolution, brightness or face proportion under the configured thresholds, when validated, then each is rejected with its own stable reason code.
- AC3.5.3 Given any rejection, when shown, then the message describes the condition of the photo (lighting, framing, angle, obstruction), points to the matching checklist item, and never describes the person's face or appearance.
- AC3.5.4 Given validation is running, when pending, then a "Checking photo" state is shown and announced.

Implementation phases:
1. P-API: MediaPipe/OpenCV adapter, built once and reused by US3.7 and US5.4 (pinned version and model hash, run off the event loop). Resolution, brightness, face-count and face-proportion checks with configurable thresholds (values pending the Photo Validation Specification). Landmarks persisted per photo.
2. P-API tests: adapter tests with licensed images; rule tests on landmark fixtures.
3. P-UI: per-photo error display.

### US3.6 — Occlusion and pose checks
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR10.1, FR10.2 (`FR-006`, `BR-005`, `ASM-002`) · **Depends on:** US3.5

As a user, I want the app to catch covered eyes or mouth and the wrong head angle, so that my analysis isn't based on an unusable photo.
- AC3.6.1 Given a fixture with eyes or mouth occluded beyond the configured threshold, when validated, then it is rejected with the occlusion reason code.
- AC3.6.2 Given a right-3/4 photo in the front slot, when validated, then it is rejected for pose mismatch.

Implementation phases:
1. P-API: occlusion and pose checks (approach from the US0.11 findings).
2. P-API tests: landmark fixtures.

### US3.7 — Make sure all three photos are of me
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR11.1–FR11.4 (`BR-005` items 1–6, `ASM-010`, Q7) · **Depends on:** US3.5, US3.4

As a user, I want the app to check that all three photos show the same person and tell me exactly which to retake, so that my report is based on me.
- AC3.7.1 Given only two angles have passed, when the status is read, then no identity result exists and uploads are not blocked.
- AC3.7.2 Given all three angles have passed, when the check runs, then the flagged list matches this table (F = front, L = left 3/4, R = right 3/4; T = the pair is within the threshold; front is always the reference and never flagged, Q7):

| Row | F–L | F–R | L–R | Flagged |
|---|---|---|---|---|
| 1 | T | T | T | `[]` |
| 2 | T | T | F | `[]` (Q7; listed for client confirmation) |
| 3 | T | F | F | `[R]` |
| 4 | F | T | F | `[L]` |
| 5 | F | F | T | `[L, R]` (Q7; listed for client confirmation) |
| 6 | F | F | F | `[L, R]` |
| 7 | T | F | T | `[R]` |
| 8 | F | T | T | `[L]` |

- AC3.7.3 Given a pair whose distance equals the configured threshold (default 0.6), when compared, then it counts as a mismatch (the comparison is strictly-less-than to match).
- AC3.7.4 Given any flagged photo, when the review screen shows, then flagged photos are marked by text and icon (not colour alone). Continue is unavailable, and its reason is linked to the button.
- AC3.7.5 Given the check is running, when the review screen shows, then a "Checking your photos match" state is shown and Continue stays unavailable.

Implementation phases (tests first):
1. P-API tests first: the 8-row table and threshold boundary with landmark-signature fixtures.
2. P-API: identity signature (5 distance ratios, configurable threshold) and vote service, recomputed from current photos.
3. P-UI: review screen.

### US3.8 — Retake a flagged photo and continue
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR9.4, FR11.3, FR11.4 (`BR-005` items 5–6) · **Depends on:** US3.7

As a user with a flagged photo, I want to retake just that photo and continue once the set matches, so that I don't redo everything.
- AC3.8.1 Given a flagged photo is retaken, when the new result arrives, then it fully replaces the old one and errors show only on currently mismatched photos.
- AC3.8.2 Given the set becomes consistent after a retake, when the result arrives, then the user stays on the review screen with Continue enabled. There is no auto-navigation.
- AC3.8.3 Given the review screen opens by a fresh page load and the stored set is already complete and consistent, when the page loads, then the user is sent past review to the next step. This decision uses only that initial load, never a retake result.
- AC3.8.4 Given a retake, when started, then no confirmation dialog appears (Q12).

Implementation phases:
1. P-UI: retake flow and review-screen state rules.
2. P-E2E: retake-to-consistent journey.

### US3.9 — Land on the onboarding step I need to finish
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR26 (`BR-005` item 7, `WF-001`) · **Depends on:** US3.7

As a returning user, I want to be taken to my next unfinished onboarding step wherever I enter, so that I never see a screen I'm not ready for.
- AC3.9.1 Given a user at any onboarding stage, when they sign in or open any route, then they land on the first incomplete step in the order: verification → consent → questionnaire → photos.
- AC3.9.2 Given an identity-inconsistent stored set, when the user enters through login, a direct dashboard URL, or the end of the questionnaire, then they land on the photo screen with the error shown.

Implementation phases:
1. P-API: journey-status endpoint (onboarding states).
2. P-UI: routing guard.
3. P-E2E: entry-point matrix.

---

## E4 — Payment

### US4.1 — Pay for my report
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR12.1, FR12.3, FR11.5 (`FR-015`, `FR-016`, `ASM-003`, `OQ-002`, `DATA-008`, `BR-005` item 8) · **Depends on:** US3.9, US0.7

As a user with a valid photo set, I want to pay once through a secure checkout, so that my analysis can begin.
- AC4.1.1 Given a consistent set, when the user continues, then the payment screen shows the configured price and currency, and no report content.
- AC4.1.2 Given the user clicks Pay, when the server creates the session, then it returns a Stripe Checkout URL and the client redirects.
- AC4.1.3 Given an inconsistent set, when checkout is requested, then 409 is returned with the mismatched angles, and no session is created.
- AC4.1.4 Given a user with a confirmed payment, when checkout is requested or the payment screen opened, then no new session is created and they are sent to Start Analysis.
- AC4.1.5 Given the user cancels at Stripe, when they return via the cancel URL, then they are back on the payment screen with a neutral message and can pay again. An open session is reused or expired, never duplicated.

Implementation phases (tests first for the 409 re-check):
1. P-API tests first: 409 re-check and the double-payment guard.
2. P-Data: payments table (DATA-008).
3. P-API: checkout-session service; real Stripe adapter.
4. P-UI: payment screen and redirect.

### US4.2 — My payment is confirmed securely
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR12.2, FR12.4 (`FR-015`, `NFR-009`) · **Depends on:** US4.1

As a user who has paid, I want my payment recorded only from Stripe's confirmation, so that I'm charged once and get access exactly once.
- AC4.2.1 Given a webhook with an invalid signature (checked against the raw body), when received, then it is rejected and nothing changes.
- AC4.2.2 Given the same event delivered twice, when processed, then the payment record changes once.
- AC4.2.3 Given a `completed` event already processed, when an older or `expired` event for the same session arrives, then the status is not downgraded.
- AC4.2.4 Given a successful payment webhook, when processed, then no analysis job exists.
- AC4.2.5 Given the user returns via the success redirect before the webhook, when the page loads, then it shows "confirming payment" by polling `GET /payments/status`. It becomes "paid" with a success toast on the next poll after the webhook, and never marks the payment paid on its own.

Implementation phases (tests first (extension)):
1. P-API tests first: signed fixture payloads covering invalid, duplicate and out-of-order events.
2. P-API: webhook endpoint with an idempotent event log; status endpoint.
3. P-UI: return page.
4. P-E2E: stubbed session plus a signed fixture webhook.

### US4.3 — Payment gate for all paid content
**Persona:** P1 Alex (buyer) · **Priority:** Must · **Traces:** FR12.1 (`FR-015`, `BR-001`), NFR4 · **Depends on:** US4.2

As a buyer, I want paid content to unlock only after my payment is confirmed, so that I never see a partial or broken report.
- AC4.3.1 Given the reusable `require_paid` dependency, when an unpaid user calls a route tagged `paid`, then 402 is returned.
- AC4.3.2 Given the route registry test, when it enumerates every route tagged `paid`, then each returns 402 for an unpaid user. The test fails if an analysis, report, PDF, visuals, home or chat route is missing the tag.

Implementation phases (tests first):
1. P-API tests first: registry test.
2. P-API: dependency and route tagging.

---

## E5 — Analysis Pipeline

### US5.1 — Start my analysis when I'm ready
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR13.1, FR13.2, FR13.5, FR13.6, FR11.5 (`BR-005` item 8, Q11, Q12) · **Depends on:** US4.3, US0.10

As a paid user, I want to start the analysis with a clear, confirmed action, so that I know it has begun.
- AC5.1.1 Given a paid user with a consistent set, when they confirm Start Analysis in the dialog, then a job is created and the progress screen opens.
- AC5.1.2 Given an unpaid user, when Start Analysis is called, then 402 is returned. Given an inconsistent set, 409 is returned and no job starts.
- AC5.1.3 Given a job is already running, when Start Analysis is requested again, then no second job is created.
- AC5.1.4 Given the per-user start limit is exceeded, when requested, then 429 is returned.
- AC5.1.5 Given analysis has started, when the user calls photo or questionnaire write endpoints, then they are refused, and those screens route to progress or Home (inputs locked, Q11).

Implementation phases (tests first):
1. P-API tests first: 402/409 re-checks and the single-job guard.
2. P-API: start endpoint; pipeline definition with an ordered step registry (later stories register their steps); input lock.
3. P-UI: Start Analysis screen with the confirm dialog.

### US5.2 — Watch my analysis progress
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR13.3, NFR3.1 · **Depends on:** US5.1

As a user waiting for results, I want to see which step is running, so that I know it's working.
- AC5.2.1 Given a running job, when the progress screen polls, then it shows the current registered step label. A step change is announced once through a polite live region.
- AC5.2.2 Given the job completes, when the next poll returns, then a "Your report is ready" message with a button to Home is shown. There is no silent auto-navigation.
- AC5.2.3 Given the user closes the tab during analysis, when they return or sign in, then they land on the progress screen with the current step.
- AC5.2.4 Given user B's job, when polled by user A, then 404 is returned.
- AC5.2.5 (performance-validation activity, not per-PR) Given at least N real-vendor analyses in a performance run (N and environment fixed before Construction, OQ7), when start→published durations are aggregated, then p95 ≤ 10 minutes.

Implementation phases:
1. P-API: job-status endpoint.
2. P-UI: progress screen (no "we'll email you" copy).

### US5.3 — Recover when a step fails, without paying again
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR13.4, NFR7 (Q2) · **Depends on:** US5.2

As a user, I want a failed analysis to resume from where it stopped, so that I'm not charged again or made to redo everything.
- AC5.3.1 Given a step fails on the configured Nth attempt, when the progress screen polls, then it names the failed step and offers "Try again".
- AC5.3.2 Given "Try again" is clicked, when the job resumes, then completed steps are not re-run.
- AC5.3.3 Given any number of retries, when payments are counted, then there is exactly one.
- AC5.3.4 Given analysis failed after retries, when the user signs in, then they land on the progress screen with the error and "Try again".

Implementation phases:
1. P-API: resume endpoint.
2. P-UI: error state.
3. P-E2E: a fake vendor failing, then succeeding.

### US5.4 — Measure eyes, eyebrows, nose and lips
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR14.1 (`FR-007`, `NFR-007`, `DATA-006`) · **Depends on:** US5.1, US3.5

As a user, I want these features measured from my front photo, so that the analysis is objective.
- AC5.4.1 Given front-photo landmark fixtures, when measurement runs, then each of the four features matches its golden values within the configured tolerance.
- AC5.4.2 Given measurement completes, when stored, then per-feature measurements are saved to the user's analysis result, reusing the landmarks persisted at validation.

Implementation phases:
1. P-Data: analysis result table (DATA-006).
2. P-API: measurement functions taking landmark coordinates; step registration.
3. P-API tests: golden fixtures.

### US5.5 — Measure jaw, chin, cheeks and ears
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR14.1, FR14.2 (`FR-007`, `ASM-010`) · **Depends on:** US5.4

As a user, I want the remaining mesh-based features and my ears measured correctly, so that every applicable feature has real numbers.
- AC5.5.1 Given landmark fixtures, when measurement runs, then jaw, chin and cheeks match their golden values within tolerance.
- AC5.5.2 Given the right 3/4 photo, when ear measurement runs, then the right ear is measured from it.
- AC5.5.3 Given the left 3/4 photo is missing, when ear measurement runs, then the left ear uses the front photo.

Implementation phases:
1. P-API: measurement functions (the ear formula is first-pass, `ASM-010`).
2. P-API tests: fixtures.

### US5.6 — A personalised, respectful narrative per feature
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR15.1–FR15.4, FR15.6, FR18.3 (`FR-008`, `FR-011`, `NFR-008`), NFR2, NFR5.4 · **Depends on:** US5.5, US0.7

As a user, I want each feature explained in a few sentences that use my measurements and goals, in a tone that fits how I feel, so that the report is useful and not hurtful.
- AC5.6.1 Given questionnaire answers, when the prompt is built, then each carries its real question text, and medical, medication and allergy answers are excluded from cosmetic context.
- AC5.6.2 Given answers that meet the configurable elevated-distress rule (fixture until the questionnaire specification arrives), when the prompt is built, then the reassuring-tone instruction is included.
- AC5.6.3 Given the structured output, when validated, then all 11 features and the closing recommendations are present, each narrative has 3–5 sentences, and any feature with a measurement cites its value. Otherwise the step fails and retries.
- AC5.6.4 Given a changed text-vendor base URL and model in configuration, when the step runs against a recorded fake, then the request goes to the configured URL and model.

Implementation phases:
1. P-API tests: prompt-builder unit tests for `FR-008` (a), (b), (e), (f).
2. P-API: narrative step; real OpenAI text adapter (multimodal); schema validation including closing recommendations.
3. P-API tests: recorded-response integration tests; opt-in live evaluation suite (not a per-PR gate).

### US5.7 — Recommendations sorted into clear tiers
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR15.5 (`FR-012`, `ASM-007`, `OQ-012`) · **Depends on:** US5.6

As a user, I want recommendations grouped into at-home, skincare/OTC and optional in-clinic, so that I can pick what suits me.
- AC5.7.1 Given "consider seeing a dermatologist about laser treatment", when tiered, then it is in-clinic. "Apply a retinol serum" is OTC/skincare-active. "Sleep 8 hours" is at-home.
- AC5.7.2 Given a recommendation matching no keyword, when tiered, then it goes to at-home (the default tier).
- AC5.7.3 Given a recommendation matching keywords from two tiers, when tiered, then the configured precedence (in-clinic > OTC > at-home) decides.
- AC5.7.4 Given every recommendation, when rendered, then it includes "consult a qualified professional" language.

Implementation phases (tests first):
1. P-API tests first: tiering rule table.
2. P-API: replaceable keyword rule set (configurable, labelled as an unconfirmed assumption).

### US5.8 — Symmetry and facial-thirds assessments
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR16.1 (`FR-018`) · **Depends on:** US5.4

As a user, I want symmetry and facial-proportion assessments, so that I see my face as a whole.
- AC5.8.1 Given landmark fixtures, when symmetry is computed, then the score is between 0 and 100 with a Regional Balance breakdown. The descriptive label comes from a configurable label set (values pending the Report & UI Specification).
- AC5.8.2 Given landmark fixtures, when facial thirds are computed, then three proportions summing to 1 (± tolerance) are returned.

Implementation phases:
1. P-API: assessment functions (first-pass heuristics).
2. P-API tests: fixtures.

### US5.9 — Dimorphism, prototypicality and face-shape data
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR16.1, FR16.2 (`FR-018`) · **Depends on:** US5.8

As a user, I want the remaining assessments computed, so that the interactive report is complete.
- AC5.9.1 Given landmark fixtures, when dimorphism is computed, then slider values within the defined range and an overall summary value are returned.
- AC5.9.2 Given landmark fixtures, when face-shape data is computed, then wireframe coordinates are returned. Displayed content follows the Report & UI Specification (FR16.2, pending list).
- AC5.9.3 Given the prototypicality score, when computed, then it uses the reference model chosen once OQ-S4 is answered. Until then the score is on the pending list and not asserted.

Implementation phases:
1. P-API: functions.
2. P-API tests: fixtures.

### US5.10 — A realistic "after" image for each feature
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR17 (`FR-022`, `FR-010`, `NFR-013`, `ASM-011`, `DATA-009`), NFR3.1 · **Depends on:** US5.6, US0.7

As a user, I want an AI-generated "potential" image for each of the 11 features that still looks like me, so that I can picture the change.
- AC5.10.1 Given a completed narrative, when the step runs, then exactly 11 feature images are stored and linked to their features.
- AC5.10.2 Given the step is re-run after a partial failure, when complete, then only missing images are generated and no existing image is regenerated. No user-triggered regeneration route or control exists.
- AC5.10.3 Given the image vendor is switched in configuration, when the adapter contract suite runs, then it passes with no code change outside the adapter.
- AC5.10.4 Given the vendor returns 429, when handled, then the sub-step backs off within the configured bound and retries.

Visual realism and identity preservation are verified by manual review (`ASM-011`), not by an automated acceptance criterion.

Implementation phases:
1. P-Data: report feature visuals table (DATA-009).
2. P-API: per-image idempotent sub-steps with bounded concurrency; real OpenAI image adapter (edit with high input fidelity).
3. P-API tests: fake-adapter count, linkage and idempotency tests.

### US5.11 — Generate my AI Visuals set
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR22.1, FR22.2, FR17.3 (`FR-020`, `DATA-010`) · **Depends on:** US5.1, US0.7

As a user, I want hairstyle, outfit and healthy-aging images generated with my analysis, so that they're ready when I open AI Visuals.
- AC5.11.1 Given the visuals step, when complete, then 5 hairstyle, 5 outfit and 3 aging images (+3, +5, +10 years) are stored, 13 in total. "Current" is the user's own photo and is not generated.
- AC5.11.2 Given a partial failure and retry, when complete, then there are no more than 13 visuals.
- AC5.11.3 Given a finished report, when all generated images are counted, then there are exactly 24.

Implementation phases:
1. P-Data: AI visual assets table (DATA-010).
2. P-API: visuals step (runs in parallel with US5.6–US5.10).
3. P-API tests: count tests.

---

## E6 — Report

### US6.1 — My report is ready as soon as it's done
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR18.1, FR18.4 (`FR-009`, `FR-014`, `BR-002`, `BR-008`, `BR-011`, `DATA-007`) · **Depends on:** US5.7, US5.9, US5.10, US5.11

As a user, I want my report published automatically when analysis finishes, so that I can read it straight away.
- AC6.1.1 Given a completed analysis, when assembled, then the report has exactly the 11 feature sections, with Smile content under Lips and no Smile section.
- AC6.1.2 Given assembly completes, when the user opens the app, then the report is available with no review step.
- AC6.1.3 Given assembly fails, when checked, then no partial report is published.
- AC6.1.4 Given the user's data, when checked, then exactly one report exists, linked 1:1 to the analysis result.

Implementation phases:
1. P-Data: reports table (DATA-007).
2. P-API: assembly step (static template copy plus the narrative and closing recommendations from US5.6; no extra AI call).

### US6.2 — Read each feature with before/after and ideas
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR18.2, FR18.3 (`FR-010`, `FR-011`) · **Depends on:** US6.1, US3.4

As a user, I want each feature section to show my photo, the "after" image, the narrative and a short summary, so that I understand my options.
- AC6.2.1 Given a feature section, when opened, then it shows the narrative, a before/after pair, projected-potential ideas and a summary callout. The "before" is the front photo [Assumption, pending the Report & UI Specification].
- AC6.2.2 Given the report, when rendered, then the introduction, "Understanding Your Results" and limitations sections come from static template copy.
- AC6.2.3 Given narrative text containing `<script>` or `<img onerror>`, when rendered, then it appears literally and no element is created.
- AC6.2.4 Given an unpaid user, when the report is requested, then 402 is returned. Given another user's report, 404 is returned.

Implementation phases:
1. P-API: report read endpoint (owner-scoped, `paid` tag).
2. P-UI: feature section components.

### US6.3 — Explore the interactive report
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR20, FR16 (`FR-018`) · **Depends on:** US6.2, US5.9

As a user, I want a table of contents, my facial assessments and per-feature metric tables, so that I can dig into details.
- AC6.3.1 Given the table of contents, when a feature is selected, then the view moves to that feature's metrics.
- AC6.3.2 Given the report, when rendered, then metric tables exist for all 11 features. Features without mesh measurements (Hair, Skin, Neck) show a not-available state, not an error.
- AC6.3.3 Given the assessment visualisations (sliders, symmetry, wireframe), when used with a screen reader, then each has an equivalent data table or list. Display-only sliders are not exposed as editable controls.

Implementation phases:
1. P-UI: interactive report screen (layout pending the Report & UI Specification).
2. P-E2E: navigation tests.

### US6.4 — Download my report as a PDF
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR19 (`FR-013`) · **Depends on:** US6.2

As a user, I want a PDF of my report, so that I can keep or share it.
- AC6.4.1 Given no PDF exists, when download is clicked, then the PDF is generated, cached and downloaded. The control shows a busy state and ignores repeat clicks.
- AC6.4.2 Given a cached PDF, when downloaded again, then the cached file is served without regeneration.
- AC6.4.3 Given generation fails, when handled, then an error toast shows and the user can retry.
- AC6.4.4 Given an unpaid user, when requested, then 402 is returned. Given another user's report, 404 is returned.

Implementation phases:
1. P-API: rendering pipeline, cache, private image retrieval (PDF engine chosen by ADR in Domain Design).
2. P-UI: download action.

### US6.5 — Branded PDF layout
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR19.1 (`FR-013`, `CON-003`) · **Depends on:** US6.4

As a user, I want the PDF to look professional and complete, so that it reads like the in-app report.
- AC6.5.1 Given a report, when rendered to PDF, then it contains the introduction, all 11 feature sections with before/after images, the assessments, the limitations and the closing recommendations, with in-house branding.

Implementation phases:
1. P-API: branded template (content per the Report & UI Specification).

---

## E7 — Post-Analysis Experience

### US7.1 — See my Home Overview and move around
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR21 (`FR-017`) · **Depends on:** US6.1, US3.4

As a returning user, I want a home screen summarising my results and a header to reach Report, AI Visuals and Chat, so that I see what matters first and never get lost.
- AC7.1.1 Given a published report, when Home loads, then it shows the before/potential image, Priority Features to Improve, report status and PDF download. Key stats and the protocol summary follow the Report & UI Specification (pending list).
- AC7.1.2 Given Home, when the radar chart renders, then it has exactly six axes, including Symmetry, and an equivalent data table.
- AC7.1.3 Given any post-analysis screen, when the header renders, then it links Home, Report, AI Visuals and Chat, and marks the current item with `aria-current="page"`.
- AC7.1.4 Given an unpaid user, when Home is requested, then 402 is returned.

Implementation phases:
1. P-API: home summary endpoint (`paid` tag).
2. P-UI: Home screen, radar chart and header (layout pending the specification).

### US7.2 — Browse my AI Visuals
**Persona:** P1 Alex · **Priority:** Must · **Traces:** FR22.1 (`FR-020`) · **Depends on:** US5.11, US3.4

As a user, I want to browse hairstyle, outfit and healthy-aging images of myself, so that I can explore looks.
- AC7.2.1 Given AI Visuals, when opened, then 5 hairstyles, 5 outfits and the 4-card aging stack are shown. The stack cards are labelled "Current", "+3 years", "+5 years" and "+10 years", with no hard-coded absolute age.
- AC7.2.2 Given a signed image link expires while the page is open, when the image is next shown, then a fresh link is fetched with no broken image. An image that fails to load shows a described placeholder.
- AC7.2.3 Given every image, when rendered, then it has a descriptive text alternative, and every stack card is reachable without swipe-only gestures.
- AC7.2.4 Given an unpaid user, when visuals are requested, then 402 is returned. Given another user's visuals, 404 is returned.

Implementation phases:
1. P-API: visuals read endpoint.
2. P-UI: AI Visuals screen (layout pending the specification).

### US7.3 — Ask the AI Beauty Assistant about my report
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR23.1, FR23.4 (`FR-019`, `DATA-011`) · **Depends on:** US6.1, US0.7

As a user, I want to ask questions about my own results in a chat, so that I understand them better.
- AC7.3.1 Given user A's report, when the chat context is built, then it contains A's measurements, narrative and questionnaire answers (excluding medical, medication and allergy answers where FR15.4 applies), and no other user's data.
- AC7.3.2 Given a conversation, when the user returns later, then previous messages are shown.
- AC7.3.3 Given a reply containing HTML, when rendered, then it appears as text. New replies are announced through a polite log region.
- AC7.3.4 Given an empty message, or one over the configured maximum length, when sent, then 422 is returned and no AI call is made.
- AC7.3.5 Given the AI call fails, when handled, then the user's message is kept, an error toast shows and they can resend.
- AC7.3.6 Given an unpaid user, when chat is called, then 402 is returned.

Implementation phases:
1. P-Data: chat conversations and messages (DATA-011).
2. P-API: chat service with a context builder (non-streaming replies [Recommendation]).
3. P-UI: chat screen with empty and pending states.

### US7.4 — Medical questions are declined safely
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** FR23.2 (`FR-019`) · **Depends on:** US7.3

As a user, I want the assistant to decline medical or medication questions kindly and point me to a professional, so that I don't act on unsafe advice.
- AC7.4.1 Given the chat system prompt, when the prompt builder is tested, then the decline-and-redirect rule is always present.
- AC7.4.2 Given a recorded reply to a medical question, when the integration test replays it, then the reply shown declines, recommends a qualified professional, and invites a question about the report.
- AC7.4.3 (opt-in evaluation suite, not per-PR) Given "can I take this with my medication?" sent to the live model, when answered, then it declines and redirects.

Implementation phases:
1. P-API tests: prompt-builder and recorded-response tests.
2. P-API: decline guard.

### US7.5 — Know when I've reached today's chat limit
**Persona:** P3 Sam · **Priority:** Should · **Traces:** FR23.3, NFR8 (Q8, Q17) · **Depends on:** US7.3, US0.8

As a user, I want to be told clearly when I've reached the daily chat limit and when I can continue, so that I'm not confused.
- AC7.5.1 Given the configured cap N, when message N is sent, then it is allowed. Message N+1 is refused with the reset time (midnight UTC), and no AI call is made.
- AC7.5.2 Given a failed AI reply, when counted, then it does not count toward the cap.
- AC7.5.3 Given the cap is reached, when the chat renders, then the input is disabled, with the reason and reset time shown as text.

Implementation phases:
1. P-API: counter.
2. P-UI: limit state.

---

## E8 — Settings

### US8.1 — Settings and my account information
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR24.1 (`FR-021`, `ASM-009`, Q14) · **Depends on:** US1.8

As a user, I want a Settings area where I can see my name, email, verification status and member-since date at any time, so that I know my account details.
- AC8.1.1 Given Settings, when opened at any journey stage, then it shows the Account Info, Password and Billing sections.
- AC8.1.2 Given Account Info, when opened, then full name, email, verification status and member-since date are shown, and the full name cannot be edited in the UI.
- AC8.1.3 Given an API request that changes the full name, when received, then it is rejected.

Implementation phases:
1. P-UI: Settings shell and Account Info using `/auth/me`.

### US8.2 — Change my password while signed in
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR6, FR24.2 (`AUTH-016`, Q15, Q19) · **Depends on:** US8.1, US1.7

As a signed-in user, I want to change my password with my current one, so that I can keep my account secure without being logged out.
- AC8.2.1 Given a wrong current password, when submitted, then the change is refused with an inline error.
- AC8.2.2 Given new and confirm-new fields that differ, or a new password breaking the Q19 rules, when submitted, then it is blocked before any request.
- AC8.2.3 Given a successful change, when complete, then the current session stays signed in and every other session of the account is revoked.

Implementation phases:
1. P-API: change-password endpoint (revoke other families).
2. P-UI: Password section.
3. P-E2E: change-password journey.

### US8.3 — See my billing history
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR24.3, FR12.5 (`FR-021`, `BR-012`, Q16) · **Depends on:** US8.1, US4.2

As a user, I want to see my payment and the payment options, so that I know what I've paid.
- AC8.3.1 Given a payment, when Billing opens, then its date, amount, currency, status, the Stripe method used (card brand and last 4) and a link to the Stripe receipt are shown. No action starts a second purchase.
- AC8.3.2 Given no payments, when Billing opens, then it shows "No payments yet".
- AC8.3.3 Given the options, when shown, then PayPal is rendered `aria-disabled` with a "not available" status in text. Activating it does nothing.
- AC8.3.4 Given another user's payments, when requested, then they are never included.

Implementation phases:
1. P-API: billing endpoint (owner-scoped).
2. P-UI: Billing section (layout pending the specification).

---

## E9 — App-wide

### US9.1 — See my name and reach my account from any screen
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR2 (`FR-002`), FR4.6, FR24 (Q14) · **Depends on:** US1.10, US8.1

As a signed-in user, I want to see my name and reach Settings or Logout from any screen, so that I know which account I'm in and can leave at any point.
- AC9.1.1 Given any signed-in screen, including onboarding, photos, payment and progress, when it renders, then a user menu shows the full name from `/auth/me`.
- AC9.1.2 Given the menu, when opened, then it offers Settings and Logout. Logout uses the confirm dialog.
- AC9.1.3 Given keyboard use, when operating the menu, then Enter or Space opens it, Escape closes it, and focus returns to the trigger.

Implementation phases:
1. P-UI: user-menu component in the authenticated layout.
2. P-E2E: logout from an onboarding screen.

### US9.2 — Land on the right post-payment screen
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR26 (`WF-001`) · **Depends on:** US6.1, US3.9

As a returning user, I want to be taken to the right place after payment, and my direct links to work once my report exists, so that I never see a screen I'm not ready for.
- AC9.2.1 Given a paid user who has not started analysis, when they sign in, then they land on Start Analysis.
- AC9.2.2 Given analysis in progress or failed, when they sign in, then they land on the progress screen.
- AC9.2.3 Given a published report, when they sign in, then they land on Home. A direct link to Report, AI Visuals, Chat or Settings opens that route.

Implementation phases:
1. P-API: extend journey status (payment → start → in progress → home).
2. P-UI: guard.
3. P-E2E: matrix.

### US9.3 — Keep my data isolated from other users
**Persona:** P2 Jordan · **Priority:** Must · **Traces:** NFR4 (`AUTH-004`, `AUTH-007`, `AUTH-008`) · **Depends on:** US1.4

As a user, I want every request for my data checked against my identity, so that nobody can read or change my records.
- AC9.3.1 Given a protected endpoint called without a valid access token, when received, then 401 is returned.
- AC9.3.2 Given a token with `alg: none` or a non-allow-listed algorithm, when presented, then it is rejected.
- AC9.3.3 Given a refresh token or `reset_token` used as a Bearer token, when presented, then it is rejected.
- AC9.3.4 Given the owner-scope registry test, when it enumerates owner-scoped routes, then each has a cross-user 404 test. A missing test fails CI.

Implementation phases (tests first (extension)):
1. P-API tests first: token-type and algorithm tests; registry test.
2. P-API: `get_current_user` dependency and ownership-scoped repository helpers.

### US9.4 — App-wide accessibility audit
**Persona:** All · **Priority:** Should · **Traces:** NFR6 (Q5, Q10) · **Depends on:** US7.1, US8.3

As a user who relies on a keyboard, screen reader, zoom or a small screen, I want the whole app to meet WCAG 2.1 AA, so that I can complete the whole journey.
- AC9.4.1 Given every route, when axe runs in CI, then there are no violations.
- AC9.4.2 Given every screen at 320 CSS px width and at 200% text zoom, when viewed, then no content or function is lost and there is no two-dimensional scrolling.
- AC9.4.3 Given text and meaningful non-text UI, when measured, then contrast is ≥ 4.5:1 for text and ≥ 3:1 for non-text.
- AC9.4.4 (manual checklist, not CI) Given a keyboard and screen-reader pass across the full journey, when performed, then no blocker is found.

Implementation phases:
1. P-E2E: axe per route, reflow and contrast checks.
2. P-E2E: manual audit checklist in docs.

### US9.5 — Keep the app fast and measurable
**Persona:** P1 Alex · **Priority:** Should · **Traces:** NFR3.2, NFR3.3, NFR12 (Q4) · **Depends on:** US0.3

As a user, I want pages and actions to respond quickly, so that the app feels reliable.
- AC9.5.1 (blocked on OQ7; not a CI gate) Given the load profile (concurrent users, request mix, duration) fixed before Construction, when non-AI endpoints are load-tested, then p95 ≤ 500 ms.

Implementation phases:
1. P-E2E: load-test script and profile (after OQ7).

---

## Dependencies and Critical Path

**Critical path:**
US0.1 → US0.3 → US0.4 / US0.5 → US0.6 → US1.2 → US1.3 → US1.4 → US1.7 → US1.8 → US2.1 → US2.2 → US2.3 → US2.4 → US3.1 → US3.2 → US3.3 → US3.5 → US3.6 → US3.7 → US3.9 → US4.1 → US4.2 → US4.3 → US5.1 → US5.4 → US5.5 → US5.6 → US5.10 → US6.1 → US6.2 → US7.1

**Must land before it is needed:**

| Item | Needed before |
|------|---------------|
| US0.7 adapters | US1.2 |
| US0.8 rate limits | US1.2 |
| US0.10 job runner | US5.1 |
| US0.11 CV spike | US3.5 |
| US3.4 signed links | US3.7 |

**Parallel once dependencies land:**
- US0.2, US0.9;
- US1.5, US1.6, US1.9, US1.10, US1.11, US9.3;
- US3.8;
- US5.2, US5.3, US5.7, US5.8, US5.9, US5.11 (visuals run in parallel with narrative and images);
- US6.3, US6.4, US6.5, US7.2, US7.3–US7.5;
- E8, US9.1, US9.2, US9.4, US9.5.

## Open Questions for the Client

| ID | Question | Affects |
|----|----------|---------|
| OQ-S1 | Confirm the identity-vote reading for rows 2 and 5 (front never flagged, Q7) | US3.7 |
| OQ-S2 | What a user who cannot truthfully confirm the disclaimer sees (supportive message, professional-help pointer) | US2.4 |
| OQ-S3 | Confirm the Billing "working Stripe option" means method + receipt link (Q16) | US8.3 |
| OQ-S4 | The reference model or dataset for the prototypicality score (none specified or licensed) | US5.9 |
| OQ-S5 | Accepted photo formats: add HEIC for iPhone uploads? | US3.2 |

Also still open from Requirements (OQ1–OQ7): market/privacy law, angle count, tiering method, price, the four pending specifications, email provider, and the load profile.

## INVEST Notes

- **Independent:**
  - Each story delivers one testable behaviour.
  - Gated and owner-scoped behaviour is enforced by registry tests (US4.3, US9.3), with a per-endpoint 402 or 404 criterion, so no story asserts endpoints built later.
  - Later journey stages are tested from seeded scenarios (US0.9).
- **Negotiable:** layout and copy wait for the pending specifications; the criteria name behaviours.
- **Valuable:** every non-Enabler story names a persona benefit. Enablers are marked.
- **Estimable and small:**
  - Stories estimated above 3 days in review were split: repository and CI, backend and frontend foundations, adapters, questionnaire, upload, validation, measurement, assessments and PDF.
  - The heaviest remaining stories (US3.5, US5.6, US5.10) keep a single responsibility.
- **Testable:**
  - Criteria are Given/When/Then with stated boundaries.
  - Criteria that cannot be automated are labelled as manual, performance-validation or opt-in evaluation.
