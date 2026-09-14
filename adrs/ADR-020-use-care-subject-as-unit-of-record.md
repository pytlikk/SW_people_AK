# ADR-020 - Use a care subject, distinct from its enclosure, as the unit of record

## Date

2026-09-14

## Status

Proposed

## Context

The estate opens a previously private collection to the public: **200+ animals across 55 displays and enclosures**, aquatic and land, exotic and poisonous, including a jumping-piranha colony. Animal care must record health, feeding ("how much / how well they are eating"), and - for the piranha - population ([1_1_Business challenges.md](../requirements/1_1_Business%20challenges.md), FR#2I, FR#2J).

Before anything can be detected, something has to be *identified*. Every later choice in this workstream - the data contract, what FR#2I can possibly detect, what FR#2J counts, what the intranet renders - depends on what a record is *about*.

The naive framing is "individual animal or enclosure". That framing hides the fact that three different things are being conflated:

| Concept | What it is | Who produces it |
|---|---|---|
| The subject of care | What a welfare claim is asserted about | Keepers, vets |
| The enclosure | A physical space with environment readings | MQTT devices, fixed infrastructure |
| The observation | What was actually seen or measured, and when | Keepers, sensors |

These do not line up. A solitary venomous snake is one subject in one vivarium. A piranha colony is one subject whose *cardinality is itself an estimate*. A troop sharing a paddock is many animals fed as one group. And an animal moved to quarantine changes enclosure without changing identity - which is precisely the moment its history matters most.

Working assumptions for this record: keeper observation is the universal data source and the only one present at every display; sensor coverage is partial and is not decided here; estate connectivity is patchy, so subject identification must be resolvable in the field with no network ([4_Assumptions and constraints.md](../requirements/4_Assumptions%20and%20constraints.md), NFR_4). Which existing collections already carry individual identifiers is **TBD** - the brief does not say.

This record does **not** decide: sensor placement or device classes; how keepers capture an observation (ADR-021); the anomaly-detection approach (ADR-022); the population-estimation method (ADR-023); or field-level schema, which follows in the data contract once this record fixes granularity.

**Foreclosed here:** treating an enclosure as the welfare identity. Environment readings are *context joined to* a subject, never the subject's health record itself. A tank cannot be sick.

## Evaluation criteria

- **Welfare continuity (driving)** - an animal's history survives a move between enclosures. Target: 0 observations orphaned by a placement change (CI).
- **Colony as first-class (driving)** - FR#2J population is an attribute of a modelled subject, not a special-case table. The piranha must not be an exception path.
- **Keeper entry cost** - a gloved keeper outdoors resolves the subject offline, and is **never** forced to invent individual identity (NFR_17, NFR_4).
- **AI baseline quality** - FR#2I detects "eating less well *than usual*". The unit of record must be the unit a baseline is computed on.
- **Retention fit** - NFR_9 requires keeper and veterinary records "for the life of the animal plus a defined legal period". Period is TBD; an animal-lifetime record must be possible where individuals exist.
- **Modifiability** - adding a collection, splitting a group, or re-housing does not change schema. Nothing hard-codes 55 (NFR_10).

## Options

- **Option A - Enclosure is the unit of record**: health notes, feed events, and environment all attach to a display/enclosure id. The 55 displays are the spine.
- **Option B - Individual animal is the unit of record**: each of the 200+ animals carries an identity record; groups are aggregations over individuals.
- **Option C - Care subject, with enclosure as a dated placement (chosen)**: the subject is whatever welfare is asserted about - individual, group, or colony. The enclosure is where it currently lives. Observations attach to a subject; environment readings attach to an enclosure; the two join through placement-at-time-of-observation.
- **Option D - Per-collection hybrid**: some collections modelled individually, others by enclosure, decided per collection, producing two record shapes.

| | Welfare continuity (driving) | Colony as first-class (driving) | Keeper entry cost | AI baseline quality | Modifiability |
|---|---|---|---|---|---|
| A enclosure | Fail - a move to quarantine severs history | Partial - population has no owner but the room | Lowest - the room is obvious | Fail - baseline is per-room, so re-housing resets it | Good |
| B individual | Pass | Fail - piranha are not individually identifiable | Fail - forces invented identity for fish and colonies | Best where identity is real, impossible where it is not | Poor - group changes churn records |
| C care subject | Pass - placement changes, subject does not | Pass - colony is a subject type with an estimated cardinality | Acceptable - sole-subject enclosures resolve automatically | Pass - baseline follows the subject | Good - new subject types, not new schema |
| D per-collection hybrid | Partial | Partial | Confusing - two mental models in one app | Two pipelines | Worst - two contracts, two eval harnesses |

Not options: individual identification by vision re-identification is excluded as the *identity mechanism* in v1 (unproven at this cost, and it would make identity depend on a model - the same objection ADR-002 raises against putting a model on the access path). A taxonomy or species-registry engine is out of scope; species is an attribute, not a structure.

## Decision

**Use the care subject as the unit of record, with the enclosure as a separate, dated placement.**

A subject is one of three types: **individual**, **group**, or **colony**. Observations, feed events, and alerts attach to a subject. Environment readings attach to an enclosure. The join is placement at the time of the observation, and placement is append-only, consistent with the field-write rule in [Appendix A](../requirements/Appendix%20A_%20Core%20functionality.md) 2-6.

Welfare continuity and colony support decide it. Option A fails continuity outright: the day you move a sick animal to quarantine is the day Option A erases the history explaining why. Option B fails on the piranha, and FR#2J is one of only three animal requirements the brief actually states - a model that cannot express the requirement is not a candidate. Option D buys nothing that C does not, at the price of two contracts and two eval harnesses for a three-person team.

The colony is the case that proves the model. A colony is not a degenerate enclosure; it is a subject whose population is an estimate with an error band. Under Option A that number has nowhere to live but the room, and under Option B it is a count of records that cannot exist. Under C it is an attribute of a subject, and FR#2J becomes an ordinary capability rather than a bolt-on.

## Key differentiators

- **Identity survives location.** Quarantine, rotation, and breeding separation are placement events, not new animals. The history a vet needs is the history that still exists.
- **The piranha are not a special case.** One subject type covers them; FR#2J reads an attribute rather than a parallel table.
- **Keepers never invent identity.** Where individuals are not distinguishable, the subject is honestly a group or a colony. Fabricated identity would poison the FR#2I training data at source.
- **Baselines follow the animal, not the room.** "Eating less than usual" stays meaningful after a move, which is the only reason FR#2I can work at all.
- **Environment stays context.** Water and temperature readings are joined evidence on an alert, never a welfare claim on their own.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Data integrity | **Improved (driving)** | Welfare history cannot be severed by a re-housing. Placement is append-only, so what was true at a point in time stays recoverable. |
| Modifiability | **Improved** | New collections, new subject types, and re-housing are data. Nothing hard-codes the 55 displays (NFR_10). |
| Auditability | **Improved** | A record spanning an animal's life is expressible, which NFR_9 requires and an enclosure-keyed model cannot provide. |
| Interoperability | **Improved** | One record shape for the whole collection means one contract to the intranet. Option D would have published two. |
| Simplicity | **Weakened** | Three concepts - subject, placement, observation - where the naive case needs one. |
| Performance | **Weakened** | Every feature computation resolves placement-at-time. The enclosure model needs no join at all. |
| Usability | **Weakened** | Keepers must identify a subject, not only a room, wherever an enclosure holds more than one. |

**Deliberately downplayed: simplicity and read performance.** Both are cheap to lose here because animal care is not a hot path in the ADR-002 sense - no guest waits at a barrier while this join resolves. Spending latency to keep a welfare history intact is the right trade in this domain and would be the wrong one at a gate.

**Fit with the existing architecture.** Append-only field writes and cloud-side reconciliation are the rule already set in [Appendix A](../requirements/Appendix%20A_%20Core%20functionality.md) 2-6, and the offline-first posture matches ADR-001 and ADR-002 rather than introducing a second philosophy.

## Consequences

### Positive

- Placement changes preserve full observation history (welfare continuity).
- Population is a first-class attribute, so FR#2J needs no exception path (colony as first-class).
- Per-subject feeding baselines are stable across re-housing, which is what FR#2I scores against (AI baseline quality).
- NFR_9 animal-lifetime retention is expressible wherever individuals exist (retention fit).
- New collections and re-housing are data, not schema (modifiability).

### Negative

- **Every query and every AI feature must resolve placement-at-time.** Get that join wrong and environment readings from the wrong tank are silently mixed into a subject's features - a wrong answer that looks right. This is the primary cost of the decision.
- Keepers must identify a subject, not just a room. Where an enclosure holds one subject this should resolve automatically, but that is UX work this record hands to the intranet workstream.
- Group subjects hide individuals. "The troop ate 4kg" cannot answer "which one is off its food". Reduced resolution for groups is accepted in v1; promoting an individual out of a group is an explicit operation, shape TBD.
- Any metric normalised per animal in a colony (feed per fish) inherits the population error band and must never be rendered as precise.
- More entities than Option A, and more to build inside the kata timebox.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Placement-at-time join error | Environment features attributed to the wrong subject after a move; output looks plausible | Placement is append-only dated events; CI test on the join; observation with unresolvable placement carries a gap flag rather than a guessed enclosure |
| Wrong-subject entry | Keeper records a feed event against the wrong subject in a shared paddock | Default to the sole subject where an enclosure has one; corrections are append-only amendments, never edits (Appendix A 2-6) |
| Subject churn | Births, deaths, splits, merges, transfers off-estate | Lifecycle events on the subject; which events exist is TBD before the data contract |
| Colony cardinality misread as exact | "417 piranha" presented without its interval | Error band mandatory in the published contract; enforced in ADR-023 |
| Over-modelling | A three-person team builds a taxonomy engine instead of a welfare tracker | Exactly three subject types; species is an attribute; no registry engine |
| Group resolution gap | A sick individual inside a group is masked by group-level feeding data | High-recall posture on group subjects (ADR-022); keeper note remains the escalation path |

## Verification

**Tests (CI)**

- Moving a subject between enclosures preserves its full observation and feed history; 0 records orphaned.
- An observation resolves to exactly one subject and, through placement-at-time, at most one enclosure environment series.
- A colony subject accepts a population attribute with an interval; an individual subject does not require one.
- Adding a 56th display, or a new collection, requires no schema change.
- An observation whose placement cannot be resolved is flagged, not silently joined to the subject's last known enclosure.

**Ops check**

- A keeper can record a feed event against the correct subject with no network, in an enclosure holding more than one subject.

**Open questions**

- Which existing collections already carry individual identifiers - before the data contract.
- Subject lifecycle events (birth, death, transfer, split, merge) - before the data contract.
- Promotion of an individual out of a group subject - before beta.
- Retention period behind NFR_9 "life of the animal plus a defined legal period" - undefined in requirements.

**Revisit triggers**

- Keepers routinely need per-individual resolution inside group subjects, and work around it in free-text notes -> revisit promoting groups to individuals.
- Individual identification of a collection becomes reliable and cheap -> revisit the subject type for that collection only; the model already permits it.

## Conclusion

The unit of record is the care subject - individual, group, or colony - and the enclosure is where it currently lives, not what it is. Identity survives re-housing, the piranha colony stops being an exception, and keepers are never asked to invent an identity that does not exist.

Related: ADR-021 (how keepers capture an observation offline), ADR-022 (FR#2I anomaly detection over these subjects), ADR-023 (FR#2J population as a colony attribute). Field-level schema follows in the data contract; boundaries in [animal-care-scope.md](../docs/animal-care-scope.md).
