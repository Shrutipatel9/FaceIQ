**Collaborator:** aidlc-devsecops-agent

## Contribution

Blind support review of the lead drafts (`team-practices.md`, `discovered-rules.md`,
`evidence.md`) from a security and supply-chain angle. Evidence base:
`client_requirements.md` v1.0 (the only source of truth; greenfield, no code),
`org.md`, `phases/construction.md` (Security guardrail), and the repository root
(`.gitignore`, git remote). The labels below follow the client document's convention.
Anything not client-stated stays **[Recommendation]** or **[Assumption]** so that
nothing is invented (`client_requirements.md` "How to use this document" item 2).

### 1. Summary of gaps in the lead draft

1. **No security tooling in any section.** `## Code Style` names Ruff, mypy, ESLint,
   Prettier and `tsc` but no security lint rules. `## Deployment` proposes GitHub
   Actions CI but no SAST, secret scanning, dependency scanning, SBOM or DAST, and
   no blocking or advisory policy. For an app that holds face photos, health-related
   questionnaire answers, payments and custom auth, this is the largest gap.
2. **Missing security hard-constraint candidates.** The draft covers `AUTH-007`,
   `AUTH-010`, `AUTH-011`, `FE-002`, 402/409 and "no hardcoded secrets". It leaves
   out `AUTH-012` (rotation and reuse detection), `AUTH-013` (CSRF), `AUTH-015`
   (`reset_token` must not be accepted as Bearer), Stripe webhook signature
   verification, and any handling rule for biometric data or health-related data.
3. **Real repository finding:** the root `.gitignore` has no `.env`, `.env.*`,
   `*.pem` or `*.key` patterns. The AI-DLC block covers only logs, `node_modules` and
   the framework state. Since secrets come from the environment (`NFR-008`,
   `NFR-013`, `FR-016`), the first Construction commit should add these ignores and a
   committed `.env.example` with no values. Record this as a Construction task, not a
   rule.
4. **Test-fixture privacy is left neutral.** The draft lists "may real face photos be
   committed as fixtures" as an open question with no default. A secure default
   should apply until the human decides (see Positions).

### 2. Proposed additions to `team-practices.md`

#### `## Code Style` — security-relevant lint (all [Recommendation])

- **Python:** enable the Ruff `S` rule set (flake8-bandit), which covers hardcoded
  passwords, `subprocess` with `shell=True`, insecure `random` for secrets, unsafe
  `yaml.load`, `pickle`, `assert`-as-guard and SQL string building. Also enable `B`
  (bugbear). Ruff replaces a separate Bandit run, so it adds no extra tool. OTP and
  token generation must use `secrets`, never `random` (`AUTH-011`, `AUTH-012`).
- **TypeScript:** `eslint-plugin-security` plus `eslint-plugin-no-unsanitized`. Add a
  project lint rule that bans `dangerouslySetInnerHTML` (React
  `react/no-danger`: error). The one allowed exception is a reviewed, sanitized
  wrapper. Reason: AI narrative (`FR-008`) and chat output (`FR-019`) are untrusted
  model output and must render as text. This is the concrete "output encoding" that
  `ASM-001` asks for.
- **TypeScript:** add a lint or CI check that fails on any environment variable with
  the `NEXT_PUBLIC_` prefix whose name matches `SECRET|KEY|TOKEN|PASSWORD`, except
  for an allow-listed Stripe *publishable* key. Next.js inlines `NEXT_PUBLIC_*` into
  the browser bundle, so this is the most common way secrets leak in this stack.
- **SQLAlchemy:** use only bound parameters. `text()` with f-string or `%`
  interpolation is forbidden. This can be enforced by a Semgrep rule (see §2 CI).

#### `## Deployment` / CI — security gates (all [Recommendation], tools need a decision)

Proposed GitHub Actions security stage. It is independent of the undecided host
(`NFR-010`, `CON-007`).

| Control | Proposed tool | Runs | Proposed gate |
|---|---|---|---|
| Secret detection | Gitleaks (pre-commit + CI on full history). GitHub secret scanning **with push protection** if the repo plan allows it | Every commit / PR | **Blocking**, any finding |
| SAST | Semgrep (`p/python`, `p/fastapi`, `p/typescript`, `p/nextjs`, `p/owasp-top-ten` + project rules) **or** CodeQL (free on public repos; private repos need GitHub Advanced Security) | Every PR | **Blocking** on High/Critical; Medium advisory |
| Dependency CVEs | `pip-audit` (backend) + `npm audit --audit-level=high` (frontend) + Dependabot alerts and update PRs | Every PR + weekly schedule | **Blocking** on High/Critical *with a fix available*; otherwise advisory with a waiver |
| Lockfiles | Committed lockfiles (`uv.lock` / `poetry.lock` / hash-pinned `requirements.txt`; `package-lock.json` or `pnpm-lock.yaml`); CI installs with `npm ci` / `--require-hashes` | Every build | **Blocking** if lockfile drift |
| SBOM | Syft or Trivy → CycloneDX artifact per app | On merge to `main` | Advisory (artifact only) |
| CI supply chain | Third-party Actions pinned to a full commit SHA; workflow `permissions:` defaults to `contents: read`; no secrets exposed to `pull_request` runs from forks | Every workflow | **Blocking** (lint with `zizmor` or `actionlint`) |
| Container scan | Trivy image scan | Deferred until packaging is chosen (the lead's container assumption) | **Blocking** on Critical once images exist |
| DAST | OWASP ZAP baseline scan against an ephemeral docker-compose stack in CI (no staging exists yet) | Nightly or pre-release | Advisory at first; blocking on High once a staging host exists |

- Waivers: kept in the repo (for example `.security/waivers.yml`) with a justification,
  an owner and an expiry date. An expired waiver fails the build again.
- Branch protection on `main`: required status checks (tests, lint, security gates),
  no force-push, no direct push. Proposed `CODEOWNERS`: changes under backend auth,
  payment and photo-storage paths need a reviewer named for security.

#### `## Testing Posture` — security test floor (all [Recommendation])

The client mandate "every `AUTH-*` / `BR-*` has a test" (§12) should explicitly
include **negative / abuse tests**, not only happy paths:
- 402 bypass: calling analysis-start without a paid `DATA-008` returns 402. The
  webhook alone never starts the pipeline (`FR-015`).
- 409 bypass: a mismatched photo set is rejected at checkout **and** at analysis
  start (`BR-005` item 8).
- Reuse of an already-rotated refresh token revokes the whole token family
  (`AUTH-012`). A `reset_token` presented as Bearer is rejected (`AUTH-015`).
  `/auth/refresh` without `X-Requested-With`, or with a foreign Origin, is rejected
  (`AUTH-013`).
- OTP: lockout after 5 failures, 60-second resend cooldown, expiry (`AUTH-011`).
- Object-level authorization (IDOR): user A cannot read user B's photos, report, PDF,
  visuals, chat or payments (`DATA-005`–`DATA-011`).
- Stripe webhook with a missing, invalid or replayed signature is rejected. A
  duplicate event ID is a no-op.
- Uploads: non-image or polyglot file, oversize file, decompression bomb.
- **Test data:** synthetic or licensed/consented face images only. See the Positions
  on fixtures.

### 3. Proposed hard-constraint candidates for `discovered-rules.md`

Each candidate cites its origin. Items marked [Recommendation] are security
judgments that go beyond the client text. The human must confirm them; they must
not be presented as client-stated.

**Mandated (candidates)**
- ALWAYS reject a refresh-token presentation whose identifier was already rotated,
  and revoke the whole token family. (candidate — `AUTH-012`)
- ALWAYS require the `X-Requested-With` header and an Origin/Referer allow-list match
  on cookie-authenticated endpoints (`/auth/refresh`, `/auth/logout`). (candidate —
  `AUTH-013`)
- ALWAYS set the refresh cookie `HttpOnly`, `Secure`, with an explicit `SameSite`
  value, and scope its `Path` to the auth routes. (candidate — `FE-002`, `AUTH-013`;
  the attribute set is a [Recommendation])
- ALWAYS configure CORS with an explicit origin allow-list from configuration when
  credentials are enabled. NEVER use a wildcard origin. (candidate — `NFR-005`,
  `AUTH-013`; [Recommendation])
- ALWAYS verify JWTs with a fixed algorithm allow-list and check `exp` and a token-type
  claim, so `reset_token` and refresh tokens are never accepted as access tokens.
  (candidate — `AUTH-004`, `AUTH-007`, `AUTH-015`)
- ALWAYS hash passwords with a memory-hard KDF (Argon2id preferred, or bcrypt).
  (candidate — `WF-002` step 1 says "hashes the password" without naming an
  algorithm; the algorithm is a [Recommendation])
- ALWAYS verify the `Stripe-Signature` header against the raw request body with the
  webhook signing secret, and process each Stripe event ID idempotently. NEVER treat
  the Checkout success redirect as proof of payment. (candidate — `FR-015`, `NFR-009`;
  the verification mechanism is a [Recommendation])
- ALWAYS enforce object-level authorization: every read or write of `DATA-004`–
  `DATA-011` is scoped to the authenticated user's own records. (candidate —
  `AUTH-007`, `AUTH-008`; [Recommendation])
- ALWAYS validate uploaded photos by decoded content (magic bytes plus successful
  image decode), with size and pixel-count limits, then re-encode and strip EXIF
  metadata (including GPS) before storage or AI processing. (candidate — `FR-006`,
  `BR-005`; EXIF and re-encode are a [Recommendation])
- ALWAYS rate-limit unauthenticated auth endpoints (login, signup, OTP verify and
  resend, forgot-password) per account and per IP, in addition to the `AUTH-011`
  OTP limits. (candidate — `AUTH-011`; login and IP limits are a [Recommendation])
- ALWAYS render AI-generated text (narrative, recommendations, chat replies) as
  text, never as HTML, and ship a Content-Security-Policy on the frontend.
  (candidate — `ASM-001`, `FR-008`, `FR-019`)

**Forbidden (candidates)**
- NEVER log or send to telemetry or error trackers: passwords, OTPs, access or refresh
  tokens, `reset_token`, Stripe signatures, raw photos, landmark identity signatures
  (`ASM-010`), or questionnaire medical, medication and allergy answers. (candidate —
  `AUTH-010`, `AUTH-011`, `FR-008`(f); [Recommendation])
- NEVER commit real face photos, or any file derived from a real person's face, to the
  repository; test fixtures are synthetic or licensed with consent. (candidate —
  `FR-007`, `ASM-010`; [Recommendation] pending the human decision)
- NEVER serve user photos or generated images from public-readable storage or URLs.
  Access goes through an authenticated endpoint or short-lived signed URLs.
  (candidate — `DATA-005`, `DATA-009`, `DATA-010`; [Recommendation])
- NEVER expose a server secret through a `NEXT_PUBLIC_*` variable or any frontend
  bundle. (candidate — `NFR-008`, `NFR-013`, `FR-016`)
- NEVER disable a security gate or add an unexpiring waiver to make CI pass.
  (candidate — `org.md` Testing Posture "may not be weakened to make a step pass",
  applied to security gates; [Recommendation])

### 4. Sensitive-data note (for the interview and for later compliance and NFR work)

- Face photos, the 5-ratio landmark signature used for the identity check
  (`ASM-010`), and questionnaire answers about medical conditions, medications and
  allergies (`FR-003`) are **sensitive personal data**. The signature is effectively
  a biometric template. Depending on where users live, GDPR Art. 9, Illinois BIPA,
  and CCPA/CPRA may apply. **[Assumption]**: the target market is not stated in
  `client_requirements.md`.
- **Tension to surface, not resolve** (per "How to use this document" item 3): "Formal
  data-retention/privacy policy definition" is out of scope (§2.2). The build still
  needs baseline controls (encryption at rest, access scoping, deletion ability), and
  photos are sent to OpenAI (`FR-008`, `NFR-013`). The client should be told that the
  deferral does not remove the legal obligations. The engineering baseline above is
  proposed so that a later policy does not force a redesign.

### 5. Gaps the human interview must resolve

1. **Scanner choice:** Semgrep vs CodeQL (depends on whether the repo is public or
   has GitHub Advanced Security). Gitleaks alone, or Gitleaks plus GitHub push
   protection? Dependabot vs Renovate?
2. **Blocking vs advisory:** confirm the proposed thresholds (block High/Critical for
   SAST and dependencies with a fix; block every secret finding; DAST advisory until a
   staging host exists). Confirm the waiver process and who approves waivers.
3. **Photo and generated-image storage:** local disk vs object storage vs a Postgres
   BLOB. This is host-neutral but needs an abstraction. Decide on encryption at rest
   and on a retention or deletion stance before the data model is fixed.
4. **Real-photo fixtures:** allowed at all? If yes, where are they stored (outside the
   repo, access-controlled) and under what consent?
5. **OpenAI data handling:** is the client aware that user photos and questionnaire
   context are sent to OpenAI? Should a zero-data-retention / no-training
   configuration be requested? Is user consent captured before upload?
6. **Cookie topology:** will the frontend and backend be same-site? This decides
   `SameSite=Lax/Strict` vs `SameSite=None; Secure`, and how much weight the
   `AUTH-013` CSRF controls carry. Hosting is undecided, so the design must support
   both.
7. **Account enumeration:** `WF-002` has signup "check the email isn't registered",
   while `BR-009` exempts forgot-password for anti-enumeration. Should signup and
   login responses also be non-enumerating?
8. **Cost-abuse limits:** 24 generated images per report plus chat, with costs billed
   to the client (`BR-006`). Confirm server-side one-report-per-user
   idempotency (`FR-014`) and a per-user chat rate or quota.
9. **Security review ownership:** who is the named security reviewer for
   `CODEOWNERS` on auth, payment and photo paths?

## Positions

- AGREE: NEVER hardcode credentials, keys, price or vendor IDs; read them from configuration — matches the construction Security guardrail and `NFR-008`/`NFR-013`/`FR-016`; add the `NEXT_PUBLIC_*` leak rule and `.env` ignores to make it enforceable.
- AGREE: refresh token only in an httpOnly cookie, access token in memory, no Zustand `persist` — correct reading of `FE-002`/`ASM-001` and the right XSS blast-radius limit.
- AGREE: server-side 402 and 409 enforcement as a Mandated rule — the UI gate is bypassable; pair it with required negative tests.
- AGREE: OTPs hashed and refresh tokens stored by identifier — directly from `AUTH-010`/`AUTH-011`.
- AGREE: third-party auth-as-a-service forbidden and host-neutral configuration — client-stated (`AUTH-001`, `CON-002`, `NFR-010`).
- AGREE: external services replaced by fakes in CI, with signed Stripe webhook fixtures — no paid calls and no live secrets in CI.
- OBJECT: `discovered-rules.md` lacks candidates for `AUTH-012` reuse detection, `AUTH-013` CSRF, `AUTH-015` token-type separation and Stripe webhook signature verification — these are the security-critical auth and payment rules and belong beside `AUTH-007`/`AUTH-010`/`AUTH-011`.
- OBJECT: `## Code Style` and the CI proposal contain no security lint, secret scanning, SAST, dependency scanning or Actions pinning — "linters block the PR" is not a security gate; add the §2 controls as recommendations for the interview.
- OBJECT: real face photos as test fixtures is left as a neutral open question — secure by default says "NEVER commit real face photos" until the human explicitly decides otherwise.
- OBJECT: sensitive-data handling (photos, the biometric identity signature, health answers) has no candidate rule — at minimum the no-logging and no-public-storage rules should be put to the human, with the §2.2 privacy-policy deferral surfaced as a tension.
