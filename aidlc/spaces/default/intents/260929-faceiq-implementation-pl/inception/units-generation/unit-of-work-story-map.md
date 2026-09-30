# Unit of Work — Story Map

## Overview

Every story in `user-stories/stories.md` (67) is mapped to exactly one implementing Unit from `unit-of-work.md`. The order within each unit follows the story-level `Depends on` lines. Build order across units is chosen in Delivery Planning; story order inside a unit is the implementation sequence for that unit.

## Story-to-Unit Map

| Story | Title | Unit | Directory | Order in Unit |
|-------|-------|------|-----------|---------------|
| US0.1 | Repository scaffold and core CI (Enabler) | U1 | u1-walking-skeleton | 1 |
| US0.3 | Backend foundation (Enabler) | U1 | u1-walking-skeleton | 2 |
| US0.4 | One-command local environment (Enabler) | U1 | u1-walking-skeleton | 3 |
| US0.5 | Frontend foundation (Enabler) | U1 | u1-walking-skeleton | 4 |
| US0.6 | Auth layering walking skeleton (Enabler) | U1 | u1-walking-skeleton | 5 |
| US0.2 | Security, coverage and traceability gates (Enabler) | U2 | u2-platform-services | 1 |
| US0.7 | Vendor adapter interfaces and fakes (Enabler) | U2 | u2-platform-services | 2 |
| US0.8 | Rate-limit primitive (Enabler) | U2 | u2-platform-services | 3 |
| US0.9 | Test-data factories and E2E seeding (Enabler) | U2 | u2-platform-services | 4 |
| US1.1 | Discover the product on the landing page | U3 | u3-identity | 1 |
| US1.2 | Sign up with name, email and password | U3 | u3-identity | 2 |
| US1.3 | One-time code rules (verify service) | U3 | u3-identity | 3 |
| US1.4 | Complete signup: tokens and session | U3 | u3-identity | 4 |
| US1.5 | Resend a code without being spammed | U3 | u3-identity | 5 |
| US1.6 | Log in with password and code | U3 | u3-identity | 6 |
| US1.7 | Protect my session if a token is stolen | U3 | u3-identity | 7 |
| US1.8 | Stay signed in across reloads and restarts | U3 | u3-identity | 8 |
| US1.9 | Seamless token refresh | U3 | u3-identity | 9 |
| US1.10 | Log out safely | U3 | u3-identity | 10 |
| US1.11 | Reset a forgotten password | U3 | u3-identity | 11 |
| US9.3 | Keep my data isolated from other users | U3 | u3-identity | 12 |
| US2.1 | Consent to processing of my photos and health answers | U4 | u4-onboarding | 1 |
| US2.2 | Questionnaire definition and answers | U4 | u4-onboarding | 2 |
| US2.3 | Answer the questionnaire | U4 | u4-onboarding | 3 |
| US2.4 | Confirm the mandatory disclaimer | U4 | u4-onboarding | 4 |
| US0.11 | Computer-vision feasibility spike (Enabler, time-boxed) | U5 | u5-photos | 1 |
| US3.1 | See the photo requirements before I start | U5 | u5-photos | 2 |
| US3.2 | Upload photos securely | U5 | u5-photos | 3 |
| US3.4 | Private delivery of my photos and images | U5 | u5-photos | 4 |
| US3.3 | Upload or capture each photo angle | U5 | u5-photos | 5 |
| US3.5 | Clear feedback when a photo isn't usable (basic checks) | U5 | u5-photos | 6 |
| US3.6 | Occlusion and pose checks | U5 | u5-photos | 7 |
| US3.7 | Make sure all three photos are of me | U5 | u5-photos | 8 |
| US3.8 | Retake a flagged photo and continue | U5 | u5-photos | 9 |
| US3.9 | Land on the onboarding step I need to finish | U5 | u5-photos | 10 |
| US4.1 | Pay for my report | U6 | u6-payments | 1 |
| US4.2 | My payment is confirmed securely | U6 | u6-payments | 2 |
| US4.3 | Payment gate for all paid content | U6 | u6-payments | 3 |
| US0.10 | Background job runner (Enabler) | U7 | u7-analysis-pipeline | 1 |
| US5.1 | Start my analysis when I'm ready | U7 | u7-analysis-pipeline | 2 |
| US5.2 | Watch my analysis progress | U7 | u7-analysis-pipeline | 3 |
| US5.3 | Recover when a step fails, without paying again | U7 | u7-analysis-pipeline | 4 |
| US5.4 | Measure eyes, eyebrows, nose and lips | U8 | u8-face-analysis-engine | 1 |
| US5.5 | Measure jaw, chin, cheeks and ears | U8 | u8-face-analysis-engine | 2 |
| US5.8 | Symmetry and facial-thirds assessments | U8 | u8-face-analysis-engine | 3 |
| US5.9 | Dimorphism, prototypicality and face-shape data | U8 | u8-face-analysis-engine | 4 |
| US5.6 | A personalised, respectful narrative per feature | U9 | u9-insights | 1 |
| US5.7 | Recommendations sorted into clear tiers | U9 | u9-insights | 2 |
| US5.10 | A realistic "after" image for each feature | U10 | u10-image-generation | 1 |
| US5.11 | Generate my AI Visuals set | U10 | u10-image-generation | 2 |
| US7.2 | Browse my AI Visuals | U10 | u10-image-generation | 3 |
| US6.1 | My report is ready as soon as it's done | U11 | u11-report | 1 |
| US6.2 | Read each feature with before/after and ideas | U11 | u11-report | 2 |
| US6.3 | Explore the interactive report | U11 | u11-report | 3 |
| US6.4 | Download my report as a PDF | U11 | u11-report | 4 |
| US6.5 | Branded PDF layout | U11 | u11-report | 5 |
| US7.1 | See my Home Overview and move around | U11 | u11-report | 6 |
| US7.3 | Ask the AI Beauty Assistant about my report | U12 | u12-beauty-assistant | 1 |
| US7.4 | Medical questions are declined safely | U12 | u12-beauty-assistant | 2 |
| US7.5 | Know when I've reached today's chat limit | U12 | u12-beauty-assistant | 3 |
| US8.1 | Settings and my account information | U13 | u13-account-and-app-wide | 1 |
| US8.2 | Change my password while signed in | U13 | u13-account-and-app-wide | 2 |
| US8.3 | See my billing history | U13 | u13-account-and-app-wide | 3 |
| US9.1 | See my name and reach my account from any screen | U13 | u13-account-and-app-wide | 4 |
| US9.2 | Land on the right post-payment screen | U13 | u13-account-and-app-wide | 5 |
| US9.4 | App-wide accessibility audit | U13 | u13-account-and-app-wide | 6 |
| US9.5 | Keep the app fast and measurable | U13 | u13-account-and-app-wide | 7 |

## Implementation Order Within Each Unit

- **U1 u1-walking-skeleton**: US0.1 → US0.3 → US0.4 → US0.5 → US0.6
- **U2 u2-platform-services**: US0.2 → US0.7 → US0.8 → US0.9
- **U3 u3-identity**: US1.1 → US1.2 → US1.3 → US1.4 → US1.5 → US1.6 → US1.7 → US1.8 → US1.9 → US1.10 → US1.11 → US9.3
- **U4 u4-onboarding**: US2.1 → US2.2 → US2.3 → US2.4
- **U5 u5-photos**: US0.11 → US3.1 → US3.2 → US3.4 → US3.3 → US3.5 → US3.6 → US3.7 → US3.8 → US3.9
- **U6 u6-payments**: US4.1 → US4.2 → US4.3
- **U7 u7-analysis-pipeline**: US0.10 → US5.1 → US5.2 → US5.3
- **U8 u8-face-analysis-engine**: US5.4 → US5.5 → US5.8 → US5.9
- **U9 u9-insights**: US5.6 → US5.7
- **U10 u10-image-generation**: US5.10 → US5.11 → US7.2
- **U11 u11-report**: US6.1 → US6.2 → US6.3 → US6.4 → US6.5 → US7.1
- **U12 u12-beauty-assistant**: US7.3 → US7.4 → US7.5
- **U13 u13-account-and-app-wide**: US8.1 → US8.2 → US8.3 → US9.1 → US9.2 → US9.4 → US9.5

## Cross-Cutting Stories

These stories live in one unit but deliver something other units rely on. Other units extend them rather than rebuild them.

| Story | Home Unit | Used by |
|-------|-----------|---------|
| US0.5 | U1 | Shared toast/confirm/API client used by every UI unit (Definition of Done items 3-4) |
| US0.7 | U2 | Adapter ports consumed by U3 (email), U5 (storage), U6 (Stripe), U9 (AI text), U10 (AI image), U12 (AI text) |
| US0.8 | U2 | Rate limiter used by U3, U5, U7, U12 |
| US0.9 | U2 | Seeding scenarios used by E2E tests of every later unit |
| US3.4 | U5 | Signed links used by U10 (visuals), U11 (report, PDF, Home) |
| US4.3 | U6 | Paid-route registry: each gated endpoint in U7, U10, U11, U12 adds its route |
| US9.3 | U3 | Owner-scope registry: each owner-scoped endpoint in U4-U13 adds its cross-user test |
| US3.9 | U5 | Onboarding routing extended by U13 (US9.2) for post-payment states |

## Coverage Verification

- Stories mapped: 67 of 67; each story has exactly one unit.
- Every unit has at least two stories:
  - U1: 5 stories
  - U2: 4 stories
  - U3: 12 stories
  - U4: 4 stories
  - U5: 10 stories
  - U6: 3 stories
  - U7: 4 stories
  - U8: 4 stories
  - U9: 2 stories
  - U10: 3 stories
  - U11: 6 stories
  - U12: 3 stories
  - U13: 7 stories
- No story is unassigned, and no unit is empty.
- US0.6 (the skeleton) needs one seeded verified user. U1 creates that user with a minimal seed script. The full seeding CLI (US0.9) arrives in U2 and replaces it.
