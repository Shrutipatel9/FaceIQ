**Collaborator:** aidlc-developer-agent

## Contribution

Scope of this review: implementability, story sizing (target about 1–3 developer-days, Q3 A), dependency and critical-path correctness, per-story implementation phases (P-Data / P-API / P-UI / P-E2E), enabler sufficiency, and hidden technical dependencies. All size figures below are developer estimates **[Recommendation]**, not commitments (`CON-005`, `BR-007`). No new functional requirements are introduced. Every addition either implements an existing requirement ID or is labelled **[Recommendation]**.

### 1. Dependency and ordering defects (fix before approval)

These are correctness defects: as drafted, the story cannot meet its acceptance criteria when it is built in the stated order.

| # | Story | Defect | Proposed fix |
|---|-------|--------|--------------|
| D1 | US1.6 → US1.8 | AC1.6.1 requires `POST /auth/refresh`, but the refresh endpoint is built in US1.8 (phase 3), and US1.8 depends on US1.6. The order is inverted. | Build the refresh endpoint (rotation, CSRF header and Origin checks, cookie re-set) **before** US1.6: make US1.8 depend on US1.3, and US1.6 and US1.7 depend on US1.8. The mandated rules (`X-Requested-With` + Origin allow-list, FR4.7) then exist from the endpoint's first commit. |
| D2 | US1.3 / US1.8 | The refresh cookie is first set in US1.3, but its attributes (HttpOnly, Secure, SameSite, auth-scoped Path) are only added in US1.8 phase 3. The rotation lineage columns come in a second migration. | Move the cookie attributes into US1.3 phase 2 (Mandated: the cookie is correct from the first time it is set). Create `DATA-003` with its lineage columns (`family_id`, `parent_id`/`replaced_by`, `revoked_at`) in US1.3 P-Data, so US1.8 needs no schema change. |
| D3 | US9.5 | Depends only on US0.2, but AC9.5.2 and AC9.5.3 need the JWT service and the current-user dependency (US1.3). AC9.5.1 needs records from E2–E7. As written, the story "only makes sense in sequence" (inception rule). | Split it. **US9.5a** (depends on US1.3): the `get_current_user` dependency (fixed algorithm allow-list, `exp`, token-type claim) and ownership-scoped repository helpers, covering AC9.5.2 and AC9.5.3. **US9.5b**: each resource story (US2.2, US3.2, US4.2, US6.2, US6.4, US7.3, US7.4, US8.3) gets its own "user A cannot read user B's record" AC (`AUTH-007`, `AUTH-008`), so object-level authorization ships with each endpoint. A route-introspection test fails any owner-scoped route without that negative test. |
| D4 | US4.3 | The 402 matrix (AC4.3.1) targets analysis-start, report, visuals and chat endpoints that do not exist until E5–E7. | Keep US4.3 as the reusable `require_paid` dependency and a route-tag registry. A test enumerates every route tagged `paid` and asserts 402 for an unpaid user, so the matrix grows by itself. Add an explicit "unpaid → 402" AC to US5.1, US6.2, US6.4, US7.1, US7.3 and US7.4 (`FR-015`, `BR-001`). |
| D5 | US3.6 | AC3.6.2 (paid, not started) and AC3.6.3 (published report) need E4–E6 states, yet US3.6 sits on the critical path **before** US4.1. | Split it. **US3.6a** (critical path): routing for verification → consent → questionnaire → photos, covering AC3.6.1. **US3.6b** (after US6.1, off the critical path): payment → start → in progress → Home (AC3.6.2, AC3.6.3). US4.1 then depends on US3.4, not US3.6. |
| D6 | US3.4, US3.5, US7.1 | The review screen must show the user's own photos with the flagged one highlighted (AC3.4.4), and Home shows the before/potential image. Photos can only be served through owner-only short-lived signed links (US9.4, Forbidden: no public URLs). US9.4 is not a dependency of these stories. | Add US9.4 as a dependency of US3.4 and US7.1. Move US9.4 onto the critical path before US3.4, or fold "owner-only photo read via signed link" into US3.2. |
| D7 | US0.3 | AC0.3.3 (OpenAPI type-drift check) needs the backend OpenAPI schema, but US0.3 depends only on US0.1. | Make US0.3 depend on US0.2 as well, or move the type generation and drift check into US0.4, which already depends on both. |
| D8 | US2.1 | The server-side guard on the "questionnaire health section" (phase 2) cannot be enforced before US2.2 defines sections. The photo-upload endpoint does not exist until US3.2. | US2.1 builds a reusable `require_consent(type)` dependency and the consent records. The consent-gating ACs move to where the endpoints are created: the health-section guard in US2.2, the upload guard in US3.2 (FR7.3). AC2.1.1 stays as the UI gate. |
| D9 | US1.9 | Depends on US9.2, but the confirm-dialog component is built in US0.3. | Change the dependency to US0.3 (see §4 on US9.2). |
| D10 | US2.1 and all protected screens | Protected screens need the client-side route guard and session-restore flag (US1.6). US2.1 depends only on US1.3. | Add US1.6 (and therefore D1's US1.8) before US2.1 on the critical path. |
| D11 | US5.9 | Depends on US5.8 only for the image adapter. The hairstyle, outfit and aging images do not need the narrative. | Make US5.9 depend on US5.1 and the image adapter only, so it runs in parallel with US5.5–US5.8 (this also helps NFR3.1's 10-minute p95). |

### 2. Stories to split (estimated above 3 developer-days)

| Story | Why it is too big | Proposed split |
|-------|-------------------|----------------|
| US0.1 | Scaffolding both apps, pre-commit, lint/type/test CI, coverage floors (80% plus 90% per-module), Gitleaks, SAST, `pip-audit`/`pnpm audit`, Dependabot, SHA-pinned Actions, `permissions:` defaults, the requirement-traceability script, and two READMEs. About 4–5 days. | **US0.1a** scaffold + lint/format/type/test CI + READMEs. **US0.1b** security and supply-chain gates (Gitleaks, SAST, dependency audits, SHA pinning, `permissions: contents: read`, waiver file with expiry) + coverage floors, including the 90% module floors + the **requirement-traceability CI check** (team practice; currently in no AC). |
| US0.5 | Five interfaces, a factory, five fakes, a contract suite, *and* "real adapter skeletons" for OpenAI text, OpenAI image, Stripe, email and storage, plus encryption at rest. About 5+ days. "Skeletons" also conflicts with the construction guardrail against placeholder stubs. | **US0.5** keeps only the interfaces, the config-driven factory, the fakes and the contract-suite harness. Each **real** adapter moves into the first story that uses it: SMTP email → US1.2; storage (encrypted local disk, private) → US3.2a; Stripe → US4.1; OpenAI text → US5.5; OpenAI image (`gpt-image-1`, high input fidelity) → US5.8. Every story then ships a complete, working adapter with no stubs. |
| US2.2 | A data-driven branching engine, versioned definitions (FR8.3), server-side branch validation, a keyboard-accessible runner UI, and E2E. About 4 days. | **US2.2a** definition schema + versioning + answer/branch validation API. **US2.2b** questionnaire runner UI + E2E. Both use a **format fixture only**. Real questions come solely from the pending Onboarding Questionnaire Specification (OQ5; Forbidden: never invent requirements). |
| US3.2 | Hostile-upload hardening (magic bytes, decode, pixel cap, decompression bomb, polyglot), re-encode, EXIF/GPS strip, encrypted private storage, the per-angle UI with camera capture and fallback, and E2E. About 4–5 days. | **US3.2a** upload API + hardening + storage adapter + consent guard (D8) + per-user upload rate limit (§3 E-RL). **US3.2b** per-angle upload/capture UI + E2E. |
| US3.3 | Seven checks, including MediaPipe face count/proportion, occlusion and pose, behind an adapter, with tests. Occlusion and pose are algorithmically non-trivial and their thresholds are pending (`ASM-002`). About 4–6 days. | **US3.3a** MediaPipe/OpenCV adapter (built once, see §5) + readability, resolution, brightness, face count, face proportion. **US3.3b** occlusion + pose-match checks. Precede both with the CV spike (§3 E-CV). |
| US5.4 | Measurement definitions for seven mesh-based features plus side-photo ears. About 4 days. | **US5.4a** Eyes, Eyebrows, Nose, Lips + the `DATA-006` table. **US5.4b** Jaw, Chin, Cheeks + Ears (side photo with front fallback, AC5.4.2). |
| US5.7 | Five distinct assessments: dimorphism, prototypicality, thirds, symmetry with Regional Balance, and face-shape wireframe. About 4 days. Prototypicality needs a reference model that nothing yet provides (§5). | **US5.7a** symmetry (score, label, Regional Balance) + facial thirds. **US5.7b** dimorphism sliders + prototypicality + face-shape wireframe data. Content stays limited to the Report & UI Specification (FR16.2). |
| US6.4 | Choosing and integrating a PDF engine, a branded template covering intro, 11 sections, 22 before/after images, assessments and limitations, server-side retrieval of private images, caching, owner and payment gating. About 4 days. | **US6.4a** rendering pipeline + cache + owner-only + 402 (engine chosen by ADR, §5). **US6.4b** full branded template content (in-house branding, `CON-003`). |
| US1.3 (borderline) | OTP verify rules (tests first) + JWT issuance + refresh-session record + cookie + OTP UI + E2E. About 3–4 days once D2 moves cookie attributes in. | Optional: **US1.3a** OTP verify service and rules (tests first). **US1.3b** token service (JWT with token-type claim and algorithm allow-list, `DATA-003` with lineage, cookie attributes) + OTP screen + E2E. |

Net effect **[Recommendation]**: about 11 splits and 4 new enablers (§3), minus about 4 merges (§4), give roughly 65–67 stories. That is above the "roughly 45–60" in Q3 A. Q3's operative rule is the 1–3-day size, so this is a judgment call for the lead: either accept the higher count, or keep the larger stories and relabel them 3–5 days.

### 3. Missing enablers

| ID (proposed) | Enabler | Why it is needed | Consumers |
|---------------|---------|------------------|-----------|
| **E-RL** | Rate-limit primitive (PostgreSQL-backed counters, since no Redis is decided; injectable clock; per-account, per-IP and per-user keys) | Mandated: rate-limit unauthenticated auth endpoints per account and per IP (`AUTH-011`). NFR8 also requires per-user limits on analysis start and **photo upload**. Today only AC1.4.3 and AC5.1.4 mention limits. Signup, login, OTP verify, forgot-password and photo upload have **no AC**. | US1.2, US1.3, US1.4, US1.5, US1.10, US3.2a, US5.1, US7.6. Each needs an AC: "given the limit is exceeded, then the request is refused with 429". |
| **E-DEV** | Local dev environment: one documented command that starts PostgreSQL, a local SMTP catcher (Mailpit), the backend API, the job worker and the frontend. Matching CI service containers. | The walking skeleton's verification command must run locally with one command and no host (team practice, `NFR-010`). The E2E tests read OTPs from an SMTP catcher (project Testing Posture). No story delivers either. | US0.4, US1.2, US1.5, US1.10, and every P-E2E phase. Fold it into US0.2 phase 3 or make it a separate enabler. |
| **E-SEED** | E2E seeding CLI plus backend test-data factories (users, OTP, sessions, payments, photo sets, analysis states) that write to the database directly, never through a test-only HTTP endpoint | The INVEST notes rely on "seeded fixtures (for example a paid user with a consistent set)" for independent testability, but no story builds them. Factories are also listed in the team's testing tooling. | All E3–E8 E2E phases; US0.4 AC0.4.1 ("seeded verified user"). |
| **E-CV** | Time-boxed CV feasibility spike: MediaPipe Face Landmarker on the pinned Python version and OS, `num_faces` ≥ 2 for multi-face detection, pose (yaw) from landmarks, an occlusion signal, and landmark stability across the three angles for the 5-ratio identity signature (`ASM-010`) | Occlusion and pose detection have no specified algorithm (pending Photo Validation Specification, `ASM-002`). The 5-ratio signature compares 3/4 views, where perspective changes ratios. This is the highest technical risk on the critical path. | US3.3a, US3.3b, US3.4, US5.4. |
| **E-CFG** (small; may fold into US0.2) | Public-config endpoint exposing only non-secret values (required angle set, price and currency) | AC3.2.3 and AC4.1.3 require the frontend to reflect configuration with no code change. The frontend must not duplicate backend configuration or read it through `NEXT_PUBLIC_*`. | US3.2b, US4.1. |

Existing enablers that need additions:
- **US0.4 (walking skeleton)**: add an AC that proves CORS with credentials between two origins. A request from the allow-listed frontend origin with `credentials: include` succeeds, and a non-listed origin is refused (team Walking Skeleton explicitly lists "the two-app split and CORS with credentials"; `NFR-005`, `AUTH-013`). Also state explicitly that the skeleton stops at "OTP required" and does **not** issue or send an OTP. OTP issuance lands in US1.2/US1.5, so US0.4 needs no email adapter. Its `users` migration is the "first Alembic migration" the team practice names, so US0.2's "first empty migration" can be dropped.
- **US0.6 (job runner)**: the runner's process model is undecided. Given host neutrality and no queue broker, I recommend **[Recommendation]** a separate worker process that polls a PostgreSQL jobs table with `SELECT … FOR UPDATE SKIP LOCKED` and a lease/heartbeat, rather than in-process `BackgroundTasks`, which cannot survive a restart (AC0.6.3). Add ACs:
  - (a) two workers never claim the same job;
  - (b) a job whose worker died is reclaimed after the lease expires;
  - (c) each step has a configurable timeout.
  Record the choice in an ADR (Context / Decision / Consequences / Alternatives Rejected).
- **US0.2**: build the typed settings object **before** the database layer (the database URL comes from settings), so phase 2's settings step comes before phase 1. Move the request-duration logging from US9.6 here (AC0.2.3 and AC9.6.2 overlap).

### 4. Overlaps, duplicates and merges

| Item | Overlap | Recommendation |
|------|---------|----------------|
| US9.1, US9.2 vs US0.3 / each UI story | US0.3 builds the toast and confirm-dialog components. US9.1 and US9.2 then "apply conventions across screens", which cannot be completed on their own. | Turn US9.1 and US9.2 into Definition-of-Done items that every UI story's ACs cite (FR25.1, FR25.2, `BR-009`, `BR-010`). Keep AC9.1.2 (anti-enumeration toast wording) as an AC on US1.2, US1.5 and US1.10. If the lead keeps them as stories, they must be the last in E9 and depend on all UI stories. |
| US9.3 | "Every screen passes axe" is also cross-cutting. | Keep the per-route axe harness in US0.3 phase 3. Make "no axe violations on this screen" a DoD item per UI story (NFR6). US9.3 stays as the final manual keyboard pass (Should). |
| US9.6 vs US0.2 | Request timing and request ID are duplicated. | Timing goes to US0.2. US9.6 keeps only the load test (blocked on OQ7). |
| US3.3 phase 1 vs US5.4 phase 2 | Both build "the MediaPipe landmark extraction adapter (pinned version and model hash)". | Build it once in US3.3a. US5.4 reuses it. **[Recommendation]** Persist per-photo landmarks at validation time, so identity (US3.4) and measurement (US5.4) do not re-run inference. Landmarks and signatures are biometric data: never logged, owner-scoped. |
| US3.2 phase 2 vs AC3.3.3 | Rejecting unreadable or disguised files is specified in both. | The hostile-upload decode checks belong to US3.2a. AC3.3.3 moves there. US3.3 starts from an already-decoded image. |
| US0.4 login form vs US1.5 | Both build the login form. | US0.4 builds it; US1.5 extends it with the OTP step. Say so in US1.5's phases to avoid a rewrite. |
| US7.2 | The header nav is under 1 day. | Merge into US7.1, or keep as a tiny story. Note that a **Settings page shell** (Account / Password / Billing sections) is in no story. Add it to US8.1. |
| FR18.3 closing recommendations | US6.1 phase 2 says "AI closing recommendations" in the assembly step. That would be a second LLM call, and no story owns its prompt, schema or tests. | Put the closing recommendations into US5.5's structured-output schema (same call, validated with the 11 features, FR15.6). US6.1 only assembles. |

### 5. Hidden technical dependencies (should be surfaced to Units Generation, NFR Design and ADRs)

1. **MediaPipe on the server** (US3.3, US3.4, US5.4). The `mediapipe` wheels support a limited set of Python versions and OS/CPU platforms. Pin a Python version that has wheels for the dev OS (Windows here), CI Linux x86_64, and the future host. Fetch the `face_landmarker.task` model file by pinned URL with a SHA-256 check in setup (or commit it with its licence). Inference is CPU-bound: run it off the async event loop (thread or process pool) so upload validation does not block other requests. Use `opencv-python-headless` (no GUI system libraries in CI).
2. **Image formats from phones** (US3.2). iPhone file uploads are often HEIC, which Pillow cannot decode without `pillow-heif`. Decide the accepted formats (JPEG / PNG / WebP, HEIC yes or no). Otherwise P1's first upload fails as "unreadable". Camera capture (`getUserMedia`) needs HTTPS or `localhost`; the documented fallback is the file input.
3. **PDF rendering** (US6.4). Engine choice drives system dependencies. WeasyPrint needs Pango/Cairo (painful on Windows dev machines). Headless Chromium or Playwright printing is heavy but reuses HTML/CSS. ReportLab is pure Python but uses programmatic layout. With 22 embedded images, first-download generation may take seconds: set a timeout, or generate through the job runner. ADR required.
4. **Email provider** (US1.2; OQ6). Build one provider-neutral **SMTP adapter**. It works with Mailpit locally and in CI, and with most providers (SES, Postmark, SendGrid SMTP), so OQ6 does not block Construction. Note that SendGrid *notifications* stay out of scope. Only OTP mail is in scope.
5. **Object storage** (US0.5, US3.2a, US9.4). No host means the real adapter is local disk. "Encrypted at rest" (NFR5.3) then means application-level encryption (for example AES-GCM with a key from the settings or secrets manager, never in code), which needs a key-rotation note. "Signed links" on local disk require a backend file-serving endpoint that verifies an HMAC-signed, expiring, owner-bound token. `<img>` tags cannot carry the in-memory Bearer token, which is exactly why signed links are needed. The signed token in the URL must be scrubbed from access logs (Forbidden: never log tokens).
6. **OpenAI image volume vs NFR3.1** (US5.8, US5.9). Each `gpt-image-1` edit can take tens of seconds. 24 sequential edits risk the 10-minute p95. Needs:
   - bounded concurrency inside the image steps;
   - **per-image idempotent sub-steps**, so "Try again" regenerates only the missing images and does not rerun all 11 (cost, `BR-006`); add an AC to US5.8 and US5.9;
   - handling of account-tier rate limits (429 → bounded backoff).
   Per-feature edit prompts ("one attribute changes") have no defined source yet. Flag this against the Report & UI Specification or US5.5's output.
7. **Stripe** (US4.1, US4.2). The FastAPI webhook must read the **raw** body (`await request.body()`) before any JSON parsing, for signature verification. Document the Stripe CLI (`stripe listen`) for manual local testing. CI uses signed fixtures only.
   - Missing ACs in US4.1:
     - A user with a `paid` payment cannot create a second checkout session (charged once, FR12).
     - Repeated "Pay" clicks reuse or expire the open session instead of creating parallel ones.
   - Missing in US4.2: a `GET` payment-status endpoint for the polling return page (AC4.2.4), and a P-E2E phase (stubbed session + signed fixture webhook, per team practice).
8. **Prototypicality reference** (US5.7b). A prototypicality score needs a reference ("average") face model or dataset. None is specified or licensed. This is a knowledge gap for the Report & UI Specification or the client, and it must not be invented (Forbidden).
9. **Frontend libraries.** A radar and slider chart library (US6.3, US7.1) with text alternatives. MSW and fake timers (US1.7). Playwright browsers cached in CI. None of these is blocking; list them in US0.3.
10. **Chat transport** (US7.4). Streaming (SSE) would need streaming support in the one API client (`lib/api/client.ts`). **[Recommendation]** Use non-streaming replies in this scope unless the Report & UI Specification requires streaming.

### 6. Per-story phase completeness

- **US4.2**: add P-E2E (return page shows "confirming", then "paid" after the signed fixture webhook).
- **US5.1**: state where the pipeline's step list is registered. **[Recommendation]** The pipeline definition is created in US5.1 with an ordered step registry. US5.4–US6.1 each register their step, so the pipeline is runnable, with fewer steps, at every merge.
- **US5.2** AC5.2.1 lists four steps. The runner also has visuals and assembly steps (US5.9, US6.1). Align the progress labels with the registered steps.
- **US5.8** AC5.8.3 ("when regeneration is requested by the user, then no regeneration is available") cannot be tested as a behaviour. Rephrase it: "Given images already exist for a report, when the image step runs again (resume or retry), then no existing image is regenerated and no regeneration endpoint exists."
- **US6.4**: add a 402 AC for unpaid users (D4) alongside the existing owner AC.
- **Docs**: the team practice requires READMEs and docstrings to stay current. **[Recommendation]** Make "README and config keys updated" a DoD item rather than a P-E2E sub-bullet in only some stories.

### 7. Revised critical path (proposed)

US0.1a → US0.2 (+ E-DEV, E-CFG) → US0.3 → US0.4 → US1.2 → US1.3 → US1.8 → US1.6 → US2.1 → US2.2a → US2.2b → US2.3 → US3.1 → US3.2a → E-CV → US3.3a → US3.3b → US9.4 → US3.4 → US4.1 → US4.2 → US4.3 → (US0.6 must be done) → US5.1 → US5.4a → US5.5 → [US5.6 ∥ US5.7a/b ∥ US5.8] → US6.1 → US6.2 → US7.1

In parallel, off the path:
- US0.1b, US0.5, E-RL (E-RL must land before the US1.x auth endpoints ship), E-SEED;
- US1.4, US1.5, US1.7, US1.9, US1.10, US9.5a;
- US3.5, US3.6a;
- US5.2, US5.3, US5.4b, US5.9 (from US5.1);
- US3.6b (after US6.1), E8, the chat stories (US7.4–US7.6), US6.3, US6.4a/b, US7.3.

Two open questions for the lead (behaviour is unspecified; either answer is legitimate):
- **Unverified accounts.** For a signup with an email that exists but is unverified, and for a login to an unverified account: does the system re-issue a verification OTP, or treat it like FR2.4's existing-email case? This affects US1.2 and US1.5.
- **Other sessions after a password change.** Should a password change revoke the user's *other* sessions? FR6.2 only protects the current one. This affects US8.2.

## Positions
- AGREE: Epic structure in WF-001 order with E0 enablers — matches Q2 A and Q6 A and gives Units Generation clean seams.
- AGREE: Walking skeleton = the email + password login step through the full auth layering — matches the affirmed team practice. It needs only the CORS-with-credentials AC added (§3).
- AGREE: "Tests first" markings on OTP rules, token rotation and reuse, reset_token, the 402 gate, the identity vote and tiering — consistent with the affirmed custom Testing Posture ordering.
- AGREE: Webhook never starts analysis; success redirect is never proof of payment (US4.2) — correctly mirrors the Forbidden rules.
- OBJECT: US1.6 ordered before US1.8 while needing its refresh endpoint — the order is inverted (D1). Refresh with CSRF must precede session restore.
- OBJECT: US3.6, US4.3 and US9.5 as drafted — each asserts behaviour of endpoints or states that do not yet exist when it is built (D3–D5). This violates "stories must be independently testable".
- OBJECT: US0.5 "real adapter skeletons" — too large, and conflicts with the construction guardrail against placeholder stubs. Real adapters should ship with their first consuming story.
- OBJECT: Missing rate-limit enabler and ACs — the Mandated per-account/per-IP limits on signup, login, OTP verify and forgot-password, and the NFR8 photo-upload limit, have no acceptance criterion.
- OBJECT: Sizes of US0.1, US2.2, US3.2, US3.3, US5.4, US5.7 and US6.4 — each is estimated above the 3-day ceiling agreed in Q3 A and should be split as in §2.
- OBJECT: US9.1, US9.2 and US9.3 as standalone stories — they are cross-cutting conventions better expressed as Definition-of-Done items cited by each UI story. US9.2 duplicates US0.3.
