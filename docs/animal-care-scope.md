# Animal care - workstream scope and boundaries

> Note: This file is a **team boundary agreement**, not a design. It states what the animal-care workstream owns, what it assumes without designing, and what it publishes to the other two workstreams. Decisions live in ADRs. This file only says where the seams are, so three parallel workstreams do not produce three disconnected architectures.

| | |
|---|---|
| Owner | Krzysiek Kopacz |
| ADR block | ADR-020 to ADR-023 (reserved; 003-019 left free for the other workstreams) |
| Requirements realised | **FR#2I** (health and feeding anomalies), **FR#2J** (jumping-piranha population), [Appendix A](../requirements/Appendix%20A_%20Core%20functionality.md) 2-4 Animal care |
| Sibling workstreams | Ticketing and access; ops intranet |

## 1. In scope - what this workstream owns

- **The record of what is being cared for.** Enclosures, displays, and colonies across the 55 displays, including their environment context (water parameters, temperature, barriers). Granularity is decided in ADR-020, not here.
- **Keeper observations.** Health notes, behaviour, and condition entries made in the field, including when the animal house has no connectivity.
- **Feeding records.** Amount offered, amount refused, leftover, and aggression - the data behind "how much / how well they are eating" ([1_1_Business challenges.md](../requirements/1_1_Business%20challenges.md)).
- **Environment telemetry from animal-area MQTT devices.** Ingest, gap detection, and freshness marking for the readings this domain depends on.
- **The two AI capabilities.** FR#2I anomaly detection and FR#2J population estimation, including shadow mode, confidence and evidence on every alert, and the fallback when the model is unavailable or wrong.
- **The alert lifecycle.** Raise, route, keeper accept or reject, and resolution - including capturing the accept/reject decision as the training label (see [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md), human-in-the-loop).
- **The data contract** for all of the above, published so the intranet can render it without reading an ADR.
- **The eval harness** for both AI capabilities, under `evals/`.

## 2. Consumed, not designed here

These are **working assumptions** for this workstream, in the same sense that MQTT broker placement is a working assumption inside [ADR-001](../adrs/ADR-001-use-home-bought-token-pool.md) and [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md). Nobody owns them yet. Our ADRs must name them rather than quietly depend on them.

| Assumed | Why we depend on it | Related |
|---|---|---|
| Estate-to-cloud path exists (gateways, store-and-forward, retry, dead-letter, replay) | Keeper entries and sensor readings must survive multi-hour islanding | NFR_4, NFR_5 |
| MQTT broker topology and gateway placement | Animal-area devices need somewhere to publish | NFR_5 |
| Staff identity and role model | Keeper, vet, and duty-manager permissions on alerts | NFR_6 |
| Clock sync across field devices | Ordering feed and observation events after a partition | NFR_4 |

**Risk if they stay unowned:** the three workstreams assume incompatible backbones and judges' criterion 5 (do the additions match the existing architecture) fails. Flagged, not solved.

## 3. Published - our contract with the other workstreams

Surfaces only. Field-level detail is defined in the data contract once ADR-020 fixes granularity.

| We publish | Consumed by | Status |
|---|---|---|
| Enclosure / colony state, with data age and gap flags | Intranet (duty-manager view, keeper views) | Fields TBD - data contract |
| Welfare alerts with confidence, evidence, and suggested action | Intranet (alert inbox, FR#2I / FR#2J) | Fields TBD - data contract |
| Alert decision events (keeper accept / reject / resolve) | Our own training loop; intranet audit view | Fields TBD - data contract |
| Population estimate with error band and census reconciliation | Intranet; Countess reporting | Fields TBD - data contract |
| Containment / escape detection signal | Intranet incident flow - **we emit, they respond** | Fields TBD; response stays deterministic per NFR_7 |

## 4. Out of scope - and who has it

| Not ours | Owner | Reference |
|---|---|---|
| All staff screens (keeper, vet, duty-manager, Countess views) | Intranet workstream | Appendix A 2-1, 2-4 |
| Incident management, escape response, evacuation flows | Intranet workstream | NFR_7 |
| Gate access, entitlements, enclosure checkpoint scans | Ticketing workstream | ADR-001, ADR-002 |
| Popularity, dwell, and throughput counting | Popularity meter (planned) | FR#2D |
| Ops copilot / RAG over SOPs | Intranet workstream | FR#2L |
| **Ride and asset predictive maintenance** | **Unassigned - see note below** | FR#2K, Appendix A 2-3 |

> **Open ownership - FR#2K.** Ride predictive maintenance is specified in [2_FRs.md](../requirements/2_FRs.md) and Appendix A 2-3, and is listed in the README traceability table, but no workstream owns it. It is **not** an explicit requirement in the kata brief - the brief only states that the rides recently passed safety inspection. It must either be picked up by a workstream or explicitly marked descoped. A requirement left in the repository with no owner and no realisation is worse than one that was never claimed.

## 5. Positions taken (and why)

- **Animals only.** This workstream covers animal care. Ride maintenance was considered and dropped; see the FR#2K note above.
- **No named jurisdiction.** We do not design against a specific zoo-licensing or inspection regime, consistent with [4_Assumptions and constraints.md](../requirements/4_Assumptions%20and%20constraints.md). Audit and evidence requirements are justified on business grounds instead: a previously private poisonous collection is now open to the public, and late-discovered illness is the expensive failure ([1_0_Business goals & drivers.md](../requirements/1_0_Business%20goals%20%26%20drivers.md), Animal Welfare Cost).
- **Piranha counting is hybrid.** Feeder events and keeper data are primary, optional camera or sonar at a single tank, periodic human census as ground truth. Decided properly in ADR-023.
- **We build no UI.** Everything staff-facing is a contract handed to the intranet workstream.

## 6. Planned deliverables

| Artefact | Covers | Status |
|---|---|---|
| [ADR-020](../adrs/ADR-020-use-care-subject-as-unit-of-record.md) - care subject as the unit of record | Individual / group / colony, with enclosure as a dated placement | Proposed |
| [ADR-021](../adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md) - keeper observations off the telemetry path | Device-local append-only log, idempotent batch sync, deferred attachments | Proposed |
| [ADR-022](../adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md) - per-subject baselines before any trained model | FR#2I, tiered detector, recall posture, provider independence | Proposed |
| [ADR-023](../adrs/ADR-023-anchor-piranha-population-on-human-census.md) - census-anchored population interval | FR#2J, interval coverage, census as ground truth | Proposed |
| [`evals/animal-health-anomaly/`](../evals/animal-health-anomaly/) | NFR_13 validation - 6 golden cases, gaps listed | Specification, no runner |
| [`evals/piranha-population/`](../evals/piranha-population/) | NFR_13 validation - 5 golden cases, gaps listed | Specification, no runner |
| Data contract | Fields, semantics, freshness for section 3 | Not started |
| `diagrams/` - legend, container view, component view, one targeted AI view per capability | Kata deliverable: a targeted view for each use of AI | Not started |
| Narrative section in `docs/` | Kata deliverable: overview | Not started |

## 7. Open questions

- Which enclosures are instrumented, and with what device classes? Not every display can be assumed sensored; keeper observation is the universal fallback. Device tiers are an ADR-020/021 input.
- Is the vet on-site or external and on-call? Determines whether an external-party interface is needed on the alert path.
- Does the containment / escape signal originate from our sensors, from the intranet, or from both? Needs agreement with the intranet workstream before ADR-021.
- What is the retention period for keeper and veterinary records? NFR_9 says "life of the animal plus a defined legal period" - the period is undefined.
