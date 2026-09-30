# Requirements Analysis — Questions

`client_requirements.md` is the single source of truth and already covers most requirements with IDs. These questions cover only the gaps it leaves open, found while checking completeness across functional, non-functional, user scenarios, business, technical and quality areas. Your answers are recorded in derived documents as **[Decided by delivery team]** (or stay **[Assumption]** where you say so), never as client-stated.

Fill each `[Answer]:` with a letter, or `X` plus your own text.

## Q1 — Privacy regime for face photos and health answers

The platform sends face photos, a biometric identity signature (ASM-010) and questionnaire answers that include medical conditions and medications (FR-003) to OpenAI. The client document does not name a target market, and a formal privacy/retention policy is out of scope (section 2.2). Which assumption should the requirements carry?

A. Target market unknown: design for the strictest common baseline. Require explicit, separate consent for processing face images and health answers before upload, request OpenAI zero-data-retention where available, and flag the market/law question to the client as open (recommended)
B. United States only, excluding states with biometric-privacy laws (e.g. Illinois BIPA); plain terms-of-service acceptance, no separate consent step
C. EU/UK users included: full GDPR special-category handling (explicit consent, data-protection impact assessment) as a requirement now
D. Leave it entirely open: no consent requirement now, record as an open question only
X. Other (please specify)

[Answer]: A

## Q2 — What happens when analysis fails partway

After the user clicks Start Analysis (FR-015), the pipeline runs measurement, the OpenAI narrative and 24 AI images (FR-020, FR-022). The client document does not say what happens if a step fails (for example, OpenAI is down or an image fails to generate).

A. Analysis runs as a background job with a progress screen. Failed steps are retried automatically a limited number of times; if it still fails, the user sees a clear error and a "Try again" button that resumes from the failed step. The user never pays twice (recommended)
B. All-or-nothing: the report appears only when every step succeeds; on failure the whole analysis restarts from the beginning on retry, with no second payment
C. The text report publishes as soon as measurement and narrative succeed; images fill in later, and any image that still fails shows a placeholder
X. Other (please specify)

[Answer]: A

## Q3 — How long may analysis take?

The requirements have no time target for the analysis step, so it cannot be tested.

A. Target: the full report, including all 24 images, within 10 minutes for 95% of analyses; the progress screen shows which step is running (recommended)
B. Target: within 5 minutes for 95% of analyses
C. No target for now; measure it first and set a target after real runs (recorded as an open question)
X. Other (please specify)

[Answer]: A

## Q4 — Speed and uptime targets for the rest of the app

There are no latency or availability targets, and hosting is undecided (NFR-010).

A. Non-AI API calls respond within 500 ms at the 95th percentile under normal load; no uptime target until hosting is chosen (recommended)
B. No performance or uptime targets for now
X. Other (please specify)

[Answer]: A

## Q5 — Accessibility

The client document has no accessibility requirement.

A. WCAG 2.1 level AA for all screens (recommended: common legal baseline for consumer web apps)
B. Basic accessibility only (keyboard navigation, alt text, sufficient colour contrast) with no formal conformance level
C. No accessibility requirement for now
X. Other (please specify)

[Answer]: A

## Q6 — Revealing whether an email is registered

WF-002 says signup checks that the email is not already registered, while BR-009 hides outcomes in forgot-password to prevent account discovery. Should signup and login reveal whether an email has an account?

A. Never reveal it: signup, login and forgot-password all show the same neutral message; signing up with an existing email emails that address instead of showing an error (recommended)
B. Signup may say "email already registered"; login and forgot-password stay neutral
X. Other (please specify)

[Answer]: A

## Q7 — Where photos and generated images are stored

Photos (DATA-005) and 24 generated images per report (DATA-009, DATA-010) must be stored somewhere, and hosting is undecided.

A. Behind a storage abstraction (local disk in development; any object store later), never publicly readable, served only through short-lived signed links to the owning user, encrypted at rest. Kept until a retention policy is defined, which stays out of scope (recommended)
B. Stored as binary data inside PostgreSQL
X. Other (please specify)

[Answer]: A

## Q8 — Limits on AI usage per user

OpenAI costs are the client's (BR-006), and the AI Beauty Assistant chat (FR-019) could be used without limit.

A. A configurable per-user daily cap on chat messages, plus rate limits on analysis start and photo uploads; images are generated once per report with no user-triggered regeneration (recommended)
B. No limits for now
X. Other (please specify)

[Answer]: A

## Q9 — Unconfirmed assumptions: photo angle count and recommendation tiering

Two items are still open with the client: 3 photo angles instead of 7 (ASM-005, OQ-010), and sorting recommendations into tiers by keyword rules (ASM-007, OQ-012).

A. Plan with both as written: 3 angles as a single configurable setting, keyword-based tiering behind a replaceable rule set. Keep both labelled as unconfirmed assumptions and list them as open questions for the client (recommended)
B. Plan for 7 angles now
C. Plan tiering with AI-based classification instead of keywords
X. Other (please specify)

[Answer]: A

## Consolidated Summary Confirmation

Summary of answers:
- Privacy: design for the strictest baseline; explicit separate consent for face images and health answers before upload; request OpenAI zero-data-retention; market/law flagged to the client as open (Q1 A)
- Analysis failures: background job with progress screen; limited automatic retries; clear error and "Try again" that resumes from the failed step; never pay twice (Q2 A)
- Analysis duration: full report including 24 images within 10 minutes for 95% of analyses; progress screen shows the running step (Q3 A)
- Speed: non-AI API calls within 500 ms at the 95th percentile under normal load; no uptime target until hosting is chosen (Q4 A)
- Accessibility: WCAG 2.1 AA for all screens (Q5 A)
- Email enumeration: never reveal whether an email is registered on signup, login or forgot-password (Q6 A)
- Image storage: private storage abstraction, signed short-lived owner-only links, encrypted at rest, kept until a retention policy exists (Q7 A)
- AI usage: configurable daily chat cap, rate limits on analysis start and uploads, images generated once per report (Q8 A)
- Open client assumptions: 3 configurable angles and keyword tiering as written, still labelled unconfirmed and listed as open client questions (Q9 A)

Does this all look correct before I generate the requirements artifact?

- Looks correct
- Request changes

[Answer]: Looks correct
