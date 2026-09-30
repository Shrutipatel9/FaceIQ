**Collaborator:** aidlc-design-agent

## Contribution

Round 1 support review from the UX and persona-fidelity angle. It is based on `stories.md` and `personas.md` (drafts), `requirements.md` and `client_requirements.md`. Findings are grouped so the lead can merge them story by story. Each proposed acceptance criterion (AC) is written in Given/When/Then form with a suggested ID. Items the client did not state are labelled **[Recommendation]**. Items that need a human or client decision are labelled **[Open question]**, and no answer has been invented for them.

Overall: the draft follows the WF-001 journey order closely. The gating stories (US3.4–US3.6, US4.3, US5.1) are strong, and the draft generally avoids assuming UI content beyond the pending Report & UI Specification. The main gaps are:
- **Missing states:** loading, empty, error and "waiting" states across the photo, payment, analysis, image and chat stories.
- **Dead ends:** a few points in the journey where the user has no way forward.
- **Missing requirement:** the full-name greeting and user menu (`FR-002`) has no story.
- **Accessibility:** it sits almost entirely in one **Should** story (US9.3), so it could be cut at the MVP boundary.

### A. Persona fidelity

1. **US9.3: wrong persona.** P2 Jordan is defined by appearance-related distress (`personas.md`), not by use of assistive technology. Recommendation:
   - Set the persona to "All personas" and keep the actor "a user who relies on a keyboard or screen reader".
   - Add one line to `personas.md` → Overview: "Any persona may use assistive technology, a phone-sized screen or a slow connection. These are cross-cutting conditions, not a separate persona (NFR6)."
2. **US4.3: persona and actor mismatch.** The story is written "As the business…" but tagged P1 Alex. Either set the persona field to "— (business rule, `BR-001`)", or reframe it for P1: "As a buyer, I want the report to unlock only after my payment is confirmed, so that I never see a partial or broken report." The inception rule requires the actor to be named consistently.
3. **US2.2: add P2 as a secondary persona.** The questionnaire contains the medical, medication, allergy and appearance-thought frequency questions (`FR-003`), and those are exactly where P2's safety rules start. No behaviour change is needed, only the persona tag.
4. **US3.3 (P2 lens), new AC [Recommendation]:**
   - AC3.3.5 Given any photo rejection, when the reason is shown, then the message describes the condition of the photo (lighting, framing, angle, obstruction) and never describes the person's face or appearance.
   - Rationale: P2 sees validation messages before any of the AI tone rules (FR15.3) apply.
5. **US7.5 (P2 lens), new AC [Recommendation]:**
   - AC7.5.3 Given a declined medical question, when the reply is shown, then it is non-judgmental, recommends a qualified professional, and invites a question about the report instead.
   - Rationale: FR23.2 plus the spirit of `FR-008`(e). The exact copy stays with the prompt design.

### B. Missing story: greeting and user menu (`FR-002`)

`FR-002` states: "The full name is used for the in-app greeting and user menu." No story covers this. The draft also offers no way to log out or reach Settings **before** the report exists (onboarding, photos, payment, analysis in progress). US7.2 only covers the post-analysis header, and FR26 (US3.6) keeps users in onboarding until the report is published.

Proposed **US9.7 — See my name and reach my account from any signed-in screen**
**Persona:** P3 Sam · **Priority:** Must · **Traces:** FR2 (`FR-002`), FR4.6, FR24 (`FR-021`), FR25.2 · **Depends on:** US1.6, US9.2

As a signed-in user, I want to see my name and reach Settings or Logout from any screen, so that I know which account I'm in and can leave at any point in the journey.
- AC9.7.1 Given any signed-in screen, including onboarding, photos, payment and analysis progress, when it renders, then a user menu shows the user's full name from `/auth/me`.
- AC9.7.2 Given the user menu, when opened, then it offers Settings and Logout, and Logout uses the shared confirmation dialog (US1.9).
- AC9.7.3 Given the user menu, when operated by keyboard, then it opens with Enter or Space, closes with Escape, and returns focus to its trigger.

Implementation phases: 1. P-UI: user-menu component in the authenticated layout. 2. P-E2E: logout from an onboarding screen.

**[Open question]:** is Settings (Account Info, Password, Billing) reachable before a report exists? `FR-017` places Settings in the post-analysis header only. If Settings is reachable earlier, US8.3 needs AC8.3.3 (below).

### C. Journey edge cases and dead ends

1. **US2.1: declining consent leaves the user stuck.** AC2.1.3 blocks the user but gives them no way forward. Add:
   - AC2.1.4 Given a user who declined consent, when they return to the app from any entry point, then they land on the consent step (FR26 order) and can review and accept it. No other step is reachable.
   - **[Recommendation]:** show consent as one step **before** the questionnaire starts, rather than interrupting the questionnaire at its health section. This matches FR26's order (consent → questionnaire) and avoids breaking P2's flow part-way through.
2. **US2.3: the disclaimer is a hard gate with no exit.** `BR-003` makes the checkbox mandatory. A user who cannot truthfully confirm "no Body Dysmorphic Disorder–related concerns" has no path forward.
   - **[Open question] for the client:** what should that user see? Options include a supportive message and a pointer to professional help. The copy belongs to the pending Onboarding Questionnaire Specification. No behaviour is invented here.
   - Accessibility AC: AC2.3.4 Given the disclaimer is unchecked, when the user activates Submit, then an inline message linked to the checkbox explains why submission is blocked. The button is not silently disabled.
3. **US2.2: long questionnaire with no resume or back navigation.** The questionnaire has 23 branching questions. Nothing says whether answers survive leaving mid-way, or what happens when the user goes back and changes a branching answer.
   - **[Open question]:** are partial answers saved?
   - **[Recommendation]** ACs, if the answer is yes:
     - AC2.2.4 Given a user leaves mid-questionnaire, when they return, then they resume at the first unanswered question with earlier answers kept.
     - AC2.2.5 Given the user goes back and changes a branching answer, when they continue, then answers on the branch no longer taken are discarded and not submitted.
   - Also AC2.2.6 Given the questionnaire, when in progress, then a text progress indicator shows the position, for example "Question 5 of about 23". [Recommendation; Qoves benchmark `BC-007`.]
4. **Photo and answer editing after payment or after analysis start: [Open question].**
   - `BR-005` item 8 re-checks identity at analysis start, which implies photos can still change after payment.
   - Nothing says whether photos or answers can change **after** analysis starts or after the report is published. Changing them would break "one report per user, 1:1 with the analysis result" (`FR-014`).
   - Proposed AC for US3.6, pending the answer: AC3.6.4 Given analysis has started, when the user opens a photo or questionnaire URL, then they are routed to the progress screen or the Home Overview and cannot change inputs.
   - This needs a human decision before it is written as a rule.
5. **Start Analysis as an irreversible action (`BR-010`): [Open question].** Starting analysis uses up the single paid report. `BR-010` covers "every destructive or irreversible action". Does Start Analysis need the shared confirmation dialog? `FR-015` wants the step to be explicit and visible, and a confirmation adds friction. Please raise this with the human rather than deciding it in the stories.
6. **Retaking a photo that already passed: [Open question].** A retake replaces the existing photo (`BR-005`). Does replacing a photo that already passed count as destructive under `BR-010`? Recommendation: no confirmation for flagged photos. The case of a photo that already passed needs a decision.
7. **US3.6: missing entry-point states.** Add:
   - AC3.6.5 Given analysis is in progress, when the user signs in or reloads, then they land on the progress screen showing the current step.
   - AC3.6.6 Given analysis failed after retries, when the user signs in, then they land on the progress screen with the error and "Try again" (US5.3).
8. **US3.6 and FR26: deep links for completed users are ambiguous.** FR26 says every entry point goes to the "first incomplete step". For a user with a published report, that is the Home Overview. Does a direct link to `/report`, `/visuals`, `/chat` or `/settings` get honoured, or redirected to Home?
   - Recommendation: AC3.6.7 Given a user with a published report, when they open any post-analysis route directly, then that route opens (FR26 only redirects users away from steps they have not reached).
   - Please confirm this with the human, because it interprets FR26.
9. **US1.5 and US1.2: signed up but never verified.**
   - A user who completed step 1 of signup but never verified the OTP then logs in. Add AC1.5.4: Given an unverified account with correct credentials, when the user logs in and verifies the OTP, then the account becomes verified and the user continues to consent.
   - Also AC1.2.2 sends a "someone tried to sign up" email for an existing email. **[Open question]:** if that existing account is still unverified, should the email instead carry a fresh verification code? Otherwise the real owner cannot get past signup.
10. **US1.1: landing page for a signed-in user.** Add AC1.1.4: Given a signed-in user with a restored session, when they open the landing page or click "Get started", then they are taken to their next incomplete step (FR26). [Interpretation of "every entry point"; confirm with the human.]
11. **US1.7: session-expired feedback.** Add AC1.7.5: Given refresh returns 401, when the user is redirected to login, then a toast explains that the session ended (`BR-009`), and any unsaved form input on screen is not silently lost without that message.

### D. Loading, empty and error states (missing ACs)

**US3.2 — Upload or capture**
- AC3.2.4 Given the user chooses camera capture, when camera permission is denied or no camera exists, then an inline message explains this and offers file upload for the same angle (NFR6 "camera capture fallbacks").
- AC3.2.5 Given an upload is in progress, when the angle tile renders, then it shows a busy state that screen readers can perceive (for example "Uploading front photo"), and the user cannot submit the same angle twice.
- AC3.2.6 Given an upload fails because of a network error or 5xx, when handled, then an error toast shows, the user stays signed in (`FE-006`), and they can retry that angle.
- Note: per-angle guidance or example images come from the pending Photo Capture Specification. Do not assume the Qoves overlay design.

**US3.3 — Per-photo feedback**
- AC3.3.6 Given each per-photo check (resolution, brightness, face count, face proportion, occlusion, pose, readability), when it fails, then the message names the failed condition in plain language and points to the matching item on the 7-point checklist (US3.1). The draft's ACs cover only three of the seven checks.
- AC3.3.7 Given a photo is being validated, when the result is pending, then a "Checking photo" state is shown and announced.

**US3.4 — Identity review screen**
- AC3.4.5 Given the identity check is running, when the review screen shows, then a "Checking your photos match" state is shown, and Continue stays unavailable until the result arrives.
- AC3.4.6 Given a flagged photo, when highlighted, then the flag is conveyed by text and an icon as well as colour (WCAG 1.4.1). The reason Continue is unavailable is programmatically linked to the button (for example `aria-describedby`).

**US3.5 — Testability fix.** AC3.5.3 says "then it **may** skip review", which cannot be tested. Restate it to match `BR-005` item 6 exactly: "Given the review screen opens on a fresh page load, when the stored set is already complete and consistent, then the skip decision is made once from that load. No later retake result ever triggers navigation." Whether the skip happens at all is not client-stated. Leave that to the pending Photo Capture Specification.

**US4.1 and US4.2 — Payment**
- AC4.1.4 Given the user cancels or abandons Stripe Checkout, when they return via the cancel URL, then they are back on the payment screen with a neutral message and can pay again. No duplicate paid payment is possible.
- AC4.1.5 Given a user whose payment is already confirmed, when they reach the payment screen by any route, then they are sent to Start Analysis and no new checkout is created.
- AC4.2.5 Given the "confirming payment" state lasts longer than a configured wait, when it is still unconfirmed, then the page says confirmation is still in progress. The user can leave, and when they return, FR26 routing reflects the real status. [Recommendation; the wait length is configuration.]
- AC4.2.6 Given the webhook confirms payment, when the return page next polls, then a success toast shows (`BR-009`) and the Start Analysis action becomes available.

**US5.1 and US5.2 — Start and progress**
- AC5.2.4 Given the progress screen, when the step changes, then the change is announced once through a polite live region. It is not re-announced on every poll.
- AC5.2.5 Given the user closes the tab during analysis, when they return, then the progress screen resumes (links to AC3.6.5).
- Note (scope, `§2.2`): email notifications are out of scope. The progress screen must **not** promise "we'll email you when it's ready".
- **[Recommendation]:** tell the user the analysis can take several minutes (NFR3.1 target of 10 minutes at p95). The exact copy belongs to the Report & UI Specification.
- AC5.2.2 auto-routes to Home on completion. **[Recommendation]:** announce "Your report is ready" before navigating, or show a button, so the change of context is not unexpected (WCAG 3.2).

**US6.4 — PDF download**
- AC6.4.4 Given the PDF is being generated on first download, when the user waits, then the download control shows a busy state and ignores repeat clicks.
- AC6.4.5 Given generation fails, when handled, then an error toast shows and the user can retry (`BR-009`). The draft mentions only a success toast in its phases.

**US7.3 and US9.4 — Images (24 generated images plus the user's photos)**
- AC7.3.3 Given a signed image link expires while the page is open, when the image is next displayed, then a fresh link is fetched transparently, with no broken image.
- AC7.3.4 Given an image fails to load, when rendered, then a placeholder with a text description shows instead of a broken-image icon.
- AC7.3.5 Given every photo and generated image, when rendered, then it has a descriptive text alternative, for example "AI-generated hairstyle variation 2 of 5" or "Your front photo".
- AC7.3.6 Given the healthy-aging stack, when rendered, then the cards are labelled relative to the current photo ("Current", "+3 years", "+5 years", "+10 years"). No absolute age is shown unless the pending specifications supply the user's age. "Age 28" in `FR-020` describes the reference video and must not be hardcoded.
- AC7.3.7 Given the 4-card stack is operated by keyboard or a single pointer, when the user moves between cards, then every card is reachable without swipe-only gestures (WCAG 2.5.1).

**US7.4 — Chat**
- AC7.4.4 Given no conversation exists, when Chat opens, then a neutral empty state invites a question about the report. Any suggested prompts wait for the Report & UI Specification.
- AC7.4.5 Given a message is sent, when the reply is pending, then a "replying" indicator shows and the input prevents a duplicate send.
- AC7.4.6 Given the AI call fails, when handled, then the user's message is kept, an error toast shows, and the user can resend. **[Open question]:** does a failed reply count toward the daily cap (US7.6)? Recommendation: no.
- AC7.4.7 Given new replies arrive, when rendered, then the message list is a log region that screen readers announce politely.

**US7.6 — Chat cap**
- AC7.6.3 Given the cap is reached, when the chat renders, then the input is disabled, with the reason and reset time shown as text next to it.

**US8.3 — Billing**
- `FR-021` requires "a working Stripe option" in Billing, but the draft's "Stripe is active" does not define any behaviour. **[Open question]:** what does the Stripe option do in Billing? With one report per user and no subscription, it must not start a second purchase. It could show the payment method used or a link to a Stripe receipt. This should wait for the Report & UI Specification.
- AC8.3.2 accessibility addition: PayPal is rendered as `aria-disabled` with its "not available" status in text (not colour alone), so a screen reader announces "PayPal, not available".
- AC8.3.3 (only if Settings is reachable before payment, see B) Given no payments, when Billing opens, then an empty state reads "No payments yet".

### E. Accessibility (WCAG 2.1 AA, affirmed in NFR6)

1. **Priority and Definition of Done.** US9.3 is **Should**, which follows the Q5 rule literally (a delivery-team addition that does not protect security, privacy or payment). The risk is that NFR6 was affirmed for **all screens**, and a Should story can be dropped at the MVP boundary.
   - Recommendation, which keeps the Q5 rule intact: add an explicit cross-cutting **Definition of Done** to the Overview: "Every story with a P-UI phase passes axe checks with no violations and a manual keyboard pass for its screens (NFR6)."
   - US9.3 then keeps only the app-wide audit and the documentation. Several stories already do this ad hoc (US1.1, US3.1, US6.3, US7.1), and the DoD makes it uniform.
2. **Extend US9.3 with the AA criteria most at risk in this product:**
   - AC9.3.3 Given every screen at 320 CSS px width and at 200% text zoom, when viewed, then there is no loss of content or function and no two-dimensional scrolling (WCAG 1.4.4, 1.4.10). This also makes the phone-width capture flow a requirement, not a nice-to-have.
   - AC9.3.4 Given text and meaningful non-text UI (focus rings, chart lines, slider handles, status pills), when measured, then contrast is at least 4.5:1 for text and 3:1 for non-text (WCAG 1.4.3, 1.4.11).
   - AC9.3.5 Given every form error (signup, login, OTP, change password, disclaimer), when shown, then it is inline, linked to its field, and announced. A toast alone is never the only error channel (WCAG 3.3.1, 4.1.3).
3. **US9.1 — Toasts:**
   - AC9.1.3 Given a toast, when shown, then success toasts use a polite status region and error toasts an assertive one. Error toasts stay until dismissed or for long enough to read, and pause while hovered or focused (WCAG 2.2.1, 4.1.3).
   - AC9.1.4 Given a network error or 5xx on any action, when handled, then an error toast shows, entered form data is kept, and the user stays signed in (`FE-006`).
4. **US9.2 — Confirmation dialog:**
   - AC9.2.3 Given the dialog opens, when it renders, then focus moves into it and starts on the non-destructive action. Focus is trapped while it is open and returns to the trigger on close. The dialog is exposed as an alert dialog with an accessible name.
5. **US1.3 and US1.4 — OTP entry:**
   - AC1.3.4 Given the OTP screen, when shown, then it names the email address the code was sent to, accepts pasting the full code, and uses numeric input with one-time-code autofill hints.
   - AC1.3.5 Given a lockout, when shown, then the message states when the user can try again.
   - AC1.4.4 Given the resend countdown, when running, then its remaining time is available to screen readers without announcing every second.
6. **US1.2 — Password rules (error prevention):**
   - AC1.2.4 Given the signup form, when shown, then the password rules are visible before submission, and the password field allows paste.
7. **US7.2 — Header:**
   - AC7.2.3 Given the header, when on a section, then the current item is exposed with `aria-current="page"`.
8. **US6.3 and US7.1 — Charts:** the sliders, radar chart and face-shape wireframe already need text alternatives (AC6.3.3, AC7.1.2).
   - Recommendation: the alternative should be an equivalent data table or list of the values, not a one-line summary, so the six radar values and slider positions are all available.
   - Interactive sliders, if they are interactive, must be keyboard-operable. If they are display-only, they must not be exposed as editable controls.
9. **US1.1:** the reduced-motion AC is good. No change.

### F. UI content assumed beyond the pending Report & UI Specification

The draft is mostly careful. Four points to tighten:
- **AC5.7.1** "a label from the defined set": the client gives only one example label ("Quite Symmetric", `FR-018`). State that the label set comes from the Report & UI Specification. Do not invent labels.
- **AC6.3.2** asserts the face-shape wireframe appears. `FR-018` confirms the wireframe exists, but its content is **unconfirmed**. Add: "wireframe content per the Report & UI Specification", mirroring AC5.7.3.
- **AC7.1.1** "key stats": the content is undefined (`FR-017`). Add "as defined by the Report & UI Specification", as AC7.1.3 already does for Protocol.
- **US6.2 before/after:** which user photo is the "before" is not stated (`FR-010` says "the user's photo"). Mark it pending the specification. The front photo is the likely default, labelled [Assumption].

Nothing in the draft wrongly assumes the MyFace or Qoves layouts. Keep layout-dependent phases tagged "pending the Report & UI Specification", as US6.3 and US7.3 already are. Apply the same tag to the P-UI phases of US7.1 and US8.3.

### G. Summary of suggested new IDs

| Story | New ACs |
|-------|---------|
| US1.1 | AC1.1.4 |
| US1.2 | AC1.2.4 |
| US1.3 | AC1.3.4, AC1.3.5 |
| US1.4 | AC1.4.4 |
| US1.5 | AC1.5.4 |
| US1.7 | AC1.7.5 |
| US2.1 | AC2.1.4 |
| US2.2 | AC2.2.4–AC2.2.6 (pending the open question) |
| US2.3 | AC2.3.4 |
| US3.2 | AC3.2.4–AC3.2.6 |
| US3.3 | AC3.3.5–AC3.3.7 |
| US3.4 | AC3.4.5, AC3.4.6 |
| US3.5 | AC3.5.3 reworded |
| US3.6 | AC3.6.4 (pending the open question), AC3.6.5–AC3.6.7 |
| US4.1 | AC4.1.4, AC4.1.5 |
| US4.2 | AC4.2.5, AC4.2.6 |
| US5.2 | AC5.2.4, AC5.2.5 |
| US6.4 | AC6.4.4, AC6.4.5 |
| US7.2 | AC7.2.3 |
| US7.3 | AC7.3.3–AC7.3.7 |
| US7.4 | AC7.4.4–AC7.4.7 |
| US7.5 | AC7.5.3 |
| US7.6 | AC7.6.3 |
| US8.3 | AC8.3.3 (conditional) |
| US9.1 | AC9.1.3, AC9.1.4 |
| US9.2 | AC9.2.3 |
| US9.3 | AC9.3.3–AC9.3.5 |
| New story | US9.7 |

If the lead integrates US9.7, the story-map totals change to 57 stories and 54 Must.

**Open questions to raise with the human:**
1. Is Settings reachable before the report exists?
2. What does a user who cannot confirm the disclaimer see?
3. Are partial questionnaire answers saved?
4. Can photos or answers be edited after analysis starts?
5. Does Start Analysis need the `BR-010` confirmation?
6. Does retaking a photo that already passed need confirmation?
7. Are deep links honoured for users with a published report (FR26 interpretation)?
8. Should signup with an existing but unverified email resend a verification code?
9. What does the "working Stripe option" in Billing do?
10. Does a failed chat reply count toward the daily cap?

## Positions
- AGREE: Epic structure in WF-001 order and the three behavioural personas — they match Q1/Q2 and the single End User role in `client_requirements.md` §3.
- AGREE: Gating stories (US3.4–US3.6, US4.3, US5.1) — they carry `BR-005` items 1–8, `BR-001` and the HTTP 402/409 re-checks faithfully, including the rule that a retake never auto-navigates.
- AGREE: Handling of unconfirmed content (AC5.7.3, AC7.1.3, the P2-focused tone stories US5.5 and US7.5) — it respects `FR-017`, `FR-018` and `FR-008`(e)/(f) without inventing content.
- AGREE: US8.3 PayPal visible but disabled, and US9.1/US9.2 toasts and confirmations — faithful to `BR-009`, `BR-010` and `BR-012`, with the accessibility additions in section E.
- OBJECT: US9.3 persona assignment to P2 Jordan — assistive-technology use is a cross-cutting condition, not Jordan's defining trait. Use "All personas".
- OBJECT: Accessibility carried only by a Should story — NFR6 was affirmed for all screens, so add a P-UI Definition of Done that keeps the Q5 priority rule intact.
- OBJECT: No story covers the full-name greeting and user menu (`FR-002`), or Logout before the report exists — add US9.7.
- OBJECT: AC3.5.3 "may skip review" is not testable — restate it deterministically per `BR-005` item 6.
- OBJECT: Missing loading, empty and error states on photo upload and camera, payment cancel and delayed confirmation, analysis progress re-entry, PDF failure, image link expiry and chat failure — add the ACs in section D so each story is testable in adverse conditions.
- OBJECT: US2.1 and US2.3 dead ends (declined consent; a user unable to confirm the disclaimer) — add the re-entry AC and raise the disclaimer exit as a client question.
- OBJECT: US4.3 actor ("As the business") conflicts with its P1 persona tag — align the actor and the persona field.
