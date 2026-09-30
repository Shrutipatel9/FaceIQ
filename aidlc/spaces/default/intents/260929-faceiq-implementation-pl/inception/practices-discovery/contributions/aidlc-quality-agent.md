**Collaborator:** aidlc-quality-agent

## Contribution

Scope of this review: the lead draft's `## Testing Posture` (and the testing-related
parts of `## Way of Working`, `## Walking Skeleton`, `## Deployment`), the test-related
candidates in `discovered-rules.md`, and the open uncertainties in `evidence.md`.
Evidence: `client_requirements.md` v1.0 (greenfield, no code, no CI, no test setup in
the repository) and `org.md` `## Testing Posture`. Labels follow the lead draft:
**[Client-stated]**, **[Org default]**, **[Recommendation]**, **[Assumption]**.

### 1. Methodology and Ordering: the draft's fields are valid, but the interview needs a concrete `custom` option

- The draft correctly writes the two structured fields (`Methodology`, `Ordering`)
  with the `org.md` default (`test-after`). Nothing in the repository or the client
  document fixes a methodology, so this is purely a human choice.
- **[Recommendation]** Offer the interview a concrete `custom` option alongside
  `test-after`, because this system has two kinds of logic with different risk profiles:
  - Deterministic rule logic with exact expected results written in the client
    document: OTP expiry/cooldown/lockout (`AUTH-011`), token lifetimes, rotation and
    reuse-family revocation (`AUTH-012`), `reset_token` single use (`AUTH-015`),
    the identity-check vote (`BR-005` steps 2-5), the payment gate (`FR-015`, HTTP 402),
    the recheck (`BR-005` step 8, HTTP 409), single-flight refresh and the
    "never clear auth" rules (`FE-005`, `FE-006`), and keyword tiering (`ASM-007`).
    These are already specified as tables of inputs and outcomes, so writing the
    test first costs little and catches misreadings of the spec early.
  - UI screens, CV measurement code, and AI integration, where the expected result
    is only known after building (and, for AI and CV, only approximately).
- Proposed `custom` wording for the interview, if chosen:
  - `- **Methodology**: custom`
  - `- **Ordering**: For auth, payment-gate, identity-check and tiering rules, write the failing tests from the requirement's acceptance criteria before implementing; for all other layers, implement the layer first and then write and run its tests before moving on.`
- If the human prefers simplicity, `test-after` stays valid; the high-risk suites
  listed in the draft still apply either way.
- Note for the interviewer: `inception.md` requires Given/When/Then acceptance
  criteria in user stories. That is a requirements format, not a test methodology.
  Choosing `bdd` would additionally imply executable Gherkin (for example
  `pytest-bdd` or Playwright + Cucumber); the question should make that cost visible
  so the Given/When/Then rule is not mistaken for a BDD commitment.

### 2. Coverage floor: add branch coverage, a stricter floor for high-risk modules, and explicit exclusions

- **[Recommendation]** Keep the draft's 80% line floor per application, and add:
  - Branch coverage reported alongside line coverage (`pytest-cov --cov-branch`;
    Vitest `v8` provider reports branches). Line coverage alone does not show whether
    the lockout, reuse-detection, or vote-ambiguity branches ran.
  - A higher floor (proposal: 90% line and branch) for the backend auth services,
    the payment-gate and identity-check modules, and the frontend auth store and API
    client. Needs a decision (open uncertainty 5 in `evidence.md` already asks this).
  - An explicit, reviewed exclusion list declared in config: Alembic migration
    scripts, generated OpenAPI client types, Next.js config files. Exclusions are
    changed only by PR, never to make a gate pass (`org.md`: floors "may not be
    weakened to make a step pass").
- **[Assumption]** Scope note for the interview: the current intent's scope
  (`requirements-to-plan`) carries no floor, and the affirmed `team.md` Testing
  Posture will apply to every later intent in this space, including the future
  Construction intent. The question should state that the floor is being chosen for
  the future build.

### 3. Making "every `FR-*`, `AUTH-*`, `BR-*` maps to a test" enforceable

The draft carries this as a client-stated practice and a candidate `ALWAYS` rule, but
gives no mechanism, so at Build and Test it can only be checked by hand.

- **[Recommendation]** Tag tests with requirement IDs:
  - Backend: a pytest marker such as `@pytest.mark.req("AUTH-012")`.
  - Frontend unit tests: the requirement ID in the `describe`/`it` title.
  - Playwright: a tag in the test title (for example `@FR-015`).
- **[Recommendation]** A small CI script collects the tags, compares them with the ID
  list parsed from `client_requirements.md`, publishes a requirement-to-test matrix,
  and fails on any unmapped `FR-*`, `AUTH-*` or `BR-*` ID.
- The script needs a reviewed **pending list** for IDs whose detailed specification
  does not exist yet (`FR-003`/`FR-004` literal questions and disclaimer, `FR-005`
  angle flow, `FR-006` thresholds, `FR-009`–`FR-022` report/UI details; see the
  client document's "Detailed specifications to be provided separately"), and for
  content marked **unconfirmed** (Face Shape, Skin view in `FR-018`; Protocol section
  in `FR-017`). Tests must not assert unconfirmed content.
- Some IDs are not testable by an automated assertion as written (for example
  `BR-006` third-party costs, `BR-007` no timeline). The pending list should allow a
  reviewed "not testable" status with a reason, rather than forcing a meaningless test
  (`construction.md`: no always-passing tests).

### 4. Test layers per application (additions to the draft's tooling)

Backend **[Recommendation]**:
- `pytest`, `pytest-cov`, `httpx` `AsyncClient` / FastAPI `TestClient`, and a real
  PostgreSQL (CI service container or Testcontainers) with `alembic upgrade head`
  applied; each test runs in a rolled-back transaction for independence.
- A migration round-trip test (`upgrade head` → `downgrade base` → `upgrade head`)
  so every Alembic migration is reversible (`NFR-003`; supports the draft's
  expand-contract and rollback practices).
- An injectable clock (or `time-machine`/`freezegun`) for everything time-based:
  OTP 10-minute expiry, 60-second cooldown, 15-minute lockout, 15-minute access token,
  7-day refresh token, ~10-minute `reset_token`. Without it these tests either sleep
  or are skipped.
- Exhaustive table tests for the identity vote: with three photos there are exactly
  8 pairwise match patterns, so all of them can be enumerated against `BR-005`
  steps 2-4 (fewest matches flagged; ambiguous → front as reference; all different
  → both non-front flagged; consistent → empty list).
- Factories (for example `factory_boy`) for users, OTP records, sessions, payments.

Frontend **[Recommendation]**:
- Pick one unit runner rather than "Vitest (or Jest)": **Vitest** with React Testing
  Library (fast, native TypeScript/ESM). Needs a decision.
- Mock Service Worker (MSW) for API-client tests, plus fake timers, to cover
  proactive refresh ~1 minute before expiry, 401 → refresh → retry, single-flight
  under concurrent requests, and the `FE-006` cases where auth must **not** be
  cleared (network error, 5xx, page reload).
- A test that the auth store is not persisted (`FE-002`, `ASM-001`): after a store
  update, `localStorage` and `sessionStorage` are unchanged.
- Playwright for end-to-end journeys (`WF-001`, `WF-002`).

Contract between the two applications (gap in the draft) **[Recommendation]**:
- The FastAPI OpenAPI schema is exported in CI; the frontend's API types are
  generated from it (for example `openapi-typescript`), and CI fails if the
  generated types drift from the committed ones. This catches a backend change that
  breaks the frontend in the same PR, which supports the draft's one-repository
  rationale. Schema-based fuzzing (for example Schemathesis) is optional.

### 5. External services: how each is tested (resolves gaps the draft leaves open)

The draft's rule "no test in CI calls a paid API" is right. The additions below say
how each flow is still covered.

- **Email OTP (`NFR-012`, `AUTH-003`, `AUTH-011`):** unit/integration tests use a fake
  email provider that captures messages in memory. E2E needs the OTP value: use a
  local SMTP catcher container (for example Mailpit) or a test-only inbox endpoint
  that exists only when a test-environment flag is set. **[Assumption]** A test-only
  endpoint is a security risk if it ever ships to production; flag for
  devsecops review, and add a test that it returns 404 when the flag is off.
- **Stripe (`FR-015`, `FR-016`, `NFR-009`):**
  - Webhook tests use payloads signed with a test signing secret (Stripe's library
    can build the signature header) and cover: invalid signature rejected, duplicate
    event delivery is idempotent, out-of-order events, and **the webhook never starts
    the analysis pipeline** (assert no job is queued).
  - Price and currency come from config in tests (`FR-016`); a test asserts no
    hardcoded amount is used.
  - PR-level E2E does not drive Stripe's hosted Checkout page; it stubs session
    creation and posts a signed fixture webhook. **[Recommendation]** An optional
    nightly or pre-release run in Stripe test mode with test cards covers the real
    redirect. Needs a decision.
- **OpenAI narrative (`FR-008`, `NFR-008`) — non-deterministic output:** exact-text
  assertions are not usable. Proposed layering:
  1. Deterministic unit tests on the **prompt builder**, which is where most `FR-008`
     rules are enforceable: questions sent with real question text, not IDs (a);
     measurement values present in the context (b); tone instruction switches when
     answers indicate elevated distress (e); medical, medication and allergy answers
     absent from the payload (f).
  2. A **structured-output contract** (JSON schema: all 11 features present, per
     `FR-009`/`BR-008`) validated at runtime and in tests; checks for 3-5 sentences
     per feature (c) and for a cited measurement value (b) run as validators on the
     response, with a defined retry/fallback path when a response fails them.
  3. Integration tests replay **recorded responses** (cassette style, for example
     `vcrpy`/`pytest-recording`) with secrets scrubbed.
  4. A separate, opt-in **evaluation suite** against the real model, run on prompt or
     model changes rather than per PR: a fixed set of profiles scored against a
     rubric (tone, no medical content, measurement cited), reporting a pass rate
     rather than a binary per-case result. It costs real money (`BR-006`), so budget
     and trigger need a decision.
- **AI Beauty Assistant (`FR-019`):** a fixed list of medical and medication
  questions (red-team set) that must be declined and redirected, run in the
  evaluation suite; any deterministic pre-filter is unit-tested per PR.
- **Image generation (`FR-020`, `FR-022`, `NFR-013`):** a fake adapter returns fixed
  images; tests assert 11 per-feature images + 13 AI Visuals = 24 per report, the
  healthy-aging stack uses the user's own photo for "current" (not generated) plus
  +3/+5/+10 years, and behaviour when some generations fail. One shared contract test
  suite runs against every adapter behind the abstraction, so a vendor swap by config
  is verified. Visual quality and identity preservation (`ASM-011`) is a manual review
  activity, not an automated gate.

### 6. Computer vision heuristics and face-photo fixtures (`FR-006`, `FR-007`, `FR-018`, `ASM-002`, `ASM-010`)

- **[Recommendation]** Split CV code so most of it is tested **without photos**:
  measurement and assessment functions take landmark coordinates as input, so their
  tests use landmark fixtures (JSON arrays of 478 points, including hand-built
  symmetric/asymmetric cases). This covers the 7 geometric features, symmetry score
  and label, facial thirds, the identity signature (5 ratios normalised by
  inter-eye distance) and the 0.6 threshold comparison, with no personal data in the
  repository.
- Photo-based tests are then limited to the thin MediaPipe/OpenCV adapter and the
  per-photo checks (readability, resolution, brightness, one face, face proportion,
  occlusion, pose). Their fixtures can be derived from a small set of base images by
  transformation (downscale, darken/overexpose, crop, mirror, paste a second face,
  add occluders), and assertions use tolerances, not exact floats.
- Pin the MediaPipe package version and the Face Landmarker model file (by hash),
  because a model change silently shifts golden values.
- Thresholds are read from configuration in tests, never duplicated as literals, so
  the planned tuning pass (`ASM-002`, `ASM-010`) does not require rewriting tests.
- **Calibration is not a CI gate.** The identity threshold and ear formula
  (`ASM-010`) need a separate labelled evaluation set that reports false-accept and
  false-reject rates per threshold. The client document says they have not been
  validated on a large, diverse set, so the plan should include this as its own
  activity with a reported result, not a pass/fail test.
- **Open privacy question (sharpened from the draft):** where base face images come
  from — (a) synthetic or AI-generated faces, (b) a licensed/consented public dataset
  whose licence allows storage in a repository, or (c) consented images kept outside
  the repository (private bucket or Git LFS with restricted access) and pulled only in
  CI. Real user uploads must never become fixtures. Whatever is chosen should also
  cover demographic diversity, since the heuristics are uncalibrated.

### 7. CI quality gates and test cadence

- **[Recommendation]** Per pull request (blocking): format + lint, type check
  (`mypy`, `tsc --noEmit`), backend unit + integration against PostgreSQL, frontend
  unit, OpenAPI contract drift check, coverage floors, requirement-traceability
  check, and a small Playwright smoke set (signup with OTP, login, pay via fixture
  webhook, start analysis with fake AI) — cheap because every vendor is faked.
- **[Recommendation]** Nightly or pre-release (non-blocking for PRs, reported):
  full Playwright suite, optional Stripe test-mode run, and the AI evaluation suite
  when prompts or models changed.
- **[Recommendation]** Target PR feedback under 10 minutes; tests that flake are
  quarantined with a tracking issue and fixed or removed within an agreed period
  (proposal: one week). A quarantined test does not count toward requirement mapping.
- Because `org.md` deploys on merge to staging, anything not run per PR reaches
  staging unverified; the interview should confirm that trade-off for the nightly set.

### 8. Gaps the interview (or later requirements work) must resolve

Testing-posture questions for the interview:
1. Methodology: keep `test-after`, or `custom` (tests first for auth/payment/
   identity/tiering rules, test-after elsewhere), or `tdd`/`bdd`/`atdd`?
2. Coverage: 80% line per app — add branch coverage? Higher floor (for example 90%)
   for auth, payment-gate and identity-check code?
3. Requirement mapping: enforce by tagging tests with requirement IDs plus a CI check,
   with a reviewed pending list for IDs awaiting detailed specifications?
4. End-to-end scope: which journeys run on every PR versus nightly; is a real
   Stripe test-mode run wanted, and how often?
5. AI output: accept the layered approach (prompt-builder unit tests, output schema
   validation, recorded responses, separate paid evaluation suite)? Who owns the
   evaluation budget and when does it run?
6. Face-photo fixtures: synthetic, licensed/consented dataset, or private storage
   outside the repository?
7. Frontend unit runner: Vitest or Jest?

Not practice questions, but gaps to hand forward (they block writing measurable
tests later; per `inception.md`, requirements need a pass/fail criterion):
- No latency, duration or availability targets exist in `client_requirements.md`
  (for example maximum analysis time for one report with 24 images, API response
  targets, what the user sees if analysis fails or partly fails). Performance and load
  testing cannot be planned until these exist.
- No accessibility requirement exists; automated checks (for example axe in
  Playwright) would be a **[Recommendation]** only.
- `FR-013` PDF: test content by text extraction (all 11 sections, disclaimer present)
  and caching on second download; visual layout needs a manual or snapshot check.

## Positions

- AGREE: Testing Posture structured fields — the draft writes `Methodology` and `Ordering` as the stage requires, using the `org.md` default as a suggestion only.
- AGREE: No paid external API calls in CI, fakes behind vendor interfaces, signed Stripe webhook fixtures — correct and required by `BR-006` cost ownership and test repeatability.
- AGREE: Real PostgreSQL (not SQLite) with Alembic applied for integration tests — the client fixes PostgreSQL and Alembic (`NFR-002`, `NFR-003`), so tests must run against them.
- AGREE: High-risk suites for `AUTH-010`–`AUTH-012`, `FE-005`/`FE-006`, `FR-015` 402 and `BR-005` 409 — these are the paths where a defect is a security or revenue failure.
- AGREE: Candidate rule "ALWAYS map every `FR-*`, `AUTH-*`, `BR-*` to at least one test case" — client-stated in §12; should be paired with the tagging and CI-check mechanism in section 3 to be verifiable.
- OBJECT: Testing Posture tooling "Vitest (or Jest)" — leaving the runner open pushes a trivial decision into Construction; recommend Vitest and ask the human to confirm.
- OBJECT: Testing Posture has no contract test between the two applications — with a separate FastAPI backend and Next.js frontend (`NFR-001`, `NFR-005`), API drift is a likely defect class; add OpenAPI-generated types with a CI drift check.
- OBJECT: Testing Posture treats CV testing as "golden-image fixture tests" — most CV logic can and should be tested on landmark-coordinate fixtures without face photos; image fixtures should be limited to the MediaPipe/OpenCV adapter and per-photo checks, which also shrinks the privacy question.
- OBJECT: Testing Posture is silent on non-deterministic OpenAI output (`FR-008`, `FR-019`) — the draft should state the layered approach (prompt-builder unit tests, output schema validation, recorded responses, separate evaluation suite) or list it as an interview question.
- OBJECT: Testing Posture names Playwright for `WF-001`/`WF-002` without saying how E2E gets the emailed OTP or passes Stripe hosted Checkout — both need a stated approach (SMTP catcher or test-only inbox; stubbed checkout plus signed webhook) before E2E is plannable.
- OBJECT: Coverage floor proposal uses line coverage only — add branch coverage and a reviewed exclusion list, and ask about a higher floor for auth, payment-gate and identity-check code.
- OBJECT: Discovered rules lack a candidate for face-image test data — propose for human confirmation: "NEVER use real end-user uploaded photos as test fixtures, and NEVER commit identifiable face images to the repository without documented consent and a licence that allows it."
- OBJECT: Discovered rules lack a candidate for CI cost safety — propose for human confirmation: "NEVER let the per-PR test suite call live OpenAI, Stripe live-mode, or a real email provider."
