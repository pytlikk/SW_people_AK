# ADR-022 - Detect welfare anomalies against each subject's own baseline before any trained model

## Date

2026-09-14

## Status

Proposed

## Context

FR#2I requires detection of health and feeding anomalies - "how much / how well they are eating" - across 55 displays, with an alert carrying confidence, evidence, and a suggested action. [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) sets the rules: high recall first, keeper accept/reject is the training signal, never autonomous welfare decisions.

Four facts about this estate decide the approach, and none of them favour reaching for a model:

- **There are no labels.** The collection was private and the platform is greenfield ([4_Assumptions and constraints.md](../requirements/4_Assumptions%20and%20constraints.md)). There is no history of labelled illness to train on. On day one, supervised learning has nothing to learn from.
- **"Normal" is not shared.** A python fed once a fortnight and a piranha colony fed daily do not belong in one distribution. A refusal is routine for one subject and an emergency for another. Across 55 displays there is no population-level notion of eating well.
- **The data is thin.** A subject produces a few observations a day (ADR-021). After a full year a single subject has a few hundred feeding records. This is not a volume where a large model outperforms arithmetic.
- **Positives are rare and expensive.** Illness is infrequent, and [1_1_Business challenges.md](../requirements/1_1_Business%20challenges.md) is explicit that it is sharply more costly when missed. Rare-event detection with no labels is the hardest possible starting point for supervised methods.

Against that, [5_Risks and mitigation.md](../requirements/5_Risks%20and%20mitigation.md) warns in both directions: false negatives cause harm, and false positives train staff to ignore the system. Both are live.

Inputs available: keeper observations and feed records (ADR-021), environment telemetry joined by placement-at-time (ADR-020), and optional cameras. Alerts must reach the keeper UI within 60 seconds of the event reaching a gateway (NFR_3). Detection is not a hot path in the ADR-002 sense - nothing is gated on it.

This record does **not** decide: population estimation (ADR-023); which enclosures are instrumented; field-level schema, which follows in the data contract; or any screen, which belongs to the intranet workstream.

**Foreclosed here:** a third-party model sitting between an animal and a keeper. No vendor may be on the path that decides whether a welfare alert exists.

## Evaluation criteria

- **Cold start (driving)** - produces useful alerts on day one, with zero labelled illness events in existence.
- **Species heterogeneity (driving)** - handles 55 displays whose notions of normal differ, without a hand-maintained per-species threshold library.
- **Recall on rare events** - a missed sick animal is the expensive failure. Measured against keeper- and vet-confirmed events, including ones the system did not raise.
- **Keeper trust** - alert volume stays inside a keeper's real attention budget, or the system is ignored and the labels stop.
- **Explainability** - an accept/reject is only a valid training label if the keeper understood what was claimed.
- **Provider independence (NFR_14)** - a vendor changing price, behaviour, or existence must not stop welfare detection.
- **Cost (NFR_12)** - inference cost per check stays visible; vision is the known opex risk.

## Options

- **Option A - Fixed rules per species**: keepers and vets author thresholds; the system fires when one is crossed.
- **Option B - Supervised ML from day one**: train a classifier on labelled welfare events.
- **Option C - Tiered detector, baselines first (chosen)**: deterministic safety rules at the edge, per-subject statistical baselines as the day-one detector, supervised models earned later from keeper labels, GenAI confined to narrative.
- **Option D - GenAI as the detector**: pass keeper notes and readings to a language model and ask whether the animal is unwell.
- **Option E - Vision-first**: cameras on enclosures, behavioural analysis as the primary signal.

| | Cold start (driving) | Species heterogeneity (driving) | Recall on rare events | Explainability | Provider independence | Cost |
|---|---|---|---|---|---|---|
| A fixed rules | Pass | Fail - hundreds of hand-maintained thresholds, and no notion of "less than usual for this animal" | Weak - catches only what was anticipated | Best | Pass | Lowest |
| B supervised now | Fail - no labels exist | Fail - too few per-species positives to fit | Unmeasurable at launch | Poor | Pass | Medium |
| C tiered, baselines first | Pass - baselines need history, not labels | Pass - each subject is compared to itself | Pass - tunable, and improves as labels accrue | Pass at tiers 0-1 | Pass - vendor confined to prose | Low |
| D GenAI detector | Partial - plausible output immediately, unvalidated | Partial | Fail - no calibrated confidence on rare events; drifts on provider update | Narrative, not evidential | Fail - vendor on the welfare path | High - continuous checks across 55 displays |
| E vision-first | Fail - still needs labels | Partial | Unknown | Poor | Varies | Highest - the NFR_12 opex risk |

Not options: autonomous intervention of any kind, and A/B testing welfare thresholds - both are denied by [4_Assumptions and constraints.md](../requirements/4_Assumptions%20and%20constraints.md) and NFR_7.

## Decision

**Use a tiered detector, where the tier in use is determined by label availability rather than by what is fashionable.**

| Tier | What it is | Where it runs | Status at launch |
|---|---|---|---|
| 0 | Deterministic safety thresholds - water temperature, dissolved oxygen, filtration and containment failures | **Edge**, so it fires during islanding | Live, never model-gated |
| 1 | Per-subject baseline scoring - intake, refusal rate, feed interval, environment drift, each compared against that subject's own recent history | Cloud, after ingest | Live, the day-one detector |
| 2 | Supervised model trained on accumulated keeper accept/reject labels | Cloud | Not at launch. Shadow first, promoted per collection |
| 3 | GenAI narrative - turns tier 0-2 evidence into readable explanation, surfaces related keeper notes | Cloud | Presentation only |

Three rules bind the tiers together. **Tier 0 runs at the edge and is never gated by a model**, because an oxygen crash during a four-hour outage cannot wait for the cloud (NFR_7, NFR_4). **Suggested action is a lookup, not a generation** - keeper and vet authored playbooks keyed to the anomaly type, because Appendix B is explicit that invented veterinary doses are a failure. **Tier 3 never produces a score or a decision**, only prose over evidence that already exists.

Cold start and species heterogeneity decide it. Option B cannot start. Option A cannot express "less than usual for this animal", which is the literal wording of the business challenge. Option D puts a non-deterministic vendor between an animal and a keeper, with no calibrated confidence on exactly the rare events that matter, and bills continuously across 55 displays. Option E is the cost risk NFR_12 names, and still needs labels.

## Key differentiators

- **It works on day one with no labels.** A baseline needs history, not ground truth.
- **Each subject is compared to itself,** so 55 heterogeneous displays need no per-species threshold library and no pooled distribution that describes nothing.
- **The alert inbox is a labelling machine.** Tier 1's second job - arguably its more important one - is manufacturing the labelled dataset tier 2 needs. The ops tool and the training pipeline are the same artefact, so the system earns its way up the tiers instead of waiting for a data collection project that would never be funded.
- **Provider risk is confined to prose.** Only tier 3 touches a third party. If that vendor triples its price or shuts down tomorrow, welfare detection is unaffected and alerts simply read like a spreadsheet. That is the whole of our exposure to the kata's uncertainty question.
- **Recall is traded for rank, not for precision.** The detector stays sensitive; keeper attention is protected by a ranked, budgeted inbox rather than by raising the threshold until the system goes quiet.
- **Tiers 0 and 1 are arithmetic,** so they are exactly reproducible in CI - a golden case either passes or the code is wrong.

## Consequences

### Positive

- Detection at launch without a labelled dataset (cold start).
- No per-species threshold library to maintain across 55 displays (species heterogeneity).
- Deterministic tiers are testable to the digit, which is what NFR_13 asks for.
- Vendor change, price shock, or shutdown degrades text only (provider independence).
- Safety thresholds keep firing during a network partition (NFR_4, NFR_7).
- Cost is dominated by arithmetic over a few hundred records a day, not by continuous inference (NFR_12).

### Negative

- **Newly arrived subjects have no baseline,** and a collection that is only now opening to the public will have arrivals. Warm-up falls back to a collection prior and tier 0 rules, and new arrivals are the weakest case precisely when they are most stressed.
- **Seasonal and lifecycle states look exactly like illness.** Brumation, moulting, breeding, and gravidity all suppress feeding. Without keeper-declared expected state, the system generates a predictable false-alarm wave every autumn.
- **The alert budget is a practical recall ceiling.** A true positive ranked twelfth on a day with eleven slots is missed even though the detector found it. Reporting detector recall without reporting budget burn would be dishonest.
- **Baselines decay silently if entry discipline slips.** A subject nobody has recorded for three weeks has a confident-looking baseline built from stale data.
- **Tier 2 may never arrive** for small collections, because a year may still not yield enough confirmed positives. Tier 1 being the permanent answer for some collections is an accepted outcome, not a failure.
- Promoting tier 2 trades explainability for sensitivity, which weakens the quality of the labels it then generates.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Seasonal state read as illness | Brumation, moulting, gravidity, post-treatment suppress feeding | Keeper-declared expected state reweights or suppresses scoring; every suppression is logged and reviewable, never silent |
| Baseline poisoning | Entry rate drops, baseline is built from stale data but still looks confident | Track observation recency per subject; a stale baseline publishes as unknown, not as normal (Appendix B gap rule) |
| Label bias | Keepers reject alerts they were too busy to investigate, teaching tier 2 that busy days are healthy | Distinguish investigated rejects from dismissed ones; unexamined rejects are weak labels, weighted accordingly |
| Alert fatigue | Volume exceeds attention, trust collapses, labels stop | Ranked budget per keeper per shift; suppress repeats against an open alert; accept rate is an ops metric |
| Cold start on arrivals | A new subject has no history | Collection prior plus tier 0 until warm-up completes; alerts carry reduced confidence and say why |
| Vision cost creep | Cameras added capability by capability until inference dominates opex | Vision is excluded from v1 health detection; any addition needs its own ADR with a stated cost ceiling |
| Provider change | Tier 3 vendor changes price, behaviour, or disappears | Confined to narrative; documented fallback is template text assembled from the same evidence fields |
| Over-trust | Staff treat an alert as a diagnosis | Alerts state evidence and confidence, never a diagnosis; suggested action is a playbook lookup; keeper and vet decide (NFR_7, NFR_15) |

## Verification

**Primary metric: recall against keeper- and vet-confirmed welfare events.**

This has an architectural consequence worth stating plainly: recall cannot be measured unless the system records welfare events it **failed** to raise. A keeper must be able to open a welfare event independently of any alert, and those unprompted events form the denominator. Without that path, the system can only ever measure its own opinion of itself.

**Tracked metrics**

- Recall against confirmed events, by collection.
- Accept rate on raised alerts, as the precision proxy.
- Alert budget burn, and the share of shifts hitting the cap.
- Count of subjects whose baseline is stale, published as unknown.
- Suppressions by declared expected state, as an audit trail.

**Tests (CI)** - golden cases in [`evals/animal-health-anomaly/`](../evals/animal-health-anomaly/)

- Tiers 0 and 1 are deterministic: a golden case reproduces the identical score and decision every run.
- A stale baseline yields unknown, never normal.
- A declared expected state suppresses scoring and writes an audit record.
- Tier 3 narrative cites only evidence fields present on the alert, and states no diagnosis, dose, or treatment.
- Disabling tiers 2 and 3 leaves tiers 0 and 1 fully functional - the kill switch is real.
- Tier 0 evaluates with no cloud connectivity.

**Promotion gate**

Tier 2 runs in shadow and may only be promoted for a collection once it beats tier 1 recall at equal alert budget on held-out confirmed events (NFR_13). Promotion is per collection, never global.

**Open questions**

- Warm-up window length before a baseline is trusted, per collection - before beta.
- Alert budget per keeper per shift, agreed with ops - before first season.
- Which environment thresholds are tier 0 safety rules, and who authors and signs them - before installation.
- Expected-state vocabulary: brumation, moulting, gravid, quarantine, under treatment - before the data contract.
- Minimum confirmed positives per collection before tier 2 may be trained - before any training run.

**Revisit triggers**

- Accept rate stays below the ops-agreed floor after budget tuning -> revisit tier 1 scoring before adding any model.
- A confirmed welfare event category is missed repeatedly -> author a tier 0 rule for that category rather than waiting for tier 2.
- Tier 2 fails to beat tier 1 after a full season -> stop. Tier 1 is the answer for that collection.
- A collection develops a genuine need for vision -> separate ADR with a cost ceiling, not an extension of this one.

## Conclusion

Each subject is scored against its own history, because with no labels, 55 incompatible notions of normal, and a few hundred records per animal per year, arithmetic beats a model - and the keeper decisions that arithmetic provokes are what eventually earn the right to train one. Deterministic safety thresholds run at the edge and answer to nobody. A language model writes the explanation and nothing else, which is the entire extent of this architecture's exposure to a vendor.

Related: ADR-020 (what is scored), ADR-021 (where the labels come from and why they must not be lost), ADR-023 (population, which is a different question about the same colony). Golden cases in [`evals/animal-health-anomaly/`](../evals/animal-health-anomaly/).
