# evals/piranha-population - golden cases for the colony population estimate

Capability: colony-level population estimate for the jumping piranha ([ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)).

Authority: **L1 Inform, capped.** Fallback: the last census anchor; past maximum anchor age the capability publishes as unusable.

These are the cases [ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) specifies, and they are the clearest example in the repo of a capability being verified by what it refuses to do.

## Primary metrics

| Metric | Target | Why |
|:--|:--|:--|
| **Interval coverage** | Close to nominal - a 90% interval should contain the census about 90% of the time | The metric that proves the uncertainty model is honest. Under-coverage means overconfident; over-coverage means useless |
| Absolute error against census at each anchor | Tracked per anchor | Accuracy, but secondary to coverage |
| Change-detection latency | Days, not weeks | Time from a known loss event to the flag - where the operational value is |
| Census-request precision | Tracked | How often a census the system asked for actually found a change |

**Coverage, not error, is the headline.** A bare "417 fish" cannot be wrong in any measurable way. An interval that should contain the census 90% of the time and contains it 60% of the time is demonstrably broken, and that is what makes this capability verifiable at all.

## Refusal and guard cases

| Case | Expected | Protects |
|:--|:--|:--|
| Estimate published without an interval | **Rejected** | [ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) - foreclosed: a bare figure is not a valid output |
| Time since census increases | Interval widens **monotonically** | Confidence must decay away from the anchor |
| Time since census exceeds the maximum | Published as **unusable**, not as a number with a wide band | Refusing beats a confident-looking guess |
| Census recorded | Anchor and interval both reset | Ground truth resets drift |
| Keeper-confirmed carcass recovered | Estimate decrements by **exactly one**, deterministically | [ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md) - confirmed facts are not model inputs to be softened |
| Model estimate would contradict a confirmed decrement | The confirmed fact wins | Same |
| Vision estimator running in shadow | **Never** alters the published estimate | Shadow means silent, per the [ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) pattern |
| Consumption drops with no other evidence | **No** population-change flag raised | Temperature, season, and breeding move intake without a population change |
| Water temperature outside the stable band | Interval widens | The covariate is acknowledged rather than ignored |
| Fry observed by a model or camera | Flagged for keeper review; breeding is **never declared** by the model | A false breeding flag triggers an unnecessary intervention |
| Feed events missing `leftover` | Consumption inference disabled for the period | Feed-to-appetite destroys the signal |
| Point estimate rendered in a UI without its interval | Contract violation; rejected | The interval gets dropped between service and screen if nothing enforces it |

## Accuracy cases

| Case | Expected |
|:--|:--|
| Census at day 0, census again at day 30, no change in reality | Interval contains the second census; point estimate close to it |
| Known loss of three individuals via carcasses | Estimate decrements by exactly three; no widening from the deterministic events |
| Known loss with no carcass (suspected cannibalism) | Consumption signal may or may not detect it; a miss is recorded honestly as a change-latency failure |
| Temperature drop reducing intake, population unchanged | No population-change flag; interval widens |
| Breeding event confirmed by keeper | Estimate increases; anchor confidence reduced pending census |

## The case that is expected to fail

**Cannibalism with no carcass is close to undetectable.** A piranha eaten by tankmates leaves no body, and under a fixed-offer protocol the surviving fish simply eat more each, so total consumption barely moves.

[ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) states this as a negative consequence rather than hiding it. The eval records it as a known blind spot, and the mitigations are a hard cap on time between censuses plus injury and aggression observations as leading indicators - not a claim that the model handles it.

Writing a case we expect to fail is deliberate. It is the difference between an eval suite that documents the capability and one that flatters it.

## Open thresholds

- Maximum time since census before the estimate becomes unusable (before first season).
- Nominal interval level - 90% or otherwise - which sets what coverage is measured against.
- Per-capita intake model and its temperature covariate (before beta).
- Whether vision is funded at all, and its cost ceiling.
- Census method, frequency cap, and its own welfare and safety cost (vet, before the first census).

Related: [hld/scenarios/animal-care](../../hld/scenarios/animal-care/README.md), [ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md), [ADR-0020](../../adrs/ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md).
