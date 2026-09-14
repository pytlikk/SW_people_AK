# Von Digitalis Estates | Architectural Katas 2026

This repository is **SW people**’s working submission to O’Reilly’s [Architectural Katas 2026: AI-Assisted Software Architecture](https://www.oreilly.com/live-events/architectural-katas-2026-ai-assisted-software-architecture/0642572412906/). It is a derivation chain from the Von Digitalis Estates brief - not a finished architecture dump, and not an index of empty folders.

## Table of contents

- [Team](#team)
- [Introduction](#introduction)
- [Key objectives](#key-objectives)
- [Derivation chain](#derivation-chain)
  - [Requirements and assumptions](#requirements-and-assumptions)
  - [Event storming](#event-storming)
  - [Architecture characteristics](#architecture-characteristics)
  - [Architecture style](#architecture-style)
  - [Architecture decision records](#architecture-decision-records)
  - [C4 views](#c4-views)
  - [Use cases and AI scenarios](#use-cases-and-ai-scenarios)
  - [Deployment](#deployment)
  - [Fitness functions](#fitness-functions)
- [Traceability](#traceability)
- [Known limitations](#known-limitations)
- [Repository structure](#repository-structure)
- [Resources](#resources)
- [AI assistance](#ai-assistance)

## Team

- Krzysztof Pytlik - [GitHub](https://github.com/pytlikk)
- Mohammed - [GitHub](https://github.com/mroj4n)
- Krzysiek Kopacz - [GitHub](https://github.com/Crafterro)

## Introduction

The Countess has forty inspected eighteenth-century rides and a newly public collection of 200+ animals across 55 displays. She needs today’s 5,000 visitors a day to become at least 15,000 within three years, or the family may have to sell the carnivorous plants. Tickets - including family passes - are the obvious first product. They are not the hard problem.

The risk this architecture is being built around is **patchy connectivity on a sprawling estate**, combined with the later need for measured popularity and AI. The three challenges the brief asks us to put AI on - where to invest and staff, whether the animals are actually well, and why anyone would come back - all starve if the grounds cannot produce trustworthy events on a normal park day. If a gate has to reach a cloud database to admit a guest, access fails where Wi-Fi is thin, and every later model is guessing.

So we are designing **access that still works offline at 40 rides and 55 enclosures**. A visitor buys a fungible **token pool** at home and spends it at live attraction prices; the checkpoint camera scans a **signed static QR** and verifies it locally. That pair is the substrate: checkpoint events can later feed a popularity meter; leftover pool can later feed a return incentive. We have not yet designed the AI that sits on top.

**Depth allocation (deliberate, not a delay):** go deep on ticketing and access first, then the popularity meter, then dynamic-pricing AI, then the return incentive / AI Guide. Animal tracking is a parallel workstream. It is specified in requirements; it is not in ADRs yet.

MQTT-capable hardware is in the brief’s budget. Broker topology, kiosks, and gateway placement are **working assumptions** until their own ADRs exist - they are not decisions in this repository.

## Key objectives

Taken from the brief. Each item says what is actually in the repo. Where there is no ADR yet, we point at requirements and the ADR plan rather than inventing a solution.

### 1. Sell tickets, including family passes, and admit guests on patchy Wi-Fi

Our main contributions for this objective are:

- The guest access story in [Appendix A - Core functionality](./requirements/Appendix%20A_%20Core%20functionality.md) (1-1 discover and buy, 1-2 entitlements and family passes, 1-3 gate access). Family-pass SKUs are specified there; they do not yet have their own ADR.
- [ADR-001 - home-bought token pool, spend at live park prices](./adrs/ADR-001-use-home-bought-token-pool.md) *(Proposed)*: what is sold from home is a currency, not a catalogue of per-attraction tickets.
- [ADR-002 - signed static QR for attraction-token presentation](./adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) *(Proposed)*: the checkpoint camera scans the visitor, verifies locally, and burns the claim on reveal. No phone-to-database round-trip at the gate. Same payload type for 40 rides and 55 enclosures.
- Offline and estate-to-cloud constraints in [3_NFRs.md](./requirements/3_NFRs.md) and [4_Assumptions and constraints.md](./requirements/4_Assumptions%20and%20constraints.md). The estate→cloud path is required if we use the cloud; it is not designed yet.

### 2. Know which parts of the estates are popular, so investment and staffing are not guesswork

Our main contributions for this objective are:

- The challenge stated in [1_1_Business challenges.md](./requirements/1_1_Business%20challenges.md) and the measurement FRs **FR#2D**, **FR#2E**, **FR#2F** in [2_FRs.md](./requirements/2_FRs.md) (identifiers are in that file; Markdown cannot deep-link to a table row).
- Scenario notes in [Appendix B - AI scenarios](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md) (popularity and dwell, flow forecast, staffing and investment advice).
- A planned next decision, not a design: [adrs/README.md](./adrs/README.md) puts a **popularity-meter ADR** after ADR-002, because that record’s `validated` events are the intended counts. There is no popularity ADR in the repository yet.

### 3. Monitor animal health, feeding quality, and jumping-piranha population

Our main contributions for this objective are:

- Keeper-facing scope in [Appendix A](./requirements/Appendix%20A_%20Core%20functionality.md) (2-4 animal care) and FRs **FR#2I**, **FR#2J** in [2_FRs.md](./requirements/2_FRs.md).
- Human-in-the-loop rule in [Appendix B](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md): health and population alerts are advisory; keepers and vets decide. This workstream is **parallel**, not sequenced behind ticketing in the [ADR plan](./adrs/README.md), and it has **no ADR yet**.

### 4. Grow returning visitors and make the estates profitable

Our main contributions for this objective are:

- Drivers and goals in [1_0_Business goals & drivers.md](./requirements/1_0_Business%20goals%20%26%20drivers.md). That document *suggests* a target of ≥30% returning-visitor share of visits; that figure is **not** in the kata brief.
- Yield and loyalty FRs **FR#2A**, **FR#2B**, **FR#2C**, **FR#2G**, **FR#2H** in [2_FRs.md](./requirements/2_FRs.md).
- [ADR-001](./adrs/ADR-001-use-home-bought-token-pool.md): leftover pool survives the visit, so a later return incentive can credit the same wallet; membership and path products are meant to layer on the pool rather than fork checkout. Token expiry and refunds are still open.
- Dynamic-pricing AI and the return incentive / AI Guide are **planned** in [adrs/README.md](./adrs/README.md). They are not designed yet. Pricing is a table writer on the ADR-001 economy, not a second ticketing model.

### 5. Use AI on those three challenges - and be able to tell if it is working

Our main contributions for this objective are the **specification**, not the design:

- AI scenario table **FR#2A–FR#2L** and platform ops **FR#3** in [2_FRs.md](./requirements/2_FRs.md), expanded in [Appendix B](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md).
- Verification intent in [3_NFRs.md](./requirements/3_NFRs.md) (NFR_13–NFR_15) and a placeholder under [evals/](./evals/TEMPLATE.md).
- **No AI ADRs, no per-scenario diagrams, and no golden-case harness yet.** Core ticketing and gate access stay deterministic; that rule is already in the FR file.

## Derivation chain

A judge should be able to walk from the brief to a decision, and see the empty stages rather than infer them. **If it is not in this repository, it does not exist.**

### Requirements and assumptions

This stage is in the repo. Start at [1_0_Business goals & drivers.md](./requirements/1_0_Business%20goals%20%26%20drivers.md) and [1_1_Business challenges.md](./requirements/1_1_Business%20challenges.md), then [2_FRs.md](./requirements/2_FRs.md) and [4_Assumptions and constraints.md](./requirements/4_Assumptions%20and%20constraints.md). Stakeholders, actors, NFRs, risks, glossary, and appendices live under [requirements/](./requirements/) and are listed in [Repository structure](#repository-structure).

### Event storming

**Current boards (this session): visit access, popularity meter, and AI Guide.** See
[event-storming/event_storming.md](./event-storming/event_storming.md) for the index,
component candidates, depth allocation, and open questions.

- **Board V - Visit Access / Ticketing** (deep): pool purchase, QR-reveal, checkpoint verify,
  burn-on-reveal, MQTT audit channel (`validated`/`revoked`), kiosk rescue path. Reconstructs
  ADR-001 and ADR-002 as a domain event board. Component candidates VA-01..VA-04.
- **Board P - Popularity Meter** (next-ADR depth): ADR-002 `validated` events + ride cycles +
  optional zone occupancy - ranked counts + gap-flagging (silence = unknown, NOT zero).
  Component candidates PM-01..PM-02. This board owns the popularity feed; the intranet
  heat map (CC-06) is a consumer.
- **Board G - Return Incentive / AI Guide** (shallow): leftover pool hook (ADR-001), opt-in
  trail (FR#2G), win-back trigger (FR#2H). Stays shallow until PM ADR exists.
  Component candidates AG-01..AG-03.

**Ops boards (backup):** Animal Care and Intranet / Estate OS are in
[event-storming/ops-backup/](./event-storming/ops-backup/event_storming.md).
Those two boards are the prior session; they are not the current derivation-chain work.
Component candidates CC-01 to CC-14 are defined there; CC-12 (Popularity Aggregator) is
owned by PM-01 on the guest-lane board above.

There was no physical sticky-note workshop; all iterations are reconstructed from the
committed requirements documents and ADRs. Honesty note in each index file.

Outbound: popularity-meter ADR (next up in [adrs/README.md](./adrs/README.md)), then
the characteristics funnel.

### Architecture characteristics

**No funnel in the repository yet** - no candidate list, no cut to a top seven, no driving top three, no downplayed-characteristic ADRs.

[3_NFRs.md](./requirements/3_NFRs.md) is a requirements list, not that funnel. The two ADRs we do have name *local* evaluation criteria: ADR-001 is driven by repricing agility and SKU operability; ADR-002 by offline admission at the checkpoint and operability (commodity cameras, no wallet-cert programme). Those are not a system-wide characteristics decision.

### Architecture style

**Not in the repository yet.** No style ADR, no rejected-styles write-up.

### Architecture decision records

This is where the work currently is. Index and next-up plan: [adrs/README.md](./adrs/README.md). Template: [ADR-000-template.md](./adrs/ADR-000-template.md).

| Record | Decision (one line) | Status |
|---|---|---|
| [ADR-001](./adrs/ADR-001-use-home-bought-token-pool.md) | Home-bought **token pool**, spent at live attraction prices - not per-attraction tickets from home. | Proposed |
| [ADR-002](./adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) | **Signed static QR**; checkpoint camera scans the visitor; local verify; burn-on-reveal; 40 rides + 55 enclosures; no phone→DB at the gate. | Proposed |

Both records are **Proposed**, not Accepted. MQTT broker, kiosks, and gateway placement wait for their own ADRs ([plan](./adrs/README.md)).

### C4 views

**Not in the repository yet.** [diagrams/TEMPLATE.md](./diagrams/TEMPLATE.md) is a placeholder, not a context or container view.

### Use cases and AI scenarios

Scenarios are **named** in [Appendix B](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md) and **FR#2A–FR#2L**. Sequence diagrams, per-AI-scenario folders, and targeted AI views are **not in the repository yet**. [docs/TEMPLATE.md](./docs/TEMPLATE.md) is a placeholder for a later narrative, not an overview deliverable.

### Deployment

**Not in the repository yet.** Cloud is allowed; the brief requires an estate→cloud path. That path is a constraint in [4_Assumptions and constraints.md](./requirements/4_Assumptions%20and%20constraints.md), not a topology we have drawn.

### Fitness functions

**Not in the repository yet.** [evals/TEMPLATE.md](./evals/TEMPLATE.md) and [evals/golden-case-template.json](./evals/golden-case-template.json) are templates. There are no arithmetic fitness functions and no per-capability golden cases.

## Traceability

Capability → requirement → what (if anything) realises it today. In [2_FRs.md](./requirements/2_FRs.md) the AI rows are labelled **2A–2L** (column FR#); core platform is **FR#1** and MLOps is **FR#3**. Markdown cannot deep-link to a table row.

| Business capability | Related FRs | Realised by |
|---|---|---|
| Home purchase of spendable visit value (family passes specified, SKU not decided) | FR#1; Appendix A 1-1, 1-2 | [ADR-001](./adrs/ADR-001-use-home-bought-token-pool.md) for the pool economy. Family-pass product shape: requirements only. |
| Offline attraction access (40 rides + 55 enclosures) | FR#1; Appendix A 1-3 | [ADR-002](./adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) |
| Live per-attraction prices without reissuing home purchases | FR#1; FR#2A (algorithm later) | [ADR-001](./adrs/ADR-001-use-home-bought-token-pool.md) (economy and price table). Pricing *AI*: not yet. |
| Popularity, flow, and staffing evidence | FR#2D, FR#2E, FR#2F | Not yet. Intended input: ADR-002 `validated` events ([plan](./adrs/README.md)). |
| Returning visitors / itinerary / AI Guide | FR#2G, FR#2H | Not yet. Leftover pool in ADR-001 is the hook, not the design. |
| Yield experiments and cohort analysis | FR#2A, FR#2B, FR#2C | Not yet. |
| Animal health, feeding, piranha population | FR#2I, FR#2J | Not yet (parallel workstream). |
| Ride predictive maintenance | FR#2K | Not yet. |
| Ops copilot | FR#2L | Not yet. |
| MLOps / golden-case evaluation | FR#3 | Not yet (`evals/` is a template). |

## Known limitations

Honesty for the next iteration, not a list of regrets.

- **Proposed, not Accepted.** ADR-001 and ADR-002 can still be reversed; they should not be read as locked estate policy.
- **No C4, no style, no characteristics funnel.** Guest-lane event storming is in the repo; the chain stops before characteristics. Architecture style, C4, and fitness functions are not in the repository yet.
- **No AI ADRs yet.** Popularity meter is the next Proposed ADR (Board P). Dynamic pricing and AI Guide are later. Animal tracker and intranet have an ops-backup storm only; no ADRs.
- **No fitness functions, no per-AI diagrams.** `docs/`, `diagrams/`, and `evals/` hold templates. Placeholders are not architecture.
- **MQTT broker, kiosks, and gateway placement** are assumptions (also stated inside ADR-001/002). Do not treat them as decided.
- **Open product questions inside the ADRs we do have:** token expiry and refunds; pack sizes; which of the 55 displays are paid; fraud-window length between QR-reveal and cache write; kiosk paper vs screen reprint.
- **Next decisions** are listed in [adrs/README.md](./adrs/README.md): popularity meter → dynamic pricing AI → return incentive / AI Guide.

## Repository structure

Existing top-level folders only:

- [`requirements/`](./requirements/) - problem background: goals, challenges, FRs, NFRs, assumptions, risks, glossary, kata extract, appendices, suggested OKRs.
- [`adrs/`](./adrs/) - ADR template, two Proposed records, and the next-ADR plan.
- [`event-storming/`](./event-storming/event_storming.md) - **current** guest-lane boards (Visit Access, Popularity Meter, AI Guide); first-pass dumps, digitised SVGs, component candidates VA-01..VA-04 / PM-01..PM-02 / AG-01..AG-03. Ops boards (Animal Care, Intranet) are in the `ops-backup/` subfolder.
- [`docs/`](./docs/TEMPLATE.md) - placeholder for a later architecture narrative.
- [`diagrams/`](./diagrams/TEMPLATE.md) - placeholder; no architecture diagrams yet.
- [`evals/`](./evals/TEMPLATE.md) - placeholder; no live eval harness yet.

Requirement files (every link is a file in the repo):

- [1_0 Business goals & drivers](./requirements/1_0_Business%20goals%20%26%20drivers.md)
- [1_1 Business challenges](./requirements/1_1_Business%20challenges.md)
- [1_2 Stakeholders](./requirements/1_2_Stakeholders.md)
- [1_3 Actors and actions](./requirements/1_3_Actors%20and%20actions.md)
- [2_FRs](./requirements/2_FRs.md)
- [3_NFRs](./requirements/3_NFRs.md)
- [4_Assumptions and constraints](./requirements/4_Assumptions%20and%20constraints.md)
- [5_Risks and mitigation](./requirements/5_Risks%20and%20mitigation.md)
- [6_Glossary](./requirements/6_Glossary.md)
- [7_Kata expectations](./requirements/7_Kata%20expectations.md)
- [Appendix A - Core functionality](./requirements/Appendix%20A_%20Core%20functionality.md)
- [Appendix B - AI scenarios](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md)
- [Appendix C - Future scope](./requirements/Appendix%20C_%20Future%20scope.md)
- [Suggested OKRs](./requirements/suggested%20OKRs/OKRs.md)

## Resources

- [Kata expectations (briefing extract)](./requirements/7_Kata%20expectations.md)
- [O’Reilly live event - Architectural Katas 2026: AI-Assisted Software Architecture](https://www.oreilly.com/live-events/architectural-katas-2026-ai-assisted-software-architecture/0642572412906/)
- [The Kata Log](https://github.com/TheKataLog)

## AI assistance

The team used AI assistants while producing this submission and takes responsibility for the content.
