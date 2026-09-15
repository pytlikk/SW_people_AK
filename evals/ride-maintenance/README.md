# evals/ride-maintenance - golden cases for predictive ride maintenance

Capability: predictive maintenance and inspect-before-peak alerts across 40 rides ([ADR-0014](../../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md)).

Authority: **L2 Advise, capped permanently.** Fallback: engineer-authored vibration and cycle bands, live from day one.

**Floor, not fallback:** the statutory scheme of examination and the daily pre-opening checks run whether or not this capability exists. Every case below is written against that floor.

## The test that matters more than the rest

> **No code path produces an inspect-by date later than the statutory date.**

This is asserted as the absence of a path, not the correctness of a date. A test that checks the output is later-than-never can pass while the capability is one refactor away from being able to extend an interval; a test that checks no such path exists cannot. It is the single most important line in this folder and the reason [ADR-0014](../../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md) exists.

## Primary metrics

| Metric | Target | Why |
|:--|:--|:--|
| **Lead time gained** | Days between a recommendation and when the statutory inspection would have found the same fault | This is the capability's entire value. If it is not buying earliness it is buying nothing |
| Recommendations per day | ≤2, hard cap 4 | Two ride engineers; the same attention argument as animal health |
| False-positive rate against engineer-confirmed findings | Tracked against the 22.7% implied threshold | The eval pair `FR#2K` names |
| Faults missed by the capability and found at statutory inspection | Tracked, **not treated as failures** | The floor catching something is the system working, not the model failing |
| Share of accepted inspections placed in a below-median-demand window | Tracked | The inspect-before-peak value, and the only estate-specific part of this capability |
| Accept rate and reject reason distribution | 20-95% | Below the floor nothing is being found; above the ceiling nobody is judging |

## Refusal and guard cases

These are the cases where the correct behaviour is to refuse, and they are the reason this folder exists.

| Case | Expected | Protects |
|:--|:--|:--|
| Model output would imply an inspect-by date later than the statutory date | **No such code path exists** | The one failure mode not available to us |
| Any attempt to vary the scheme of examination on model evidence | Rejected by policy and by code | [ADR-0014](../../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md) forecloses interval extension |
| Any attempt to set ride open/closed state | No such code path exists | Ride status is deterministic and human-authored (`NFR_7`) |
| Any attempt to issue or clear an e-stop | No such code path exists | Safety paths are not outputs of an AI capability |
| Work order created without an engineer decision | Rejected | Work orders are drafted by the capability and authored by a human |
| Model in shadow for a ride class | Produces **no** engineer-visible recommendation | Shadow means silent |
| Promotion attempted past L2 | Blocked by policy | [ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) ceiling - no accuracy figure buys this |
| Promotion attempted without shadow duration, confirmed-finding count, and volume evidence | Blocked | Promotion is evidence-gated, per ride class |
| Recommendations exceed the daily cap | Eval fails rather than recommendations being delivered | Engineer attention is the scarce resource |
| Recommendation published without confidence, contributing signals, and input freshness | Rejected | [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) contract |
| Vibration feed gapped for a ride | Scored as **unknown**, never as a healthy reading | A gap is unknown, never zero ([ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)) |
| Cycle count elevated, no other signal | **No recommendation** | Cycle count is a covariate and a poor wear proxy for a ride that sat idle all winter |
| Drift alarm on a promoted ride class | Automatic demotion to engineer-authored bands | Degradation is a mode, not an outage |

## Accuracy cases

Written per ride class rather than per ride, because a class shares a failure vocabulary and a single historic ride does not generate enough observations to be learnable.

| Case | Expected |
|:--|:--|
| Rotary class: vibration amplitude rising over 200 consecutive cycles, motor current flat | Recommendation raised, with the trend as evidence |
| Tracked class: motor current spike on one cycle, vibration normal | No recommendation - a single cycle is within a loading distribution, not a trend |
| Any class: vibration rising **and** motor current rising together | Higher confidence than either alone, and an earlier inspect-by date |
| Water class: seasonal vibration change on reopening after winter shutdown | No recommendation - seasonal baseline per class |
| Carousel class: vibration signature changes after a documented bearing replacement | No recommendation - a work order in history rebaselines the class |
| Any class: gate sensor bouncing, drive signals normal | Recommendation raised, scoped to the gate rather than the drive |
| Pendulum class: gradual degradation over a full season, statutory inspection due in 3 weeks | Recommendation raised with a date inside those 3 weeks - **lead time gained is the metric, and it is recorded** |
| Pendulum class: same degradation, statutory inspection due tomorrow | **No recommendation.** The floor is about to catch it; adding a recommendation spends engineer attention for zero lead time |
| A fault engineers found at statutory inspection that telemetry did not flag | Recorded as a miss with its signature, and added as a case. **Not a test failure** |
| Ride idle for six weeks, then instrumented signals drift | No recommendation from drift alone; idle periods are a known confounder |

That second-to-last row is the one to read twice. **A capability whose misses are caught by a legally mandated backstop should record its misses without failing on them**, because the alternative is an eval suite that pressures the threshold downward until the capability becomes the safeguard - which is precisely what [ADR-0014](../../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md) forbids.

## Inspect-before-peak cases

| Case | Expected |
|:--|:--|
| Recommendation with a 10-day deadline, forecast shows Tuesday at low demand | Inspection proposed for Tuesday |
| Recommendation with a 10-day deadline, every day forecast busy | Earliest date proposed, and **yield-at-risk surfaced** for the duty manager |
| Forecast unavailable or coverage below threshold | Earliest date proposed with no scheduling claim. The recommendation does **not** wait for the forecast |
| Recommendation on a low-popularity ride | No scheduling optimisation; not worth the complexity |

The third row is a contract with [ADR-0013](../../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md): a safety-adjacent recommendation never blocks on an advisory forecast. Degraded scheduling, never degraded safety.

## Reviewer discipline

Injected golden cases are placed into the live recommendation queue periodically, as in [evals/animal-health](../animal-health/README.md), to confirm engineers are still discriminating. An accept rate near 100% is investigated rather than celebrated.

The intranet carries a standing line on the maintenance panel: **absence of a recommendation is not evidence of health.** It is there because the predictable human failure here is not distrusting the model, it is trusting it as a clean bill of health.

## The honest gap

This capability buys **earliness, not detection.** A 22.7% alert threshold - forty-one times more conservative than animal health's - means real developing faults pass without a recommendation. That is accepted, and it is only acceptable because the statutory inspection catches them. Every metric above is written to keep that distinction visible, because the day it blurs is the day someone argues for extending an interval.

It is also, by volume and by cost, the estate's most expensive AI capability: 74% of all telemetry and the largest line in [cost-analysis](../../cost-analysis/README.md), for the benefit with the tightest ceiling. That is why it is phased last.

## Open thresholds

- The six ride classes and their membership (ride engineers, before instrumentation).
- M4 and M5 from [ADR-0014](../../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md) - in-service failure cost and progression probability - **with the insurer**, who has better numbers than we do. The 22.7% threshold moves with them.
- Minimum shadow duration and confirmed-finding count per class for promotion.
- Whether 60-second vibration sampling carries the same signal as 10-second. Worth 83% of estate ingest volume, and untested.
- Whether historic-fabric conservation permits sensor mounting on every ride. **This can veto the capability entirely.**

Related: [ADR-0014](../../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md), [evals/animal-health](../animal-health/README.md) (the pattern this reuses and the cost asymmetry it inverts), [hld/core-func/4_Maintenance_and_Intranet](../../hld/core-func/4_Maintenance_and_Intranet.md), [cost-analysis](../../cost-analysis/README.md).
