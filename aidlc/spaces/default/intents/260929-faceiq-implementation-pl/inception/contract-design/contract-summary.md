# Contract Summary — FaceIQ

## Overview

This document pins down every formal contract FaceIQ must honour before building starts. It covers three kinds of boundary: the HTTP API between the Next.js frontend and the FastAPI backend, the external boundaries (the Stripe webhook and the browser's signed-link downloads), and the in-process contracts between the 13 backend units.

- **Inputs:**
  - `unit-of-work` and `unit-of-work-dependency` (units, kinds, edges, integration points);
  - `components` (entity shapes, ownership);
  - `requirements` (auth rules, 402/404/409/422/429 conventions, NFRs);
  - answers Q1–Q6 in `contract-design-questions.md`.
- **Project rules applied:**
  - UI never calls auth APIs directly;
  - only the API client performs HTTP requests;
  - cross-user access returns 404, rate-limited requests return 429, and unpaid requests to gated routes return 402;
  - AI text is plain text;
  - no hardcoded price or model.
- **Conventions for every HTTP contract:**
  - Base path `/api/v1`. FastAPI's generated OpenAPI is the source of truth; the frontend types are generated from it and checked for drift in CI (Q1).
  - JSON field names are camelCase (Q2).
  - Authentication: `Authorization: Bearer <access token>` on protected routes. The refresh cookie is httpOnly, scoped to `/api/v1/auth`, and sent with `credentials: include`.
  - Cookie-authenticated routes (`/auth/refresh`, `/auth/logout`) also require `X-Requested-With: XMLHttpRequest` and an allow-listed `Origin`/`Referer` (`AUTH-013`).
  - Errors use one envelope (Q3):

    ```
    {"error": {"code", "message", "details", "requestId"}}
    ```

    429 responses add a `Retry-After` header (Q6).
  - Lists are cursor-paginated: `?limit=` (default 20, max 100) and `?cursor=`, returning `nextCursor`.
  - Content still pending the Report & UI Specification (key stats, protocol summary, priority features, radar axes, label sets) travels in extensible objects (`additionalProperties: true`), so filling it in later is an additive change.

## Contracts

| # | Provider Unit | Consumer | Mechanism | Owner |
|---|---------------|----------|-----------|-------|
| C1 | U1 walking-skeleton + U2 platform-services (Platform) | External: public web (frontend U1–U13) | REST/HTTP (OpenAPI) | U1 |
| C2 | U3 identity (Identity) | Frontend Auth Store via API Client (U1, U3) | REST/HTTP (OpenAPI) | U3 |
| C3 | U4 onboarding (Onboarding) | Frontend (U4) | REST/HTTP (OpenAPI) | U4 |
| C4 | U5 photos (Photos, MediaStore, JourneyStatus) | Frontend (U5, U13) | REST/HTTP (OpenAPI) | U5 |
| C5 | U6 payments (Payments) | Frontend (U6, U13) | REST/HTTP (OpenAPI) | U6 |
| C6 | U6 payments (Payments) | External: Stripe | Webhook REST/HTTP (OpenAPI) | U6 |
| C7 | U7 analysis-pipeline (AnalysisOrchestrator) | Frontend (U7) | REST/HTTP (OpenAPI), polling | U7 |
| C8 | U10 image-generation (ImageGeneration) | Frontend (U10) | REST/HTTP (OpenAPI) | U10 |
| C9 | U11 report (Report) | Frontend (U11, U13) | REST/HTTP (OpenAPI) | U11 |
| C10 | U12 beauty-assistant (BeautyAssistant) | Frontend (U12) | REST/HTTP (OpenAPI) | U12 |
| C11 | U5 photos (MediaStore) | External: browser (img/PDF download via signed link) | HTTP GET with signed token | U5 |
| C12 | U4 onboarding | U5 photos, U7 pipeline, U9 insights, U12 assistant, JourneyStatus | In-process interface (shared schema) | U4 |
| C13 | U5 photos | U6 payments, U7 pipeline, U11 report, JourneyStatus | In-process interface (shared schema) | U5 |
| C14 | U6 payments | U7 pipeline, U10 images, U11 report, U12 assistant, U13, JourneyStatus | In-process interface (shared schema) | U6 |
| C15 | U5 photos (MediaStore) | U10 images, U11 report | In-process interface (shared schema) | U5 |
| C16 | U7 analysis-pipeline | U8 engine, U9 insights, U10 images, U11 report (step implementers) | In-process step contract (shared schema) | U7 |
| C17 | U11 report | U12 assistant, JourneyStatus | In-process interface (shared schema) | U11 |
| C18 | U3 identity | All backend units | In-process dependency (shared schema) | U3 |

Pipeline data flowing through C16 (measurements, narrative, images) is carried in the AnalysisResult entity owned by U7 (`components`), so U8–U11 never call each other directly.

## HTTP Contracts (Frontend ↔ Backend)

### C1 — Platform: health, public config, error envelope

```yaml
openapi: 3.1.0
info: {title: FaceIQ Platform API, version: "1.0.0"}
paths:
  /api/v1/health:
    get:
      summary: Liveness and database connectivity
      responses:
        "200":
          description: Healthy
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: {type: string, enum: [ok, degraded]}
                  database: {type: string, enum: [ok, unavailable]}
  /api/v1/config/public:
    get:
      summary: Non-secret configuration the frontend must reflect (AC0.3.5, NFR2)
      responses:
        "200":
          description: Public config
          content:
            application/json:
              schema:
                type: object
                required: [photoAngles, price, analysisPollIntervalSeconds, passwordPolicy]
                properties:
                  photoAngles: {type: array, items: {type: string, example: front}}
                  price:
                    type: object
                    properties:
                      amountMinor: {type: integer}
                      currency: {type: string, example: USD}
                  analysisPollIntervalSeconds: {type: integer, default: 3}
                  passwordPolicy:
                    type: object
                    properties:
                      minLength: {type: integer, default: 10}
                      maxLength: {type: integer, default: 128}
components:
  schemas:
    ErrorEnvelope:
      type: object
      required: [error]
      properties:
        error:
          type: object
          required: [code, message, requestId]
          properties:
            code:
              type: string
              description: Stable machine code
              enum: [unauthenticated, forbidden, not_found, payment_required, identity_mismatch,
                     validation_failed, rate_limited, otp_invalid, otp_expired, otp_locked, otp_cooldown,
                     csrf_rejected, consent_required, inputs_locked, analysis_already_running,
                     chat_cap_reached, photo_rejected, internal_error]
            message: {type: string, description: Safe user-facing text; never reveals account existence}
            details: {type: object, additionalProperties: true}
            requestId: {type: string}
  responses:
    Unauthenticated: {description: "401", content: {application/json: {schema: {$ref: "#/components/schemas/ErrorEnvelope"}}}}
    PaymentRequired: {description: "402", content: {application/json: {schema: {$ref: "#/components/schemas/ErrorEnvelope"}}}}
    NotFound: {description: "404 (also returned for another user's resource)", content: {application/json: {schema: {$ref: "#/components/schemas/ErrorEnvelope"}}}}
    IdentityMismatch:
      description: "409 identity_mismatch; details.mismatchedAngles lists flagged angles"
      content: {application/json: {schema: {$ref: "#/components/schemas/ErrorEnvelope"}}}
    RateLimited:
      description: "429; Retry-After header in seconds"
      headers: {Retry-After: {schema: {type: integer}}}
      content: {application/json: {schema: {$ref: "#/components/schemas/ErrorEnvelope"}}}
```

### C2 — Identity: signup, OTP, login, sessions, reset, change password

```yaml
openapi: 3.1.0
info: {title: FaceIQ Identity API, version: "1.0.0"}
paths:
  /api/v1/auth/register:
    post:
      summary: Signup step 1 (FR2.1). Neutral outcome for existing emails (FR2.4); resend for unverified (Q8)
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [fullName, email, password]
              properties:
                fullName: {type: string, minLength: 1, maxLength: 200}
                email: {type: string, format: email}
                password: {type: string, minLength: 10, maxLength: 128}
      responses:
        "202": {description: "Always the same body: {otpRequired: true, purpose: signup}"}
        "422": {description: "password_policy or field validation (inline field errors in details)"}
        "429": {description: rate_limited}
  /api/v1/auth/login:
    post:
      summary: Login step 1 (FR3.1). Identical 401 for unknown email and wrong password (FR3.3)
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email: {type: string, format: email}
                password: {type: string}
      responses:
        "202": {description: "{otpRequired: true, purpose: login}; for unverified accounts purpose=signup (Q8)"}
        "401": {description: "unauthenticated (generic)"}
        "429": {description: rate_limited}
  /api/v1/auth/otp/verify:
    post:
      summary: Verify OTP for signup or login (FR2.2, FR2.3, FR3.2); issues tokens
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [email, purpose, code]
              properties:
                email: {type: string, format: email}
                purpose: {type: string, enum: [signup, login]}
                code: {type: string, pattern: "^[0-9]{6}$"}
      responses:
        "200":
          description: "Signed in. Sets httpOnly Secure refresh cookie (Path=/api/v1/auth, 7 days)"
          content:
            application/json:
              schema: {$ref: "#/components/schemas/TokenResponse"}
        "400": {description: "otp_invalid or otp_expired"}
        "429": {description: "otp_locked (Retry-After = remaining lockout seconds)"}
  /api/v1/auth/otp/resend:
    post:
      summary: Resend OTP (60 s cooldown; never resets lockout)
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [email, purpose]
              properties:
                email: {type: string, format: email}
                purpose: {type: string, enum: [signup, login, password_reset]}
      responses:
        "202": {description: Neutral outcome}
        "429": {description: "otp_cooldown or otp_locked or rate_limited, with Retry-After"}
  /api/v1/auth/refresh:
    post:
      summary: Rotate refresh cookie and issue a new access token (FR4.2, FR4.7). Requires X-Requested-With and allow-listed Origin/Referer
      responses:
        "200":
          description: New access token; refresh cookie rotated
          content: {application/json: {schema: {$ref: "#/components/schemas/TokenResponse"}}}
        "401": {description: "Expired, revoked, missing or reused (reuse revokes the family). Frontend clears auth ONLY on this"}
        "403": {description: csrf_rejected}
  /api/v1/auth/logout:
    post:
      summary: Revoke current session and clear cookie (FR4.6). CSRF headers required
      responses:
        "204": {description: Logged out}
  /api/v1/auth/me:
    get:
      summary: Current user (AUTH-014)
      security: [{bearer: []}]
      responses:
        "200":
          description: User
          content: {application/json: {schema: {$ref: "#/components/schemas/User"}}}
        "401": {description: unauthenticated}
  /api/v1/auth/password/forgot:
    post:
      summary: Request reset code (FR5.1). Always neutral (FR5.4)
      requestBody:
        content: {application/json: {schema: {type: object, required: [email], properties: {email: {type: string, format: email}}}}}
      responses:
        "202": {description: Neutral outcome}
        "429": {description: rate_limited}
  /api/v1/auth/password/verify-code:
    post:
      summary: Verify 6-digit reset code; issues single-use resetToken (FR5.2)
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [email, code]
              properties:
                email: {type: string, format: email}
                code: {type: string, pattern: "^[0-9]{6}$"}
      responses:
        "200":
          description: Reset token (TTL default 600 s)
          content:
            application/json:
              schema:
                type: object
                properties:
                  resetToken: {type: string}
                  expiresInSeconds: {type: integer}
        "400": {description: "otp_invalid or otp_expired"}
        "429": {description: otp_locked}
  /api/v1/auth/password/reset:
    post:
      summary: Set new password with resetToken; revokes all sessions (FR5.3). resetToken is never accepted as Bearer
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [resetToken, newPassword]
              properties:
                resetToken: {type: string}
                newPassword: {type: string, minLength: 10, maxLength: 128}
      responses:
        "204": {description: Password reset; all sessions revoked}
        "400": {description: "Token used, expired or invalid"}
        "422": {description: password_policy}
  /api/v1/auth/password/change:
    post:
      summary: Change password while signed in (AUTH-016); keeps current session, revokes other families (Q15)
      security: [{bearer: []}]
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [currentPassword, newPassword]
              properties:
                currentPassword: {type: string}
                newPassword: {type: string, minLength: 10, maxLength: 128}
      responses:
        "204": {description: Changed}
        "400": {description: "Current password wrong"}
        "422": {description: password_policy}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    TokenResponse:
      type: object
      required: [accessToken, expiresInSeconds, user]
      properties:
        accessToken: {type: string, description: "JWT, 15 min, typ=access; held in memory only"}
        expiresInSeconds: {type: integer, example: 900}
        user: {$ref: "#/components/schemas/User"}
    User:
      type: object
      properties:
        id: {type: string, format: uuid}
        fullName: {type: string}
        email: {type: string, format: email}
        verified: {type: boolean}
        role: {type: string, enum: [user]}
        memberSince: {type: string, format: date-time}
```

### C3 — Onboarding: consent, questionnaire, disclaimer

```yaml
openapi: 3.1.0
info: {title: FaceIQ Onboarding API, version: "1.0.0"}
paths:
  /api/v1/onboarding/consents:
    get:
      summary: Current consent state and text versions (FR7)
      security: [{bearer: []}]
      responses:
        "200":
          description: Consents
          content:
            application/json:
              schema:
                type: object
                properties:
                  images: {$ref: "#/components/schemas/ConsentState"}
                  health: {$ref: "#/components/schemas/ConsentState"}
    post:
      summary: Record consents (two records, Q18)
      security: [{bearer: []}]
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [images, health, textVersion]
              properties:
                images: {type: boolean}
                health: {type: boolean}
                textVersion: {type: string}
      responses:
        "200": {description: Recorded}
  /api/v1/onboarding/questionnaire:
    get:
      summary: Active definition plus saved answers for resume (FR8.1, Q13)
      security: [{bearer: []}]
      responses:
        "200":
          description: Definition and progress
          content:
            application/json:
              schema:
                type: object
                properties:
                  definitionVersion: {type: string}
                  questions: {type: array, items: {type: object, additionalProperties: true}}
                  answers: {type: array, items: {$ref: "#/components/schemas/Answer"}}
                  nextQuestionId: {type: string, nullable: true}
                  locked: {type: boolean}
        "403": {description: consent_required}
  /api/v1/onboarding/questionnaire/answers:
    put:
      summary: Save one answer incrementally; server validates branch and discards abandoned-branch answers
      security: [{bearer: []}]
      requestBody:
        content: {application/json: {schema: {$ref: "#/components/schemas/Answer"}}}
      responses:
        "200": {description: "Saved; returns nextQuestionId"}
        "403": {description: "consent_required (health section without health consent)"}
        "409": {description: "inputs_locked (analysis started)"}
        "422": {description: "Branch not allowed or invalid value"}
  /api/v1/onboarding/questionnaire/submit:
    post:
      summary: Submit; disclaimer must be confirmed (FR8.2, BR-003)
      security: [{bearer: []}]
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [disclaimerConfirmed]
              properties:
                disclaimerConfirmed: {type: boolean, const: true}
      responses:
        "200": {description: Completed}
        "422": {description: "Disclaimer not confirmed or questions unanswered"}
        "409": {description: inputs_locked}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    ConsentState:
      type: object
      properties:
        accepted: {type: boolean}
        textVersion: {type: string}
        acceptedAt: {type: string, format: date-time, nullable: true}
    Answer:
      type: object
      required: [questionId, value]
      properties:
        questionId: {type: string}
        value: {description: "Shape depends on question type from the pending Onboarding Questionnaire Specification"}
```

### C4 — Photos, identity check and journey status

```yaml
openapi: 3.1.0
info: {title: FaceIQ Photos API, version: "1.0.0"}
paths:
  /api/v1/photos:
    get:
      summary: Current photo set, validation results and identity result (review screen)
      security: [{bearer: []}]
      responses:
        "200":
          description: Photo set
          content:
            application/json:
              schema:
                type: object
                properties:
                  photos: {type: array, items: {$ref: "#/components/schemas/Photo"}}
                  identity:
                    type: object
                    nullable: true
                    description: "Null until every configured angle has passed (FR11.1)"
                    properties:
                      consistent: {type: boolean}
                      mismatchedAngles: {type: array, items: {type: string}}
                      computedAt: {type: string, format: date-time}
                  locked: {type: boolean}
  /api/v1/photos/{angle}:
    put:
      summary: Upload or replace the photo for one angle (FR9.2, FR9.4, FR10.3)
      security: [{bearer: []}]
      parameters:
        - {name: angle, in: path, required: true, schema: {type: string}}
      requestBody:
        content:
          multipart/form-data:
            schema:
              type: object
              required: [file]
              properties:
                file: {type: string, format: binary, description: "JPEG, PNG or WebP; size and pixel limits from config"}
      responses:
        "200":
          description: "Stored; validation result (accepted or rejected with stable reason codes)"
          content: {application/json: {schema: {$ref: "#/components/schemas/Photo"}}}
        "403": {description: consent_required}
        "409": {description: inputs_locked}
        "422": {description: "Angle not configured, or unreadable/hostile file (photo_rejected, reason unreadable)"}
        "429": {description: rate_limited}
  /api/v1/journey:
    get:
      summary: Next step for routing guards (FR26; JourneyStatus)
      security: [{bearer: []}]
      responses:
        "200":
          description: Next step
          content:
            application/json:
              schema:
                type: object
                properties:
                  nextStep:
                    type: string
                    enum: [verify, consent, questionnaire, photos, payment, start_analysis, analysis_in_progress, analysis_failed, home]
                  photoIssues: {type: array, items: {type: string}, description: "Mismatched angles when nextStep=photos"}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    Photo:
      type: object
      properties:
        angle: {type: string}
        status: {type: string, enum: [pending, accepted, rejected]}
        reasons:
          type: array
          items:
            type: string
            enum: [unreadable, resolution, brightness, face_count, face_proportion, occlusion, pose]
        imageUrl: {type: string, description: "Signed owner-only link (C11); short TTL"}
        uploadedAt: {type: string, format: date-time}
```

### C5 — Payments: checkout, status, billing

```yaml
openapi: 3.1.0
info: {title: FaceIQ Payments API, version: "1.0.0"}
paths:
  /api/v1/payments/checkout:
    post:
      summary: Create or reuse a Stripe Checkout session (FR12.1). Identity re-check (409) and double-payment guard
      security: [{bearer: []}]
      responses:
        "200":
          description: Redirect target
          content:
            application/json:
              schema:
                type: object
                properties:
                  checkoutUrl: {type: string, format: uri}
        "409": {description: "identity_mismatch (details.mismatchedAngles), or already paid (code payment_exists)"}
  /api/v1/payments/status:
    get:
      summary: Payment state for the return page polling (AC4.2.5). Never marked paid by the redirect
      security: [{bearer: []}]
      responses:
        "200":
          description: Status
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: {type: string, enum: [none, pending, paid, failed, expired]}
  /api/v1/payments/billing:
    get:
      summary: Billing history (FR24.3, Q16). Owner's payments only
      security: [{bearer: []}]
      parameters:
        - {name: limit, in: query, schema: {type: integer, default: 20, maximum: 100}}
        - {name: cursor, in: query, schema: {type: string}}
      responses:
        "200":
          description: Payments
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items:
                      type: object
                      properties:
                        paidAt: {type: string, format: date-time}
                        amountMinor: {type: integer}
                        currency: {type: string}
                        status: {type: string}
                        cardBrand: {type: string}
                        cardLast4: {type: string}
                        receiptUrl: {type: string, format: uri}
                  nextCursor: {type: string, nullable: true}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
```

### C6 — Stripe webhook (external provider → backend)

```yaml
openapi: 3.1.0
info: {title: FaceIQ Stripe Webhook, version: "1.0.0"}
paths:
  /api/v1/payments/webhook:
    post:
      summary: Stripe events. Signature verified against the RAW body with the signing secret; idempotent by event id; never starts analysis (FR12.2, FR12.4)
      parameters:
        - {name: Stripe-Signature, in: header, required: true, schema: {type: string}}
      requestBody:
        content:
          application/json:
            schema:
              type: object
              description: "Stripe Event object; handled types: checkout.session.completed, checkout.session.expired, checkout.session.async_payment_failed"
      responses:
        "200": {description: "Processed, or duplicate already processed, or older event ignored without downgrade"}
        "400": {description: "Invalid signature; nothing changes"}
```

### C7 — Analysis: start, status, resume

```yaml
openapi: 3.1.0
info: {title: FaceIQ Analysis API, version: "1.0.0"}
paths:
  /api/v1/analysis/start:
    post:
      summary: Start analysis after explicit confirmed click (FR13.1). Re-checks 402 and 409, locks inputs, single job
      security: [{bearer: []}]
      responses:
        "202":
          description: Job created
          content: {application/json: {schema: {$ref: "#/components/schemas/JobStatus"}}}
        "402": {description: payment_required}
        "409": {description: "identity_mismatch or analysis_already_running"}
        "429": {description: rate_limited}
  /api/v1/analysis/status:
    get:
      summary: Polled every analysisPollIntervalSeconds (Q5)
      security: [{bearer: []}]
      responses:
        "200":
          description: Status
          content: {application/json: {schema: {$ref: "#/components/schemas/JobStatus"}}}
        "402": {description: payment_required}
        "404": {description: "No job for this user"}
  /api/v1/analysis/resume:
    post:
      summary: '"Try again" after failure; resumes from failed step, no new payment (FR13.4)'
      security: [{bearer: []}]
      responses:
        "202": {description: Resumed}
        "409": {description: "Job not in failed state"}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    JobStatus:
      type: object
      properties:
        status: {type: string, enum: [queued, running, failed, completed]}
        currentStep: {type: string, enum: [measurement, assessments, narrative, feature_images, visuals, report]}
        failedStep: {type: string, nullable: true}
        errorCode: {type: string, nullable: true}
        startedAt: {type: string, format: date-time}
        completedAt: {type: string, format: date-time, nullable: true}
```

### C8 — AI Visuals

```yaml
openapi: 3.1.0
info: {title: FaceIQ Visuals API, version: "1.0.0"}
paths:
  /api/v1/visuals:
    get:
      summary: 5 hairstyles, 5 outfits, aging stack (FR22). Paid, owner-only
      security: [{bearer: []}]
      responses:
        "200":
          description: Visuals
          content:
            application/json:
              schema:
                type: object
                properties:
                  hairstyles: {type: array, maxItems: 5, items: {$ref: "#/components/schemas/Visual"}}
                  outfits: {type: array, maxItems: 5, items: {$ref: "#/components/schemas/Visual"}}
                  aging:
                    type: array
                    description: "Cards: current (own photo), plus3, plus5, plus10"
                    items:
                      allOf:
                        - {$ref: "#/components/schemas/Visual"}
                        - type: object
                          properties:
                            step: {type: string, enum: [current, plus3, plus5, plus10]}
        "402": {description: payment_required}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    Visual:
      type: object
      properties:
        id: {type: string}
        imageUrl: {type: string, description: "Signed link (C11); refetch this endpoint when expired"}
        altText: {type: string}
        status: {type: string, enum: [ready, failed]}
```

### C9 — Report, Home and PDF

```yaml
openapi: 3.1.0
info: {title: FaceIQ Report API, version: "1.0.0"}
paths:
  /api/v1/report:
    get:
      summary: Published report (FR18, FR20). Paid, owner-only; AI text is plain text
      security: [{bearer: []}]
      responses:
        "200":
          description: Report
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: {type: string}
                  publishedAt: {type: string, format: date-time}
                  introduction: {type: string}
                  understandingYourResults: {type: string}
                  limitations: {type: string}
                  features:
                    type: array
                    minItems: 11
                    maxItems: 11
                    items:
                      type: object
                      properties:
                        feature: {type: string, enum: [hair, eyebrows, eyes, nose, cheeks, jaw, lips, chin, skin, neck, ears]}
                        narrative: {type: string}
                        summary: {type: string}
                        beforeImageUrl: {type: string}
                        afterImageUrl: {type: string}
                        projectedPotential: {type: array, items: {type: string}}
                        metrics: {type: object, additionalProperties: true, description: "Empty for features without mesh measurements"}
                        recommendations:
                          type: array
                          items:
                            type: object
                            properties:
                              text: {type: string}
                              tier: {type: string, enum: [at_home, otc, in_clinic]}
                  assessments: {type: object, additionalProperties: true, description: "Dimorphism, prototypicality, thirds, symmetry, face shape; content per Report & UI Specification"}
                  closingRecommendations: {type: string}
        "402": {description: payment_required}
        "404": {description: "No published report"}
  /api/v1/home:
    get:
      summary: Home Overview summary (FR21). Paid, owner-only
      security: [{bearer: []}]
      responses:
        "200":
          description: Home
          content:
            application/json:
              schema:
                type: object
                properties:
                  beforeImageUrl: {type: string}
                  potentialImageUrl: {type: string}
                  priorityFeatures: {type: array, items: {type: object, additionalProperties: true}}
                  radar:
                    type: array
                    minItems: 6
                    maxItems: 6
                    items:
                      type: object
                      properties:
                        axis: {type: string, description: "Includes symmetry; other axes per Report & UI Specification"}
                        value: {type: number}
                  keyStats: {type: object, additionalProperties: true}
                  protocolSummary: {type: object, additionalProperties: true}
                  reportStatus: {type: string, enum: [published]}
        "402": {description: payment_required}
  /api/v1/report/pdf:
    get:
      summary: Download PDF (FR19). Generated on first request and cached
      security: [{bearer: []}]
      responses:
        "200": {description: "{downloadUrl} signed link to the cached PDF"}
        "202": {description: "Generation in progress; retry after Retry-After seconds"}
        "402": {description: payment_required}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
```

### C10 — Beauty Assistant chat

```yaml
openapi: 3.1.0
info: {title: FaceIQ Chat API, version: "1.0.0"}
paths:
  /api/v1/chat/messages:
    get:
      summary: Conversation history (FR23.4), cursor-paginated
      security: [{bearer: []}]
      parameters:
        - {name: limit, in: query, schema: {type: integer, default: 20, maximum: 100}}
        - {name: cursor, in: query, schema: {type: string}}
      responses:
        "200":
          description: Messages and usage
          content:
            application/json:
              schema:
                type: object
                properties:
                  items: {type: array, items: {$ref: "#/components/schemas/Message"}}
                  nextCursor: {type: string, nullable: true}
                  usage: {$ref: "#/components/schemas/Usage"}
        "402": {description: payment_required}
    post:
      summary: Send a message; non-streaming reply (Proposed ADR-016). Medical questions declined (FR23.2)
      security: [{bearer: []}]
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [content]
              properties:
                content: {type: string, minLength: 1, maxLength: 2000}
      responses:
        "200":
          description: Reply
          content:
            application/json:
              schema:
                type: object
                properties:
                  userMessage: {$ref: "#/components/schemas/Message"}
                  reply: {$ref: "#/components/schemas/Message"}
                  usage: {$ref: "#/components/schemas/Usage"}
        "402": {description: payment_required}
        "422": {description: "Empty or too long"}
        "429": {description: "chat_cap_reached (Retry-After until 00:00 UTC) or rate_limited"}
        "502": {description: "AI reply failed; user message kept, not counted toward the cap (Q17)"}
components:
  securitySchemes:
    bearer: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    Message:
      type: object
      properties:
        id: {type: string}
        role: {type: string, enum: [user, assistant]}
        content: {type: string, description: Plain text}
        createdAt: {type: string, format: date-time}
    Usage:
      type: object
      properties:
        used: {type: integer}
        cap: {type: integer}
        resetsAt: {type: string, format: date-time}
```

### C11 — Signed media download (browser → backend)

```yaml
openapi: 3.1.0
info: {title: FaceIQ Media Download, version: "1.0.0"}
paths:
  /api/v1/media/{objectId}:
    get:
      summary: >
        Serves an encrypted stored object (photo, generated image, PDF) decrypted, only with a valid HMAC token
        binding objectId, owner and expiry (Proposed ADR-015). Opened directly by img tags and download links,
        so it is the one documented exception to "only the API client calls the backend"; the URL itself is
        obtained through the API client from C4/C8/C9. No Bearer token. Token scrubbed from logs.
      parameters:
        - {name: objectId, in: path, required: true, schema: {type: string}}
        - {name: token, in: query, required: true, schema: {type: string}}
      responses:
        "200": {description: "File bytes; Cache-Control: private, no-store"}
        "404": {description: "Expired, tampered, wrong owner or missing (indistinguishable)"}
```

## In-Process Contracts (Between Backend Units)

These are Python `Protocol` interfaces with Pydantic models. A module may import another module's interface only, never its internals or tables (import-linter enforced, Q4). They are written in shared-schema form: method names, inputs, outputs and errors.

### C12 — Onboarding interface (provider U4)

```yaml
shared_schema: OnboardingPort
provider: u4-onboarding
consumers: [u5-photos, u7-analysis-pipeline, u9-insights, u12-beauty-assistant, JourneyStatus]
methods:
  has_consent:
    input: {user_id: uuid, consent_type: "images | health"}
    output: bool
  get_answers_for_ai:
    input: {user_id: uuid}
    output:
      answers:
        - {question_id: str, question_text: str, value: any, category: "general | medical | medication | allergy"}
    notes: "Consumers must exclude medical/medication/allergy categories from cosmetic prompts (FR15.4)"
  lock_inputs:
    input: {user_id: uuid}
    output: none
    notes: "Idempotent; after lock all answer writes return inputs_locked"
  progress:
    input: {user_id: uuid}
    output: {consents_complete: bool, questionnaire_complete: bool}
errors: [not_found]
```

### C13 — Photos interface (provider U5)

```yaml
shared_schema: PhotosPort
provider: u5-photos
consumers: [u6-payments, u7-analysis-pipeline, u11-report, JourneyStatus]
methods:
  recheck_identity:
    input: {user_id: uuid}
    output: {consistent: bool, mismatched_angles: [str]}
    notes: "Recomputed from current photos; caller maps inconsistent to HTTP 409 identity_mismatch"
  lock_photos:
    input: {user_id: uuid}
    output: none
  get_photos_for_analysis:
    input: {user_id: uuid}
    output:
      photos: [{angle: str, image_bytes: bytes, landmarks: "list[478 x (x,y,z)]"}]
  get_before_photo_ref:
    input: {user_id: uuid}
    output: {stored_object_id: uuid}
    notes: "Front photo [Assumption pending Report & UI Specification]"
  progress:
    input: {user_id: uuid}
    output: {complete: bool, consistent: bool, mismatched_angles: [str]}
errors: [not_found, inputs_locked]
```

### C14 — Payments interface (provider U6)

```yaml
shared_schema: PaymentsPort
provider: u6-payments
consumers: [u7-analysis-pipeline, u10-image-generation, u11-report, u12-beauty-assistant, u13-account-and-app-wide, JourneyStatus]
methods:
  is_paid:
    input: {user_id: uuid}
    output: bool
    notes: "Backs the require_paid dependency; routes tagged paid return 402 when false (registry test US4.3)"
  status:
    input: {user_id: uuid}
    output: {status: "none | pending | paid | failed | expired"}
errors: []
```

### C15 — Media Store interface (provider U5)

```yaml
shared_schema: MediaStorePort
provider: u5-photos (MediaStore component)
consumers: [u5-photos, u10-image-generation, u11-report]
methods:
  put:
    input: {owner_user_id: uuid, kind: "photo | feature_image | visual | pdf", content_type: str, data: bytes}
    output: {stored_object_id: uuid}
    notes: "Encrypts at rest (ADR-015)"
  get_bytes:
    input: {stored_object_id: uuid}
    output: bytes
    notes: "Server-side only (pipeline, PDF rendering)"
  signed_url:
    input: {stored_object_id: uuid, requesting_user_id: uuid}
    output: {url: str, expires_at: datetime}
    notes: "Returns not_found when requesting user is not the owner (404 convention)"
errors: [not_found]
```

### C16 — Pipeline step contract (provider U7; implemented by U8–U11)

```yaml
shared_schema: PipelineStep
provider: u7-analysis-pipeline
implementers:
  - {unit: u8-face-analysis-engine, steps: [measurement, assessments]}
  - {unit: u9-insights, steps: [narrative]}
  - {unit: u10-image-generation, steps: [feature_images, visuals]}
  - {unit: u11-report, steps: [report]}
registration: "Each implementing unit registers its step object with the step registry at start-up; order is fixed by the registry: measurement, assessments, narrative, feature_images, visuals, report"
interface:
  name: str
  run:
    input:
      context:
        job_id: uuid
        user_id: uuid
        analysis_result_id: uuid
        attempt: int
    output: {status: "succeeded | failed", error_code: "str | null"}
  rules:
    - "Idempotent: re-running after partial success must not duplicate outputs (e.g. only missing images regenerated)"
    - "Reads inputs from AnalysisResult and the provider ports (C12, C13, C15); writes outputs to AnalysisResult or its own tables"
    - "Must finish within the configured per-step timeout; transient vendor errors (429/5xx) are retried by the runner with bounded attempts"
    - "Never logs photos, landmarks, biometric signatures or health answers; error_code only"
analysis_result_fields:
  measurements: "written by measurement step (U8)"
  assessments: "written by assessments step (U8)"
  narratives: "per feature, written by narrative step (U9)"
  recommendations: "tiered, written by narrative step (U9)"
  closing_recommendations: "written by narrative step (U9)"
```

### C17 — Report interface (provider U11)

```yaml
shared_schema: ReportPort
provider: u11-report
consumers: [u12-beauty-assistant, JourneyStatus]
methods:
  get_grounding_content:
    input: {user_id: uuid}
    output:
      report_id: uuid
      features: [{feature: str, narrative: str, metrics: dict, recommendations: [str]}]
      assessments: dict
    errors: [not_found]
  is_published:
    input: {user_id: uuid}
    output: bool
```

### C18 — Identity dependency (provider U3)

```yaml
shared_schema: IdentityPort
provider: u3-identity (skeleton in u1-walking-skeleton)
consumers: [all backend units]
methods:
  current_user:
    input: {authorization_header: str}
    output: {user_id: uuid, verified: bool, role: "user"}
    errors: [unauthenticated]
    notes: "Rejects tokens with alg outside the allow-list and any typ other than access (refresh and reset tokens never accepted as Bearer)"
  verification_state:
    input: {user_id: uuid}
    output: {verified: bool}
```

## Contract Ownership Rules

- **Owners:** the provider unit in the Contracts table owns its spec (C1–C18) and the code that generates it: the FastAPI routes and Pydantic models for HTTP contracts, and the `Protocol` classes for in-process contracts.
- **Additive changes** (new optional fields, new endpoints, new enum values that consumers may ignore) are safe within `/api/v1`. The generated types are regenerated in the same pull request, and CI fails on drift. Consumers must ignore unknown fields.
- **Breaking changes** (removing or renaming a field, changing a type or status code, making an optional field required) need:
  - agreement from every consumer unit listed in the table;
  - for HTTP, a parallel `/api/v2` route until the frontend moves;
  - for in-process contracts, one pull request that updates all consumers.
- **Error codes** are part of the contract. A new code is additive. Renaming or removing a code is breaking, because the frontend maps codes to toast copy.
- **Pending specifications:** fields reserved for the pending Report & UI and Onboarding Questionnaire specifications are extensible objects. Filling them in is additive, and until then tests assert only their presence.
- **Security invariants (unchangeable without an ADR):**
  - cross-user access returns 404;
  - unpaid access to a gated route returns 402;
  - no token or secret in any URL except the C11 signed-link token;
  - the refresh token is never in a response body.

## Open Questions

| Contract | Question | Blocks |
|----------|----------|--------|
| C2 | Final email/OTP provider (`NFR-012`, OQ6); SMTP adapter assumed | U3 real-provider configuration only |
| C3 | Question types and value shapes from the Onboarding Questionnaire Specification (OQ5) | U4 `Answer.value` schema |
| C4 | Accepted formats: add HEIC (OQ-S5); final pixel and size limits (Photo Validation Specification) | U5 upload validation values |
| C5 | Report price and currency (`OQ-002`) | U6 configuration values only |
| C9 | Key stats, protocol summary, priority-feature rule, the five non-symmetry radar axes, symmetry label set (Report & UI Specification) | U11 home and report content fields |
| C9 | PDF generation mode: synchronous with timeout or background (202 + retry); owner of async PDF work (domain review R-01, Proposed ADR-014) | U11 PDF endpoint behaviour |
| C11 | Signed-link TTL default (proposed 300 s) and key rotation (Proposed ADR-015) | U5 MediaStore configuration |
| C16 | Prototypicality reference model (OQ-S4) | U8 assessments step output |
| C1 | Normal-load profile for the 500 ms p95 target (OQ7) | U13 load test |
