# Requirements — FaceIQ (AI Facial Analysis Platform)

## Sources

- **Primary source (single source of truth):** `client_requirements.md` v1.0 (2026-09-29). Every requirement below cites the client IDs it derives from (`BC-*`, `FR-*`, `AUTH-*`, `FE-*`, `NFR-*`, `BR-*`, `WF-*`, `DATA-*`, `CON-*`, `ASM-*`, `OQ-*`).
- **Clarifications:** `requirements-analysis-questions.md` answers Q1–Q9 (2026-09-29), cited as `[Q<n>]`. These are **[Decided by delivery team]**, not client-stated.
- **Project practices:** the affirmed `team-practices` artifact (`inception/practices-discovery/team-practices.md`) and project rules in `aidlc/spaces/default/memory/project.md`. They supply the testing posture (custom ordering, 80%/90% coverage floors, requirement-ID test tagging) used to shape the acceptance criteria below.
- **Not available (skipped by this plan):** intent-statement and scope-document (ideation skipped because the client document already fixes context and scope, `BC-001`–`BC-007`, section 2); brownfield artifacts (greenfield, `CON-001`).
- **Labels:** requirements keep the client document's labels. **[Client-stated]** comes from the client. **[Recommendation]**, **[Assumption]** and **[Decided by delivery team]** do not. Assumptions stay assumptions until the client confirms them.

## Intent Analysis

- **Business goal:** automate Qoves-style expert facial-aesthetics reports with computer vision and AI, so a similar product can be sold at consumer scale and price (`BC-001`, `BC-002`).
- **Commercial model:** a paywalled, one-time-payment report per user. Nothing is analysed or shown before payment (`BC-003`, `FR-015`, `FR-016`).
- **Primary actor:** the End User, the only functional role in scope. The data model must still allow a future Admin role (`AUTH-009`, section 3).
- **Success means:**
  - a user can complete the WF-001 journey end to end;
  - a validated, identity-consistent photo set produces an auto-published 11-feature report with 24 AI images;
  - the codebase is clean and documented enough for the client's own team to extend it with AI-assisted tools (`BC-005`, `NFR-011`).
- **Request type:** greenfield, multi-component, system-wide (`CON-001`). Two separate applications: a FastAPI backend and a Next.js frontend (`NFR-001`, `NFR-005`).
- **Complexity:** complex. It combines custom JWT+OTP auth, Stripe payment gating, biometric photo validation, a CV + LLM + image-generation pipeline, and health-adjacent AI advice. Depth: Standard.

## Functional Requirements

Acceptance criteria use Given/When/Then. Exact thresholds that the client deferred to detailed specifications are marked as pending specification rather than invented.

### Entry and account

**FR1 — Landing page** (`FR-001`, `CON-003`) [Client-stated]
- FR1.1 The landing page shows a primary "Get started" call-to-action that begins signup/onboarding, and a separate header "Sign in" link for returning visitors.
- FR1.2 The landing page shows an animated, scroll-triggered "How it works" section illustrating questionnaire → photos → analysis → report.
- FR1.3 All other page structure and copy is [Recommendation], owned by the delivery team (`CON-003`).
- AC1.1.1 Given a visitor on the landing page, when they click "Get started", then they reach the signup screen.
- AC1.1.2 Given a visitor on the landing page, when they click the header "Sign in" link, then they reach the login screen, not the signup screen.
- AC1.2.1 Given a visitor scrolls to "How it works", when the section enters the viewport, then the four-step flow animates in order. With reduced-motion preference set, the steps appear without animation (`NFR6`).

**FR2 — Signup with mandatory email OTP** (`FR-002`, `AUTH-003`, `WF-002` step 1–2, `DATA-001`, `DATA-002`) [Client-stated]
- FR2.1 Signup step 1 collects full name, email and password. The backend hashes the password and creates an unverified user. No tokens are issued at this step.
- FR2.2 Step 2 emails a one-time code (`NFR-012`), stored hashed with a 10-minute expiry, a 60-second resend cooldown, and a 15-minute lockout after 5 failed attempts (`AUTH-011`).
- FR2.3 On correct OTP entry, the account is marked verified, an access token is issued and the httpOnly refresh cookie is set (`AUTH-002`, `AUTH-012`).
- FR2.4 Signup never reveals whether an email is already registered. Submitting an existing email shows the same neutral "check your email" outcome, and an email is sent to that address instead of an error on screen [Decided by delivery team, Q6].
- AC2.1.1 Given a new email, when the user submits full name, email and a valid password, then an unverified user exists, an OTP email is sent, and the OTP screen shows.
- AC2.2.1 Given an OTP older than 10 minutes, when the user submits it, then it is rejected and the user may request a new one.
- AC2.2.2 Given 5 failed OTP attempts, when a 6th is submitted within 15 minutes, then it is rejected with a lockout message, regardless of correctness.
- AC2.2.3 Given an OTP was sent less than 60 seconds ago, when the user requests a resend, then it is refused with the remaining wait time.
- AC2.4.1 Given an already-registered email, when someone submits signup with it, then the screen shows the same neutral outcome as a new email, and no account detail is disclosed.

**FR3 — Login with mandatory email OTP** (`AUTH-003`, `AUTH-006`, `WF-002`) [Client-stated]
- FR3.1 Login step 1 checks email + password against the stored hash. It issues no tokens.
- FR3.2 On success, step 2 applies the same OTP rules as FR2.2. A correct OTP issues the access token and sets the refresh cookie.
- FR3.3 A wrong email or wrong password produces one generic failure message that does not say which part was wrong [Decided by delivery team, Q6].
- AC3.1.1 Given a verified user, when they submit correct credentials and then the correct OTP, then they are signed in and land on the correct entry-point screen (FR26).
- AC3.3.1 Given an unknown email, and separately a known email with a wrong password, when each is submitted, then both show the identical generic message.

**FR4 — Session lifecycle** (`AUTH-002`, `AUTH-004`, `AUTH-006`, `AUTH-007`, `AUTH-010`, `AUTH-012`, `AUTH-013`, `AUTH-014`, `FE-001`–`FE-007`, `WF-002` steps 3–6, `DATA-003`, section 5.3) [Client-stated]
- FR4.1 The access token lives 15 minutes and is held only in memory in the Zustand auth store. The refresh token lives 7 days, only in an httpOnly cookie, and is rotated on every use.
- FR4.2 Presenting an already-rotated refresh token revokes its whole token family and forces re-authentication.
- FR4.3 The frontend refreshes about 1 minute before access-token expiry. It also handles 401 → refresh → retry. Concurrent refreshes share one in-flight request.
- FR4.4 On app start the frontend calls `POST /auth/refresh` and then `GET /auth/me` to restore the session. Protected routes wait while this restore is in progress.
- FR4.5 Auth state is cleared and the user redirected to login only when refresh returns 401 (expired, revoked, missing or reused) or the user logs out. Access-token expiry, reload, tab close, network errors and 5xx never clear it.
- FR4.6 Logout requires confirmation (`BR-010`), revokes the server-side session and clears the cookie.
- FR4.7 `/auth/refresh` and `/auth/logout` require the `X-Requested-With` header and an Origin/Referer allow-list match.
- FR4.8 The architectural layering `UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services` is preserved. UI components never call auth APIs or touch tokens.
- AC4.2.1 Given a refresh token that was already rotated, when it is presented again, then every session in that family is revoked and the response is 401.
- AC4.3.1 Given three API calls fail with 401 at the same moment, when the client refreshes, then exactly one refresh request is sent and all three calls are retried once.
- AC4.4.1 Given a signed-in user closes and reopens the browser within 7 days, when the app starts, then the session is restored without showing the login screen.
- AC4.5.1 Given the backend returns 503 during refresh, when the client handles it, then the user stays signed in and no redirect happens.
- AC4.7.1 Given a refresh request without `X-Requested-With`, or from a non-allow-listed Origin, when it reaches the backend, then it is rejected.

**FR5 — Forgot password** (`AUTH-015`, `WF-002` step 7, `BR-009` exception) [Client-stated / Decided by delivery team]
- FR5.1 The flow has three screens: request email → verify a 6-digit emailed code (FR2.2 rules) → set a new password.
- FR5.2 Code verification issues a short-lived (~10 minutes), single-use `reset_token`. It is never the raw OTP or email, and it is never accepted as a Bearer access token.
- FR5.3 A successful reset revokes all of the account's sessions on all devices.
- FR5.4 The request screen shows the same neutral outcome whether or not the email is registered (`BR-009` exception).
- AC5.2.1 Given a `reset_token` already used once, when it is presented again, then it is rejected.
- AC5.2.2 Given a `reset_token`, when it is sent as a Bearer token to a protected endpoint, then it is rejected with 401.
- AC5.3.1 Given a user signed in on two devices, when they complete a password reset, then both sessions are invalid at their next refresh.

**FR6 — Change password while signed in** (`AUTH-016`, `FR-021`, `WF-002` step 8) [Client-stated]
- FR6.1 Settings → Password accepts current, new and confirm-new password, and calls `POST /auth/change-password`.
- FR6.2 The current session is not revoked afterwards.
- AC6.1.1 Given a wrong current password, when the form is submitted, then the change is refused with an error toast (`BR-009`).
- AC6.1.2 Given new and confirm fields that differ, when the form is submitted, then it is blocked client-side before any request.
- AC6.2.1 Given a successful change, when the user continues using the app, then they stay signed in.

### Consent, onboarding and photos

**FR7 — Consent for sensitive data** [Decided by delivery team, Q1; supports `FR-003`, `FR-005`, `FR-008`, `ASM-010`]
- FR7.1 Before the user answers health-related questions or uploads photos, they give explicit, separately recorded consent covering:
  - processing their face images and derived biometric signature;
  - processing their health-related questionnaire answers;
  - sending both to the AI vendor.
- FR7.2 Each consent is stored with a timestamp and the consent-text version.
- FR7.3 Without the relevant consent, the questionnaire's health section and the photo upload stay blocked.
- AC7.1.1 Given a new user who has not consented, when they try to reach photo upload, then they are shown the consent step first and cannot skip it.
- AC7.2.1 Given a user consents, when the record is read, then it carries the consent type, timestamp and text version.
- Note: consent wording and the applicable law are open (OQ1). The requirement is the strictest-baseline assumption, not a legal determination.

**FR8 — Onboarding questionnaire** (`FR-003`, `FR-004`, `BR-003`, `DATA-004`, `WF-001` step 2) [Client-stated]
- FR8.1 The questionnaire presents 23 questions with branching logic. Content, answer options and branching come only from the pending Onboarding Questionnaire Specification (no invented questions).
- FR8.2 It ends with a mandatory, checkbox-gated disclaimer (no Body Dysmorphic Disorder–related concerns; informational only, not medical guidance). Submission is impossible while it is unchecked, enforced on client and server.
- FR8.3 Answers are stored against the user, together with the question text needed by FR15.
- AC8.2.1 Given all questions are answered but the disclaimer is unchecked, when the user submits (including by direct API call), then submission is refused.
- AC8.1.1 Given a branching answer, when the user selects it, then the next question shown follows the specification's branch. Test data comes from the specification once supplied.

**FR9 — Photo requirements and capture** (`FR-005`, `ASM-005`, `DATA-005`, `WF-001` steps 3–4) [Client-stated; angle count Assumption]
- FR9.1 A Photo Requirements screen shows the 7-point checklist:
  - remove glasses/hat;
  - natural, even lighting;
  - plain white background;
  - tie back long hair;
  - remove makeup;
  - avoid neck-covering clothing;
  - no filters.
- FR9.2 The user provides the front, left 3/4 and right 3/4 photos by file upload or camera capture.
- FR9.3 The required angle set is a single configuration value [Assumption `ASM-005`, unconfirmed, OQ2].
- FR9.4 A retake replaces the existing photo for that angle. No duplicate photo records.
- AC9.2.1 Given a device with a camera, when the user chooses capture for an angle, then the captured image is submitted the same way as an uploaded file.
- AC9.4.1 Given an existing front photo, when the user retakes it, then exactly one front photo exists for the user.

**FR10 — Per-photo validation** (`FR-006`, `BR-005`, `BR-004`, `ASM-002`, `CON-006`) [Client-stated; thresholds pending the Photo Validation Specification]
- FR10.1 The backend rejects a photo that fails any check:
  - file readability;
  - minimum resolution;
  - brightness/exposure;
  - exactly one face;
  - face-to-frame proportion;
  - eyes/mouth not occluded;
  - pose matches the requested angle.
- FR10.2 Each rejection states a clear, user-readable reason.
- FR10.3 Photos are validated by decoded content (not file extension), within size and pixel limits. They are re-encoded with EXIF/GPS metadata stripped before storage or AI use (project rule).
- AC10.1.1 Given a photo with two faces, when uploaded for any angle, then it is rejected with the "exactly one face" reason.
- AC10.1.2 Given a right-3/4 photo uploaded into the front slot, when validated, then it is rejected for pose mismatch.
- AC10.3.1 Given a text file renamed to `.jpg`, when uploaded, then it is rejected as unreadable.

**FR11 — Set-level identity check** (`BR-005` items 1–8, `ASM-010`) [Client-stated; heuristic and 0.6 threshold are Assumption `ASM-010`]
- FR11.1 The identity check runs only after all three angles pass FR10 individually. It never blocks a single upload.
- FR11.2 The flagged photo is chosen by match-count vote. On an ambiguous vote, front is the reference. If all three photos differ, both non-front photos are flagged.
- FR11.3 The result is a list of mismatched angles (empty when consistent), fully recomputed after every retake.
- FR11.4 The review screen highlights every flagged photo and disables Continue while any is flagged. A retake never auto-navigates forward. Skipping the review screen is decided only from the initial page load.
- FR11.5 The check is re-run server-side at checkout and at analysis start, returning HTTP 409 when inconsistent.
- AC11.2.1 Given front matches left, front matches right, but left does not match right, when the vote runs, then front is the reference and the result follows `BR-005` item 2's ambiguity rule. All 8 pairwise match patterns are covered by table tests.
- AC11.2.2 Given all three photos are different people, when the check runs, then left 3/4 and right 3/4 are both flagged.
- AC11.4.1 Given one flagged photo is retaken and the set becomes consistent, when the result returns, then the user stays on the review screen with Continue enabled.
- AC11.5.1 Given an identity-inconsistent set, when checkout or analysis start is called directly via the API, then the response is HTTP 409.

### Payment and analysis

**FR12 — Payment before analysis** (`FR-015`, `FR-016`, `BR-001`, `ASM-003`, `ASM-008`, `DATA-008`, `NFR-009`, `OQ-002`) [Client-stated; one-time model is Recommendation]
- FR12.1 The payment screen appears immediately after a valid, identity-consistent photo set. No teaser or partial content is shown before payment.
- FR12.2 Payment uses Stripe hosted Checkout. Payment status is updated only from a signature-verified, idempotently processed webhook, never from the success redirect.
- FR12.3 Price and currency are configuration values (`OQ-002` open).
- FR12.4 The webhook never starts analysis.
- FR12.5 Billing shows payment history (FR24).
- AC12.2.1 Given a webhook with an invalid signature, when received, then it is rejected and no payment status changes.
- AC12.2.2 Given the same Stripe event delivered twice, when processed, then the payment record changes once.
- AC12.4.1 Given a successful payment webhook, when processed, then no analysis job has started.
- AC12.1.1 Given an unpaid user, when they request any analysis or report endpoint, then the response is HTTP 402.

**FR13 — Start and run analysis** (`FR-015`, `WF-001` step 6, `BR-004`, `DATA-006`) [Client-stated; failure/timing behaviour Decided by delivery team, Q2/Q3]
- FR13.1 After payment, analysis starts only when the user clicks "Start Analysis".
- FR13.2 The server re-checks payment (402) and identity consistency (409) before starting.
- FR13.3 Analysis runs as a background job. A progress screen shows which step is running (measurement, narrative, assessments, images) [Q2, Q3].
- FR13.4 Failed steps are retried automatically a limited, configurable number of times. If a step still fails, the user sees a clear error and a "Try again" action that resumes from the failed step. Completed steps are kept, and no second payment is ever required [Q2].
- FR13.5 Starting analysis again while a job is running does not start a second job.
- FR13.6 Analysis-start is rate-limited per user [Q8].
- AC13.1.1 Given a paid user with a consistent set, when they click Start Analysis, then a job starts and the progress screen shows the first step.
- AC13.4.1 Given image generation fails after retries, when the user clicks "Try again", then measurement and narrative are not re-run and image generation resumes.
- AC13.5.1 Given a running job, when Start Analysis is requested again, then no second job is created.

**FR14 — Facial measurement** (`FR-007`, `NFR-007`, `ASM-010`) [Client-stated]
- FR14.1 MediaPipe Face Landmarker (478 points) plus OpenCV produce geometric measurements for seven of the 11 features. Hair, Skin and Neck use simpler heuristics or none.
- FR14.2 Mesh-based features are measured from the front photo. Each ear is measured from its own side's 3/4 photo, falling back to front only if that photo is missing [first-pass formula, `ASM-010`].
- AC14.2.1 Given landmark-coordinate fixtures for a front photo, when measurement runs, then each mesh-based feature yields its defined measurements deterministically.
- AC14.2.2 Given the left 3/4 photo is missing, when ear measurement runs, then the left ear uses the front photo.

**FR15 — AI narrative and recommendations** (`FR-008`, `FR-012`, `ASM-007`, `NFR-008`) [Client-stated; tiering heuristic Assumption `ASM-007`]
- FR15.1 A multimodal OpenAI call sends the photos, the CV measurements and the questionnaire answers with their real question text. The vendor, base URL, key and model are configuration.
- FR15.2 Each feature narrative:
  - is 3–5 sentences;
  - cites an actual measurement value when one exists;
  - is personalised to the user's stated goal, motivation and liked/disliked features.
- FR15.3 When answers indicate elevated appearance-related distress, the tone is especially measured and reassuring.
- FR15.4 Narratives never reference medical, medication or allergy answers.
- FR15.5 Recommendations are sorted into three tiers: at-home/lifestyle, OTC/skincare-active and optional in-clinic. Tiering uses a replaceable keyword rule set [Assumption `ASM-007`, unconfirmed, OQ3]. Recommendations always include "consult a qualified professional" language and are never prescriptive.
- FR15.6 AI output is validated against a structured schema (all 11 features present) before use.
- AC15.1.1 Given a questionnaire answer, when the prompt is built, then it contains the question text, not an opaque ID.
- AC15.4.1 Given answers that list a medication, when the prompt and output are checked, then no medication or allergy content appears in cosmetic commentary.
- AC15.5.1 Given the recommendation "consider seeing a dermatologist about laser treatment", when tiered, then it lands in the in-clinic tier.
- AC15.6.1 Given an AI response missing a feature, when validated, then the step is treated as failed and retried (FR13.4).

**FR16 — Facial Assessments** (`FR-018`) [Client-stated; first-pass heuristics]
- FR16.1 The system computes the following assessments:
  - masculine/feminine dimorphism slider values with an overall summary;
  - a prototypicality score;
  - facial-thirds proportions;
  - symmetry (score out of 100, a descriptive label, and a Regional Balance breakdown);
  - a face-shape wireframe.
- FR16.2 Face Shape content and the Skin analysis view are unconfirmed and must not be assumed beyond what the Report & UI Specification later defines.
- AC16.1.1 Given landmark fixtures, when symmetry is computed, then it yields a score from 0 to 100 and a label from the defined label set.

**FR17 — Per-feature AI "after" images** (`FR-010`, `FR-022`, `NFR-013`, `DATA-009`, `ASM-011`) [Client-stated]
- FR17.1 Each of the 11 features gets one AI-generated "after"/potential image, produced by identity-preserving image editing.
- FR17.2 Generation goes through a swappable image-generation abstraction selected by configuration.
- FR17.3 Images are generated once per report, with no user-triggered regeneration [Q8].
- AC17.1.1 Given a completed analysis, when the report is assembled, then exactly 11 feature images exist and each is linked to its feature.
- AC17.2.1 Given the image vendor setting is switched to a second adapter, when generation runs, then no code outside the adapter changes. The shared adapter contract suite passes.

**FR18 — Report assembly and auto-publish** (`FR-009`, `FR-010`, `FR-011`, `FR-012`, `FR-014`, `BR-002`, `BR-008`, `BR-011`, `DATA-007`) [Client-stated]
- FR18.1 The report always has exactly 11 feature sections: Hair, Eyebrows, Eyes, Nose, Cheeks, Jaw, Lips, Chin, Skin, Neck, Ears. Smile content sits under Lips.
- FR18.2 Each feature section has:
  - a narrative;
  - a before/after pair (the user's photo and the FR17 image) with projected-potential ideas;
  - a short summary callout.
- FR18.3 The report has a static branded introduction, an "Understanding Your Results" preamble and a limitations/disclaimer section, plus AI-synthesised closing recommendations.
- FR18.4 The report auto-publishes on generation, with no review step. There is one report per user, 1:1 with their analysis result.
- AC18.1.1 Given any completed analysis, when the report is generated, then it contains exactly the 11 named sections, with no "Smile" section.
- AC18.4.1 Given generation completes, when the user opens the app, then the report is immediately available.

**FR19 — PDF export** (`FR-013`, `CON-003`) [Client-stated; caching Decided by delivery team]
- FR19.1 The report downloads as a PDF with the delivery team's in-house branding.
- FR19.2 The PDF is generated on first download and cached.
- AC19.2.1 Given a report whose PDF was already generated, when downloaded again, then the cached file is served without regeneration.

### Post-analysis experience

**FR20 — Interactive report screen** (`FR-018`) [Client-stated]
- FR20.1 The interactive report has a table of contents, the Facial Assessments (FR16) and deeper per-feature metric tables for all 11 features.
- AC20.1.1 Given a published report, when the user opens it and selects a feature in the table of contents, then the view scrolls to that feature's metrics.

**FR21 — Home Overview** (`FR-017`) [Client-stated]
- FR21.1 Once a report exists, the landing screen shows:
  - key stats;
  - a before/potential image;
  - a "Priority Features to Improve" list;
  - a protocol summary;
  - a six-axis facial-harmony radar chart including Symmetry;
  - report status and PDF download.
- FR21.2 A persistent header navigation links Home, Report, AI Visuals and Chat. Account and billing are in Settings.
- FR21.3 Protocol section content is unconfirmed and must not be assumed.
- AC21.1.1 Given a published report, when the user signs in, then the Home Overview shows the radar chart with exactly six axes, one of them Symmetry.

**FR22 — AI Visuals** (`FR-020`, `NFR-013`, `DATA-010`) [Client-stated]
- FR22.1 The AI Visuals screen shows:
  - 5 hairstyle variations;
  - 5 outfit variations;
  - a healthy-aging 4-card stack: current (the user's own photo, not generated), +3, +5 and +10 years. That is 13 generated images.
- FR22.2 Together with FR17, each report has 24 generated images.
- AC22.1.1 Given a completed analysis, when AI Visuals is opened, then 13 generated images and 1 original photo are shown in the defined layout.

**FR23 — AI Beauty Assistant chat** (`FR-019`, `DATA-011`) [Client-stated; usage cap Decided by delivery team, Q8]
- FR23.1 The user can ask questions about their own report. Answers are grounded in their measurements, narrative and questionnaire answers.
- FR23.2 Medical and medication questions are declined with a redirect to a qualified professional.
- FR23.3 A configurable per-user daily message cap applies. When it is reached, the user is told when they can continue [Q8].
- FR23.4 Conversations and messages are stored per user.
- AC23.2.1 Given the question "can I take this with my medication?", when sent, then the assistant declines and recommends a qualified professional.
- AC23.3.1 Given the daily cap is reached, when another message is sent, then it is refused with a clear message and no AI call is made.

**FR24 — Settings** (`FR-021`, `BR-012`, `AUTH-016`, `ASM-009`) [Client-stated]
- FR24.1 Account Info shows full name (not editable), email, verification status and member-since date.
- FR24.2 Password uses FR6.
- FR24.3 Billing shows payment history and a working Stripe option. PayPal is visible but disabled (muted button with a "not available" status pill), with no PayPal integration.
- AC24.3.1 Given the Billing screen, when the user clicks PayPal, then nothing is initiated and the not-available state stays visible.

### App-wide behaviour

**FR25 — Feedback and confirmations** (`BR-009`, `BR-010`) [Decided by delivery team]
- FR25.1 Every user action with a real success/failure outcome shows a top-right toast, except anti-enumeration flows (FR2.4, FR3.3, FR5.4).
- FR25.2 Every destructive or irreversible action (for example logout) requires the shared confirmation dialog.
- AC25.2.1 Given a signed-in user clicks Logout, when the dialog appears and they cancel, then they stay signed in.

**FR26 — Entry-point routing** (`BR-005` item 7, `WF-001`) [Client-stated]
- FR26.1 Every entry point sends the user to the first incomplete step in this order:
  1. verification;
  2. consent;
  3. questionnaire;
  4. photos (an identity-inconsistent set counts as incomplete and shows the error);
  5. payment;
  6. start analysis;
  7. analysis in progress;
  8. Home Overview.
- AC26.1.1 Given a paid user whose stored set is identity-inconsistent, when they visit the dashboard URL directly, then they land on the photo screen with the mismatch shown.

## Non-Functional Requirements

**NFR1 — Technology stack** (`NFR-001`–`NFR-006`, `CON-002`) [Client-stated]
- The backend is Python + FastAPI, using SQLAlchemy and Alembic on PostgreSQL. It is vendor-neutral, with no Supabase in any role.
- The frontend is Next.js + TypeScript with Zustand. It is a separate application that talks to the backend over HTTP/CORS.
- No third-party auth service.

**NFR2 — Vendor configurability** (`NFR-008`, `NFR-013`, `FR-016`, `ASM-005`) [Client-stated]
- The following are changeable by configuration without code changes:
  - the AI text vendor base URL, key and model;
  - the image-generation vendor;
  - price and currency;
  - the required photo-angle set.
- Pass/fail: switching each value in configuration changes behaviour with zero code edits.

**NFR3 — Performance** [Decided by delivery team, Q3/Q4]
- NFR3.1 The full analysis, including all 24 images, completes within 10 minutes for 95% of analyses, measured from Start Analysis to report published.
- NFR3.2 Non-AI API endpoints respond within 500 ms at the 95th percentile under normal load (load level to be defined in NFR Requirements when that stage runs).
- NFR3.3 No availability target until hosting is chosen (`NFR-010`).

**NFR4 — Security** (`AUTH-010`–`AUTH-016`, `ASM-001`, `BR-005`, `CON-006`, project rules) [Client-stated + affirmed project rules]
- Refresh tokens are revocable and rotated with reuse detection.
- OTPs and passwords are hashed (Argon2id or bcrypt).
- CSRF defences apply on cookie endpoints, and CORS uses an explicit origin allow-list.
- JWTs use a fixed algorithm list and a token-type claim.
- Object-level authorization is enforced on every `DATA-004`–`DATA-011` record.
- Unauthenticated auth endpoints are rate-limited.
- AI text is rendered as text, and a Content-Security-Policy is applied.
- Tokens, OTPs, photos, biometric signatures and health answers are never logged.
- Pass/fail: each is covered by a negative test (bypass, reuse, CSRF, cross-user access, webhook replay, malicious upload).

**NFR5 — Privacy and data protection** [Decided by delivery team, Q1/Q7; formal retention policy out of scope]
- NFR5.1 Explicit consent is captured per FR7.
- NFR5.2 OpenAI zero-data-retention is requested for the account where the vendor offers it. Its status is recorded as a deployment prerequisite.
- NFR5.3 Photos and generated images sit behind a storage abstraction. They are never publicly readable, are served only to their owner through short-lived signed links, are encrypted at rest, and are kept until a retention policy is defined.
- NFR5.4 Only the data each AI call needs is sent. Medical, medication and allergy answers are excluded from prompts where FR15.4 forbids their use.

**NFR6 — Accessibility** [Decided by delivery team, Q5]
- All screens meet WCAG 2.1 level AA, including keyboard operation of the questionnaire, camera capture fallbacks, chart text alternatives (radar chart, sliders), and reduced-motion support for the landing animation.
- Pass/fail: automated axe checks report no violations, plus a manual keyboard pass per screen.

**NFR7 — Reliability of the analysis pipeline** [Decided by delivery team, Q2]
- Each pipeline step is idempotent and resumable.
- Retries are bounded and configurable.
- A failed analysis never loses completed steps and never needs a second payment (FR13.4).

**NFR8 — Cost and abuse controls** (`BR-006`) [Decided by delivery team, Q8]
- A configurable daily chat cap applies.
- Analysis start and photo upload are rate-limited per user.
- Images are generated once per report.
- The per-PR test suite never calls live OpenAI, Stripe live mode or a real email provider.

**NFR9 — Maintainability and handover** (`BC-005`, `NFR-011`) [Client-stated]
- The code is clean and modular, with a single repository holding `backend/` and `frontend/`.
- Each app has a README (setup, run, test, configuration keys).
- Public modules have docstrings/TSDoc, and ADRs record decisions.
- Linters enforce the layering.
- Frontend API types are generated from the backend OpenAPI schema.

**NFR10 — Testability and quality gates** (affirmed `team-practices`, client section 12)
- Every `FR-*`, `AUTH-*` and `BR-*` client ID maps to at least one tagged test.
- Coverage floors are 80% line and branch per app, and 90% for auth, payment-gate and identity-check code.
- GitHub Actions runs lint, type-check, tests, coverage and blocking security scans on every push and PR.

**NFR11 — Hosting neutrality** (`NFR-010`, `CON-007`) [Client-stated]
- No host is assumed. All environment-specific values come from environment variables.
- There are no deployment jobs until a host is chosen.

**NFR12 — Observability baseline** [Recommendation]
- Each backend exposes a health endpoint and emits structured logs with request IDs.
- The analysis pipeline records per-step duration and outcome, so NFR3.1 can be measured.
- Log content follows NFR4's exclusions.

## Constraints

- From-scratch build with no existing code (`CON-001`).
- No third-party auth-as-a-service (`CON-002`, `AUTH-001`).
- No client branding assets. The delivery team owns UI and branding (`CON-003`).
- Third-party usage costs belong to the client (`CON-004`, `BR-006`).
- No committed timeline or estimate (`CON-005`, `BR-007`).
- Enforced photo validation is a hard requirement because reports auto-publish (`CON-006`).
- Hosting is undecided and must not be assumed (`CON-007`).
- Four detailed specifications (Onboarding Questionnaire, Photo Capture, Photo Validation, Report & UI) are still to be supplied. Requirements that depend on them state behaviour, not the pending content.

## Assumptions

| ID | Assumption | Status |
|----|-----------|--------|
| A1 | Three photo angles (front, left 3/4, right 3/4) are enough; the angle set is configurable (`ASM-005`) | Unconfirmed with client (OQ2) |
| A2 | Keyword-heuristic recommendation tiering (`ASM-007`) | Unconfirmed with client (OQ3) |
| A3 | Identity signature from 5 landmark-distance ratios with a 0.6 threshold; ear formula first-pass (`ASM-010`) | Needs a tuning pass on diverse photos |
| A4 | Image-generation quality and cost at 24 images per report are acceptable (`ASM-011`) | Not yet validated |
| A5 | Per-photo validation thresholds are set during development (`ASM-002`) | Pending the Photo Validation Specification |
| A6 | One-time payment per report (`ASM-003`, `FR-016`) | Working model; price open (`OQ-002`) |
| A7 | The strictest privacy baseline is enough until the target market is known [Q1] | Delivery-team assumption |
| A8 | Refresh token in httpOnly cookie, access token in memory; residual XSS risk limited to 15 minutes (`ASM-001`) | Recorded; mitigated by CSP and output encoding |

## Out of Scope

Deferred by the client; do not build without a scope change (`BC-006`, section 2.2):
- Admin panel and the Draft → Pending Review → Approved → Published workflow. The `role` field still exists (`AUTH-009`).
- Email notifications (SendGrid). OTP email is in scope.
- Meta Pixel and Google Tag Manager.
- A formal data-retention/privacy policy.
- A working PayPal integration (visible-but-disabled only).

Also not included:
- Deployment jobs or environments (practices: build and push only).
- User-triggered image regeneration.
- Editing the full name after signup (`ASM-009`).

## Open Questions

| ID | Question | Owner | Blocks |
|----|----------|-------|--------|
| OQ1 | Target market and applicable privacy law (GDPR Art. 9, BIPA, CCPA) for face images, biometric signature and health answers; final consent wording [Q1] | Client | FR7 wording, NFR5 |
| OQ2 | Photo angle count: 3 vs the 7-pose reference (`OQ-010`, `ASM-005`) | Client | FR9, FR10 |
| OQ3 | Recommendation tiering: keyword heuristic vs AI classification (`OQ-012`, `ASM-007`) | Client | FR15.5 |
| OQ4 | Report price and currency (`OQ-002`) | Client | FR12.3 configuration values |
| OQ5 | Content of the four pending detailed specifications | Client | FR8, FR9, FR10, FR16, FR18, FR20, FR21 |
| OQ6 | Email/OTP provider selection (`NFR-012`) | Delivery team | FR2.2 integration |
| OQ7 | Normal-load level for NFR3.2 | Delivery team | NFR3.2 test |

## Traceability

| Client ID(s) | Requirement(s) |
|--------------|----------------|
| FR-001, CON-003 | FR1 |
| FR-002, AUTH-003, WF-002, DATA-001, DATA-002, NFR-012 | FR2, FR3 |
| AUTH-001, AUTH-005, CON-002 | NFR1, NFR4 |
| AUTH-002, AUTH-004, AUTH-006, AUTH-007, AUTH-008, AUTH-010, AUTH-012, AUTH-013, AUTH-014, FE-001–FE-007, DATA-003, section 5.3 | FR4, NFR4 |
| AUTH-009 | Intent Analysis, Out of Scope (role field) |
| AUTH-011 | FR2.2, FR3.2, FR5.1 |
| AUTH-015 | FR5 |
| AUTH-016, ASM-009 | FR6, FR24 |
| FR-003, FR-004, BR-003, DATA-004 | FR8 |
| FR-005, ASM-005, DATA-005 | FR9 |
| FR-006, BR-004, BR-005, ASM-002, CON-006 | FR10, FR11 |
| ASM-010 | FR11, FR14 |
| FR-015, FR-016, BR-001, ASM-003, ASM-008, DATA-008, NFR-009, OQ-002 | FR12, FR13 |
| FR-007, NFR-007 | FR14 |
| FR-008, FR-012, ASM-007, NFR-008, OQ-012 | FR15, NFR2 |
| FR-018 | FR16, FR20 |
| FR-010, FR-022, NFR-013, DATA-009, ASM-011 | FR17 |
| FR-009, FR-011, FR-014, BR-002, BR-008, BR-011, DATA-007 | FR18 |
| FR-013 | FR19 |
| FR-017 | FR21 |
| FR-020, DATA-010 | FR22 |
| FR-019, DATA-011 | FR23 |
| FR-021, BR-012 | FR24 |
| BR-009, BR-010 | FR25 |
| WF-001, BR-005 item 7 | FR26 |
| NFR-001–NFR-006 | NFR1 |
| NFR-010, CON-007 | NFR11 |
| NFR-011, BC-005 | NFR9 |
| BR-006, CON-004 | NFR8, Constraints |
| BR-007, CON-005 | Constraints |
| BC-006, section 2.2 | Out of Scope |
| OQ-010 | OQ2 |
