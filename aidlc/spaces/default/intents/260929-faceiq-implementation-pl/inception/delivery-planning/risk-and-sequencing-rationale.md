# Risk and Sequencing Rationale — FaceIQ

## Overview

This document explains why the 15 **Bolts** in `bolt-plan.md` come in the order they do. A Bolt is one build pass over one or more units of work, ending in something runnable.

The sequencing uses two heuristics:
- **Walking skeleton first** (Cockburn): build a thin end-to-end slice first to prove the architecture connects.
- **Risk first** (Boehm, Spiral Model): surface the biggest unknowns before dependent work commits to them.

Both apply within the dependency graph from `unit-of-work-dependency`. We did not use a formal WSJF score (value and urgency weighed against size, Q1).

## Heuristic Chosen

1. **Walking skeleton first (B1).** This is the affirmed team practice. U1 has no dependencies and proves the riskiest architectural constraint: the client-mandated auth layering across two separate apps (`client_requirements.md` §5.3, `NFR-001`, `NFR-005`).
2. **Dependency order afterwards.** The unit graph is largely a chain (U1 → U2 → U3 → U4 → U5 → U6 → U7 → U8 → U9 → U10 → U11 → {U12, U13}), because the product is a gated journey:
   - consent before photos;
   - photos before payment;
   - payment before analysis;
   - analysis before report.
3. **Risk pulled forward through spikes rather than reordering.** Two throw-away spikes move the highest-uncertainty questions ahead of the Bolts that depend on them, without breaking the dependency order:
   - the **CV feasibility spike** runs alongside B1–B2;
   - the **AI-image quality and cost spike** runs alongside B9–B10.
4. **Splitting the two largest units.**
   - Identity (U3) becomes B3 and B4. B3 covers the signup and verify happy path. B4 covers sessions and account safety.
   - Photos (U5) becomes B6 and B7. B6 covers capture and private storage. B7 covers validation and identity.

   Each half ends in its own demo and keeps the security-critical rules reviewable in smaller pull requests.

## Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Where handled |
|---|------|-----------|--------|------------|---------------|
| R1 | Landmark-based identity check, pose and occlusion detection are not accurate enough (`ASM-010`, `ASM-002`) | Medium | High (auto-published reports built on the wrong person's photos, `CON-006`) | Early CV spike; configurable thresholds; exhaustive vote tests; tuning pass on licensed images | CV spike, B6, B7 |
| R2 | AI image quality or identity preservation is poor, or 24 images per report cost too much or take too long (`ASM-011`, NFR3.1) | Medium | High | AI-image spike before B12 with measured cost, latency and quality; per-image idempotency; bounded concurrency; vendor swap by configuration (`NFR-013`) | AI-image spike, B12 |
| R3 | Custom auth defects: token reuse, CSRF, enumeration (`AUTH-010`–`AUTH-016`) | Low | Critical | Tests written first for rotation, reuse and lockout; 90% coverage floor; security reviewer on B3 and B4; blocking SAST | B3, B4 |
| R4 | Payment gate bypass or double charge (`FR-015`, `BR-001`) | Low | Critical | Tests written first for 402 and webhook; raw-body signature check; idempotent events; gated-route registry test | B8 |
| R5 | Pending client specifications arrive late (questionnaire, photo capture and validation, report and UI) | High | Medium | Format fixtures and extensible contract fields; Bolts finish on fixtures, with small follow-ups to load real content | B5, B6, B7, B13 |
| R6 | Privacy law is unknown for biometric and health data (OQ1) | Medium | High | Strictest-baseline consent; encryption at rest; OpenAI zero-data-retention request; flagged to the client | B5, B6, B11 |
| R7 | The job worker or PDF engine choices (Proposed ADR-013 and ADR-014) do not hold up | Low | Medium | B9 proves the worker before real steps depend on it; the PDF engine is confirmed at B13 | B9, B13 |
| R8 | Open contract-review findings, including the journey-status ownership loop, surface during the build (contract review R-01 to R-09) | Medium | Medium | Resolve journey-status ownership at the start of B7 (onboarding routing) and B15 (post-payment routing); fix the contract document then | B7, B15 |
| R9 | MediaPipe wheels are unavailable for the pinned Python version or OS | Low | Medium | Pin the Python version in B1; verify in the CV spike | B1, CV spike |

## Deviations from Pure Dependency Order

| Deviation | Justification |
|-----------|---------------|
| The CV feasibility spike (part of US0.11, U5) starts during B1–B2, before U2–U4 are complete | It is exploratory only and merges no production code. It answers risk R1 early, so B6 and B7 start with a proven approach. The formal write-up lands in B6. |
| The AI-image spike runs during B9–B10, before U9 (its dependency) is done | It is exploratory only. It answers risk R2 before B12 commits to a vendor setup, and the client can review cost early. |
| B14 and B15 run in parallel | They are independent in the dependency graph (both depend only on U11, and U13 also on U6). This is allowed by Q3. |

No Bolt depends on a Bolt that comes after it: B1–B15 respect every edge in `unit-of-work-dependency`.

## Why Not the Alternatives

- **Value-first only (Q1 B):** this would leave the CV and AI-image uncertainty until late B7 and B12. If either failed there, the pipeline and report work would have to be redone.
- **WSJF scoring (Q1 C):** the gated journey chain leaves almost no ordering freedom, so a scoring model adds ceremony without changing the order.
- **Seven bundled Bolts (Q2 B):** larger Bolts would put the auth and payment rules into big pull requests, which are harder to review for security.
