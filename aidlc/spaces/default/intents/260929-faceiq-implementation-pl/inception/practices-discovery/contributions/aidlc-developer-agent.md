**Collaborator:** aidlc-developer-agent

## Contribution

Scope of this review: naming, layer boundaries, error handling, file and folder
organization for the two applications, code-style conventions, and maintainability
for the client team that will continue the work with AI-assisted tools (`BC-005`,
`NFR-011`). The review covers the lead drafts `team-practices.md`, `discovered-rules.md`
and `evidence.md`, checked against `client_requirements.md` and `org.md`. The project
is greenfield (`CON-001`), so every item below is a **[Recommendation]** unless it
cites a client-stated ID. Nothing here is affirmed.

### 1. Repository and folder organization (additions for `## Way of Working` / `## Code Style`)

**[Recommendation]** One repository, two top-level applications. This agrees with the
lead. Proposed layout (names are open to the interview):

```
FaceIQ/
  backend/                  # Python + FastAPI app (NFR-001)
    pyproject.toml          # deps, Ruff, mypy, pytest config
    alembic.ini
    alembic/versions/       # one migration per schema change (NFR-003)
    app/
      main.py               # app factory: routers, CORS, exception handlers
      core/                 # settings (pydantic-settings), logging, errors, security primitives
      api/
        deps.py             # FastAPI dependencies (DB session, current_user, JWT check)
        routes/             # one module per resource: auth.py, photos.py, payments.py, reports.py ...
      schemas/              # Pydantic request/response models (API boundary DTOs)
      services/             # business logic: token_service, otp_service, photo_validation_service ...
      repositories/         # SQLAlchemy queries, one per aggregate
      models/               # SQLAlchemy ORM models (DATA-*)
      integrations/         # vendor adapters behind interfaces: llm/, image_gen/, payments/, email/
      cv/                   # MediaPipe + OpenCV measurement and identity signature (FR-007, ASM-010)
    tests/
      unit/  integration/  api/   # mirror app/ structure
    .env.example
    README.md
  frontend/                 # Next.js + TypeScript app (NFR-005)
    package.json  tsconfig.json  .eslintrc / eslint.config  .prettierrc
    src/
      app/                  # routes (App Router, if chosen)
      components/           # presentational UI; ui/ for shared primitives (toast, confirm dialog)
      features/<feature>/   # feature-scoped components + hooks (onboarding, photos, report, chat ...)
      stores/auth-store.ts  # the ONLY auth store (FE-001, FE-003)
      lib/api/client.ts     # the ONLY place that calls fetch (FE-004, FE-005)
      lib/api/<resource>.ts # typed endpoint functions (auth.ts, photos.ts, ...)
      types/                # shared types; generated API types (see section 5)
    tests/ or colocated *.test.ts(x);  e2e/ for Playwright
    .env.example
    README.md
  docs/adr/                 # ADRs (Context, Decision, Consequences, Alternatives Rejected)
  .github/workflows/        # one CI job per app, path-filtered
  README.md                 # how the two apps fit together, local run of both
```

Rationale: each app keeps its own toolchain and can be built and tested alone. The
`backend/app/` split matches the §5.3 wording that "JWT/OTP Services" are
"backend-internal services ... decoupled from route handlers". The single
`auth-store.ts` and single `client.ts` files make the frontend half of §5.3 visible in
the file tree, which helps AI tools infer where new code belongs.

### 2. Layer boundaries — making them enforceable, not just written down

The lead records the §5.3 layering (`UI → Zustand Auth Store → API Client → Backend
Auth APIs → JWT/OTP Services`) as a Code Style statement and as a Mandated candidate.
I agree with that. I propose three additions:

**2a. [Recommendation] Backend layering applies to every feature, not only auth.** The
client states separation only for auth (§5.3). Applying the same shape everywhere keeps
the codebase uniform for the client team (`NFR-011`):

- `api/routes/*` (thin handlers): parse the request through a Pydantic schema, call
  one service, map the result to a response schema. No SQLAlchemy queries, no business
  rules, no vendor SDK calls.
- `services/*`: business rules and orchestration (payment gate `FR-015`, identity
  vote `BR-005`, OTP limits `AUTH-011`, token rotation and reuse detection
  `AUTH-012`). Services never import `fastapi` (no `Request`, `Response`,
  `HTTPException`); they raise domain errors (section 3).
- `repositories/*`: the only layer that builds SQLAlchemy queries.
- `integrations/*`: the only layer that imports vendor SDKs (`openai`, `stripe`, the
  email provider). Each vendor sits behind a small interface (`typing.Protocol`),
  and a factory in `core/` or `integrations/` picks it from settings (`NFR-008`,
  `NFR-013`).
- `cv/*`: pure functions over images and landmarks, with no DB or HTTP access, so they
  can be tested with golden fixtures.
- Direction: `routes → services → (repositories | integrations | cv) → models`. Lower
  layers never import higher layers.

**2b. [Recommendation] Enforce the boundaries in CI with a linter.**
- Backend: `import-linter` contracts in `pyproject.toml`: a layers contract for the
  direction above, plus "`app.services` may not import `fastapi`" and "only
  `app.integrations` may import `openai` / `stripe`".
- Frontend: ESLint `no-restricted-imports` so that `src/components/**`,
  `src/features/**` and `src/app/**` cannot import `@/lib/api/client` or
  `@/lib/api/auth` (they call auth-store actions such as `login()` and `logout()`
  instead, per `FE-007`). Add `no-restricted-globals` / `no-restricted-syntax` to ban
  raw `fetch` outside `src/lib/api/**` (`FE-004`). Also ban importing `zustand/middleware`
  `persist` in `stores/auth-store.ts` (`FE-002`).
- Why this matters for `BC-005`: AI coding tools often take the shortest path, for
  example calling `fetch` directly from a component. A failing lint rule catches that
  on the PR. Prose alone does not.

**2c. [Recommendation] Next.js-specific boundary.** The access token lives only in
memory (`FE-002`), and the backend sets the refresh cookie on its own origin (§5.3:
"The Next.js frontend calls these directly from the browser"). Because of that:
- Next.js `middleware.ts`, Server Components and Route Handlers cannot see auth state.
  Protected-route gating is a client-side guard that waits for the auth store's
  `initializing` flag (`FE-002`).
- No Next.js API route may proxy or touch auth traffic, because that would add an
  extra layer §5.3 does not list.
- This needs architect and devsecops confirmation (cookie `SameSite` / domain across
  two origins, CORS `allow_credentials`). It is flagged here because it decides which
  folder the guard code goes in.

### 3. Error handling (missing from the lead draft — proposed addition to `## Code Style`)

The construction guardrail requires boundary error handling and forbids silent
failures. The drafts have no error-handling convention. Proposal:

- **[Recommendation] One domain-error hierarchy in the backend** (`app/core/errors.py`):
  a base `AppError` with a stable machine-readable `code` and a safe `message`, and
  subclasses mapped to HTTP **only** in exception handlers registered in `main.py`:
  - `AuthenticationError` → 401
  - `PermissionDeniedError` → 403 (`AUTH-008`)
  - `PaymentRequiredError` → 402 (`FR-015`, `BR-001`)
  - `IdentityMismatchError` → 409, carrying the list of mismatched angles (`BR-005` items 4 and 8)
  - `ValidationFailedError` → 422, carrying a per-photo reason (`BR-005` "clear reason")
  - `RateLimitedError` / lockout → status to be decided (see open question 9)
- **[Recommendation] One JSON error envelope for every non-2xx response**, for example
  `{"error": {"code": "payment_required", "message": "...", "details": {...}}}`. The
  frontend toast layer (`BR-009`) maps `code` to user copy. It must not parse English
  messages.
- **[Recommendation] Integration boundary:** every vendor call (OpenAI text and image,
  Stripe, email) has an explicit timeout and wraps vendor exceptions in an
  `IntegrationError` subclass. Transient failures (timeouts, 429, 5xx) may retry with
  a bounded backoff. Auth and validation failures fail fast. Stripe webhook handlers
  verify the signature and are idempotent by event id.
- **[Recommendation] Logging:** unexpected exceptions are logged once, at the handler,
  with a request id. Never log passwords, OTPs, access or refresh tokens, `reset_token`,
  raw photos, or questionnaire medical answers (`AUTH-011`, `AUTH-015`, `FR-008`(f)).
  Responses never include stack traces.
- **[Recommendation] Frontend:** `lib/api/client.ts` turns every failure into one typed
  `ApiError { status, code, message, details }`. Only a 401 from `/auth/refresh`, or an
  explicit logout, clears auth (`FE-006`). Network errors and 5xx surface as a toast and
  keep the session. Components never use empty `catch {}` blocks.

### 4. Naming conventions (additions to `## Code Style`)

The org default is language-idiomatic naming. Proposed concrete forms:

- **Python:** modules and functions `snake_case`; classes `PascalCase`; suffix by role:
  `*_service.py`, `*_repository.py`; route modules named after the resource
  (`routes/auth.py`); Pydantic models `XxxRequest` / `XxxResponse`; ORM models are
  singular (`User`, `OtpRecord`, `RefreshSession`) with plural `snake_case` table names.
  Alembic revision messages are imperative and cite the `DATA-*` ID
  (`add_otp_records_table (DATA-002)`).
- **TypeScript:** component files `PascalCase.tsx` (`LoginForm.tsx`); other files
  `kebab-case.ts` (`auth-store.ts`, `api-client.ts`); hooks `useXxx`; the store hook is
  `useAuthStore`; types and interfaces `PascalCase`, with no `I` prefix.
- **API paths:** lower-case, plural resources, kebab-case where needed; auth paths as
  the client already names them (`/auth/refresh`, `/auth/me`, `/auth/change-password`).
  The API version prefix (`/api/v1`) is open.
- **Gap: JSON field casing across the boundary.** Python is `snake_case` and TS is
  `camelCase`. Options: (A) the backend emits `camelCase` through a Pydantic alias
  generator, or (B) the frontend consumes `snake_case` as-is. A mix causes steady bugs.
  This needs a decision, here or in contract design.
- **Gap: product and package name.** The repository is `FaceIQ` but the client document
  says "AI Facial Analysis Platform". Choose one identifier for the Python package,
  the npm `name` field, log prefixes and cookie names.

### 5. Maintainability for AI-assisted continuation (`BC-005`, `NFR-011`)

The lead's README / docstring / ADR proposal is good. Additions:

- **[Recommendation] `.env.example` per app**, listing every configuration key the lead
  names (DB URL, OpenAI base URL / key / model, image vendor, Stripe keys, email
  provider, price and currency, CORS origins, photo-angle set, token and OTP lifetimes)
  with placeholders only. All env access goes through one typed `Settings` object in
  the backend and one validated `env.ts` in the frontend. No scattered `os.environ` or
  `process.env` reads.
- **[Recommendation] Agent-facing instructions file per app** (for example
  `AGENTS.md` / `CLAUDE.md` in `backend/` and `frontend/`). It states the layering,
  where each kind of code goes, the error convention and the test commands. This is
  low cost and directly serves the client's "vibe coding" continuation (`BC-005`).
- **[Recommendation] Typed API contract shared across apps:** the backend's FastAPI
  OpenAPI schema is the source, and frontend request and response types are generated
  from it (for example `openapi-typescript`) into `src/types/api.gen.ts`, with a CI
  check for drift. Hand-written duplicate types across two apps drift quickly.
- **[Recommendation] Type-checking bar:** mypy `strict` on `backend/app/` (tests may be
  looser); TS `strict` plus `noUncheckedIndexedAccess`. Type hints on every function,
  not only public ones. Strict typing is the cheapest guard against confident but wrong
  AI-generated code.
- **[Recommendation] Prefer explicit code:** no metaprogramming or dynamic imports beyond
  the settings-selected vendor factory. Keep modules small and single-purpose.

### 6. Gaps the human interview must resolve (developer view)

Add these to `evidence.md` → Open Uncertainties (items 1 and 8 there already cover some):

1. **Monorepo vs two repositories.** I support one repository (see Positions).
2. **Frontend package manager:** npm, pnpm or yarn. Commit exactly one lockfile.
3. **Python toolchain:** `uv`, Poetry or pip-tools, plus the **Python version**. Caveat:
   MediaPipe publishes wheels only for some CPython versions, so check the pinned
   version against the MediaPipe release at setup time **[Assumption — verify]**. Also
   pin the **Node.js LTS** version (`.nvmrc` / `engines`).
4. **Next.js App Router vs Pages Router.** I recommend App Router, since it is the
   current default and the most common in AI-tool training data. Either router works
   with section 2c, because auth pages are Client Components either way.
5. **SQLAlchemy sync vs async** (SQLAlchemy 2.x `AsyncSession` + asyncpg vs sync +
   psycopg). This touches every repository, dependency and test fixture, so decide it
   before any code exists.
6. **JSON casing convention** (section 4).
7. **Error envelope shape and error-code catalogue** (section 3). The format can be
   affirmed here; the catalogue belongs in contract design.
8. **API versioning prefix** (`/api/v1` or none).
9. **HTTP status for OTP lockout and resend cooldown** (`AUTH-011` gives the limits, not
   the status code: 429 vs 423 vs 400). Contract-design item; noted so it is not
   silently chosen in code.
10. **Where the long-running pipeline runs** (in-request vs background worker for CV +
    OpenAI + 24 images, `FR-020`/`FR-022`). This is an architecture decision, but it
    decides whether a `workers/` package exists in the layout. Flag only.
11. **Ruff-only vs Black + Ruff** (already lead item 8). I recommend Ruff alone for both
    lint and format, to have one tool and one config.
12. **Pre-commit hooks** (already lead item 8). I recommend them, running the same
    Ruff / ESLint / Prettier configs as CI, so AI-tool output is fixed before the push.

### 7. Corrections to the lead's `discovered-rules.md`

- The Mandated candidate "store OTPs hashed and refresh tokens by identifier" and the
  Forbidden candidate "NEVER store an OTP in plaintext" state the same OTP rule twice.
  Keep the Forbidden form for OTPs and reword the Mandated one to cover refresh tokens
  only. This avoids two rules to maintain for one constraint.
- "NEVER hardcode ... AI vendor/model identifiers in code" should say whether a
  non-secret default inside the `Settings` class (for example `model = "gpt-4o"`) counts
  as hardcoding. Proposed wording: secrets have no in-code default; non-secret vendor
  and model values are read from configuration, and `.env.example` documents their
  defaults. `NFR-008` requires only that they can be swapped "without code changes",
  which a settings default with an env override satisfies. The human should confirm
  which reading applies.
- Suggested new candidate (developer, `[Recommendation]`, derived from §5.3 +
  `FE-004` + `FE-007`): "NEVER call `fetch` / HTTP directly outside the centralized API
  client module in the frontend." This is the testable, lint-enforceable form of
  `FE-004`.
- Suggested new candidate (`[Recommendation]`, derived from §5.3 "decoupled from route
  handlers"): "NEVER put token, OTP, payment-gate or identity-check logic in FastAPI
  route handlers; it lives in services." The client states this only for JWT/OTP; the
  extension to payment and identity logic is labelled as a recommendation.

## Positions

- AGREE: Single repository with `backend/` and `frontend/` top-level apps — one PR can change an API contract and its consumer together, and the client team gets one place to continue work (`BC-005`).
- AGREE: Mandated §5.3 layering candidate — client-stated and central to the auth design; it should also carry enforceable lint rules (section 2b).
- AGREE: Ruff + mypy for Python and TS `strict` + ESLint + Prettier for TypeScript — recommend Ruff alone for both format and lint, and mypy `strict` on `app/`.
- AGREE: Vendor integrations behind configuration-selected interfaces — required by `NFR-008` / `NFR-013`; keep them in one `integrations/` package.
- AGREE: README per app, docstrings and ADRs for `NFR-011` — extend with `.env.example` and an agent-instructions file per app.
- AGREE: Walking skeleton as the infrastructure slice (page → API client → FastAPI → PostgreSQL via Alembic) — it is the smallest integrated slice per `org.md`; auth can be the first feature Unit on top of it.
- AGREE: Requirement IDs in commit and PR titles — gives cheap traceability to `client_requirements.md` §12.
- OBJECT: Code Style section has no error-handling convention — the construction guardrail requires boundary error handling, so add the domain-error hierarchy, single envelope and logging redaction from section 3.
- OBJECT: Layer boundaries are stated only for auth and only as prose — extend the route/service/repository/integration split to every backend feature and enforce it with `import-linter` and ESLint `no-restricted-imports`.
- OBJECT: Duplicate OTP-hashing rule across Mandated and Forbidden in `discovered-rules.md` — merge it into one rule so there is one rule per constraint.
- OBJECT: Open uncertainties omit build-shaping choices — add package manager, Python and Node versions, App Router vs Pages Router, SQLAlchemy sync vs async, and JSON casing, because each one changes the folder layout or every data-access file.
