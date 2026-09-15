# evals/animal-health - golden cases for health and feeding anomaly detection

Capability: health and feeding anomaly detection across 55 displays ([ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)).

Authority: **L2 Advise, capped permanently.** Fallback: keeper-authored deterministic rules, live from day one.

## Primary metrics

| Metric | Target | Why |
|:--|:--|:--|
| Recall against keeper-labelled health events | Tracked per subject type; a drop is a failure | FR#2I wants recall first - a missed sick animal is the expensive outcome |
| Alert volume per keeper per shift | Within the attention budget | A flooded inbox is ignored, and then the true positive is missed too |
| Accept rate and reject reason distribution | Tracked; both extremes investigated | The drift indicator that moves before offline metrics |
| Time-to-detect a confirmed anomaly | Materially faster than paper rounds | OKR 5.1 |
| Shadow model recall and volume vs rules, per subject type | The promotion evidence | Promotion is earned, per subject type |

## Refusal and guard cases

| Case | Expected | Protects |
|:--|:--|:--|
| Model is in shadow for a subject type | Produces **no** keeper-visible alert | [ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) - shadow means silent |
| Promotion attempted without recorded shadow duration, label count, recall vs rules, and volume evidence | Blocked | Promotion is evidence-gated |
| Promotion attempted past L2 for any welfare capability | Blocked by policy | [ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) ceiling - no confidence buys this |
| Alert volume exceeds the shift budget | Eval fails rather than alerts being delivered | Keeper attention is the scarce resource |
| Three consecutive days of refusal on one subject | **One** open case with a worsening trend, not three alerts | Where the attention budget is won |
| Alert published without confidence, evidence, contributing signals, and input freshness | Rejected | [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) contract; keepers triage on evidence |
| Any attempt to resolve an alert without a keeper decision | Rejected | No autonomous welfare action, ever |
| Any attempt to auto-medicate, auto-cull, or auto-escalate to treatment | No such code path exists | Foreclosed in [ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) |
| Drift alarm fires on a promoted subject type | Automatic demotion to rules | Degradation is a mode, not an outage |
| Feed event missing `leftover` | Subject excluded from consumption-based scoring | [ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md) - incomplete protocol data must not be inferred over |
| Environment sensor gapped | Scored as unknown covariate, not as a normal reading | Gap is never zero |

## Accuracy cases

Species heterogeneity is the core difficulty: a fortnightly-feeding python and a daily-feeding colony are different distributions, so cases are written per subject type rather than in aggregate.

| Case | Expected |
|:--|:--|
| Daily feeder, leftover rising over three feeds, temperature in band | Alert raised with the consumption trend as evidence |
| Fortnightly feeder, one missed feed | No alert on a single observation; the trailing window is too sparse |
| Water temperature drifts outside band, feeding normal | Alert raised on the environment signal alone |
| Post-water-change stress, feeding drops for one cycle | Ideally no alert; if raised, the keeper reject reason becomes a label |
| Seasonal appetite reduction for a species with a known cycle | No alert - seasonal baseline per subject |
| Gravid or breeding-condition animal, intake changes | No health alert; not confused with illness |
| Aggression observed plus leftover rising | Alert raised at higher confidence than either signal alone |
| Keeper-labelled illness event in history | Detected by rules and/or the shadow model; a miss is recorded as a recall failure |

## Reviewer discipline

Injected golden cases are placed into the live inbox periodically to confirm keepers are still discriminating rather than accepting by reflex ([ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)). An accept rate near 100% with no rejections is investigated, not trusted.

## The honest gap

When a capability hits its attention budget it raises its threshold, which **suppresses some true positives**. That trade is deliberate - a queue nobody reads has negative value - but it is measured rather than hidden: recall is reported per period so a budget-driven threshold rise shows up as a recall cost.

Keeper rounds remain the primary safeguard. This capability shortens time-to-detect; it does not replace someone looking at the animals.

## Open thresholds

Four of these are now derived in [ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md#the-cost-asymmetry-and-the-attention-budget-it-sets) from a £900 miss against a £4.94 false alarm, and remain open only for confirmation with keepers rather than for invention:

- **Attention budget: 4 alerts per shift estate-wide**, hard cap 6. Not the 18 that available keeper time would allow - the binding constraint is that illness is rare, not that keepers are busy.
- **Minimum shadow duration: 6 months and 30 labelled events** per subject type, from a base rate of 6 genuine events per subject per year.
- **Overnight escalation above 67% confidence**, because an out-of-hours call-out costs £600 against a £900 miss.
- **Starter thresholds** are constrained to an aggregate under 4 a shift, allocated by risk rather than evenly. The per-subject values are still a keeper-and-vet exercise.

Genuinely still open:

- Reject reason taxonomy (keepers) - "rejected: other" teaches nothing. The minimum set is whatever lets precision be computed per subject type.
- **The attention curve** relating alert volume to the probability a keeper acts. It is the softest input in the budget above and it is calibrated during shadow, by measuring accept rate against delivered volume.

Related: [hld/scenarios/animal-care](../../hld/scenarios/animal-care/README.md), [ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md), [ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md).
