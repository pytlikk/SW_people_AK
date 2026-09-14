# Eval - jumping-piranha population estimate (FR#2J)

Golden cases for the estimator decided in [ADR-023](../../adrs/ADR-023-anchor-piranha-population-on-human-census.md).

> **Status: specification, not a running harness.** These files state what correct behaviour is. There is no runner in this repository yet ([evals/TEMPLATE.md](../TEMPLATE.md): store evaluation placeholders only).

## What is being verified

| | |
|---|---|
| Ground truth | Periodic human census |
| Primary metrics | Absolute error against census; **interval coverage** |
| Operational metrics | Change-detection latency; census-request precision |
| Fallback | The last census result, with the interval marking how old it is |

**Interval coverage is the metric that matters most, and it cannot be asserted by a single golden case.** If the published interval is nominally 90 percent, the census should land inside it close to 90 percent of the time across many anchors. Under-coverage means the system is overconfident, which is the failure Appendix B warns about when it rejects "a single magic number". Over-coverage means the interval is so wide it carries no information. Coverage is measured across anchors over a season; the cases here defend the behaviours that make coverage possible.

## The shape of the answer

ADR-023 forecloses publishing a bare number. Every case below is, in one way or another, defending that: the estimate is an interval anchored on census, it decays away from that anchor, confirmed facts move it deterministically, and nothing that merely correlates with population is allowed to move it at all.

Fixture values are illustrative. The colony is `SUBJ-0500` in tank `ENC-007`, the same subject and enclosure used in [`evals/animal-health-anomaly/golden-004`](../animal-health-anomaly/golden-004-tier0-edge-offline.json), so the two capabilities can be read against one another.

## Case index

| Case | Asserts |
|---|---|
| [golden-001](golden-001-interval-widens-from-anchor.json) | The interval widens monotonically with time since census, and a census resets it |
| [golden-002](golden-002-carcass-deterministic-decrement.json) | A recovered carcass decrements by exactly one, and does not change the interval width |
| [golden-003](golden-003-stale-anchor-unusable.json) | Past the maximum anchor age the estimate publishes as unusable, not as a wide number |
| [golden-004](golden-004-consumption-confound.json) | A consumption change alone never raises a population-change flag |
| [golden-005](golden-005-vision-shadow-no-effect.json) | Vision running in shadow records a comparison and alters nothing |

## Not yet written

- **Breeding flag** - a fry sighting flags for keeper confirmation and never declares breeding by itself.
- **Census-request precision** - the system asks for a census, and the census finds a real change often enough to justify the netting.
- **Cannibalism blind spot** - the case that would demonstrate the loss mode ADR-023 admits it cannot see. Writing it requires the fixed-offer feeding protocol to be agreed first, which is an open question.
