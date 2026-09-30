# Units of Work — FaceIQ

## Overview

- **Sources:**
  - the component catalogue in `domain-design/components.md` (16 components);
  - the architecture decisions in `domain-design/decisions.md` (ADR-001 to ADR-016);
  - `requirements-analysis/requirements.md` (FR1–FR26, NFR1–NFR12);
  - `user-stories/stories.md` (67 stories);
  - the answers Q1–Q5 and the approved plan in `units-generation-questions.md`.
- **Decomposition:** 13 **vertical feature units** (Q1). Each unit delivers one feature's backend module(s) together with the screens that use them. Three high-risk capabilities have their own units:
  - the CV engine;
  - AI text;
  - AI images.
- **Deployment model (Q2):** a modular monolith with **two deployables**:
  - **Backend:** one FastAPI codebase, run as an API process and a background-worker process. The worker model is Proposed ADR-013. Components are modules whose boundaries are enforced by import-linter contracts (ADR-001).
  - **Frontend:** one Next.js application.

  Units are modules inside these two deployables, not separate services. There are no deployment jobs until a host is chosen (NFR11).
- **Walking skeleton (Q3; affirmed team Walking Skeleton):** U1 is the smallest working integrated slice: the email + password login step running through UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services, on the real PostgreSQL. It runs end to end on its own.
- **Ordering:** this document records scope only. Dependencies are in `unit-of-work-dependency.md`. Deciding what ships first is left to Delivery Planning.

## Unit Index

| Unit ID | Directory | Name | Kind | Complexity | Components | Stories |
|---------|-----------|------|------|------------|------------|---------|
| U1 | u1-walking-skeleton | walking-skeleton | service | L | Platform (base), Identity (login step), WebUI (shell), AuthStore, ApiClient | 5 |
| U2 | u2-platform-services | platform-services | library | M | Platform | 4 |
| U3 | u3-identity | identity | service | XL | Identity, AuthStore, ApiClient, WebUI (auth screens) | 12 |
| U4 | u4-onboarding | onboarding | service | M | Onboarding, WebUI (consent, questionnaire) | 4 |
| U5 | u5-photos | photos | service | XL | Photos, MediaStore, FaceAnalysisEngine (adapter and checks), JourneyStatus (onboarding part), WebUI (photo screens) | 10 |
| U6 | u6-payments | payments | service | M | Payments, WebUI (payment screens) | 3 |
| U7 | u7-analysis-pipeline | analysis-pipeline | service | L | AnalysisOrchestrator, WebUI (start, progress) | 4 |
| U8 | u8-face-analysis-engine | face-analysis-engine | library | L | FaceAnalysisEngine (measurements, assessments) | 4 |
| U9 | u9-insights | insights | library | M | InsightGeneration | 2 |
| U10 | u10-image-generation | image-generation | service | L | ImageGeneration, WebUI (AI Visuals) | 3 |
| U11 | u11-report | report | service | L | Report, WebUI (report, home) | 6 |
| U12 | u12-beauty-assistant | beauty-assistant | service | M | BeautyAssistant, WebUI (chat) | 3 |
| U13 | u13-account-and-app-wide | account-and-app-wide | ui | M | WebUI (settings, user menu), JourneyStatus (post-payment part), Identity/Payments reads | 7 |

Complexity: S = under 3 stories of simple scope; M = 3–5 stories; L = 4–6 stories with a risky integration; XL = 8 or more stories or a safety-critical rule set.

## Unit Definitions

### U1 — walking-skeleton (`u1-walking-skeleton`)

- **Kind:** service
- **Complexity:** L
- **Deployment:** backend API and frontend (both deployables)
- **Description:** The integrated first slice. It proves the two-app split, CORS with credentials, PostgreSQL through SQLAlchemy/Alembic, and the mandated auth layering, before any feature work starts.
- **Boundaries:**
  - The login step stops at "OTP required". No code is issued or sent.
  - No signup, OTP or refresh logic yet.
- **Responsibilities:**
  - Repository scaffold and core CI (US0.1).
  - Backend foundation: settings, error envelope, redacted logging, health, public config (US0.3).
  - One-command local environment with PostgreSQL, Mailpit, API, worker and frontend (US0.4).
  - Frontend foundation: shell, API client, toast and confirm components, OpenAPI type generation, CSP, axe harness (US0.5).
  - The skeleton login step through the full layering, including the `users` table with its `role` field (US0.6).
- **Delivers:** a running app, started by one documented command, whose Playwright test logs in to the "OTP required" state. This is the candidate verification command for the skeleton checkpoint.
- **Implementation notes:**
  - Pin a Python version that has MediaPipe wheels for Windows, Linux x86_64 and macOS, even though MediaPipe is first used in U5.
  - Import-linter and ESLint layering contracts start here.

### U2 — platform-services (`u2-platform-services`)

- **Kind:** library
- **Complexity:** M
- **Deployment:** embedded in the backend
- **Description:** Shared backend services that later units depend on.
- **Boundaries:** No endpoints of its own. Real vendor adapters ship with their first consuming unit (ADR-011).
- **Responsibilities:**
  - Blocking security, coverage and traceability gates (US0.2).
  - Vendor adapter ports, fakes and contract harness, plus network blocking in tests (US0.7).
  - Rate-limit primitive (US0.8).
  - Test-data factories and E2E seeding CLI (US0.9).
- **Delivers:**
  - CI that fails on secrets, High/Critical findings, coverage below the 80%/90% floors, and unmapped requirement IDs.
  - Reusable limiter, adapters and seeding.

### U3 — identity (`u3-identity`)

- **Kind:** service
- **Complexity:** XL
- **Deployment:** backend API and frontend
- **Description:** The complete custom authentication feature (`AUTH-001`–`AUTH-016`, `FE-001`–`FE-007`).
- **Boundaries:** Covers all auth flows and the owner-scoping foundation. Settings screens are in U13.
- **Responsibilities:**
  - Landing page (US1.1).
  - Signup with neutral and resume paths and the real SMTP adapter (US1.2).
  - OTP rules (US1.3).
  - Tokens, sessions and cookie (US1.4).
  - Resend (US1.5).
  - Login (US1.6).
  - Rotation, reuse detection and CSRF (US1.7).
  - Session restore (US1.8).
  - Frontend refresh (US1.9).
  - Logout (US1.10).
  - Forgot password (US1.11).
  - Current-user dependency, token-type and algorithm checks, and the owner-scope registry test (US9.3).
- **Delivers:** Complete signup, login, session and reset flows. Auth services are at 90% coverage.
- **Implementation notes:** Tests are written first for the OTP, token-rotation, reuse and reset-token rules (affirmed Testing Posture).

### U4 — onboarding (`u4-onboarding`)

- **Kind:** service
- **Complexity:** M
- **Deployment:** backend API and frontend
- **Description:** Consent capture and the questionnaire with its mandatory disclaimer (FR7, FR8).
- **Boundaries:** Questionnaire content comes only from the pending Onboarding Questionnaire Specification; until then it uses a format fixture.
- **Responsibilities:**
  - Two consents and the `require_consent` check (US2.1).
  - Versioned definition, branch validation and incremental save (US2.2).
  - Questionnaire runner UI with resume (US2.3).
  - Disclaimer enforcement (US2.4).
  - The input lock that the analysis pipeline sets (ADR-012).
- **Delivers:** A verified user can consent, answer, resume and submit with the disclaimer confirmed.

### U5 — photos (`u5-photos`)

- **Kind:** service
- **Complexity:** XL
- **Deployment:** backend API and frontend
- **Description:** The photo flow end to end:
  - requirements screen;
  - secure upload and private storage;
  - per-photo validation;
  - identity check and retake;
  - onboarding-stage routing.
- **Boundaries:**
  - Includes the MediaStore component (private encrypted files, signed links).
  - Includes the FaceAnalysisEngine adapter and per-photo check functions.
  - Includes the onboarding part of JourneyStatus.
  - Measurement and assessment functions are in U8.
- **Responsibilities:**
  - CV feasibility spike (US0.11).
  - Requirements screen (US3.1).
  - Hardened upload with MediaStore storage (US3.2).
  - Capture UI (US3.3).
  - Signed-link delivery (US3.4).
  - Basic checks with the MediaPipe adapter and persisted landmarks (US3.5).
  - Occlusion and pose checks (US3.6).
  - Identity vote with the 8-row table (US3.7).
  - Retake (US3.8).
  - Onboarding routing (US3.9).
- **Delivers:** A user reaches a validated, identity-consistent photo set with photos stored encrypted and served privately.
- **Implementation notes:**
  - The spike comes before the check stories.
  - Tests are written first for the identity vote.
  - Validation thresholds come from configuration (pending Photo Validation Specification).

### U6 — payments (`u6-payments`)

- **Kind:** service
- **Complexity:** M
- **Deployment:** backend API and frontend
- **Description:** One-time Stripe payment with the server-side 402 gate (FR12, `BR-001`).
- **Boundaries:** PayPal is visible but disabled in the UI only. Billing history screens are in U13.
- **Responsibilities:**
  - Checkout with the 409 identity re-check and double-payment guard, plus the real Stripe adapter (US4.1).
  - Verified, idempotent webhook, status endpoint and return page (US4.2).
  - `require_paid` dependency and gated-route registry (US4.3).
- **Delivers:** A consistent-set user can pay once. Paid state is confirmed only by a verified webhook.

### U7 — analysis-pipeline (`u7-analysis-pipeline`)

- **Kind:** service
- **Complexity:** L
- **Deployment:** backend API, backend worker and frontend
- **Description:** The analysis job framework and its screens.
- **Boundaries:** Owns the job runner, step registry, start/progress/resume and the AnalysisResult entity. Units U8–U11 each register their own steps.
- **Responsibilities:**
  - Background worker with row-lock claims and leases (US0.10).
  - Start with 402/409 re-checks, confirm dialog, single-job guard and input locks (US5.1).
  - Progress screen and duration capture (US5.2).
  - Resume after failure (US5.3).
- **Delivers:** A paid user can start analysis and watch progress. The pipeline runs, retries and resumes with fake steps before the real steps land.

### U8 — face-analysis-engine (`u8-face-analysis-engine`)

- **Kind:** library
- **Complexity:** L
- **Deployment:** embedded in the backend worker
- **Description:** Pure CV computation used by the pipeline (ADR-002).
- **Boundaries:**
  - Reuses the MediaPipe adapter and the landmarks persisted by U5.
  - Owns no data. Results go into AnalysisResult (U7).
  - Prototypicality waits for the reference model (OQ-S4).
- **Responsibilities:**
  - Eyes, eyebrows, nose and lips measurements (US5.4).
  - Jaw, chin, cheeks and ears measurements (US5.5).
  - Symmetry and thirds (US5.8).
  - Dimorphism, prototypicality and face-shape data (US5.9).
  - Registering the measurement and assessment steps.
- **Delivers:** Deterministic measurements and assessments against landmark golden fixtures.

### U9 — insights (`u9-insights`)

- **Kind:** library
- **Complexity:** M
- **Deployment:** embedded in the backend worker
- **Description:** AI narrative, closing recommendations and tiering (FR15, ADR-004).
- **Boundaries:** Stateless. The real OpenAI text adapter ships here.
- **Responsibilities:**
  - Prompt builder, safety and tone rules, schema-validated output, and the real text adapter (US5.6).
  - Keyword tiering rule set (US5.7).
  - Registering the narrative step.
- **Delivers:** Validated narratives for all 11 features, plus tiered recommendations, from recorded responses.
- **Implementation notes:** Tests are written first for tiering. The live-model evaluation suite is opt-in.

### U10 — image-generation (`u10-image-generation`)

- **Kind:** service
- **Complexity:** L
- **Deployment:** backend worker, backend API (visuals read) and frontend
- **Description:** 24 AI images per report and the AI Visuals screen (FR17, FR22).
- **Boundaries:** The real OpenAI image adapter ships here. Visual quality is checked by manual review (`ASM-011`).
- **Responsibilities:**
  - 11 feature "after" images with per-image idempotency (US5.10).
  - 13 visuals (US5.11).
  - AI Visuals screen (US7.2).
- **Delivers:** Exactly 24 images per report, stored privately, with visuals browsable by the owner.

### U11 — report (`u11-report`)

- **Kind:** service
- **Complexity:** L
- **Deployment:** backend worker (assembly), backend API and frontend
- **Description:**
  - Auto-published report assembly;
  - report reads;
  - interactive report;
  - PDF;
  - Home Overview (FR18–FR21, ADR-005).
- **Boundaries:** Stores a content snapshot at assembly. The PDF engine follows Proposed ADR-014.
- **Responsibilities:**
  - Assembly and publish (US6.1).
  - Feature sections (US6.2).
  - Interactive report (US6.3).
  - PDF pipeline and cache (US6.4).
  - Branded PDF (US6.5).
  - Home Overview with header navigation (US7.1).
- **Delivers:** A completed analysis produces exactly one published 11-section report, viewable, downloadable and summarised on Home.

### U12 — beauty-assistant (`u12-beauty-assistant`)

- **Kind:** service
- **Complexity:** M
- **Deployment:** backend API and frontend
- **Description:** Grounded chat about the user's own report (FR23).
- **Boundaries:** Non-streaming replies (Proposed ADR-016). Grounded only in the requesting user's data.
- **Responsibilities:**
  - Chat with grounding context and failure handling (US7.3).
  - Medical-question decline (US7.4).
  - Daily cap (US7.5).
- **Delivers:** A paid user with a report can chat, and medical questions are declined.

### U13 — account-and-app-wide (`u13-account-and-app-wide`)

- **Kind:** ui
- **Complexity:** M
- **Deployment:** frontend, plus small backend endpoints in Identity and Payments
- **Description:** Settings, the global user menu, post-payment routing, and app-wide quality work.
- **Boundaries:** Account and billing reads call the Identity and Payments modules. The load test waits for OQ7.
- **Responsibilities:**
  - Settings shell and Account Info (US8.1).
  - Change password (US8.2).
  - Billing history with method and receipt (US8.3).
  - Name greeting and user menu (US9.1).
  - Post-payment routing and deep links (US9.2).
  - App-wide accessibility audit (US9.4).
  - Performance load test (US9.5).
- **Delivers:** Settings reachable at any time, a user menu on every screen, correct post-payment routing, and the full-app WCAG 2.1 AA audit.

## Kind Summary

| Kind | Units | Design documents implied in Construction |
|------|-------|------------------------------------------|
| service | U1, U3, U4, U5, U6, U7, U10, U11, U12 | Full set (functional, NFR, infrastructure as applicable) |
| library | U2, U8, U9 | Functional and NFR design; no infrastructure or scalability documents |
| ui | U13 | Frontend component and interaction design; no backend scalability documents |
