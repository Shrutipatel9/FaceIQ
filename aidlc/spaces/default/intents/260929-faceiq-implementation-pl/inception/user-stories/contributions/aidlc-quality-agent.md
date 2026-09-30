**Collaborator:** aidlc-quality-agent

## Contribution

Scope: whether the acceptance criteria (ACs) in `stories.md` can be tested. This covers pass/fail clarity, stated boundaries, a happy path plus at least one error or edge case per story, the placement of the "tests first" markers against the affirmed Testing Posture (`team.md`), and coverage of FR1–FR26 / NFR1–NFR12 and the client `FR-*` / `AUTH-*` / `BR-*` IDs. Findings are grouped by severity, and each one names the story or AC to change. Proposed AC text is written so it can be pasted into the story.

### 1. Blocking: criteria with no pass/fail oracle

| # | Where | Problem | Proposed replacement / action |
|---|-------|---------|-------------------------------|
| B1 | AC3.4.2 (identity vote) | The AC says "front is the reference" but never gives the **expected flagged list** for each pattern. A table test needs that expected output. `BR-005` item 2 ("fewest matches is flagged; ambiguous → front is reference") does not settle two of the 8 patterns (rows 2 and 5 below). | Replace AC3.4.2 with an explicit 8-row oracle table in the story (F = front, L = left 3/4, R = right 3/4; T = pair matches within threshold). Rows 2 and 5 are **[Assumption]** and must be raised to the human / Photo Validation Specification (OQ5), not decided silently. See the table after this one. |
| B2 | AC3.5.3 | "then it **may** skip review" is not a pass/fail statement. | "Given the review screen is opened by a fresh page load and the stored set is identity-consistent, when the page loads, then the user is sent past review to the next step (FR26). Given the same set becomes consistent through a retake on that screen, when the result arrives, then the user stays on the review screen (AC3.5.2)." |
| B3 | AC1.2.3 | "fails the password rules". No rules are defined anywhere (requirements.md is silent). | Add a **[Decided by delivery team]** password policy as configuration (for example a minimum length). Rewrite as boundary cases: length min−1 is refused with a field-level message; length min is accepted. The same policy applies to US1.10 set-password and US8.2 new-password. |
| B4 | AC1.10.2 | "valid **about** 10 minutes". A test cannot assert "about". | "…a single-use `reset_token` whose lifetime is the configured value (default 10 minutes). With an injectable clock, it is accepted at TTL−1 s and rejected at TTL+1 s." |
| B5 | AC5.8 title / persona text ("realistic", "still looks like me") and FR17.1 "identity-preserving" | These cannot be automated. Posture says visual quality (`ASM-011`) is manual review. | Keep the words in the narrative, but add a note: "Visual realism and identity preservation are verified by manual review (`ASM-011`) and are not an automated AC." Automated ACs stay count, linkage and adapter-contract only. |
| B6 | AC7.4.1 ("grounded"), AC7.5.1 (the live model declines) | Model behaviour is not deterministic, and per-PR tests may not call live OpenAI (`NFR8`, project Forbidden). | AC7.4.1 becomes: "Given user A's report, when the chat context is built, then it contains A's measurements, narrative and questionnaire answers (excluding medical, medication and allergy answers where FR15.4 applies) and no other user's data." AC7.5.1 moves to the **opt-in evaluation suite** and is labelled "not a per-PR gate". Its per-PR proof is AC7.5.2 plus a recorded-response test. |
| B7 | AC5.7.3, AC7.1.3 ("only content defined in the Report & UI Specification") | This is a negative scope guard over content that does not exist yet. Posture says tests never assert unconfirmed content (`FR-017` Protocol, `FR-018` Face Shape and Skin). | Keep them as scope notes, not ACs. Put `FR16.2` / `FR21.3` on the reviewed **pending list** with the reason "awaiting Report & UI Specification". |
| B8 | AC9.6.1 | The p95 500 ms target has no load definition until OQ7 is answered. | Keep it, but mark it **Blocked on OQ7** and "not a CI gate". Add to the AC: "load profile (concurrent users, request mix, duration) is fixed in NFR Requirements before the test counts." |
| B9 | NFR3.1 (10-minute p95 for the whole analysis) | No AC validates it. AC5.2.3 only records durations. With fakes in CI the target cannot be measured. | Add AC5.2.4: "Given ≥ N completed analyses run against the real vendors in a performance-validation run (N and environment fixed in NFR Requirements), when the Start→published durations are aggregated, then p95 ≤ 10 min." Label it "performance-validation activity, not per-PR". |

**Proposed oracle for AC3.4.2** (`BR-005` items 2–3). Counts are the number of other photos each one matches.

| Row | F–L | F–R | L–R | Counts F/L/R | Expected flagged | Basis |
|---|---|---|---|---|---|---|
| 1 | T | T | T | 2/2/2 | `[]` | Consistent |
| 2 | T | T | F | 2/1/1 | `[]` **or** `[L,R]`? | **Ambiguous in `BR-005`.** Front is the reference and both match it, so `[]` is the likely reading. **Raise to human.** |
| 3 | T | F | F | 1/1/0 | `[R]` | Fewest = R |
| 4 | F | T | F | 1/0/1 | `[L]` | Fewest = L |
| 5 | F | F | T | 0/1/1 | `[F]` **or** `[L,R]`? | **Conflict:** the vote flags front, but "front is the reference" would keep it. **Raise to human.** |
| 6 | F | F | F | 0/0/0 | `[L,R]` | `BR-005` item 3 |
| 7 | T | F | T | 1/2/1 | `[R]` | Tie between F and R, so front is the reference and R does not match it |
| 8 | F | T | T | 1/1/2 | `[L]` | Tie between F and L, so front is the reference and L does not match it |

Also add to US3.4 the **threshold boundary** case: "similarity exactly at the configured threshold (default 0.6) counts as match / no-match". The comparison operator must be stated, so tests do not duplicate the config value (posture: thresholds read from config).

### 2. Tests-first marker placement (affirmed ordering)

The affirmed list is: auth token rotation, reuse detection and OTP lockout; the 402 payment gate; the identity vote and 409 recheck; recommendation tiering.

| Story | Current | Verdict |
|---|---|---|
| US1.3 OTP expiry/lockout | tests first | Correct |
| US1.8 rotation/reuse/CSRF | tests first | Correct |
| US1.10 reset_token | tests first | Correct (auth token rule) |
| US3.4 vote | tests first | Correct |
| US4.1 409 at checkout | tests first | Correct |
| US4.3 402 matrix | tests first | Correct |
| US5.6 tiering | tests first | Correct |
| **US5.1 start analysis (402 + 409 recheck)** | **not marked** | **Missing.** AC5.1.2 is the 409 recheck at analysis start (`BR-005` item 8, FR11.5), and FR13.2 re-checks 402 here. Mark phase 1 "P-API tests first: 402/409 re-checks and the single-job guard". |
| **US1.5 login OTP step** | not marked | The login OTP reuses the US1.3 lockout rules. Add a tests-first line: "lockout and expiry tests run for purpose `login`" (see C4). |
| US1.4 resend cooldown | tests first | Acceptable. Cooldown is part of `AUTH-011`. |
| US1.7 (FE refresh), US4.2 (webhook), US9.5 (cross-user) | tests first | Beyond the affirmed list. They are exactly-specified rules, so this is harmless and I support it. Note in the Overview that these are a deliberate extension, so the list does not look inconsistent. |

### 3. Criteria needing boundaries or exact outcomes (non-blocking, but the lead should tighten them)

- **AC1.3.2**: add the boundary. With an injectable clock, an OTP is accepted at 9 min 59 s and rejected at 10 min 00 s (state whether the limit is inclusive or exclusive).
- **AC1.3.3**: state what the 5 attempts are counted against (per account and purpose, not per code) and when the 15 minutes start (the 5th failure). Add **AC1.3.4**: "Given a lockout started at T, when a correct OTP is submitted at T+15 min+1 s, then it is accepted." Add **AC1.3.5** (lockout bypass): "Given an active lockout, when a resend is requested, then no new code is issued, and the lockout and attempt count are not reset."
- **AC1.4.1**: boundary at 59 s refused / 60 s allowed.
- **AC1.4.3, AC5.1.4**: "repeated", "many" and "rate limit" need numbers. Rewrite as: "Given the configured limit N per window, when request N+1 arrives inside the window, then it is refused with HTTP 429 and error code `rate_limited`. Request N succeeds."
- **AC1.5.2**: "identical" should cover status code and response body. Add an edge case the draft misses: **login to an unverified account.** State the expected outcome (for example, the same generic path plus a signup OTP). This must be decided, not invented. Raise it if requirements.md is silent (it is).
- **AC1.7.1**: "Given an access token issued with 15-min lifetime, when fake time reaches expiry−60 s, then exactly one `POST /auth/refresh` is sent, and no request receives 401 before it."
- **AC1.8.2**: state the rejection status (for example 403 with code `csrf_rejected`), and cover missing Origin with a Referer fallback.
- **AC1.8 (new AC1.8.4)**: "Given a successful login or refresh, when the `Set-Cookie` header is inspected, then the refresh cookie has `HttpOnly`, `Secure`, an explicit `SameSite` and a `Path` scoped to the auth routes." Today this exists only as a phase, and the project Mandated rule needs a test.
- **AC1.9.3**: make it observable. "…then no request to `/auth/logout` is sent on `beforeunload`, `visibilitychange` or route change."
- **AC2.1.2**: says "**both** consents", but FR7.1 lists **three** consent subjects (face images and biometric signature; health answers; sending to the AI vendor). Align the count, or state that the AI-vendor consent is folded into the other two. A test needs the exact record count.
- **AC2.1.1**: add a server-side clause. "…and a direct API call to photo upload or health answers without consent returns 403." As written, the AC can be met by the UI alone.
- **AC2.1.3**: add the partial-consent edge case (images accepted, health declined), which gives photos allowed and health section blocked.
- **AC3.1.1**: list the 7 checklist items (FR9.1), so the test asserts the content and not just a count.
- **AC3.3**: FR10.1 has 7 checks, but the ACs cover only face count, pose and readability. Add a parametrised AC: "Given the configured thresholds, when a fixture fails resolution / brightness / face proportion / occlusion, then it is rejected with that check's **distinct, stable reason code**." Stable codes are what make FR10.2 ("clear reason") testable.
- **AC4.1.2**: split it. The happy path is "server returns a Stripe Checkout session URL and the client redirects". The error path is "inconsistent set → 409 with mismatched angles, no session created".
- **AC4.2**: add **AC4.2.5** for out-of-order events: "Given a `completed` event already processed, when an older or `expired` event for the same session arrives, then the status is not downgraded." Add **AC4.2.6**: "Given the return page is polling, when the webhook is processed, then the page shows paid within the next poll."
- **AC5.3.1**: "clear error" becomes "an error state naming the failed step and a 'Try again' action, shown after exactly the configured N attempts".
- **AC5.4.1**: "defined measurements" becomes "match the golden values for the fixture within the configured tolerance". Add the happy path from FR14.2: "the right ear is measured from the right 3/4 photo".
- **AC5.5**: add **AC5.5.4** for FR15.2 / `FR-008`(b): "Given a feature with a measurement, when the structured output is validated, then the narrative cites the measurement value; if it is missing, the output fails validation." AC5.5.2 needs a defined trigger for "elevated distress" (which answers). Until the questionnaire specification arrives, use a configurable rule and a fixture, and label it as such.
- **AC5.6**: add the at-home tier example; the **no-keyword-match default tier**; and **multi-keyword precedence** (for example "at-home retinol" matches two tiers). These are the edge cases the tests-first table needs.
- **AC5.8.3**: the wording contradicts itself. Change to: "No UI control or API route exists for user-triggered regeneration. Re-running the image step after a partial failure does not create more than 11 feature images (idempotency, NFR7)."
- **AC6.2.3, AC7.4.3**: give the test input. "Given text containing `<script>` and `<img onerror>`, when rendered, then it appears literally and no element is created."
- **AC6.4.3, AC9.4.2, AC9.5.1**: agree **one** cross-user status code for every story. I recommend 404 so resources cannot be enumerated. Do not mix "refused" / 403 / 404. Otherwise each story's tests will assert something different.
- **AC7.3.2, AC9.4.1**: "short-lived" / "short expiry" becomes "the configured signed-link TTL; accepted at TTL−1 s, refused at TTL+1 s".
- **AC7.6.1**: define the reset boundary for the daily cap (UTC midnight or the user's time zone). Add the boundary: message N is allowed, N+1 is refused.
- **AC8.1.2**: add the server side. "An API request that changes the full name is rejected or ignored."
- **AC9.1.1 / AC9.2.1**: "any action" and "any destructive action" cannot be enumerated in a test. Add an **action inventory**: destructive actions are currently only logout; toast actions are listed per story. Then AC9.1.1 becomes: "the API client maps every `ApiError.code` to toast copy; an unmapped code falls back to a generic message." Note: AC9.1.2 (a neutral toast on anti-enumeration flows) differs slightly from FR25.1, which *excludes* those flows from toasts. Pick one reading, because the test depends on it.
- **AC9.3.2** is manual. Label it "manual checklist, not CI".

### 4. Stories with no error or edge case (construction guardrail: happy path + ≥ 2 error/edge cases per test file)

| Story | Add |
|---|---|
| US1.1 | Only happy paths. Add: "Given a signed-in user with a valid session, when visiting `/`, then they are routed per FR26" (or state N/A). |
| US1.6 | "Given no refresh cookie, or an expired or revoked one, when the app starts, then the login screen shows with no error toast and no redirect loop." |
| US3.1 | "Given a user who has not completed the questionnaire, when opening the requirements URL, then they are routed per FR26." |
| US3.2 | "Given an angle not in the configured set, when uploaded, then 422." Plus the hostile-upload AC (see C1). |
| US5.2 | "Given user B's job ID, when polled by A, then 404." "Given a poll network error, when retried, then the screen keeps polling without clearing auth." |
| US5.9 | Idempotent retry: no more than 13 visuals after a partial failure. The generated aging images are +3 / +5 / +10 only ("current" is not generated). |
| US6.1 | "Given assembly fails, when checked, then no partial report is published (status is not `published`)." |
| US6.3 | "Given a feature with no mesh measurement (Hair, Skin, Neck), when its metric table renders, then it shows the not-available state rather than an error." Also: "metric tables exist for all 11 features" (FR20.1). |
| US7.1 / US7.2 | "Given no published report, when Home is requested, then the user is routed per FR26." |
| US7.4 | "Given an empty message, or one over the configured maximum length, when sent, then it is refused with 422 and no AI call is made." |
| US8.1 / US8.3 | Billing: the empty-history state. Only the caller's own payments are listed. |

### 5. Independence (INVEST "Independent")

- **US4.3** asserts 402 on report, visuals, chat and PDF endpoints that are built in later stories (US6.x, US7.x). As written, it cannot pass until those stories exist. Proposal: US4.3 delivers the shared payment-gate dependency plus a **gated-route registry test** ("every route registered as gated returns 402 for an unpaid user"). Each later gated-endpoint story (US5.1, US6.2, US6.4, US7.1, US7.3, US7.4, US8.3 if gated) then gets a tests-first AC: "unpaid → 402". Also add **PDF** and **Home summary** to AC4.3.1's list, which currently omits them.
- US9.5 has the same pattern for cross-user access. Each owner-scoped endpoint story adds its own cross-user AC, and US9.5 owns the helper plus a registry test.

### 6. Coverage gaps: requirements with no AC

**FR sub-requirements**

| ID | Gap | Proposed home |
|---|---|---|
| FR2.2 | "OTP stored hashed" has no AC (project Forbidden: never plaintext) | US1.3: "Given an issued OTP, when the `otp_records` row is read, then no column equals the plaintext code." |
| FR4.1 | The 15-min access and **7-day refresh** lifetimes are not asserted | US1.6/US1.8: "refresh at day 7 + 1 s returns 401"; "access token `exp` = iat + 15 min". |
| FR5.1 | The reset code reuses the FR2.2 expiry and lockout rules; no AC | US1.10: parametrise the US1.3 lockout tests with purpose `password_reset`. |
| FR10.3 | Content-decode validation, size and pixel limits, re-encode, **EXIF/GPS strip**: phase only, no AC. The posture's hostile-upload list is untested. | New AC in US3.2: "non-image / polyglot / oversize / decompression-bomb → rejected"; "a stored photo has no EXIF (GPS) tags." |
| FR11.1 | "never blocks a single upload" has no negative AC | US3.4: "Given only two angles have passed, when the status is read, then no identity result exists and upload is not blocked." |
| FR13.2 | 402 re-check at analysis start | US5.1 (see §2). |
| FR15.2 | Measurement citation and personalisation | US5.5 AC5.5.4 (see §3). |
| FR16.1 | **Dimorphism sliders + summary, prototypicality score, face-shape wireframe data** have no computation AC (only symmetry and thirds) | US5.7: add AC5.7.4 / AC5.7.5 for fixture-based output shape and range. |
| FR20.1 | Metric tables for all 11 features | US6.3 (see §4). |

**NFR**

| ID | Gap | Proposed home |
|---|---|---|
| NFR2 | The **AI text vendor base URL / key / model** switch has no AC (image vendor, price and angle set are covered) | US0.5: "Given a changed text-vendor base URL and model, when the narrative step runs against a recorded fake, then the request goes to the configured URL and model." |
| NFR3.1 | See B9. | US5.2 |
| NFR4 | **JWT algorithm allow-list** (for example `alg: none` or a mismatched alg is rejected); **password hashing** (stored hash is Argon2id or bcrypt, never plaintext); **rate limits on signup, login, OTP verify and forgot-password** (only resend is covered); **CSP header present** | US9.5 (alg), US1.2 (hash), US1.2/1.5/1.10 (limits, pattern as AC1.4.3), US0.3 (CSP header AC). |
| NFR4 logging | AC0.2.3 "never contain" needs a method | "Given sentinel values for password, OTP, token, `reset_token`, Stripe signature and health answer passed through each auth or pipeline flow, when captured logs are scanned, then no sentinel appears." |
| NFR5.2 | ZDR request recorded as a deployment prerequisite | A documentation AC in US0.1 README, or pending list (not automatable). |
| NFR5.3 | "Encrypted at rest" (AC0.5.3) has no method | "Given a stored file, when the raw bytes are read from the backing store, then they do not equal and do not contain the plaintext image." |
| NFR8 | **Photo-upload rate limit per user** has no AC | US3.2. |
| NFR8 / AC0.5.2 | "no live call is made" needs an enforcement mechanism | "Outbound network is blocked in the per-PR test run (for example a socket-blocking plugin); any attempt fails the test." |
| NFR9 | Backend import-linter layering contract (only the FE lint is an AC); ADRs; docstrings | US0.4: "a service importing `fastapi`, or a route importing a repository, fails CI." |
| NFR10 | The **90% floor** for auth, payment-gate and identity-check code (backend) and auth store / API client (frontend), and the **requirement-traceability CI check** (fail on an unmapped `FR-*`/`AUTH-*`/`BR-*` ID). Neither has an AC; AC0.1.4 covers only 80%. | US0.1: AC0.1.5 (90% per-module floor) and AC0.1.6 ("Given a client `FR-*`/`AUTH-*`/`BR-*` ID with no tagged test and not on the reviewed pending list, when CI runs, then the build fails"). |
| NFR11 | "No deployment jobs" | US0.1: "the workflow contains no deploy job" (phase text already says so; promote it to an AC). |
| FE-002 | The auth store does not import Zustand `persist` | US1.6: lint AC. |

**Client IDs not cited by any story** (the traceability CI check will flag these):
- `AUTH-001`, `AUTH-005`: cite in US0.4 (custom backend auth; a dependency check that no third-party auth SDK is present).
- `AUTH-004`: cite in US1.3 / US1.8 / US9.5.
- `AUTH-009`: add to US0.4 an AC that `users.role` exists and defaults to `"user"`.
- `FE-001`, `FE-003`: cite in US0.4 / US1.6 (a single Zustand store).
- `BR-006`, `BR-007`: add to the **reviewed pending list** with the reason "commercial rule, not automatable". This is already anticipated in `team.md`.

### 7. Traceability.json note

Several ACs combine a happy path and an error path in one ID (AC4.1.2, AC1.10.2, AC9.5.3). Splitting them (see §3) keeps one AC mapped to one test tag, which is what the `traceability` sensor and the CI tag check both count.

## Positions
- AGREE: Given/When/Then format and the `AC{group}.{seq}.{n}` IDs — they are consistent across all 56 stories and suit requirement-ID test tagging.
- AGREE: tests-first markers on US1.3, US1.8, US1.10, US3.4, US4.1, US4.3 and US5.6 — they match the affirmed ordering exactly.
- AGREE: landmark-coordinate fixtures (US3.4, US5.4, US5.7), and fakes and recorded responses for vendors — they match the posture and keep CI deterministic.
- OBJECT: AC3.4.2 as written — without an 8-row expected-output oracle the table test has nothing to assert, and two rows (2 and 5) are ambiguous in `BR-005` and must be raised to the human, not decided silently.
- OBJECT: US5.1 not marked tests first — it contains the 402/409 re-check at analysis start, which the affirmed ordering names.
- OBJECT: AC3.5.3 "may skip review", AC1.2.3 undefined password rules and AC1.10.2 "about 10 minutes" — they have no pass/fail criterion (inception Requirements Quality guardrail).
- OBJECT: the live-model and visual-quality ACs (AC7.4.1, AC7.5.1, US5.8 "realistic") presented as automated ACs — per the posture they belong in the opt-in evaluation suite or manual review.
- OBJECT: coverage gaps NFR2 (text vendor), NFR4 (JWT alg, password hash, auth-endpoint rate limits, CSP), NFR10 (90% floor, traceability check), FR10.3 (hostile upload / EXIF), FR16.1 (dimorphism, prototypicality) and the uncited IDs `AUTH-001/004/005/009`, `FE-001/003`, `BR-006/007` — each requirement needs at least one AC or a pending-list entry.
- OBJECT: US4.3 as a single 402 matrix over endpoints from later stories — it breaks story independence; use a gated-route registry test plus a per-endpoint 402 AC.
