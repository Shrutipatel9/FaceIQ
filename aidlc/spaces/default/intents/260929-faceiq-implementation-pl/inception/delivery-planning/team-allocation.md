# Team Allocation — FaceIQ

## Overview

This document assigns the 15 **Bolts** in `bolt-plan.md` to the people and agents who build and approve them. A Bolt is one build pass over one or more units of work that ends in something runnable and demonstrable.

- **Team setup:** team formation was not run in this plan. Per Q6, every Bolt is built by the **AI developer agent**, and the **Crest Infosystems** delivery team reviews and approves the work. The same review applies to the specifications and designs under `inception/`: `requirements`, `stories`, `components`, `unit-of-work`, `contract-summary` and `team-practices`.
- **Named people:** the client document names Tejash Patel as PM and Jainesh Bhatt as Business Manager (`BC-004`). The individual Crest reviewers for each area are still to be named.

## Roles

| Role | Who | Responsibility |
|------|-----|----------------|
| Builder | AI developer agent (with quality, security and design agents for reviews) | Implements each story's phases, writes tests, opens pull requests |
| Code reviewer / approver | Crest Infosystems developer (to be named) | Reviews and approves every pull request (1 approval required, affirmed Way of Working) |
| Security reviewer | Crest Infosystems (to be named) | Mandatory reviewer on auth, payment and photo/biometric pull requests (B3, B4, B6, B7, B8) |
| Delivery lead | Tejash Patel (PM, `BC-004`) | Bolt acceptance at each demo; requests client inputs listed in `external-dependency-map.md` |
| Client sponsor | Jay Michaels (`BC-004`) | Supplies accounts, keys, specifications and confirmations; accepts spike findings |

## Bolt Assignment

| Bolt | Units | Builder | Approver | Extra reviewer |
|------|-------|---------|----------|----------------|
| B1 walking skeleton | U1 | AI developer agent | Crest developer | — |
| B2 platform services | U2 | AI developer agent | Crest developer | Security reviewer (CI gates) |
| B3 signup and verification | U3 | AI developer agent | Crest developer | Security reviewer |
| B4 login, sessions, account safety | U3 | AI developer agent | Crest developer | Security reviewer |
| B5 consent and questionnaire | U4 | AI developer agent | Crest developer | — |
| B6 photo capture and storage | U5 | AI developer agent | Crest developer | Security reviewer |
| B7 photo validation and identity | U5 | AI developer agent | Crest developer | Security reviewer |
| B8 payment | U6 | AI developer agent | Crest developer | Security reviewer |
| B9 analysis pipeline | U7 | AI developer agent | Crest developer | — |
| B10 measurement and assessments | U8 | AI developer agent | Crest developer | — |
| B11 AI narrative and tiering | U9 | AI developer agent | Crest developer | — |
| B12 AI images and visuals | U10 | AI developer agent | Crest developer | — |
| B13 report, PDF, home | U11 | AI developer agent | Crest developer | — |
| B14 beauty assistant | U12 | AI developer agent | Crest developer | — |
| B15 account and app-wide | U13 | AI developer agent | Crest developer | — |

## Working Agreements

- **Workflow:** one short-lived branch per story or small group of stories, merged into `main` through a squash-merged pull request with CI green and 1 approval (affirmed Way of Working).
- **Parallel work:** B14 and B15 are the only Bolts that run in parallel. They touch different modules (chat versus settings and navigation), so the risk of merge conflicts is low.
- **Spikes:** the two exploratory spikes (CV feasibility and AI-image quality/cost) are run by the AI developer agent. The client sponsor accepts their findings.
