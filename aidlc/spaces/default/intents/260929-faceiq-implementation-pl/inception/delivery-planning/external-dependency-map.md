# External Dependency Map — FaceIQ

## Overview

This document lists the items from outside the delivery team that the build needs, and maps each to the **Bolt** that needs it. A Bolt is one build pass over one or more units of work, ending in something runnable; the Bolts are listed in `bolt-plan.md`.

- **Ownership:** the client sponsor owns and supplies each item (Q4). The delivery lead requests it at least one Bolt ahead of the Bolt that needs it.
- **Starting without the item:** every Bolt can start on fakes or format fixtures. Only its "done" depends on the real item. This follows the adapter and fake approach (ADR-011) and the extensible contract fields in `contract-summary`.
- **Excluded costs:** third-party usage costs are the client's (`BR-006`, `CON-004`).

## Dependencies

| # | Item | Owner | Needed by Bolt | Request by | Lead time (estimate) | If it slips |
|---|------|-------|----------------|-----------|----------------------|-------------|
| X1 | Licensed or consented face-image test set (varied ages, skin tones, angles) | Client sponsor | CV spike, B6, B7 | Start of B1 | 1–2 weeks | Spike runs on a very small consented set; accuracy claims stay provisional |
| X2 | Email/SMTP provider account (`NFR-012`, OQ6) | Client sponsor | B3 (real sends) | Start of B2 | Days | Build and tests use Mailpit; the real provider is configured later with no code change |
| X3 | Onboarding Questionnaire Specification (23 questions, branching, disclaimer wording) | Client sponsor | B5 | Start of B3 | Unknown | B5 completes on a format fixture; loading the real content is a small follow-up |
| X4 | Consent wording and target market / privacy law (OQ1) | Client sponsor (legal) | B5 | Start of B3 | Unknown | Strictest-baseline wording as a placeholder, flagged as not legally reviewed |
| X5 | Photo Capture Specification; HEIC decision (OQ-S5) | Client sponsor | B6 | Start of B5 | Unknown | JPEG, PNG and WebP only; standard per-angle layout |
| X6 | Photo Validation Specification (thresholds); identity-vote confirmation (OQ-S1); 3-angle confirmation (OQ2) | Client sponsor | B7 | Start of B6 | Unknown | Configurable defaults from the spike; vote reading per Q7 stays flagged |
| X7 | Stripe account (test mode keys, webhook secret) | Client sponsor | B8 | Start of B7 | Days | Signed fixture webhooks and a stubbed session; the real test account is needed before B8's demo |
| X8 | Report price and currency (`OQ-002`) | Client sponsor | B8 | Start of B7 | Days | Placeholder configuration value; no code change later |
| X9 | OpenAI API key with `gpt-4o` and `gpt-image-1` edit access; zero-data-retention request status (NFR5.2) | Client sponsor | AI-image spike, B11, B12 | Start of B8 | Days to weeks (ZDR approval) | Recorded responses and fakes; the spike waits for the key |
| X10 | Acceptance of AI-image quality and cost from the spike (`ASM-011`) | Client sponsor | B12 | End of the AI-image spike | Days | B12 proceeds with the vendor swap option (`NFR-013`) under consideration |
| X11 | Tiering method confirmation (OQ3, `ASM-007`) | Client sponsor | B11 | Start of B10 | Days | Keyword rule set as written, labelled unconfirmed |
| X12 | Prototypicality reference model or dataset (OQ-S4) | Client sponsor | B10 | Start of B9 | Unknown | Prototypicality stays on the pending list; the other assessments ship |
| X13 | Report & UI Specification (key stats, protocol summary, radar axes, labels, layouts) | Client sponsor | B13 (also B12 and B15 layouts) | Start of B11 | Unknown | Extensible fields and a basic layout; the real content is a follow-up |
| X14 | Load profile for the 500 ms p95 target (OQ7) | Delivery team with client | B15 (US9.5) | Start of B14 | Days | Load test deferred; not a CI gate |
| X15 | Named Crest reviewers, including a security reviewer | Crest Infosystems | B1 onward (security from B3) | Before B1 | Days | Delivery lead reviews until they are named |

## Bolts With No External Dependency

- **B2:** platform services.
- **B4:** login, sessions and account safety. It uses X2 only indirectly, through Mailpit.
- **B9:** analysis pipeline.
- **B14:** beauty assistant. It reuses X9.
