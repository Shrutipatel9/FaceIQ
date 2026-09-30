# Bolt Plan — FaceIQ

## Overview

A **Bolt** is one build pass over one or more units of work that ends in something that runs and can be demonstrated. This plan orders the 13 units from `unit-of-work` into **15 Bolts** (Q2): one Bolt per unit, except identity (U3) and photos (U5), which are each split in two.

- **Sources:**
  - `unit-of-work`, `unit-of-work-dependency` and `unit-of-work-story-map`: units, dependency graph and story order;
  - `stories`: the 67 stories with acceptance criteria and per-story implementation phases;
  - `requirements`;
  - `components`;
  - `contract-summary`: contracts C1–C18;
  - the affirmed `team-practices`.
- **Order (Q1):**
  - The **walking skeleton** comes first. This is a thin end-to-end slice built first to prove the pieces connect.
  - After that, Bolts follow the dependency graph, with the riskiest unknowns pulled forward through two throw-away spikes (see `risk-and-sequencing-rationale.md`).
- **Parallel work (Q3):** Bolts run in parallel only where the graph allows, meaning B14 and B15 at the end, plus fixture-based early work inside a Bolt.
- **Builders:** every Bolt is built by the AI developer agent and reviewed and approved by Crest Infosystems (see `team-allocation.md`).

**Every Bolt's Definition of Done includes:**
- the story Definition of Done in `stories.md`;
- CI green, including coverage floors of 80% per app and 90% on critical modules, the security scans, and the requirement-tag check;
- the Bolt's stories' acceptance criteria automated as tagged tests;
- the README configuration keys updated;
- one squash-merged pull request per story or small group of stories into `main`, with requirement IDs in the title (affirmed Way of Working).

**Deployment:** there is none in this plan (affirmed Deployment). "Demo" means running the app locally with the one documented start command.

## Bolt Sequence

| Order | Bolt | Units | Stories | Skeleton | Runs in parallel with |
|-------|------|-------|---------|----------|------------------------|
| 1 | B1 walking skeleton | U1 | US0.1, US0.3, US0.4, US0.5, US0.6 | **Yes** | CV spike (exploratory) |
| 2 | B2 platform services | U2 | US0.2, US0.7, US0.8, US0.9 | — | CV spike (exploratory) |
| 3 | B3 signup and verification | U3 (part 1) | US1.1, US1.2, US1.3, US1.4, US1.5 | — | — |
| 4 | B4 login, sessions and account safety | U3 (part 2) | US1.6, US1.7, US1.8, US1.9, US1.10, US1.11, US9.3 | — | — |
| 5 | B5 consent and questionnaire | U4 | US2.1, US2.2, US2.3, US2.4 | — | — |
| 6 | B6 photo capture and private storage | U5 (part 1) | US0.11, US3.1, US3.2, US3.4, US3.3 | — | — |
| 7 | B7 photo validation and identity check | U5 (part 2) | US3.5, US3.6, US3.7, US3.8, US3.9 | — | — |
| 8 | B8 payment | U6 | US4.1, US4.2, US4.3 | — | — |
| 9 | B9 analysis pipeline | U7 | US0.10, US5.1, US5.2, US5.3 | — | AI-image spike (exploratory) |
| 10 | B10 facial measurement and assessments | U8 | US5.4, US5.5, US5.8, US5.9 | — | AI-image spike (exploratory) |
| 11 | B11 AI narrative and tiering | U9 | US5.6, US5.7 | — | — |
| 12 | B12 AI images and visuals | U10 | US5.10, US5.11, US7.2 | — | — |
| 13 | B13 report, PDF and home | U11 | US6.1, US6.2, US6.3, US6.4, US6.5, US7.1 | — | — |
| 14 | B14 beauty assistant | U12 | US7.3, US7.4, US7.5 | — | B15 |
| 15 | B15 account and app-wide | U13 | US8.1, US8.2, US8.3, US9.1, US9.2, US9.4, US9.5 | — | B14 |

## Bolt Details

### B1 — Walking skeleton (U1) · skeleton marker

- **Stories, in order:**
  - US0.1 repository and core CI;
  - US0.3 backend foundation;
  - US0.4 one-command local environment;
  - US0.5 frontend foundation;
  - US0.6 auth-layering skeleton.
- **What it proves:** the two-app split, CORS with credentials, PostgreSQL through SQLAlchemy/Alembic, and the mandated layering UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services.
- **Definition of Done:**
  - The login form submits a seeded user's email and password and gets "OTP required" end to end, with no token issued.
  - Lint and CI contracts reject UI code that touches auth or tokens.
  - One command starts PostgreSQL, Mailpit, the API, a placeholder worker and the frontend.
  - A minimal seed script creates the verified test user (the full seeding CLI arrives in B2).
- **Confidence hypothesis:** the chosen stack and layering work together on Windows development machines and Linux CI with one command. If they do not, we learn it before any feature depends on them.
- **Demo:** run the start command, open the login page, sign in with the seeded user, and see the "enter your code" state. The Playwright skeleton test passes in CI.
- **Proposed verification command for skeleton and unit checkpoints:** `./scripts/verify.sh`. It runs backend pytest, frontend Vitest, and the Playwright E2E suite against the local stack. It is recorded when building starts, because this plan has no Construction phase.
- **Exploratory, alongside B1–B2:** the CV feasibility spike (the first part of US0.11) on licensed or consented images. No production code is merged. Its findings feed B6 and B7.

### B2 — Platform services (U2)

- **Stories:** US0.2 security, coverage and traceability gates; US0.7 adapter ports and fakes; US0.8 rate limiter; US0.9 seeding.
- **Definition of Done:** CI blocks on secrets, High/Critical findings, coverage below the floors and unmapped requirement IDs. Adapter factory and fakes are in place. Per-PR tests cannot reach the network. Seeding scenarios exist.
- **Confidence hypothesis:** the quality and security gates run fast enough to keep short-lived branches practical (target: CI under 10 minutes).
- **Demo:** a pull request with a fake secret, and one with an untested client requirement ID, both fail CI.

### B3 — Signup and verification (U3, part 1)

- **Stories:** US1.1 landing; US1.2 signup; US1.3 OTP rules (tests first); US1.4 tokens and session; US1.5 resend.
- **Definition of Done:**
  - A new user signs up, receives a code in Mailpit, verifies it, and is signed in with an in-memory access token and an httpOnly refresh cookie.
  - Existing-email and unverified paths behave per Q6 and Q8 (neutral response, fresh code).
  - The real SMTP adapter ships in this Bolt.
- **Confidence hypothesis:** the OTP rules (10-minute expiry, 60-second cooldown, lockout after 5 attempts) are correct at their boundaries, and signup never reveals whether an email is already registered.
- **Demo:** sign up, read the code in Mailpit, sign in. A second signup with the same email shows the identical screen.
- **External items:** email provider for real sends (the build uses Mailpit until then).

### B4 — Login, sessions and account safety (U3, part 2)

- **Stories:** US1.6 login; US1.7 rotation, reuse detection and CSRF (tests first); US1.8 session restore; US1.9 seamless refresh; US1.10 logout; US1.11 forgot password; US9.3 data isolation foundation.
- **Definition of Done:**
  - Full login, reload and restart, silent refresh, logout and reset flows work.
  - A reused refresh token revokes the whole token family.
  - A cross-user request returns 404, backed by the owner-scope registry.
  - Auth modules reach 90% coverage.
- **Confidence hypothesis:** sessions survive reloads and restarts, yet a stolen token is useless. This is the client's central auth requirement set (`AUTH-010`–`AUTH-016`, `FE-002`–`FE-006`).
- **Demo:** log in, close and reopen the browser and stay signed in; replay an old refresh token and watch the session end; reset a password and watch both devices sign out.

### B5 — Consent and questionnaire (U4)

- **Stories:** US2.1 consent; US2.2 questionnaire API; US2.3 questionnaire UI; US2.4 disclaimer.
- **Definition of Done:**
  - Two consent records are stored.
  - A format-fixture questionnaire branches, saves and resumes.
  - Submission is impossible without the disclaimer, on both client and server.
  - Health answers are blocked without consent.
- **Confidence hypothesis:** the data-driven questionnaire can take the client's pending specification without code changes.
- **Demo:** consent, answer half the questions, leave, return and resume, then try to submit without ticking the disclaimer.
- **External items:** the Onboarding Questionnaire Specification (real content), and consent wording and privacy law (OQ1). Until they arrive, the Bolt is done on fixtures, with a follow-up to load the real content.

### B6 — Photo capture and private storage (U5, part 1)

- **Stories:** US0.11 CV spike (findings formalised); US3.1 requirements screen; US3.2 secure upload; US3.4 signed-link delivery; US3.3 capture UI.
- **Definition of Done:**
  - Three angles can be uploaded or captured.
  - Files are re-encoded with EXIF removed, encrypted at rest and served only through expiring owner-only links.
  - Hostile files are rejected.
  - Spike findings are written up.
- **Confidence hypothesis:** biometric photos can be handled privately without a cloud host (Proposed ADR-015).
- **Demo:** upload three photos. Copy an image link and watch it fail after expiry and for another user. Inspect the storage to confirm the bytes are unreadable.
- **External items:** licensed or consented face images for tests; the HEIC decision (OQ-S5); the Photo Capture Specification.

### B7 — Photo validation and identity check (U5, part 2)

- **Stories:** US3.5 basic checks with the MediaPipe adapter; US3.6 occlusion and pose; US3.7 identity vote (tests first); US3.8 retake; US3.9 onboarding routing.
- **Definition of Done:**
  - Per-photo checks return stable reason codes.
  - The 8-row identity table passes, including the boundary at the threshold.
  - The retake flow works.
  - Every entry point routes to the first incomplete onboarding step.
- **Confidence hypothesis:** the landmark-ratio identity check separates same-person from different-person sets on the licensed test set well enough to proceed (`ASM-010`, tuning expected).
- **Demo:** upload a mismatched set, see the flagged photo, retake it, and continue.
- **External items:** the Photo Validation Specification thresholds (configurable defaults until then); client confirmation of the identity-vote reading (OQ-S1).

### B8 — Payment (U6)

- **Stories:** US4.1 checkout; US4.2 verified webhook (tests first); US4.3 payment gate (tests first).
- **Definition of Done:**
  - A consistent-set user pays once through Stripe test mode.
  - The paid state comes only from a signed, idempotent webhook.
  - A double payment is impossible.
  - Every paid route returns 402 to unpaid users.
- **Confidence hypothesis:** nothing is analysed or shown before payment, and the gate cannot be bypassed from the API (`FR-015`, `BR-001`).
- **Demo:** pay with a Stripe test card, watch "confirming" become "paid", and call a gated endpoint as an unpaid user to see 402.
- **External items:** the client's Stripe test account and keys; price and currency (`OQ-002`, configurable).

### B9 — Analysis pipeline (U7)

- **Stories:** US0.10 worker; US5.1 start (tests first for 402/409); US5.2 progress; US5.3 resume.
- **Definition of Done:**
  - A confirmed start locks inputs and runs a job of fake steps in the worker.
  - Progress polls correctly.
  - A failing fake step is retried, then resumes from the failed step, with no second payment.
  - Two workers never claim the same job.
- **Confidence hypothesis:** the job model (Proposed ADR-013) survives restarts and resumes correctly, before any real CV or AI step depends on it.
- **Demo:** start analysis, kill the worker mid-job, restart it, and watch it continue.
- **Exploratory, alongside B9–B10:** an AI-image quality and cost spike. A few `gpt-image-1` edits on consented images measure identity preservation, latency and cost per image against the 24-image and 10-minute targets (`ASM-011`, NFR3.1). The findings go to the client before B12.

### B10 — Facial measurement and assessments (U8)

- **Stories:** US5.4, US5.5 measurements; US5.8, US5.9 assessments.
- **Definition of Done:**
  - The seven mesh features and the ears match their golden fixture values.
  - Symmetry, thirds, dimorphism and face-shape data are produced.
  - Prototypicality waits for its reference model (OQ-S4, on the pending list).
  - The steps are registered in the pipeline.
- **Confidence hypothesis:** the measurements are deterministic and explainable from landmarks alone.
- **Demo:** run analysis for a seeded consistent set and inspect the measurements in the analysis result.

### B11 — AI narrative and tiering (U9)

- **Stories:** US5.6 narrative; US5.7 tiering (tests first).
- **Definition of Done:**
  - Prompt-builder tests cover `FR-008` (a), (b), (e) and (f).
  - Structured output is validated for 11 features, 3–5 sentences each, a cited measurement, and closing recommendations.
  - Recommendations are tiered.
  - The real OpenAI text adapter works through recorded responses.
  - The opt-in live evaluation suite exists.
- **Confidence hypothesis:** tone and safety rules hold for a distressed persona, and medical answers never appear in commentary.
- **Demo:** run the narrative step for two fixture users with different goals and distress levels and compare the tone.
- **External items:** the OpenAI API key and zero-data-retention status; tiering method confirmation (OQ3).

### B12 — AI images and visuals (U10)

- **Stories:** US5.10 feature images; US5.11 visuals; US7.2 AI Visuals screen.
- **Definition of Done:**
  - Exactly 24 images per report.
  - A retry generates only the missing images.
  - The shared image-adapter contract suite passes.
  - The visuals screen works with expiring links.
- **Confidence hypothesis:** image generation fits the 10-minute p95 with bounded concurrency. The spike's cost figure is accepted by the client.
- **Demo:** browse the 5 hairstyles, 5 outfits and the 4-card aging stack for a real test analysis.
- **External items:** OpenAI image access; the client's acceptance of image quality and cost (`ASM-011`).

### B13 — Report, PDF and home (U11)

- **Stories:** US6.1 assembly and publish; US6.2 feature sections; US6.3 interactive report; US6.4 PDF pipeline; US6.5 branded PDF; US7.1 Home and header.
- **Definition of Done:**
  - Exactly one published report per user, with 11 sections.
  - AI text is rendered as text.
  - The PDF is generated once and then served from cache.
  - Home shows the six-axis radar with a data-table alternative.
  - Content from the pending specification is kept in configurable fields.
- **Confidence hypothesis:** the whole paid journey completes end to end, from signup to a published report and PDF, on the local stack.
- **Demo:** the full journey from landing page to a downloaded PDF.
- **External items:** the Report & UI Specification (key stats, protocol summary, radar axes, labels); the PDF engine confirmation (Proposed ADR-014).

### B14 — Beauty assistant (U12) · parallel with B15

- **Stories:** US7.3 chat; US7.4 medical decline; US7.5 daily cap.
- **Definition of Done:**
  - Replies are grounded only in the user's own data.
  - The decline rule is always present.
  - The daily cap resets at 00:00 UTC, and failed replies do not count toward it.
- **Confidence hypothesis:** the assistant is useful without ever giving medical advice.
- **Demo:** ask a feature question, then a medication question, then hit the cap.

### B15 — Account and app-wide (U13) · parallel with B14

- **Stories:** US8.1 Settings; US8.2 change password; US8.3 billing; US9.1 user menu; US9.2 post-payment routing; US9.4 accessibility audit; US9.5 load test.
- **Definition of Done:**
  - Settings are reachable at any time.
  - Changing the password signs out other devices.
  - Billing shows the payment method and receipt link, with PayPal disabled.
  - The user menu appears on every screen.
  - Every entry point routes correctly.
  - The full WCAG 2.1 AA audit passes.
  - The load test waits for OQ7.
- **Confidence hypothesis:** the finished app is navigable and accessible end to end.
- **Demo:** a keyboard-only run through the complete journey, and a Settings visit during onboarding.
- **External items:** the load profile (OQ7).

## Construction Settings to Record When Building Starts

This planning workflow has no Construction phase. When a build workflow starts, record:
- **Iteration:** unit by unit (the default), with checkpoints enabled.
- **Execution:** serial, except B14 and B15.
- **Verification command:** `./scripts/verify.sh`, created in B1, after human approval.
- **Staffing:** one session, AI-built, with Crest review.
