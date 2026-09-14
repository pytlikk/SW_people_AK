# ADR-023 - Publish a piranha population interval anchored on human census, never a count

## Date

2026-09-14

## Status

Proposed

## Context

FR#2J requires a colony-level estimate of the jumping-piranha population, with a trend and breeding or loss flags, verified against periodic human census. [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) is blunt about the shape of the answer: an error band, "not a single magic number".

Under ADR-020 the colony is already a first-class care subject whose cardinality is an estimate. This record decides how that estimate is produced and published.

Three things make counting the wrong goal:

- **They cannot be counted from outside.** Fish occlude fish. In a dense school the undercount is itself a function of density, which is the quantity being measured. Turbidity makes it worse.
- **Nobody acts on the level.** The Countess does not deploy staff because there are 417 piranha. She acts when the colony is *losing* individuals - disease, cannibalism, predation, escape - or *gaining* them, which means overcrowding and intervention. The brief says "check population levels"; the business need in [1_1_Business challenges.md](../requirements/1_1_Business%20challenges.md) is population *control*.
- **Measuring harms the thing measured.** Censusing piranha means netting or draining, which is stressful for the colony and hazardous for keepers. Any design that assumes frequent census has quietly imposed a welfare and safety cost to buy a number nobody needed that precisely.

There are also no labels and no history, exactly as in ADR-022. Nobody has ever counted this colony in a way a model could learn from.

This record does **not** decide: individual animal health, which is ADR-022; whether vision is funded, which needs a cost ceiling of its own; the census method, which is a veterinary and safety question; or field-level schema, which follows in the data contract.

**Foreclosed here:** publishing a bare population figure. An estimate without an interval is not a valid output of this capability.

## Evaluation criteria

- **Honest uncertainty (driving)** - every published estimate carries an interval whose width reflects time since the last anchor and the quality of the signals feeding it.
- **Change-detection latency (driving)** - a loss or breeding event surfaces in days, not at the next census. This is where the operational value is.
- **Welfare cost of measurement** - the design minimises census frequency rather than assuming it (NFR_8 purpose limitation, and plain animal welfare).
- **Cold start** - must work with no labelled counts in existence.
- **Cost (NFR_12)** - vision is the known opex risk and must be scoped or excluded, not assumed.
- **Verifiability (NFR_13)** - census is ground truth; error and interval calibration against it are the metrics.

## Options

- **Option A - Human census only**: keepers count on a schedule; nothing in between.
- **Option B - Vision counting as primary**: camera over the tank, a model counts fish per frame.
- **Option C - Sonar or hydroacoustic counting**: works in turbid water, as used in fisheries.
- **Option D - Feed-derived estimation only**: infer population from total consumption and assumed per-capita intake.
- **Option E - Census-anchored hybrid (chosen)**: human census anchors the absolute value; feed-derived consumption tracks change between anchors; keeper observations contribute deterministic events; optional vision is a third estimator, in shadow first.

| | Honest uncertainty (driving) | Change latency (driving) | Welfare cost | Cold start | Cost |
|---|---|---|---|---|---|
| A census only | Pass - it is ground truth | Fail - change is found weeks late | Fail - accuracy is bought with repeated netting | Pass | Low |
| B vision primary | Fail - occlusion bias scales with the thing being measured | Pass | Pass - non-invasive | Fail - needs labelled counts | High |
| C sonar primary | Partial - calibration is specialist work | Pass | Pass | Partial | High |
| D feed-derived only | Fail - drifts with no anchor, and stays confident while drifting | Pass | Pass | Pass | Lowest |
| E census-anchored hybrid | Pass - interval widens away from the anchor | Pass | Pass - census frequency is minimised by design | Pass | Low, vision optional |

Not options: continuous individual identification of fish, and tagging a colony of this size and temperament.

## Decision

**Publish an interval anchored on human census, with each signal doing one job.**

| Signal | Job | Not its job |
|---|---|---|
| Human census | Anchor the absolute value and reset accumulated drift | Frequent monitoring |
| Feed consumption | Track direction and rough magnitude of change between anchors | Producing a count |
| Keeper observations | Deterministic events - a recovered carcass, an observed fry | Inference |
| Vision (optional, one tank) | Third estimator, shadow first | Sole basis for any published figure |

Three rules follow:

**The interval widens with time since the last census.** Confidence decays away from the anchor, which means the system can say when it needs ground truth. Census is then scheduled by uncertainty rather than by calendar habit, and the welfare cost of netting is spent only when it buys something.

**A recovered carcass is a deterministic decrement of exactly one.** Confirmed facts are not model inputs to be softened.

**If vision is funded, it is scoped to surface-feeding frames.** Jumping piranha break the surface at feed time, and that is the one moment when individuals are separable and occlusion is at its lowest. Counting a settled school is the hard problem; counting a feeding frenzy at the surface is a different and more tractable one.

Honest uncertainty and change latency decide it. Option A buys truth at the price of weeks of blindness and repeated netting. Option D is the dangerous one - it produces a plausible number that drifts with nothing to correct it and no reason to lose confidence. Options B and C put the expensive, cold-start-blocked estimator at the centre when it should be, at most, a third opinion.

## Key differentiators

- **The output is a decision-support signal, not a number.** Trend and change flags answer the question that is actually asked; the interval keeps the estimate honest about what it does not know.
- **The system requests its own ground truth.** When the interval exceeds a threshold, it asks for a census, which closes the loop between uncertainty and measurement.
- **Welfare cost is treated as a design input.** Minimising how often the colony is netted is an architectural goal here, not an afterthought.
- **Confirmed facts outrank estimates.** A carcass decrements deterministically; no model gets to smooth it away.
- **Vision is scoped to the moment the domain makes it easy,** if it is funded at all, and it starts in shadow under the same promotion pattern as ADR-022 tier 2.

## Consequences

### Positive

- Every published figure carries a calibrated interval (honest uncertainty).
- Loss and breeding surface between censuses rather than at them (change latency).
- Census happens when uncertainty demands it, not on a fixed schedule (welfare cost).
- Works on day one with no labelled counts (cold start).
- Vision is optional and bounded, so the capability does not carry the NFR_12 opex risk by default.

### Negative

- **Cannibalism is close to invisible, and it is the most likely loss mode in this colony.** A piranha eaten by its tankmates leaves no carcass, and under a feed-to-appetite regime the tank still consumes the same total - fewer fish simply eat more each. The signal is destroyed by the feeding practice.
- **That forces an operational change: a fixed offered amount with measured leftover.** The feeding protocol becomes part of the instrument. If keepers feed to appetite, per-capita intake is unobservable and the whole between-census signal collapses. This is a real constraint imposed on ops by an architectural choice, and it needs keeper agreement before instrumentation.
- Consumption confounds with water temperature, season, and breeding condition, so a consumption change is not a population change and must never be flagged as one on its own.
- Between anchors the point estimate can be systematically wrong while the interval only tells you it is uncertain, not in which direction.
- If ops skips censuses, the estimate degrades - and would look merely stale rather than unusable unless the interval is enforced hard.
- Vision, if funded, needs labels too, so its shadow period produces nothing decision-useful for months.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Cannibalism invisible | No carcass, no change in total consumption | Fixed-offer and measured-leftover protocol; injury and aggression observations as leading indicators; hard cap on time between censuses |
| Consumption confounding | Temperature, season, or gravid females move intake without a population change | Water temperature as a covariate; widen the interval when temperature is outside a stable band; never flag on consumption alone |
| Census skipped | Anchor ages, estimate drifts, output still looks like a number | Interval widens monotonically; beyond the maximum, publish as unusable rather than as a figure with a wide band |
| Presented as exact | An interval is dropped somewhere between service and screen | Contract requires the interval; CI rejects a point estimate without one; the intranet workstream is told explicitly |
| False breeding flag | A model declares breeding from ambiguous evidence | Fry sightings require keeper confirmation; the model flags for review and never declares |
| Vision cost creep | One camera becomes a programme | One tank, shadow first, stated cost ceiling, its own revisit trigger |
| Measurement harm | Censusing to improve accuracy stresses the colony and endangers keepers | Census is requested by uncertainty, not scheduled by habit; method and frequency need vet sign-off |

## Verification

**Primary metrics**

- **Absolute error against census** at each anchor.
- **Interval coverage** - if the published interval is nominally 90 percent, the census result should fall inside it close to 90 percent of the time. Under-coverage means the system is overconfident; over-coverage means the interval is too wide to be useful. Coverage, not error alone, is what proves the uncertainty model is honest.
- **Change-detection latency** - time from a known loss event to the flag.
- **Census-request precision** - how often a census the system asked for actually found a change.

**Tests (CI)** - golden cases in [`evals/piranha-population/`](../evals/piranha-population/)

- A recovered carcass decrements the estimate by exactly one, deterministically.
- An estimate published without an interval is rejected.
- The interval widens monotonically with time since the last census.
- Past the maximum age, the estimate publishes as unusable rather than as a number.
- A census resets both the anchor and the interval.
- Vision running in shadow never alters the published estimate.
- A consumption change alone does not raise a population-change flag.

**Open questions**

- Maximum time since census before the estimate becomes unusable - before first season.
- Feeding protocol change to fixed offer with measured leftover - needs keeper agreement, before instrumentation.
- Whether vision is funded at all, and its cost ceiling - before any camera is specified.
- Per-capita intake model and its temperature covariate - before beta.
- Census method and its own welfare and safety cost - with the vet, before the first census.

**Revisit triggers**

- Interval coverage sits below nominal across several anchors -> the uncertainty model is wrong; fix it before adding any estimator.
- Census repeatedly finds changes the system missed -> the feed signal is insufficient; escalate to more frequent census or fund vision.
- Vision in shadow beats feed-derived accuracy at acceptable cost -> promote it under the ADR-022 promotion pattern.

## Conclusion

The colony's population is published as an interval anchored on human census, with feed consumption tracking change between anchors and keeper-confirmed events decrementing it deterministically. The interval widens away from its anchor so the system asks for ground truth when it needs it, which is also the only way to keep netting a tank of piranha down to the times it is worth doing.

Related: ADR-020 (the colony as a subject whose cardinality is an estimate), ADR-021 (census and carcass entries travel the keeper path), ADR-022 (shadow-before-promote, reused here). Golden cases in [`evals/piranha-population/`](../evals/piranha-population/).
