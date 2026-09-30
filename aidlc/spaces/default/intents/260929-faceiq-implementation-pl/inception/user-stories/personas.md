# Personas — FaceIQ

## Overview

The client defines one functional role, the **End User** (`client_requirements.md` section 3). The three personas below are behavioural variants of that one role, not separate permission levels. They exist so stories can state *why* a user needs something and which rules protect them (`requirements.md` FR15.3, FR15.4, FR23.2). The **Admin** is a future role (deferred, section 2.2). It is recorded for data-model awareness only (`AUTH-009`), and no stories are written for it.

Any persona may use assistive technology, a phone-sized screen or a slow connection; these are cross-cutting conditions, not a separate persona (NFR6). Stories written for them are tagged "All" (for example US9.4).

Persona details beyond the client document are **[Recommendation]**: they are behavioural archetypes derived from the requirements, not client research.

## P1 — Alex, the Goal-Driven Self-Improver (primary)

| Field | Value |
|-------|-------|
| Role | End User |
| Goals | Get an objective, measurement-backed view of their face; learn concrete at-home, skincare and optional in-clinic options; see a realistic "potential" image per feature |
| Pain points | Expert reports (Qoves-style) are slow and expensive (`BC-002`); generic beauty advice ignores their actual features |
| Context | Arrives from the landing page, completes signup, questionnaire and photos in one session, pays and waits for the report |
| Tech comfort | Medium–high; comfortable with camera capture and online payment |
| Frequency | One intensive first session, then occasional return visits |
| Key rules that serve them | Narratives cite real measurements and match their stated goals (FR15.2); recommendations are tiered and never prescriptive (FR15.5) |

## P2 — Jordan, the Appearance-Anxious User (safety-critical)

| Field | Value |
|-------|-------|
| Role | End User |
| Goals | Understand their appearance without being hurt by the result; get gentle, practical ideas |
| Pain points | Appearance-related worry; risk of harm from blunt or medicalised commentary |
| Context | Their questionnaire answers indicate elevated appearance-related distress; must pass the mandatory disclaimer (FR8.2, `FR-004`); may list medical conditions or medications |
| Tech comfort | Medium |
| Frequency | Same as Alex, but more likely to reread the report and ask the assistant questions |
| Key rules that protect them | Measured, reassuring tone (FR15.3); no medical, medication or allergy content in cosmetic commentary (FR15.4); the assistant declines medical questions (FR23.2); consent before health data is processed (FR7) |

## P3 — Sam, the Returning User

| Field | Value |
|-------|-------|
| Role | End User |
| Goals | Come back days later to review the report, download the PDF, browse AI Visuals, ask the assistant follow-up questions, check billing |
| Pain points | Being forced to log in repeatedly; losing their place; unclear account or payment status |
| Context | Session restored from the refresh cookie within 7 days (FR4.4); lands on the Home Overview (FR21, FR26) |
| Tech comfort | Medium |
| Frequency | Several short visits after the report exists |
| Key rules that serve them | Session survives reload and restart (FR4.4, FR4.5); entry-point routing to the right screen (FR26); daily chat cap explained clearly (FR23.3) |

## Future — Admin (no stories)

A future role that reviews, edits, verifies and publishes AI reports and manages recommendation content (section 3). It is **Won't Have** in this scope (section 2.2). The only present-day impact is the `role` field on the user record (`AUTH-009`).

## Persona Relationships and Priority

| Rank | Persona | Why |
|------|---------|-----|
| 1 | P1 Alex | Primary buyer; the core WF-001 journey is designed around them |
| 2 | P2 Jordan | Same journey, but the safety and tone rules exist for them; a failure here is the most harmful |
| 3 | P3 Sam | Drives the post-analysis and session-continuity stories |

All three are the same account role. A single real user can move from P1 or P2 in the first session to P3 on later visits.
