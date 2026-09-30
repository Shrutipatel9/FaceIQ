# Project-Level Rules

> Project-specific specialisation and corrections. Loaded after `org.md` and
> `team.md` as strict-additive guidance; contradictions with broader policy
> are rejected. Populated by practices-discovery and the self-learning loop.
>
> Use sparingly: most teams don't need a project layer. Reach for it
> only when this specific project needs stable, durable guidance beyond the
> team practice (for example, package-specific release checks or an additional
> regression suite for a legacy component).

## Way of Working

<!-- Project-specific specialisation. Example: -->
<!-- This monorepo requires package-scoped branch names and a package owner -->
<!-- review in addition to the team's normal merge policy. -->

## Walking Skeleton

<!-- Project-specific specialisation. Example: -->
<!-- The walking skeleton must exercise the legacy service adapter as well -->
<!-- as the new service boundary. -->

## Testing Posture

<!-- Project-specific specialisation. -->

- End-to-end tests read emailed OTPs from a local SMTP catcher, never from a test-only inbox endpoint in the application. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:practices-discovery:a74cbf9de0da0dc6c836e2ad0c81d7327817429372d9108bb5babcfda25496f6 -->

- App-wide status conventions: a cross-user request for another user's resource returns 404 (no enumeration), rate-limited requests return 429 rate_limited, unpaid requests to gated routes return 402; every story's tests assert these same codes. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:user-stories:79a86e094bc79fa3dd781a472c0198f5267ecaea0c8a031ed9fa849b10261842 -->

## Guard Policy

<!-- Project-specific. Mode: strict, relaxed, or off. Strict here holds for every intent and cannot be changed from chat. A section under the retired Change Control heading, written by an earlier release, is still read. -->

## Deployment

<!-- Project-specific specialisation. -->

## Code Style

<!-- Project-specific specialisation. -->

## Tech Stack

<!-- Technology choices locked for this project. -->

## Decided

<!-- Decisions made in earlier stages that should not be re-asked. -->
<!-- Format: DECIDED: [decision] (Stage [slug], [date]) -->

## Scope Overrides

<!-- Custom scope rules for this project. -->

## Forbidden

<!-- Populated by practices-discovery affirmation gate. -->
<!-- Format: NEVER [behavior] (affirmed [date]) -->
<!-- Example: NEVER throw exceptions across service layer boundaries (affirmed 2026-05-17) -->

- NEVER use a third-party authentication service (Supabase Auth, Auth0, Firebase Auth or similar) as the application's authentication mechanism. (`AUTH-001`, `AUTH-005`, `CON-002`) (affirmed 2026-09-29)

- NEVER use Supabase in any role. (`NFR-002`) (affirmed 2026-09-29)

- NEVER store the refresh token in Zustand, `localStorage` or `sessionStorage`, and NEVER use Zustand `persist` for the auth store. (`FE-002`, `ASM-001`) (affirmed 2026-09-29)

- NEVER store an OTP in plaintext; OTPs are stored hashed. (`AUTH-011`) (affirmed 2026-09-29)

- NEVER let UI components call auth APIs or read or handle tokens directly. (`FE-007`) (affirmed 2026-09-29)

- NEVER call `fetch` or any other HTTP API directly in the frontend outside the centralized API client module. (`FE-004`, `FE-007`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER put token, OTP, payment-gate or identity-check logic in FastAPI route handlers; that logic lives in services. (`client_requirements.md` §5.3 for JWT/OTP; extension to payment and identity [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER use a wildcard CORS origin. (`AUTH-013`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER treat the Stripe Checkout success redirect as proof of payment. (`FR-015`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER start the analysis pipeline from the Stripe webhook, and NEVER run the AI pipeline on photos that failed validation. (`FR-015`, `BR-004`) (affirmed 2026-09-29)

- NEVER hardcode credentials, API keys or secrets. Also NEVER hardcode the report price, the currency, or AI vendor or model identifiers outside the typed settings object (a configuration-overridable non-secret default inside it is permitted, see Mandated). (`FR-016`, `NFR-008`, `NFR-013`, construction-phase Security guardrail) (affirmed 2026-09-29)

- NEVER expose a server secret through a `NEXT_PUBLIC_*` variable or any frontend bundle. (`NFR-008`, `NFR-013`, `FR-016`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER log or send any of the following to telemetry or error trackers: passwords, OTPs, access or refresh tokens, `reset_token` values, Stripe signatures, raw photos, landmark identity signatures, or questionnaire medical, medication and allergy answers. (`AUTH-010`, `AUTH-011`, `AUTH-015`, `ASM-010`, `FR-008`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER serve user photos or generated images from public-readable storage or URLs; access goes through an authenticated endpoint or short-lived signed URLs. (`DATA-005`, `DATA-009`, `DATA-010`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER use real user photos as test fixtures, and NEVER commit real face photos (or any file derived from a real person's face) to the repository. Face-image fixtures are licensed or consented only, and most CV tests use landmark-coordinate fixtures. (`FR-007`, `ASM-010`; [Recommendation — affirmed Q6, Q10]) (affirmed 2026-09-29)

- NEVER let the per-push or per-PR test suite call live OpenAI, Stripe live mode, or a real email provider. (`BR-006`, `NFR-008`, `NFR-009`, `NFR-012`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- NEVER disable or weaken a security gate or coverage floor, and NEVER add a waiver without an expiry, to make CI pass. (`org.md` Testing Posture "may not be weakened to make a step pass"; [Recommendation — affirmed Q5, Q9, Q10]) (affirmed 2026-09-29)

- NEVER assume a hosting provider (including Replit) in any design, configuration or deployment artifact. (`NFR-010`, `CON-007`) (affirmed 2026-09-29)

- NEVER build out-of-scope items without an explicit client scope change. Out-of-scope items are the admin panel and review workflow, SendGrid email notifications, Meta Pixel / Google Tag Manager, and a working PayPal integration. (`BC-006`, `client_requirements.md` §2.2) (affirmed 2026-09-29)

- NEVER invent requirements that `client_requirements.md` does not support. (`client_requirements.md` "How to use this document" item 2) (affirmed 2026-09-29)

## Mandated

<!-- Populated by practices-discovery affirmation gate. -->
<!-- Format: ALWAYS [behavior] (affirmed [date]) -->
<!-- Example: ALWAYS use Result<T,E> for fallible operations in service layer (affirmed 2026-05-17) -->

- ALWAYS build the backend (Python + FastAPI) and the frontend (Next.js + TypeScript) as two separate applications that communicate over HTTP/CORS. (`NFR-001`, `NFR-005`) (affirmed 2026-09-29)

- ALWAYS use PostgreSQL through SQLAlchemy, and deliver every schema change as an Alembic migration. (`NFR-002`, `NFR-003`, `NFR-004`) (affirmed 2026-09-29)

- ALWAYS manage frontend authentication state in one centralized Zustand auth store, and attach the access token through one centralized API client. (`FE-001`, `FE-003`, `FE-004`, `NFR-006`) (affirmed 2026-09-29)

- ALWAYS preserve the layering `UI → Zustand Auth Store → API Client → Backend Auth APIs → JWT/OTP Services` in design and code. (`client_requirements.md` §5.3, `FE-007`) (affirmed 2026-09-29)

- ALWAYS keep the refresh token only in an httpOnly cookie that the backend sets, rotates and clears, and keep the access token only in memory. (`FE-002`, `ASM-001`) (affirmed 2026-09-29)

- ALWAYS set the refresh cookie `HttpOnly` and `Secure`, give it an explicit `SameSite` value, and scope its `Path` to the auth routes. (`FE-002`, `AUTH-013`; attribute set [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS validate the JWT access token on every protected backend endpoint. (`AUTH-007`) (affirmed 2026-09-29)

- ALWAYS verify JWTs against a fixed algorithm allow-list and check `exp` and a token-type claim, so that a `reset_token` or refresh token is never accepted as a Bearer access token. (`AUTH-004`, `AUTH-007`, `AUTH-015`; mechanism [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS store refresh tokens by token identifier (never the raw token) so sessions are revocable server-side, and when an already-rotated refresh token is presented, revoke its whole token family. (`AUTH-010`, `AUTH-012`) (affirmed 2026-09-29)

- ALWAYS require the `X-Requested-With` header and an Origin/Referer allow-list match on cookie-authenticated endpoints (`/auth/refresh`, `/auth/logout`). (`AUTH-013`) (affirmed 2026-09-29)

- ALWAYS configure CORS with an explicit origin allow-list read from configuration. (`NFR-005`, `AUTH-013`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS hash passwords with a memory-hard password KDF: Argon2id preferred, bcrypt acceptable. (`WF-002`; algorithm [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS rate-limit the unauthenticated auth endpoints (signup, login, OTP verify and resend, forgot-password) per account and per IP, on top of the OTP attempt and lockout limits. (`AUTH-011`; login and IP limits [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS enforce object-level authorization, so that every read or write of `DATA-004`–`DATA-011` is scoped to the authenticated user's own records. (`AUTH-007`, `AUTH-008`; [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS enforce the payment gate (HTTP 402) and the photo identity check (HTTP 409 at checkout and at analysis start) on the server, never only in the UI. (`FR-015`, `BR-001`, `BR-005`) (affirmed 2026-09-29)

- ALWAYS verify the `Stripe-Signature` header against the raw request body with the webhook signing secret, and process each Stripe event ID idempotently. (`FR-015`, `NFR-009`; mechanism [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS validate uploaded photos by their decoded content (magic bytes plus a successful decode) within size and pixel-count limits, then re-encode them and strip EXIF metadata (including GPS) before storage or AI processing. (`FR-006`, `BR-005`; EXIF strip and re-encode [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS render AI-generated text (narrative, recommendations, chat replies) as text, never as HTML, and ship a Content-Security-Policy on the frontend. (`ASM-001`, `FR-008`, `FR-019`) (affirmed 2026-09-29)

- ALWAYS read the following through the typed settings object, so they can change by configuration without a code change: AI vendor settings (base URL, API key, model, image-generation vendor), the report price and currency, and the required photo-angle set. A non-secret default declared in that settings object (for example a default model name) is allowed as long as configuration overrides it and `.env.example` documents it. Secrets have no in-code default. (`NFR-008`, `NFR-013`, `FR-016`, `ASM-005`) (affirmed 2026-09-29)

- ALWAYS map every `FR-*`, `AUTH-*` and `BR-*` requirement to at least one test case, checked in CI through requirement-ID test tags. (`client_requirements.md` §12; tagging mechanism [Recommendation — affirmed Q10]) (affirmed 2026-09-29)

- ALWAYS cite the `client_requirements.md` requirement IDs that a derived document or change implements, and keep non-client-stated content labelled `[Recommendation]`, `[Assumption]` or `[Decided by delivery team]`. (`client_requirements.md` "How to use this document" items 2 and 4, §12) (affirmed 2026-09-29)

- ALWAYS raise a conflict with `client_requirements.md` to the human instead of resolving it silently. (`client_requirements.md` "How to use this document" item 3) (affirmed 2026-09-29)

- ALWAYS ship each application with a README covering setup, run, test and configuration keys, and document its public modules, services and API handlers. (`BC-005`, `NFR-011`) (affirmed 2026-09-29)

## Corrections

<!-- Project-specific corrections from human feedback. -->
<!-- Format: NEVER/ALWAYS [behavior] (learned [date]) -->
- In a planning-only workflow that skips Construction, record the walking-skeleton choice as guidance for the later build workflow rather than running a skeleton ceremony. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:practices-discovery:36f9946c144c19db2a407bbf8f100eae393eafac553996e44adb4923a112e46c -->
- Treat client_requirements.md as the single source of truth: derive project rules and every downstream document from it, cite its requirement IDs, and use org.md only as suggested defaults. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:practices-discovery:ffa409e1436a70196aa52eb34101e696ffdc01e1c035dc03bf899cb0bc1dc02d -->
- Keep product and business behaviour rules (e.g. FR-008(f), FR-019, BR-008..BR-012) out of project-wide engineering rules; they belong in requirements and design documents. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:practices-discovery:795b57ae8580b57e24f64a90236558681febb28661e791208e064b39bfdf1593 -->
- Until a hosting provider is chosen (NFR-010, CON-007), the project has no deployment jobs: build and push to GitHub only, with GitHub Actions running blocking checks on every push and pull request. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:practices-discovery:1b694c54049981c218fc03b346ac5a536cb00dd7484ceaef927e86e5d47b7cec -->
- The walking skeleton for this project is the auth slice: the email + password login step through UI -> Zustand Auth Store -> API Client -> Backend Auth APIs -> JWT/OTP Services, chosen over a lighter infra slice to de-risk the most constrained path. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:practices-discovery:2a3dec485cbe00fec019e7e7f4acab04e9d92a01876b4a7dd048457b1edf9af3 -->
- Derived requirements use stable FR{n}/NFR{n} keys with a traceability table back to client_requirements.md IDs; client IDs remain the primary citation so client-document changes can be traced to affected requirements. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:requirements-analysis:5bd167290898db25333fdff2f237dc62fefe61f56b168b874f5e207ebdd9f8a7 -->
- When a story is split, give each part its own sequential USx.y ID within the epic (no a/b suffixes), because traceability checks only recognise USx.y IDs. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:user-stories:7cea85c4af2a84e56797a4eefe8136a697685ccf0bc860b7a4723c856905f83a -->
- Cross-cutting UX rules (toasts, confirmation dialogs, per-screen accessibility checks) are Definition of Done items for every UI story rather than standalone stories; keep one final app-wide accessibility audit story. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:user-stories:f9789c1583c332a7a521b2c898a67d5a22d89f1ad5f5ab13a10c9baa94c900cc -->
- Real vendor adapters (email, storage, Stripe, OpenAI text/image) ship with the first story that uses them, never as placeholder skeletons; keep stories at 1-3 developer-days even if the story count grows. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:user-stories:09931cca1dc49ed67fe907453373603cfdf88f2b7a94c0c49cd3222bf31cd25c -->
- The frontend is three components mirroring the mandated layering (Web UI, Zustand Auth Store, API Client); the Auth Store injects a token provider and session-ended callback into the API Client so the API Client depends on no frontend component. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:domain-design:f788e06ad19e9bf463222a82cc937e8bfcc73772a7e0c2cb90443cf5e77ec9f2 -->
- When NFR or infrastructure design stages are skipped, record the needed technical choices as Proposed ADRs with alternatives, to be confirmed when building starts. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:domain-design:835c270810fc589e9305fb2952a95261105d0622c6d8ec7b5b67d691346354fa -->
- When grouping stories into units creates a unit-level dependency cycle, move the offending story into the unit that first needs it (for example the CV feasibility spike belongs with photo validation) rather than adding back-edges. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:units-generation:8df37c5b9f902efad5c07207e9e3e0e3ef4b7639c85473872631a886346663f0 -->
- Signed media links (photos, generated images, PDFs) are the one documented exception to 'only the API client calls the backend': the browser opens them directly because img tags cannot carry the in-memory Bearer token, but the URL itself is always obtained through the API client. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:contract-design:89dacdbe4c823e9b8d7d82044042a5ecc6cf3139a62a89f1ee847ab7e1e82d05 -->
- API fields for content still pending a client specification (Report & UI, Onboarding Questionnaire) are modelled as extensible objects so filling them in later is an additive, non-breaking change. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:contract-design:c8cc6b38231e77c39ac1748d930180fa7c16cafcb60067d9ddd4f8da0039af0c -->
- Exploratory spikes that merge no production code may run ahead of their unit to pull risk forward (e.g. CV feasibility, AI-image quality/cost) without breaking the unit dependency order; their formal write-up lands with the owning unit. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:delivery-planning:fe4ab1983a67d0004bd932c3739387c9235936a84f63f9cbb92340a4b201723f -->
- In a planning-only workflow, write proposed construction settings (iteration order, execution, verification command, staffing) into the Bolt plan for the build workflow to confirm, rather than recording them in workflow state. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:delivery-planning:525db5829a04b0ccaa64ad698df0ac33882215369c33d0fb0283e92ea7a3569a -->
- Plan one Bolt per unit and split only XL units into two Bolts, keeping security-critical pull requests small without fragmenting the plan. (learned 2026-09-29) <!-- cid:260929-faceiq-implementation-pl:delivery-planning:ea20b6d2023475ff88eea489dc421f177a9e39efb8ec99880f3422c06d09772e -->
