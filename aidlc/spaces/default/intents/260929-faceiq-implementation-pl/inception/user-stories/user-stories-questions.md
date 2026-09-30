# User Stories — Story Plan and Questions

## Story Plan (draft)

- **Input:** `inception/requirements-analysis/requirements.md` (FR1–FR26, NFR1–NFR12), traced to `client_requirements.md` IDs.
- **Format:** "As a [persona], I want [goal], so that [benefit]", following INVEST. Each story gets a stable `US{group}.{seq}` ID, and each criterion a Given/When/Then `AC{group}.{seq}.{n}` ID (inception rules require Given/When/Then).
- **Priority:** MoSCoW per story. The MVP boundary is decided later, in Delivery Planning.
- **Review:** the designer, developer and quality engineer each review the draft independently, then the product lead reviews it.

The questions below settle personas, how stories are split, how big they are, and how the "phases for implementation" you asked for are expressed.

Fill each `[Answer]:` with a letter, or `X` plus your own text.

## Q1 — Personas

The client names one functional role, the End User (section 3). The Admin is deferred (section 2.2, `AUTH-009`).

A. One End User role with three behavioural personas: a goal-driven self-improver; a user with elevated appearance-related distress, for whom tone and safety rules matter (`FR-008`(e), `FR-004`); and a returning user who comes back to the report, visuals and chat. The Admin is noted as a future persona only, with no stories (recommended)
B. A single End User persona only
C. End User plus a fully described Admin persona (still no Admin stories, since out of scope)
X. Other (please specify)

[Answer]: A

## Q2 — How stories are grouped

A. By user-journey epics in the WF-001 order: Account & Session, Consent & Onboarding, Photos & Validation, Payment, Analysis Pipeline, Report, Post-Analysis Experience, Settings, App-wide. Each epic is split into vertical-slice stories (recommended)
B. By feature area only, without journey ordering
C. By persona
X. Other (please specify)

[Answer]: A

## Q3 — Story size

A. Small stories, each about 1–3 developer-days, giving roughly 45–60 stories (recommended: fits INVEST and the short-lived-branch practice)
B. Medium stories, each about 3–5 developer-days, giving roughly 25–35 stories
X. Other (please specify)

[Answer]: A

## Q4 — "Each story has phases for implementation"

You asked for each story to have phases for implementation. How should that look?

A. Every story lists its own implementation phases in order, for example: Phase 1 data model and migration; Phase 2 backend service and API with tests; Phase 3 frontend UI and state with tests; Phase 4 end-to-end test and docs. Only the phases a story needs are listed. Units Generation and Delivery Planning later group and sequence stories into build phases (recommended)
B. Stories stay purely user-facing; implementation phases appear only later, in Units Generation and Delivery Planning
C. No per-story phases; stories are grouped into release phases (Phase 1 foundation, Phase 2 core journey, ...) in this document
X. Other (please specify)

[Answer]: A

## Q5 — Priorities

The client defines scope as a fixed feature list (section 2.1, `BC-006`).

A. Every client in-scope feature is **Must Have**. Delivery-team additions are **Must Have** when they protect security, privacy or payment (consent, rate limits, enumeration protection), otherwise **Should Have**. Deferred items (section 2.2) are **Won't Have** (recommended)
B. Everything is **Must Have**
X. Other (please specify)

[Answer]: A

## Q6 — Technical enabler work

Some needed work is not a user story: repository and CI setup, the thin end-to-end login slice (walking skeleton), and the pluggable vendor adapters.

A. Include it as clearly marked **Enabler** items (not "As a user" stories) with their own acceptance criteria and phases, so the plan covers it (recommended)
B. Leave enabler work out of stories; Units Generation and Delivery Planning cover it
X. Other (please specify)

[Answer]: A

## Follow-up Questions from the Story Review

The designer, developer and quality engineer reviewed the draft (`contributions/`). Where both options are legitimate, the choice is yours. Answers are recorded as **[Decided by delivery team]**, and the client-facing ones are also listed as client confirmations.

## Q7 — Identity check: two photo patterns the client rule leaves ambiguous

`BR-005` says the photo with the fewest matches is flagged, and front is the reference whenever the vote is ambiguous. Two of the 8 match patterns are not settled by that: (row 2) front matches both side photos but the two side photos don't match each other; (row 5) the two side photos match each other but neither matches front.

A. Front is always the reference and is never flagged. Row 2 → nothing flagged (both side photos match front). Row 5 → both side photos flagged. Also listed as a client confirmation (recommended: consistent with `BR-005` item 3, which keeps front when all three differ)
B. Row 2 → both side photos flagged. Row 5 → front flagged (pure match-count vote)
X. Other (please specify)

[Answer]: A

## Q8 — Accounts that signed up but never verified their email

Requirements don't cover a user who completed signup step 1 but never entered the code.

A. Treat it as unfinished signup. Signing up again with that email, or logging in with the correct password, sends a fresh verification code, and verifying it completes the account. Verified accounts keep the neutral "existing email" behaviour (recommended)
B. Unverified accounts can only continue through the original code. After it expires, the user must wait for cleanup and sign up again
X. Other (please specify)

[Answer]: A

## Q9 — Story count after splitting

The developer estimates 8 stories exceed the agreed 1–3 day size and proposes splitting them, and also proposes new enablers (rate limiting, one-command local setup, test-data seeding, a computer-vision feasibility spike, a public config endpoint). That gives about 65 stories, above the "roughly 45–60" in Q3.

A. Accept the splits and enablers; the 1–3 day size rule wins over the rough count (recommended)
B. Keep about 56 stories and relabel the large ones as 3–5 day stories
X. Other (please specify)

[Answer]: A

## Q10 — Accessibility, notifications and confirmation dialogs

Should these be separate stories or a "Definition of Done" that every screen-building story must meet?

A. Definition of Done: every story with a UI part passes automated accessibility checks and a keyboard pass, and uses the shared notification and confirmation conventions. Keep one final app-wide accessibility audit story (recommended: accessibility can't be cut at the MVP line)
B. Keep them as separate stories, as drafted
X. Other (please specify)

[Answer]: A

## Q11 — Changing photos or answers after analysis starts

Nothing says whether photos or questionnaire answers can change after analysis starts. Changing them would break "one report per user, matching its analysis" (`FR-014`).

A. Inputs lock once analysis starts; photo and questionnaire screens then send the user to progress or Home (recommended)
B. Inputs stay editable, but changes have no effect on the existing report
X. Other (please specify)

[Answer]: A

## Q12 — Which actions need the confirmation dialog

`BR-010` requires confirmation for every destructive or irreversible action. Starting analysis uses up the one paid report, and retaking a photo replaces the old one.

A. Logout and Start Analysis ask for confirmation. Photo retakes don't (recommended)
B. Logout, Start Analysis and retaking a photo that had already passed
C. Logout only
X. Other (please specify)

[Answer]: A

## Q13 — Leaving the questionnaire part-way

The questionnaire has 23 branching questions.

A. Answers save as the user goes, and they resume at the first unanswered question. Changing an earlier branching answer discards answers on the branch no longer taken (recommended)
B. No saving: leaving restarts the questionnaire
X. Other (please specify)

[Answer]: A

## Q14 — Name, user menu and Settings before the report exists

`FR-002` says the full name appears in a greeting and user menu, but no story covers it, and there is no way to log out during onboarding.

A. Add a story: every signed-in screen shows the user's name and a menu with Settings and Logout. Settings is reachable at any time; Billing shows "No payments yet" when empty (recommended)
B. Add the name and menu, but with Logout only before the report exists; Settings only after the report
X. Other (please specify)

[Answer]: A

## Q15 — Other sessions after a password change

`AUTH-016` keeps the current session signed in after a password change. It doesn't say what happens to the user's other devices.

A. Sign out all other devices; keep the current one (recommended: a common protection when a password is changed)
B. Keep all sessions signed in
X. Other (please specify)

[Answer]: A

## Q16 — The "working Stripe option" in Billing

`FR-021` asks for a working Stripe option in Billing, but there is one payment per report and no subscription.

A. Show the Stripe payment used (card brand and last 4 digits) and a link to its Stripe receipt; never start a second purchase. Also listed as a client confirmation (recommended)
B. Show only a Stripe label as the active method, with no link
X. Other (please specify)

[Answer]: A

## Q17 — Chat daily limit details

A. Failed replies don't count toward the limit; the limit resets at midnight UTC (recommended: simplest to test and explain)
B. Failed replies don't count; the limit resets at midnight in the user's own time zone
X. Other (please specify)

[Answer]: A

## Q18 — Consent records

FR7 names three subjects: face images and biometric signature, health answers, and sending both to the AI vendor.

A. Two consents: (1) face images and biometric signature, including sending them to the AI vendor; (2) health answers, including sending them to the AI vendor. The consent step comes before the questionnaire (recommended)
B. Three separate consents
X. Other (please specify)

[Answer]: A

## Q19 — Password rules

No password rules are defined anywhere.

A. Minimum 10 characters, maximum 128, no forced character mix, paste allowed, and a check against a list of common passwords; the values are configuration (recommended: current NIST-style guidance)
B. Minimum 8 characters, with at least one letter and one number
X. Other (please specify)

[Answer]: A

## Consolidated Summary Confirmation

Summary of answers:
- Personas: one End User role with three behavioural personas (goal-driven self-improver, user with elevated appearance-related distress, returning user); Admin noted as future only, no stories (Q1 A)
- Grouping: user-journey epics in WF-001 order, each split into vertical-slice stories (Q2 A)
- Size: small stories of about 1-3 developer-days (Q3 A); oversized stories are split and new enablers added, giving about 65 stories (Q9 A)
- Implementation phases: every story lists its own ordered implementation phases (only those it needs); later stages sequence stories into build phases (Q4 A)
- Priorities: client in-scope features Must Have; delivery-team additions Must Have when protecting security, privacy or payment, otherwise Should Have; deferred items Won't Have (Q5 A)
- Enablers: repo/CI, the thin login slice, adapters, job runner, rate limiting, one-command local setup, test seeding, CV feasibility spike and public config as marked Enabler items (Q6 A, Q9 A)
- Identity vote: front is always the reference and never flagged; ambiguous row 2 flags nothing, row 5 flags both side photos; listed for client confirmation (Q7 A)
- Unverified accounts: signing up again or logging in with the correct password sends a fresh verification code (Q8 A)
- Accessibility, notifications and confirmations: a Definition of Done for every UI story, plus one final app-wide accessibility audit story (Q10 A)
- Inputs lock once analysis starts (Q11 A)
- Confirmation dialog for Logout and Start Analysis; photo retakes need none (Q12 A)
- Questionnaire saves progress and resumes; changing a branching answer discards the abandoned branch (Q13 A)
- New story: name greeting and user menu with Settings and Logout on every signed-in screen; Settings reachable any time (Q14 A)
- Password change signs out all other devices and keeps the current session (Q15 A)
- Billing shows the Stripe payment method (brand and last 4) and receipt link, never a second purchase; listed for client confirmation (Q16 A)
- Chat daily limit: failed replies don't count; resets at midnight UTC (Q17 A)
- Two consent records (images and biometric signature; health answers), each including AI-vendor sharing, shown before the questionnaire (Q18 A)
- Password rules: 10-128 characters, no forced character mix, paste allowed, common-password blocklist, configurable (Q19 A)

Does this all look correct before I generate the artifact?

- Looks correct
- Request changes

[Answer]: Looks correct
