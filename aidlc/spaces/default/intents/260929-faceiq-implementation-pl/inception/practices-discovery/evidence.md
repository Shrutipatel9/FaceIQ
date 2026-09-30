# Evidence — Practices Discovery

## Sources Inspected

| Participant | What was inspected | Main inferences |
|---|---|---|
| Lead (`aidlc-pipeline-deploy-agent`) | `org.md` (five practice sections and `## Mandated`); `team.md` and `project.md` (empty templates, so no re-run baseline); `phases/inception.md`; `phases/construction.md` Security; `client_requirements.md` v1.0 in full; `<record>/aidlc-state.md`; `.claude/scopes/aidlc-requirements-to-plan.md`; git (`log`, `branch -a`, `remote -v`, `status`); repository root listing | Greenfield with no observable conventions: one commit `c069529`, no code, CI, linter or test setup (`CON-001`). The stack is fixed by the client (`NFR-001`–`NFR-006`). Hosting cannot be designed (`NFR-010`, `CON-007`). The remote is GitHub. The scope declares no `skeleton:` and skips Construction. Maintainability is a first-class requirement (`BC-005`, `NFR-011`). |
| `aidlc-quality-agent` | Lead drafts; `client_requirements.md`; `org.md` Testing Posture | Proposed a concrete `custom` methodology; branch coverage, a 90% floor for high-risk code and a reviewed exclusion list; requirement-ID test tagging with a CI check and pending list; test layers per app (real PostgreSQL, injectable clock, migration round-trip, MSW, a non-persistence test); the OpenAPI drift check; layered testing of OpenAI, Stripe, email and image generation; landmark-coordinate fixtures for CV; quarantine policy. |
| `aidlc-developer-agent` | Lead drafts; `client_requirements.md`; `org.md` | Proposed the folder layout for both apps; backend layering for every feature, enforced with `import-linter`, and frontend boundaries enforced with ESLint; client-side route guarding given in-memory tokens; the error-handling convention (domain errors, one envelope, `ApiError`); naming conventions; `.env.example`, agent-instruction files, generated API types and mypy/TS strictness. Found the duplicated OTP rule and the "is a settings default hardcoding?" ambiguity. |
| `aidlc-devsecops-agent` | Lead drafts; `client_requirements.md`; `org.md`; `phases/construction.md`; root `.gitignore`; git remote | Proposed security lint (Ruff `S`/`B`, `eslint-plugin-security`, `no-unsanitized`, `react/no-danger`, a `NEXT_PUBLIC_*` secret check); CI security gates (Gitleaks, Semgrep/CodeQL, `pip-audit`/audit, Dependabot, lockfiles, SHA-pinned Actions); missing auth, payment, upload and data-handling rules; security and abuse tests. Found that `.gitignore` lacks `.env`, `.env.*`, `*.pem` and `*.key`. Raised biometric and health data sensitivity. |

## Interview Decisions

| Question | Answer | Effect on the artifacts |
|---|---|---|
| Q1 Repository layout | A — one repository with `backend/` and `frontend/` | Way of Working; folder layout in Code Style |
| Q2 Branching and review | A — short-lived branches, a PR with CI green and 1 approval, squash-merge, titles cite requirement IDs | Way of Working |
| Q3 Thin end-to-end slice first | B — the email + password login step through the full §5.3 auth layering | Walking Skeleton |
| Q4 Test ordering | B — tests first for the exactly-specified rules (token rotation, reuse and lockout; 402 gate; identity vote and 409; tiering); tests after elsewhere | Testing Posture `Methodology: custom` and `Ordering` |
| Q5 Coverage floor | B — 80% line and branch per app; 90% for auth, payment-gate and identity-check code; reviewed exclusion list | Testing Posture coverage bullets; "never weaken a gate" rule |
| Q6 Face photos as test data | A — no real user photos; landmark-coordinate fixtures; licensed or consented images only, kept outside the public repository if required | Testing Posture CV bullets; Forbidden fixture rule |
| Q7 Deployment | X — "Currently no need to deploy anywhere.just build and push to github" (verbatim) | Deployment: no deployment and no deployment jobs; the org default (staging on merge, manual production approval) is **not** adopted for now |
| Q7a Checks on GitHub | A — GitHub Actions runs the checks on every push and PR, blocking merge; no deployment jobs | Deployment CI check list |
| Q8 Tooling | A — Ruff, mypy strict, pytest; TS strict, ESLint, Prettier, Vitest + RTL, Playwright, pnpm; pre-commit hooks; OpenAPI-generated types with a drift check | Code Style; Testing Posture tooling |
| Q9 Security checks | A — blocking on High/Critical (Gitleaks, Semgrep or CodeQL, `pip-audit`, `npm`/`pnpm audit`, Dependabot); waivers in the repository with an expiry | Deployment security scans; Forbidden gate-weakening rule |
| Q10 Hard rules | A — all client-stated rules plus the reviewers' security and test-data rules | `discovered-rules.md` in full; rules beyond the client text are labelled `[Recommendation — affirmed Q10]` |
| Summary confirmation | "Looks correct" | Integration proceeds |

## Integration Notes and Maintained Dissent

- **Skeleton slice (maintained dissent).** The developer reviewer preferred the infrastructure slice (page → API → PostgreSQL) as the smallest integrated slice, with auth as the first feature on top of it. The human chose the auth slice (Q3 B). The artifacts follow the human; the dissent is recorded here only.
- **Deployment default replaced.** The org default (deploy on merge to staging, manual production approval) is not affirmed. The human chose no deployment for now (Q7 X, Q7a A). SBOM publication, container scanning and DAST from the devsecops review are deferred until a host exists, because there is nothing to package or scan yet.
- **Duplicate OTP rule merged.** The Mandated and Forbidden forms of the OTP hashing rule were merged into one Forbidden rule. The Mandated rule now covers refresh-token storage and reuse detection only.
- **"Hardcoding" clarified.** A non-secret default declared in the typed settings object and overridable by configuration without a code change is not hardcoding (`NFR-008`: "swapped without code changes"). Secrets never have an in-code default. Both rules are worded this way.
- **Scanner choice.** Semgrep **or** CodeQL stays open (Q9 A named both). It depends on whether the repository is public or has GitHub Advanced Security.
- **E2E OTP retrieval.** The artifacts use a local SMTP catcher rather than a test-only inbox endpoint. This avoids the risk the quality reviewer flagged of such an endpoint shipping to production.
- **AI evaluation suite.** The paid, opt-in real-model evaluation suite is kept out of the per-PR gates, consistent with the no-live-vendor rule (Q10). Its budget and trigger are not decided.
- **Product rules left out.** Product and business rules (`FR-008`(f) as a content rule, `FR-019`, `BR-008`–`BR-012`) are not project-wide engineering rules. They carry into requirements-analysis. The logging rule covers the data-handling side of `FR-008`(f).

## Unresolved Uncertainty (handed forward)

1. **Privacy law and target market.** Face photos, the landmark identity signature (effectively a biometric template) and health-related questionnaire answers are sensitive data, and photos are sent to OpenAI (`FR-008`, `NFR-013`). The target market is not stated in `client_requirements.md`, so which regimes apply (GDPR Art. 9, BIPA, CCPA/CPRA) is unknown. This is in tension with the out-of-scope privacy-policy definition (§2.2). Raise it with the client; do not resolve it silently. Also open: user consent before upload, and OpenAI zero-data-retention / no-training settings.
2. **`.gitignore` gaps.** The root `.gitignore` lacks `.env`, `.env.*`, `*.pem` and `*.key` patterns. This is a Construction task for the first commit, together with a committed `.env.example` per app.
3. **No performance or failure targets.** There are no latency, duration or availability targets, and no defined behaviour when analysis fails or partly fails (24 images per report). Performance testing cannot be planned until requirements-analysis supplies measurable criteria.
4. **No accessibility requirement.** Automated accessibility checks would only be a recommendation.
5. **MediaPipe wheel availability.** MediaPipe publishes wheels for specific CPython versions. The pinned Python version must be checked against the MediaPipe release at setup. The Python toolchain (uv, Poetry or pip-tools) and the Node.js LTS pin are also still open.
6. **Next.js routing and auth guarding.** App Router (recommended) plus an in-memory access token means protected routes are guarded on the client side. `middleware.ts` and Server Components cannot see auth state. Cookie `SameSite` and domain behaviour across the two origins depends on the future hosting topology.
7. **Build-shaping decisions for domain-design and contract-design:**
   - SQLAlchemy sync vs async
   - JSON field casing across the boundary
   - API version prefix
   - error-code catalogue
   - HTTP status for OTP lockout and resend cooldown
   - background worker vs in-request analysis pipeline
   - photo and image storage abstraction with encryption at rest
   - account-enumeration stance for signup and login
   - per-user chat and cost-abuse limits
   - the named security reviewer for `CODEOWNERS`
   - product/package identifier
