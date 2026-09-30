# Architecture Decisions — FaceIQ

## Overview

This is the Architecture Decision Record log for the domain design in `components.md`.
- **Inputs:** `requirements-analysis/requirements.md`, `user-stories/stories.md`, the affirmed `team-practices`, and answers Q1–Q10 in `domain-design-questions.md`.
- **ADR structure:** every ADR has Context, Decision, Consequences, Alternatives Rejected, and a Security & compliance note (inception architecture guardrails).
- **Status:** ADR-001 to ADR-012 are **Accepted** (domain decisions confirmed by the human). ADR-013 to ADR-016 are **Proposed** technical starting points (Q10). The NFR and infrastructure design stages are skipped in this plan, so these four are confirmed or replaced when building starts.

## ADR Index

| ADR | Title | Status | Source |
|-----|-------|--------|--------|
| ADR-001 | Backend decomposed into 13 domain-aligned components | Accepted | Q1–Q9 |
| ADR-002 | Face Analysis Engine is pure computation and owns no data | Accepted | Q1 |
| ADR-003 | Onboarding owns consent and questionnaire together | Accepted | Q2 |
| ADR-004 | Separate Insight Generation and Image Generation components | Accepted | Q3 |
| ADR-005 | Report owns report, PDF and Home; stores a content snapshot | Accepted | Q4 |
| ADR-006 | Read-only Journey Status component for entry routing | Accepted | Q5 |
| ADR-007 | Media Store owns all private files and signed links | Accepted | Q6 |
| ADR-008 | Orchestrator calls components directly inside a background worker | Accepted | Q7 |
| ADR-009 | Three frontend components mirroring the mandated auth layering | Accepted | Q8 |
| ADR-010 | Platform component for cross-cutting backend concerns | Accepted | Q9 |
| ADR-011 | Ports and adapters for every external vendor | Accepted | NFR2, `NFR-008`, `NFR-013` |
| ADR-012 | Input lock owned by Onboarding and Photos, set by the orchestrator | Accepted | Stories Q11 |
| ADR-013 | Background worker polling PostgreSQL with row locks and leases | Proposed | Q10 |
| ADR-014 | HTML-to-PDF rendering with headless Chromium | Proposed | Q10 |
| ADR-015 | Application-level encryption and HMAC-signed links for local storage | Proposed | Q10 |
| ADR-016 | Non-streaming chat replies | Proposed | Q10 |

## Accepted Decisions

### ADR-001: Backend decomposed into 13 domain-aligned components

- **Status:** Accepted
- **Context:**
  - FaceIQ has distinct concerns with very different risk and change rates: custom auth (`AUTH-*`), consent and questionnaire (FR7, FR8), biometric photo validation (FR10, FR11), payments (FR12), a long-running CV + AI pipeline (FR13–FR17), the report (FR18–FR21) and chat (FR23).
  - The client's team will extend the code with AI-assisted tools, so boundaries must be obvious and enforced (`BC-005`, `NFR-011`).
  - Deployment shape is decided later, in Units Generation.
- **Decision:** Model the backend as 13 components: Platform, Identity, Onboarding, MediaStore, FaceAnalysisEngine, Photos, Payments, AnalysisOrchestrator, InsightGeneration, ImageGeneration, Report, BeautyAssistant and JourneyStatus.
  - Each component owns its own entities, and the dependency graph is acyclic.
  - Cross-component access goes only through public interfaces, never another component's tables. Linters enforce this (affirmed Code Style: import-linter contracts).
- **Consequences:**
  - Positive:
    - Each safety-critical rule has a single owner: auth, the 402 gate, the identity vote and tiering. This matches the affirmed 90% coverage modules.
    - Components can be tested in isolation.
    - Any deployment shape remains possible.
  - Negative:
    - There are more modules to wire together than a coarse split would have.
    - Some reads cross boundaries through interfaces (for example Journey Status and Report).
- **Alternatives Rejected:**
  - A coarse 5-component split (Auth, Onboarding, Analysis, Report, Chat): mixes payments, photos and the pipeline, so the gate and identity rules lose a clear owner.
  - One component per entity: too fine, with chatty interfaces and no real benefit.
- **Security & compliance:**
  - Isolates biometric data (Photos, MediaStore) and credentials (Identity) behind narrow interfaces.
  - Object-level authorization is applied per component with a cross-user 404 convention.

### ADR-002: Face Analysis Engine is pure computation and owns no data

- **Status:** Accepted
- **Context:** Landmarks feed three uses:
  - per-photo checks and the identity signature, at upload time (FR10, FR11);
  - measurements (FR14);
  - assessments (FR16).

  MediaPipe and OpenCV are heavy native dependencies. The heuristics are first-pass and will be re-tuned (`ASM-002`, `ASM-010`).
- **Decision:** FaceAnalysisEngine exposes pure functions (images or landmarks in, numbers out) and persists nothing.
  - Photos stores landmarks, signatures and identity results.
  - AnalysisOrchestrator stores the analysis result (DATA-006).
- **Consequences:**
  - Positive:
    - Most CV logic is tested with landmark-coordinate fixtures and no face photos, as affirmed in Testing Posture and the test-data rules.
    - Landmarks are computed once at validation and reused by measurement.
    - Tuning never touches storage.
  - Negative: Callers must pass all inputs explicitly.
- **Alternatives Rejected:**
  - The engine owns the analysis result and landmarks: this couples tuning with persistence and forces the engine to know users.
  - Separate engines for validation and for measurement: this duplicates the MediaPipe adapter.
- **Security & compliance:** No biometric data is stored or logged by the engine. Persisted landmarks and signatures stay in owner-scoped Photos records.

### ADR-003: Onboarding owns consent and questionnaire together

- **Status:** Accepted
- **Context:**
  - Consent (FR7, Q18) and the questionnaire (FR8) are both pre-analysis inputs.
  - Both change with the pending Onboarding Questionnaire Specification.
  - Both gate other components: images consent gates upload, and health consent gates health answers.
- **Decision:** One Onboarding component owns ConsentRecord, QuestionnaireDefinition and QuestionnaireResponse, and exposes a consent-check interface.
- **Consequences:**
  - Positive:
    - One place for the consent wording version (open question OQ1) and the questionnaire version.
    - The consent check sits next to the data it protects.
  - Negative: The component grows if consent later expands to more purposes.
- **Alternatives Rejected:** Separate Consent and Questionnaire components: two components that always change together, with an extra hop for every health-answer write.
- **Security & compliance:**
  - Consent is recorded with timestamp and text version for evidence.
  - Health answers are stored owner-scoped and never logged.
  - The strictest-baseline privacy assumption is recorded (Q1).

### ADR-004: Separate Insight Generation and Image Generation components

- **Status:** Accepted
- **Context:**
  - Text generation (FR15) changes with prompt, tone and safety rules (`FR-008` a–f) and the text vendor (`NFR-008`).
  - Image generation (FR17, FR22) changes with the image vendor, cost and throughput (`NFR-013`, `ASM-011`, NFR3.1: 24 images within a 10-minute p95).
- **Decision:**
  - **InsightGeneration** is stateless. It covers the narrative, closing recommendations and tiering.
  - **ImageGeneration** owns the image records and the AI Visuals read endpoint.
  - Each uses its own swappable adapter (ADR-011).
- **Consequences:**
  - Positive:
    - Each vendor can be swapped by configuration independently.
    - Image concurrency and back-off do not affect text logic.
  - Negative: Two AI integration surfaces to maintain.
- **Alternatives Rejected:** A single AI component: mixes unrelated change drivers and makes vendor swaps riskier.
- **Security & compliance:**
  - Only needed data is sent to vendors (NFR5.4).
  - Medical, medication and allergy answers are excluded from cosmetic prompts.
  - OpenAI zero-data-retention is a deployment prerequisite (NFR5.2).

### ADR-005: Report owns report, PDF and Home; stores a content snapshot

- **Status:** Accepted
- **Context:**
  - The report auto-publishes (`BR-002`) and must have exactly 11 sections (`BR-008`).
  - Home Overview and the PDF are views of the same report (FR19, FR21).
  - The orchestrator assembles the report at the end of the pipeline. If Report later read from the orchestrator, there would be a dependency cycle.
- **Decision:** Report owns the report entity, the Home summary view and the PDF (cached through MediaStore). Assembly receives the analysis result from the orchestrator and stores a sections snapshot, so later reads never call the orchestrator.
- **Consequences:**
  - Positive:
    - No cycle.
    - The report reflects exactly what was published.
    - Home and PDF stay consistent with the report.
  - Negative: The snapshot duplicates some analysis-result data. This is acceptable, because the report is immutable once published.
- **Alternatives Rejected:**
  - A separate Dashboard component for Home and Visuals: a pass-through with no own data.
  - Report reading live from AnalysisResult: creates a cycle and couples reads to pipeline internals.
- **Security & compliance:** Report reads are owner-scoped and behind the 402 gate. AI text is returned as plain text for text-only rendering (XSS defence, `ASM-001`).

### ADR-006: Read-only Journey Status component for entry routing

- **Status:** Accepted
- **Context:**
  - FR26 requires every entry point to route to the first incomplete step, based on state owned by six components.
  - Identity-inconsistent photo sets count as incomplete (`BR-005` item 7).
- **Decision:** JourneyStatus computes the next step by calling each owner's public interface. It owns no data. Once a report exists, direct links to post-analysis routes are honoured.
- **Consequences:**
  - Positive: One routing rule; no component reads another's tables.
  - Negative: One request fans out to several components. This is cheap, because they are in-process calls within the same backend.
- **Alternatives Rejected:**
  - Identity computes the status: this couples auth to every downstream domain.
  - Each screen computes its own guard: the rules would drift.
- **Security & compliance:** Server-side routing state backs client-side guards, but the 402 and 409 checks remain enforced at each gated endpoint regardless.

### ADR-007: Media Store owns all private files and signed links

- **Status:** Accepted
- **Context:**
  - Face photos, 24 generated images and PDFs are sensitive (biometric).
  - They must never be public, must be encrypted at rest, and must be served only to their owner through expiring links (NFR5.3, project Forbidden rules).
  - Hosting is undecided (`NFR-010`).
- **Decision:** MediaStore owns StoredObject metadata, encryption, the storage adapter, and signed-link issue and verification. Photos, ImageGeneration and Report never touch storage directly.
- **Consequences:**
  - Positive:
    - The privacy rules are implemented and tested once.
    - The storage backend can change by configuration.
  - Negative: File reads go through one more interface.
- **Alternatives Rejected:** Each component stores its own files: this duplicates encryption and signing and risks an unsafe path.
- **Security & compliance:**
  - The design directly implements the Forbidden rule "never serve user photos from public storage".
  - Link tokens are scrubbed from logs.

### ADR-008: Orchestrator calls components directly inside a background worker

- **Status:** Accepted
- **Context:**
  - Analysis must run outside the request (FR13.3), survive restarts and resume from a failed step (FR13.4, NFR7).
  - The team is small, and there is no message broker because hosting is undecided.
- **Decision:** AnalysisOrchestrator executes an ordered step registry in a background worker. Each step calls the owning component's public interface directly. Job and step tables provide retries, timeouts, leases and resume. There is no event bus.
- **Consequences:**
  - Positive:
    - Simple control flow that is easy to debug and host-neutral.
    - One place for retry and resume logic.
  - Negative: The orchestrator knows every pipeline participant (high fan-out). This is acceptable, because the pipeline order is fixed by the client (`WF-001` step 6).
- **Alternatives Rejected:** Domain events through a transactional outbox: looser coupling, but adds an outbox, event schemas and eventual-consistency handling that no requirement needs.
- **Security & compliance:** The worker runs with the same data-access rules. Steps log no biometric or health data, and failures are recorded as error codes only.

### ADR-009: Three frontend components mirroring the mandated auth layering

- **Status:** Accepted
- **Context:**
  - The client mandates `UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services` (§5.3, `FE-001`–`FE-007`).
  - The API client needs the access token and must report an ended session. The store calls auth endpoints through the client. A naive design creates a cycle.
- **Decision:**
  - Three components: WebUI, AuthStore and ApiClient.
  - AuthStore registers a token provider and a session-ended callback with ApiClient at start-up (dependency inversion), so ApiClient depends on no frontend component.
  - UI never imports auth APIs or tokens (enforced by lint).
- **Consequences:**
  - Positive:
    - The client's layering is visible in the code structure.
    - Single-flight refresh lives in one place.
  - Negative: Wiring happens at app start, and tests must provide the injected callbacks.
- **Alternatives Rejected:**
  - One Web App component: the layering becomes a convention that is easy to erode.
  - ApiClient importing the store directly: creates a cycle.
- **Security & compliance:**
  - The access token is held in memory only.
  - The refresh token lives only in an httpOnly cookie.
  - `persist`, `localStorage` and `sessionStorage` are forbidden for auth (`FE-002`, `ASM-001`).

### ADR-010: Platform component for cross-cutting backend concerns

- **Status:** Accepted
- **Context:** Every backend component needs the same settings, error envelope, redacted logging and rate limiting. Inconsistent copies would break the app-wide 404/429/402 conventions and the log-redaction rule.
- **Decision:** A Platform component owns these concerns and the vendor-adapter factory. It depends on no domain component, and every backend component may depend on it. FaceAnalysisEngine stays independent, because its thresholds are passed in by callers.
- **Consequences:**
  - Positive: The conventions stay consistent and are tested once.
  - Negative: Platform must stay small, or it becomes a dumping ground. Only concerns needed by at least three components belong there.
- **Alternatives Rejected:** Helpers inside each component: this duplicates the redaction and limiter code, and the copies drift.
- **Security & compliance:**
  - Central redaction enforces "never log tokens, OTPs, photos or health answers".
  - The central limiter enforces the mandated auth rate limits.

### ADR-011: Ports and adapters for every external vendor

- **Status:** Accepted
- **Context:**
  - The AI text vendor, base URL, key and model must be swappable without code changes (`NFR-008`), and so must the image vendor (`NFR-013`).
  - Email provider selection is open (`NFR-012`).
  - Storage depends on the undecided host (`NFR-010`).
  - Per-PR tests may not call live vendors (project Forbidden).
- **Decision:**
  - Each vendor (email, AI text, AI image, payments, storage) sits behind a port, with a configuration-selected adapter and a fake.
  - Real adapters ship with their first consuming story.
  - An image-adapter contract test suite is shared by all image adapters.
- **Consequences:**
  - Positive:
    - Vendor swaps need configuration only.
    - Deterministic CI.
    - Adapters can be tested in isolation.
  - Negative: One more layer per integration.
- **Alternatives Rejected:** Calling vendor SDKs directly from services: breaks the swap requirement and makes CI depend on live vendors.
- **Security & compliance:**
  - Secrets come only from configuration and have no in-code defaults.
  - Stripe signature verification lives in the payments adapter path.

### ADR-012: Input lock owned by Onboarding and Photos, set by the orchestrator

- **Status:** Accepted
- **Context:** Photos and answers must lock once analysis starts, because the report is 1:1 with its analysis (`FR-014`, stories Q11). Having Onboarding and Photos ask the orchestrator would create cycles.
- **Decision:**
  - Onboarding and Photos each hold a `lockedAt` on their own records and refuse writes when it is set.
  - The orchestrator sets both locks, through their interfaces, as the first action of a successful start.
- **Consequences:**
  - Positive: No cycles; each owner enforces its own invariant.
  - Negative: The start operation spans two owners. It runs locks, then job creation, in that order, and is idempotent on retry.
- **Alternatives Rejected:** Onboarding and Photos query the orchestrator's job state on each write: creates a dependency cycle.
- **Security & compliance:** Prevents altering inputs behind a published, auto-published report.

## Proposed Decisions (confirm when building starts)

### ADR-013: Background worker polling PostgreSQL with row locks and leases

- **Status:** Proposed
- **Context:**
  - The analysis runs for minutes (NFR3.1: 10-minute p95).
  - It must survive process restarts, never run twice for a job, and resume from a failed step (NFR7).
  - There is no broker, and hosting is undecided (`NFR-010`).
- **Decision (proposed):**
  - A separate worker process polls the job table using `SELECT … FOR UPDATE SKIP LOCKED`, with a lease and heartbeat per step.
  - Retry counts and per-step timeouts are configurable.
  - An expired lease makes the job reclaimable.
- **Consequences:**
  - Positive:
    - Only PostgreSQL is needed, so it runs the same locally and on any host.
    - Exactly-one claim and restart-safe.
  - Negative:
    - Polling adds small latency and database load.
    - The lease length must exceed the longest step heartbeat interval.
- **Alternatives Rejected:**
  - FastAPI `BackgroundTasks`: does not survive a restart and blocks scaling.
  - Celery or RQ with Redis: adds a broker to host and secure without a stated need.
  - A cloud queue: assumes a host (`CON-007`).
- **Security & compliance:** The worker uses the same database credentials scope as the API. Step errors are stored as codes, without payloads that could contain health data.

### ADR-014: HTML-to-PDF rendering with headless Chromium

- **Status:** Proposed
- **Context:**
  - The branded PDF must contain 11 sections with 22 before/after images, the assessments and the limitations (FR19, `FR-013`).
  - It is generated on first download and cached.
  - The team develops on Windows and runs CI on Linux.
- **Decision (proposed):**
  - Render the report HTML template to PDF with headless Chromium (Playwright for Python), run through the background worker with a timeout.
  - While generation runs, the download control shows a busy state.
- **Consequences:**
  - Positive:
    - One HTML/CSS template serves both screen and print styling.
    - Playwright is already in the toolchain.
  - Negative:
    - Chromium is a large runtime dependency.
    - Generation takes seconds, so it runs as a job.
- **Alternatives Rejected:**
  - WeasyPrint: needs Pango/Cairo system libraries, which are painful on Windows development machines.
  - ReportLab: programmatic layout duplicates the design and is slow to change.
- **Security & compliance:**
  - Images are fetched from MediaStore server-side, never via public URLs.
  - The PDF itself is an encrypted stored object served by signed link.

### ADR-015: Application-level encryption and HMAC-signed links for local storage

- **Status:** Proposed
- **Context:**
  - No host means no managed object store yet (`NFR-010`).
  - Files must be encrypted at rest and served only through expiring owner-bound links (NFR5.3).
  - `<img>` tags cannot carry the in-memory Bearer token.
- **Decision (proposed):**
  - The local-disk storage adapter encrypts each file with AES-256-GCM (envelope encryption with a key from settings or a secrets manager, rotation noted by key ID).
  - MediaStore issues HMAC-signed tokens binding object ID, owner and expiry, verified by its file-serving endpoint.
  - When a host is chosen, an object-store adapter can use provider encryption and presigned URLs behind the same port.
- **Consequences:**
  - Positive:
    - Meets the privacy rules without a cloud dependency.
    - Portable to any host later.
  - Negative:
    - Key management is our responsibility until a secrets manager exists.
    - The backend serves the file bytes.
- **Alternatives Rejected:**
  - Unencrypted local disk: violates NFR5.3.
  - Storing images in PostgreSQL: rejected in requirements Q7.
- **Security & compliance:**
  - Keys are never in code.
  - Signed-link tokens are scrubbed from logs.
  - Link TTL is configurable, with boundary-tested expiry.

### ADR-016: Non-streaming chat replies

- **Status:** Proposed
- **Context:**
  - FR23 needs grounded replies with a medical-decline rule and a daily cap.
  - Streaming would need streaming support in the single API client and partial-output handling for the decline rule.
- **Decision (proposed):** The chat endpoint returns the complete reply in one response. The UI shows a "replying" state.
- **Consequences:**
  - Positive:
    - Simpler client and error handling.
    - The full reply can be validated before display.
  - Negative: Longer perceived wait on long replies.
- **Alternatives Rejected:** Server-sent events streaming: better perceived speed, but more client complexity. Revisit if the Report & UI Specification requires streaming.
- **Security & compliance:** The whole reply passes through output checks and text-only rendering before it reaches the user.
