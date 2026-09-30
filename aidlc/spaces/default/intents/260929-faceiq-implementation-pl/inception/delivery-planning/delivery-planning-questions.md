# Delivery Planning — Questions

This last planning step orders the build into **Bolts**. A Bolt is one build pass over one or more units of work that ends in something that runs and can be demonstrated. The 13 units and their dependency graph come from `units-generation/`; this step chooses the path through that graph and explains why.

Affirmed practices used here:
- **Way of Working:** trunk-based development, squash-merged pull requests into `main`.
- **Walking Skeleton:** the login step first. A walking skeleton is a thin end-to-end slice built first to prove the pieces connect.
- **Deployment:** none yet; build and push to GitHub.

Fill each `[Answer]:` with a letter, or `X` plus your own text.

## Q1 — What to build first

A. The walking skeleton first (U1), then follow the dependency order, pulling the riskiest unknowns forward where the graph allows. The computer-vision feasibility spike (US0.11) runs as early throw-away exploration alongside U2–U4, and a small AI-image quality and cost spike runs before U10 (recommended)
B. Value-first: follow the user journey strictly in dependency order, with no early spikes
C. Formal scoring: rank Bolts by value plus urgency plus risk reduction, divided by size (WSJF), within the dependency limits
X. Other (please specify)

[Answer]: A

## Q2 — How big a Bolt is

A. One Bolt per unit, except the two largest units, identity (U3) and photos (U5), which are split into two Bolts each. That gives 15 Bolts, each ending in a demo (recommended)
B. Bundle related units into about 7 larger Bolts
C. Thin slices that cut across several units
X. Other (please specify)

[Answer]: A

## Q3 — Building Bolts at the same time

A. Follow the dependency graph; build in parallel only where it allows (the final chat and account/app-wide Bolts, plus fixture-based early work inside a Bolt) (recommended)
B. Strictly one Bolt at a time
X. Other (please specify)

[Answer]: A

## Q4 — Things outside the team that could hold us up

Known items:
- Stripe test-mode account and keys;
- an OpenAI API key with image-editing access, plus the zero-data-retention request;
- an email/SMTP provider;
- the four pending client specifications (Onboarding Questionnaire, Photo Capture, Photo Validation, Report & UI);
- the report price;
- the target market and privacy law;
- the photo-angle and tiering confirmations;
- a licensed or consented face-image set for tests;
- the prototypicality reference.

A. The client (sponsor) owns the accounts, keys, specifications and confirmations. The delivery team requests each one by the Bolt that needs it. Every Bolt can start with fakes or format fixtures, and only its "done" waits on the real item (recommended)
B. The delivery team sets up all accounts itself; the client only supplies the specifications
X. Other (please specify)

[Answer]: A

## Q5 — What worries you most

A. Computer-vision accuracy (identity check, pose, occlusion) and AI image quality and cost. Tackle them early with the two spikes from Q1 (recommended)
B. Security of custom auth and payments
C. Late arrival of the client specifications
X. Other (please specify)

[Answer]: A

## Q6 — Who builds each Bolt

Team formation was skipped in this plan.

A. The AI developer agent builds every Bolt. The delivery team (Crest Infosystems) reviews and approves every pull request and checkpoint (recommended)
B. Named Crest developers own Bolts (names to be supplied)
X. Other (please specify)

[Answer]: A

## Consolidated Summary Confirmation

Summary of answers:
- Walking skeleton (U1) first, then dependency order with the riskiest unknowns pulled forward: an early throw-away CV feasibility spike alongside U2-U4 and an AI-image quality/cost spike before U10 (Q1 A)
- One Bolt per unit, with identity (U3) and photos (U5) each split in two: 15 Bolts, each ending in a demo (Q2 A)
- Parallel work only where the dependency graph allows (final chat and account/app-wide Bolts, plus fixture-based early work) (Q3 A)
- The client owns accounts, keys, specifications and confirmations, requested by the Bolt that needs them; Bolts start on fakes or format fixtures and only their "done" waits on the real item (Q4 A)
- Top risks: computer-vision accuracy and AI image quality/cost, tackled by the two early spikes (Q5 A)
- The AI developer agent builds every Bolt; Crest Infosystems reviews and approves every pull request and checkpoint (Q6 A)

Does this all look correct before I generate the artifact?

- Looks correct
- Request changes

[Answer]: Looks correct
