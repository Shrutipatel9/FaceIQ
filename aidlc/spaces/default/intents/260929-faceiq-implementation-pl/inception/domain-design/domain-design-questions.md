# Domain Design — Questions

These questions settle the **logical building blocks** of FaceIQ: the parts of the code we write, which data each one owns, and how they call each other. Deployment shape (one backend process or several) is decided later in Units Generation. The stack is already fixed by the client: FastAPI + SQLAlchemy/Alembic/PostgreSQL backend, and a Next.js + TypeScript + Zustand frontend (`NFR-001`–`NFR-006`).

Inputs:
- `requirements-analysis/requirements.md`, FR1–FR26 and NFR1–NFR12;
- `user-stories/stories.md`, US0.1–US9.5;
- the affirmed practices in `team-practices.md`.

Draft decomposition (about 14 building blocks):
- **Backend:**
  - Identity: accounts, codes, sessions, reset.
  - Onboarding: consent, questionnaire.
  - Photos: upload, validation, identity check.
  - Face Analysis Engine: computer-vision measurements and assessments.
  - Payments.
  - Analysis Orchestrator: the pipeline job.
  - Insight Generation: AI narrative and tiering.
  - Image Generation: per-feature and visuals images.
  - Report: assembly, read, PDF, home summary.
  - Beauty Assistant: chat.
  - Journey Status.
  - Media Store: private files and signed links.
  - Platform: settings, errors, rate limits.
- **Frontend:** Web UI, Auth Store, API Client.

The questions below cover the boundary calls where more than one design is reasonable.

Fill each `[Answer]:` with a letter, or `X` plus your own text.

## Q1 — Where computer-vision logic and its results live

Measurements (FR14), assessments (FR16) and the identity signature (FR11) all come from the same face landmarks.

A. A **Face Analysis Engine** component holds the pure computation (landmarks in, numbers out) and owns no data. Photos stores landmarks and identity results; the Analysis Orchestrator stores the analysis result (DATA-006). The engine can then be tested with landmark fixtures alone, and swapped later (recommended)
B. The Face Analysis Engine owns the analysis result and landmarks itself
X. Other (please specify)

[Answer]: A

## Q2 — Consent and questionnaire

A. One **Onboarding** component owns consent records and the questionnaire (definition and answers). Both are pre-analysis user inputs that change together with the onboarding specification (recommended)
B. Separate **Consent** and **Questionnaire** components
X. Other (please specify)

[Answer]: A

## Q3 — AI text and AI images

A. Two components. **Insight Generation** covers the narrative, closing recommendations and tiering (FR15). **Image Generation** covers the 11 feature images and 13 visuals (FR17, FR22). Each sits behind its own swappable vendor adapter (NFR2), because they change for different reasons: prompt/tone rules versus image vendor and cost (recommended)
B. One **AI** component for both
X. Other (please specify)

[Answer]: A

## Q4 — Report, Home and PDF

A. The **Report** component owns the report, its PDF and the Home Overview summary (Home is a view over the report). Image Generation serves AI Visuals reads directly (recommended)
B. A separate **Dashboard** component builds Home and Visuals read views
X. Other (please specify)

[Answer]: A

## Q5 — Journey status (where the user should land, FR26)

A. A read-only **Journey Status** component asks Identity, Onboarding, Photos, Payments, Orchestrator and Report for their state through their public interfaces. It owns no data (recommended: one place for the routing rule, and no component reaches into another's tables)
B. Identity computes it
X. Other (please specify)

[Answer]: A

## Q6 — Where files (photos, generated images, PDFs) are stored

A. A **Media Store** component owns stored-file metadata, encryption at rest and signed owner-only links. Photos, Image Generation and Report call it (recommended: one place for the privacy rules in NFR5.3)
B. Each component stores and serves its own files
X. Other (please specify)

[Answer]: A

## Q7 — How the analysis pipeline calls the other components

A. The **Analysis Orchestrator** runs each step inside the background worker by calling the other components' public interfaces directly. There is no event bus; the job and step tables provide retries and resume (recommended: simplest for a small team, and host-neutral)
B. Components publish domain events through a transactional outbox, and steps react to events
X. Other (please specify)

[Answer]: A

## Q8 — Frontend building blocks

A. Three components that mirror the client's required layering (`client_requirements.md` §5.3): **Web UI** (screens), **Auth Store** (Zustand, the single source of auth state) and **API Client** (the only HTTP caller, with refresh logic) (recommended)
B. One **Web App** component with the layering kept as internal folders
X. Other (please specify)

[Answer]: A

## Q9 — Shared cross-cutting code

A. A **Platform** component holds the typed settings, error envelope, request logging and rate limiter (it owns the rate-limit counters). Every backend component depends on it, and it depends on none (recommended)
B. Put these helpers inside each component
X. Other (please specify)

[Answer]: A

## Q10 — Technical decisions handed forward from earlier reviews

The stages that would normally decide some technical choices (NFR design, infrastructure) are skipped in this plan. Four choices came up in the story reviews: the background job runner model, the PDF engine, file encryption and signed links on local storage, and whether chat replies stream.

A. Record each as a **Proposed** ADR here, with alternatives, so implementation has a starting point. Each stays Proposed until confirmed when building starts (recommended)
B. Don't record them now; leave them to a later workflow
X. Other (please specify)

[Answer]: A

## Consolidated Summary Confirmation

Summary of answers:
- Face Analysis Engine is pure computation with no data; Photos stores landmarks and identity results, the Analysis Orchestrator stores the analysis result (Q1 A)
- One Onboarding component owns consent records and the questionnaire (Q2 A)
- Two AI components: Insight Generation (narrative, closing recommendations, tiering) and Image Generation (feature images and visuals), each behind its own vendor adapter (Q3 A)
- Report owns the report, its PDF and the Home Overview summary; Image Generation serves AI Visuals reads (Q4 A)
- A read-only Journey Status component computes where the user lands from other components' public interfaces (Q5 A)
- A Media Store component owns file metadata, encryption at rest and signed owner-only links (Q6 A)
- The Analysis Orchestrator calls components directly inside the background worker; job and step tables give retry and resume; no event bus (Q7 A)
- Frontend: three components, Web UI, Auth Store and API Client, mirroring the required layering (Q8 A)
- A Platform component holds settings, error envelope, request logging and rate limiter; everything depends on it, it depends on none (Q9 A)
- Job runner model, PDF engine, local encryption plus signed links, and chat streaming recorded as Proposed ADRs (Q10 A)

Does this all look correct before I generate the artifact?

- Looks correct
- Request changes

[Answer]: Looks correct
