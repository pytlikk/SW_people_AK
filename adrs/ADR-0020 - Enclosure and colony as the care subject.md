# ADR-0020 - The care subject is the enclosure or colony, and its cardinality is a property

## Date

2026-09-14

## Status

Proposed

## Context

The estate has over 200 animals across 55 displays and enclosures, aquatic and land-based, including a jumping-piranha colony. The brief asks for tracking of animal health, how much and how well they are eating, and - for the piranha - population levels.

The instinctive data model is one record per animal. It is wrong here, and the reason is not squeamishness about databases: **you cannot reliably identify an individual piranha, and pretending you can corrupts everything built on top of it.**

The 200+ figure is itself informative. Spread across 55 displays it averages under four animals per display, which means the collection is a mix: some displays hold one or two large identifiable animals, others hold a shoal or a group whose members are indistinguishable from outside. A uniform individual-animal model forces the second case into fiction - inventing `piranha_0417` and then guessing which fish it was this morning.

[4_Assumptions and constraints](../requirements/4_Assumptions%20and%20constraints.md) already states the position: enclosure- or colony-level tracking is sufficient except where individual IDs already exist, and piranha are colony-level with periodic census.

What follows architecturally is the interesting part. If the subject can be a group, then **how many members it has is uncertain**, and that uncertainty has to be a first-class property of the subject rather than a caveat in a report. This is the record that makes [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)'s interval possible.

This record decides the care subject model. It does not decide the keeper data path ([ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md)), anomaly detection ([ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)), or the population method ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)).

**Foreclosed here:** a data model requiring an individual animal identity for every observation, and any subject whose member count is stored as a bare integer when it is actually an estimate.

## Evaluation criteria

- **Honest about identifiability (driving)** - the model must not require an identity the keepers cannot establish. Fabricated identity is worse than acknowledged aggregation.
- **Cardinality uncertainty is representable (driving)** - a colony's count must be able to be an estimate with an interval, structurally.
- **Supports both ends of the collection** - a named venomous snake and a piranha shoal in one schema, without special-casing every query.
- **Matches keeper practice** - keepers already work in rounds by enclosure; the model should not fight that.
- **Welfare record keeping** - veterinary records must persist for the life of the animal where an individual exists (NFR_9).

## Options

- **Option A - Individual animal per record, always**: every animal gets an identity; groups are collections of individuals.
- **Option B - Enclosure-only**: the enclosure is the only subject; individuals are not modelled at all.
- **Option C - Polymorphic care subject (chosen)**: one subject type with a `subjectKind` of individual, group, or colony, and cardinality carrying its own certainty.
- **Option D - Two parallel models**: separate individual-animal and colony subsystems.

| | Honest about identifiability (driving) | Cardinality uncertainty (driving) | Both ends of collection | Keeper practice | Complexity |
|---|---|---|---|---|---|
| A Individual always | Fail - invents identities for shoals and then requires guessing which one was observed | Fail - count is implied by row count, which is a lie for a colony | Partial | Fail - keepers do not observe individual fish | Low |
| B Enclosure only | Pass | Partial - possible but no place to put an individual's veterinary history | Fail - loses the named snake's record | Pass | Lowest |
| C Polymorphic subject | Pass - kind is declared | Pass - certainty is a property of the subject | Pass | Pass - rounds are by enclosure either way | Medium |
| D Two models | Pass | Pass | Pass | Partial | High - every query, alert, and UI written twice |

Not options: individual identification of fish (foreclosed in [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) - occlusion and temperament make it infeasible, and tagging a colony of this size is a welfare cost for no decision benefit).

## Decision

**One care subject type. `subjectKind` declares whether it is an individual, a group, or a colony, and cardinality carries its own certainty.**

| Field | Meaning |
|---|---|
| `subjectId` | The thing care is recorded against |
| `displayId` | Which of the 55 displays it lives in |
| `subjectKind` | `individual` \| `group` \| `colony` |
| `cardinality` | Member count |
| `cardinalityCertainty` | `exact` \| `estimated` |
| `speciesOrCollection` | What it is |

Three rules follow.

**`cardinalityCertainty` is structural, and consumers must handle both values.** A colony's count is `estimated` and arrives with an interval ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)). A named snake's count is `exact` and is 1. Every consumer - alert, report, Countess dashboard - has to deal with the estimated case rather than reading an integer and moving on. Making this a required field is what stops a wide-interval estimate being rendered as a confident number three screens later.

**Observations attach to the subject, at the granularity actually observed.** A keeper who sees one fish behaving oddly records an observation against the colony with a note, not against an invented individual. A keeper treating a specific snake records against that individual. The model accepts the granularity of the real observation instead of demanding one it cannot supply.

**A subject can be promoted, never silently.** If individual tagging is later funded for a species, those animals become `individual` subjects and the group subject is superseded explicitly, with history preserved. This is the migration path [Appendix C](../requirements/Appendix%20C_%20Future%20scope.md) leaves open for individual tagging.

Identifiability honesty and representable uncertainty decided it. Option A fails both and is the default most systems pick: it creates identities nobody can verify, then produces confident per-animal records that are quietly fictional. Option B is clean and loses the veterinary history of the animals that *do* have identities, which is a real welfare and legal record (NFR_9). Option D is correct and costs double implementation of every query, alert, and screen for a three-person team.

## Key differentiators

- **The model never asks a keeper for an identity they cannot establish,** so the data stays true to what was actually seen.
- **Uncertainty about how many animals there are is a field, not a footnote,** which is what lets the population capability publish an interval and refuse to publish a number.
- **One schema spans a named venomous snake and a piranha shoal,** so alerts, reports, and the keeper UI are written once.
- **Promotion to individual tracking is a defined migration,** so today's aggregation is not a dead end.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Data integrity | **Improved (driving)** | No fabricated identities; cardinality certainty is explicit and unavoidable. |
| Evolvability | **Improved (driving)** | Individual tagging later is a subject-kind promotion, not a schema rewrite. |
| Usability for keepers | **Improved** | Matches how rounds are actually done - by enclosure (NFR_17). |
| Simplicity | **Improved** | One model, one set of queries and alerts, versus two parallel subsystems. |
| Cost efficiency | **Improved** | No tagging programme, no per-animal identification hardware. |
| Query complexity | **Weakened (deliberate)** | Every consumer must handle `estimated` cardinality, including the reporting layer and the Countess's dashboard. |
| Resolution | **Weakened** | Within a colony, an individual animal's history is not reconstructable. A specific sick fish cannot be followed over time. |
| Welfare traceability | **Weakened** | For colony subjects there is no per-animal treatment record, which is a genuine limitation if a regulator or vet asks. |

**Deliberately downplayed: per-individual resolution inside colonies.** A tagged, individually tracked collection would support far better veterinary medicine. We refuse it for the colonies because the identification does not work - occlusion in a dense shoal is not an engineering problem to be solved with a better camera - and because tagging piranha imposes a welfare and keeper-safety cost to obtain a record nobody can act on individually. For the animals where identity *is* establishable, `subjectKind: individual` gives full resolution, so the loss is confined to exactly the cases where the alternative was fiction.

**Fit with the existing architecture.** Same rule as the rest of the estate: represent what is known, mark what is not, and never let an estimate masquerade as a fact. `cardinalityCertainty: estimated` is the animal-domain sibling of a gapped zone publishing unknown ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)) and a popularity figure carrying its coverage ([ADR-0013](ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)).

## Consequences

### Positive

- Observations record what was actually seen (driving criterion).
- Population uncertainty has somewhere to live, structurally (driving criterion).
- One schema, one alert path, one keeper UI across a very mixed collection.
- No tagging programme required to open the collection to the public.

### Negative

- **No per-animal history inside a colony.** If a vet asks what happened to a specific fish over three months, there is no answer, and for a hazardous collection newly open to the public that gap may eventually attract attention.
- Every consumer must handle estimated cardinality, and the reporting layer is where that will be forgotten - the Countess will want a number.
- "How many animals does the estate have?" is answerable only as a range, which is an awkward sentence for a marketing page and an annual report.
- Subject kind is a judgement call at setup, and a wrong call (group where individuals are identifiable) loses resolution that was available.
- Promotion to individual subjects is a real migration with history-linking work, even though the path exists.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Estimated read as exact | An interval flattened into a number downstream | `cardinalityCertainty` is required; CI rejects a consumer that ignores it; the population contract requires the interval ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)) |
| Wrong subject kind at setup | A group modelled where individuals were identifiable | Keeper and vet review at setup; promotion path exists and is documented |
| Lost individual welfare record | Colony subjects have no per-animal treatment history | Individuals modelled wherever identity is establishable; colony-level treatment events recorded against the subject with keeper notes |
| Cardinality drift | A group's count silently becomes wrong | Census resets the anchor; mortality decrements deterministically; certainty degrades with anchor age |
| Regulatory expectation | A welfare regime may assume per-animal records | Stated as an open question for the jurisdiction ADR; individuals modelled where feasible already |

## Verification

**Primary metrics**

- Share of the 55 displays with a defined care subject and subject kind - target 100% before opening.
- Share of subjects whose cardinality certainty is correctly asserted, reviewed with keepers.
- Consumers handling estimated cardinality - must be all of them.

**Tests (CI)**

- A subject with `subjectKind: colony` and `cardinalityCertainty: exact` is rejected unless a census is its anchor.
- A consumer rendering cardinality without checking certainty fails the check.
- An observation may attach to a colony subject without an individual identity.
- A mortality event against a colony decrements cardinality by exactly one.
- Promoting a group to individual subjects preserves prior observation history against the superseded subject.

**Ops check**

- Keepers walk the 55 displays and confirm the subject kind assigned to each matches what they can actually observe.

**Open questions**

- Which displays hold individually identifiable animals - with keepers, before the animal module is populated. This is a walk-round, not a design question.
- Whether species-level or collection-level grouping is used where a display holds several species - with keepers, before setup.
- Whether any welfare regulation in the eventual jurisdiction requires per-animal records - with the jurisdiction ADR.
- How treatment events are recorded against a colony - with the vet, before the collection opens.

**Revisit triggers**

- Individual tagging is funded for a species - promote those subjects using the defined path.
- A welfare regulator or insurer requires per-animal records for a colony - revisit, knowing identification is the blocker rather than the schema.
- Keepers routinely find themselves writing "the one with the scar" in free text - identity is establishable after all, and the subject kind is wrong.

## Conclusion

There is one care subject type whose kind declares whether it is an individual, a group, or a colony, and whose member count carries its own certainty. Chosen because a per-animal model would fabricate identities for shoals, and because population uncertainty needs a structural home rather than a caveat. The cost is no per-animal history inside colonies and a reporting layer that must always handle an estimate - both narrower than the alternative, which was confident fiction.

Related: [ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md) (how observations arrive), [ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) (what scores them), [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) (the estimated cardinality this record makes possible), [hld/scenarios/animal-care](../hld/scenarios/animal-care/README.md).
