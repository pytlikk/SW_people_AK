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

**Depth allocation (deliberate, not a delay):** go deep on ticketing and access first, then the popularity meter, then dynamic-pricing AI, then the return incentive / AI Guide. **Animal care runs as a parallel workstream and is now designed** - [ADR-020 to ADR-023](./adrs/README.md), five diagrams, and eleven golden cases. Its boundaries are stated in [animal-care-scope.md](./docs/animal-care-scope.md).

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
- Human-in-the-loop rule in [Appendix B](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md): health and population alerts are advisory; keepers and vets decide.
- [ADR-020 - care subject as the unit of record](./adrs/ADR-020-use-care-subject-as-unit-of-record.md) *(Proposed)*: what a welfare record is *about* is a subject - individual, group, or colony - and the enclosure is a dated placement. History survives a move to quarantine, and the piranha colony stops being an exception.
- [ADR-021 - keeper observations off the telemetry path](./adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md) *(Proposed)*: a lost sensor reading is a gap; a lost keeper note is unrecoverable and a missing training label. Device-local append-only log, idempotent batch sync, deferred attachments.
- [ADR-022 - per-subject baselines before any trained model](./adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md) *(Proposed)*: **FR#2I**. No labelled illness events exist on day one, so the detector starts as arithmetic and earns its way up a tier ladder. The alert inbox is the labelling machine.
- [ADR-023 - census-anchored population interval](./adrs/ADR-023-anchor-piranha-population-on-human-census.md) *(Proposed)*: **FR#2J**. The answer is an interval that widens away from its anchor, never a count - so the system asks for a census when it needs one.
- Targeted AI views for both capabilities, a container view, a component view, and a sequence: [diagrams/00-legend.md](./diagrams/00-legend.md).
- The [data contract](./docs/animal-care-data-contract.md): what is recorded, what it means, and ten machine-checkable invariants. This is the interface the ops intranet builds against, and what the golden cases are typed against.
- Golden cases: [`evals/animal-health-anomaly/`](./evals/animal-health-anomaly/) (6) and [`evals/piranha-population/`](./evals/piranha-population/) (5). Specifications, not a runner.

### 4. Grow returning visitors and make the estates profitable

Our main contributions for this objective are:

- Drivers and goals in [1_0_Business goals & drivers.md](./requirements/1_0_Business%20goals%20%26%20drivers.md). That document *suggests* a target of ≥30% returning-visitor share of visits; that figure is **not** in the kata brief.
- Yield and loyalty FRs **FR#2A**, **FR#2B**, **FR#2C**, **FR#2G**, **FR#2H** in [2_FRs.md](./requirements/2_FRs.md).
- [ADR-001](./adrs/ADR-001-use-home-bought-token-pool.md): leftover pool survives the visit, so a later return incentive can credit the same wallet; membership and path products are meant to layer on the pool rather than fork checkout. Token expiry and refunds are still open.
- Dynamic-pricing AI and the return incentive / AI Guide are **planned** in [adrs/README.md](./adrs/README.md). They are not designed yet. Pricing is a table writer on the ADR-001 economy, not a second ticketing model.

### 5. Use AI on those three challenges - and be able to tell if it is working

Specification for most scenarios; design for animal care.

- AI scenario table **FR#2A–FR#2L** and platform ops **FR#3** in [2_FRs.md](./requirements/2_FRs.md), expanded in [Appendix B](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md).
- Verification intent in [3_NFRs.md](./requirements/3_NFRs.md) (NFR_13–NFR_15).
- **Designed and verifiable: FR#2I and FR#2J.** Two AI ADRs with trade-off tables, two targeted AI views, eleven golden cases, and a stated primary metric for each - recall against keeper- and vet-confirmed events, and interval coverage against census. Both answer the uncertainty question concretely: only the narrative tier touches a vendor, so a provider vanishing costs prose, not detection.
- **Still specification only: FR#2A-FR#2H, FR#2K, FR#2L, FR#3.** No ADRs, no diagrams, no golden cases for pricing, popularity, loyalty, ride maintenance, the ops copilot, or MLOps.
- Core ticketing and gate access stay deterministic; that rule is already in the FR file.

## Derivation chain

A judge should be able to walk from the brief to a decision, and see the empty stages rather than infer them. **If it is not in this repository, it does not exist.**

### Requirements and assumptions

This stage is in the repo. Start at [1_0_Business goals & drivers.md](./requirements/1_0_Business%20goals%20%26%20drivers.md) and [1_1_Business challenges.md](./requirements/1_1_Business%20challenges.md), then [2_FRs.md](./requirements/2_FRs.md) and [4_Assumptions and constraints.md](./requirements/4_Assumptions%20and%20constraints.md). Stakeholders, actors, NFRs, risks, glossary, and appendices live under [requirements/](./requirements/) and are listed in [Repository structure](#repository-structure).

### Event storming

**Not in the repository yet.** There is no event-storming board, photo, or component-candidate list.

### Architecture characteristics

**No funnel in the repository yet** - no candidate list, no cut to a top seven, no driving top three, no downplayed-characteristic ADRs.

[3_NFRs.md](./requirements/3_NFRs.md) is a requirements list, not that funnel. Every ADR names *local* evaluation criteria instead: ADR-001 is driven by repricing agility and SKU operability; ADR-002 by offline admission and operability; ADR-020 by welfare continuity and colony support; ADR-021 by not losing a human observation and by label integrity; ADR-022 by cold start and species heterogeneity; ADR-023 by honest uncertainty and change latency. Those are six local decisions, not a system-wide characteristics decision.

### Architecture style

**Not in the repository yet.** No style ADR, no rejected-styles write-up.

### Architecture decision records

This is where the work currently is. Index and next-up plan: [adrs/README.md](./adrs/README.md). Template: [ADR-000-template.md](./adrs/ADR-000-template.md).

| Record | Decision (one line) | Status |
|---|---|---|
| [ADR-001](./adrs/ADR-001-use-home-bought-token-pool.md) | Home-bought **token pool**, spent at live attraction prices - not per-attraction tickets from home. | Proposed |
| [ADR-002](./adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) | **Signed static QR**; checkpoint camera scans the visitor; local verify; burn-on-reveal; 40 rides + 55 enclosures; no phone→DB at the gate. | Proposed |
| [ADR-020](./adrs/ADR-020-use-care-subject-as-unit-of-record.md) | A **care subject** - individual, group, or colony - is the unit of record; the enclosure is a dated placement. | Proposed |
| [ADR-021](./adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md) | Keeper observations stay **off the MQTT telemetry path**: device-local append-only log, idempotent batch sync, deferred attachments. | Proposed |
| [ADR-022](./adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md) | Score each subject against **its own baseline** before any trained model; tier ladder set by label availability. | Proposed |
| [ADR-023](./adrs/ADR-023-anchor-piranha-population-on-human-census.md) | Publish a **census-anchored interval** for the piranha colony, never a count. | Proposed |

All records are **Proposed**, not Accepted. Numbers are allocated in blocks so parallel workstreams do not collide: 001-019 ticketing and its follow-ons, 020-029 animal care ([index](./adrs/README.md)). MQTT broker, kiosks, and gateway placement still wait for their own ADRs, and the animal-care records name that dependency as an assumption rather than designing it.

### C4 views

**Animal care only.** [Container view](./diagrams/c2-containers-animal-care.md) and [component view](./diagrams/c3-components-animal-care.md), with a [legend](./diagrams/00-legend.md) because shapes carry meaning. PNG exports in [`diagrams/png/`](./diagrams/png/).

**No system context view, and nothing for ticketing or the intranet.** There is no C1, and the two ticketing ADRs have no diagram of any kind.

### Use cases and AI scenarios

Scenarios are **named** in [Appendix B](./requirements/Appendix%20B_%20AI%20scenarios%20explained.md) and **FR#2A–FR#2L**. Targeted AI views exist for two of them: [animal health and feeding anomalies](./diagrams/ai-animal-health-anomaly.md) (FR#2I) and [piranha population](./diagrams/ai-piranha-population.md) (FR#2J), plus a [sequence](./diagrams/seq-offline-observation-to-label.md) tracing one keeper observation from a dead zone to a training label.

**The other ten AI scenarios have no targeted view.** [docs/TEMPLATE.md](./docs/TEMPLATE.md) is still a placeholder; the only narrative in `docs/` is the animal-care [scope and boundaries](./docs/animal-care-scope.md), which is a team agreement rather than an overview deliverable.

### Deployment

**Not in the repository yet.** Cloud is allowed; the brief requires an estate→cloud path. That path is a constraint in [4_Assumptions and constraints.md](./requirements/4_Assumptions%20and%20constraints.md), not a topology we have drawn.

### Fitness functions

**Golden cases for two capabilities, and no runner for any of them.** [`evals/animal-health-anomaly/`](./evals/animal-health-anomaly/) holds six and [`evals/piranha-population/`](./evals/piranha-population/) holds five, each stating the behaviour it defends, what must never happen, and which ADR it comes from. Both folders also list the cases we did **not** write, so the gaps are visible rather than discovered.

Deterministic tiers are exactly reproducible, so most of the animal-care detector can be tested like ordinary software. The GenAI tier is not, so its case asserts constraints - names no disease, states no dose, cites only fields present on the alert - instead of an expected string.

**Nothing executes them.** There is no harness, no CI job, and no golden cases for the other ten AI scenarios.

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
| Animal welfare record and keeper capture | FR#2I, FR#2J; Appendix A 2-4 | [ADR-020](./adrs/ADR-020-use-care-subject-as-unit-of-record.md) (unit of record), [ADR-021](./adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md) (offline capture). [Boundaries](./docs/animal-care-scope.md). |
| Animal health and feeding anomalies | FR#2I | [ADR-022](./adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md), [targeted AI view](./diagrams/ai-animal-health-anomaly.md), [6 golden cases](./evals/animal-health-anomaly/). |
| Jumping-piranha population | FR#2J | [ADR-023](./adrs/ADR-023-anchor-piranha-population-on-human-census.md), [targeted AI view](./diagrams/ai-piranha-population.md), [5 golden cases](./evals/piranha-population/). |
| Ride predictive maintenance | FR#2K | Not yet - and **no workstream owns it**. Not an explicit requirement in the brief. |
| Ops copilot | FR#2L | Not yet. |
| MLOps / golden-case evaluation | FR#3 | Partly. Golden cases and a promotion gate are specified for FR#2I and FR#2J; nothing runs them, and there is no model registry or drift monitor. |

## Known limitations

Honesty for the next iteration, not a list of regrets.

- **Proposed, not Accepted.** All six records can still be reversed; they should not be read as locked estate policy.
- **No style, no event storming, no characteristics funnel, no C1.** The derivation chain is still incomplete. C4 container and component views exist for animal care only.
- **Coverage is uneven by design, and it shows.** Ticketing has two ADRs and no diagram. Animal care has four ADRs, five diagrams, and eleven golden cases. The intranet has neither. That is the cost of splitting three ways with the time available, not a claim that the halves are balanced.
- **Eight of twelve AI scenarios are undesigned.** Popularity meter, dynamic pricing, return incentive / AI Guide, cohort analysis, itineraries, win-back, ride maintenance, and the ops copilot are specified in requirements only.
- **Nothing executes the golden cases.** They are specifications of correct behaviour, not a harness. `docs/TEMPLATE.md` is still a placeholder.
- **FR#2K has no owner.** Ride predictive maintenance sits in the requirements with no workstream behind it. It is not an explicit requirement in the brief either - the brief only says the rides recently passed inspection.
- **MQTT broker, kiosks, and gateway placement** are assumptions (also stated inside ADR-001/002). Do not treat them as decided.
- **Open product questions inside the ADRs we do have:** token expiry and refunds; pack sizes; which of the 55 displays are paid; fraud-window length between QR-reveal and cache write; kiosk paper vs screen reprint.
- **Next decisions** are listed in [adrs/README.md](./adrs/README.md): popularity meter → dynamic pricing AI → return incentive / AI Guide.

## Repository structure

Existing top-level folders only:

- [`requirements/`](./requirements/) - problem background: goals, challenges, FRs, NFRs, assumptions, risks, glossary, kata extract, appendices, suggested OKRs.
- [`adrs/`](./adrs/README.md) - ADR template, six Proposed records in two number blocks, and the next-ADR plan.
- [`docs/`](./docs/) - animal-care [scope and boundaries](./docs/animal-care-scope.md) and [data contract](./docs/animal-care-data-contract.md); the architecture narrative is still a template.
- [`diagrams/`](./diagrams/00-legend.md) - legend, C4 container and component views for animal care, two targeted AI views, one sequence, and PNG exports.
- [`evals/`](./evals/) - golden cases for [animal health anomalies](./evals/animal-health-anomaly/) and [piranha population](./evals/piranha-population/). No runner.

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
