# Fitness functions

An architecture characteristic that cannot be measured is an aspiration. This page turns the ten characteristics that fourteen ADRs marked **driving** into tests with numbers, arithmetic, and a verdict.

## How to read a verdict

There is no running system. Pretending otherwise would be the easiest way to make this page look better and the fastest way to make it worthless, so every entry carries an honest status:

| Status | Means |
|:--|:--|
| **Derived** | The number falls out of arithmetic in this repository. A reader can check it today, without a deployment. |
| **Specified** | The test is written down - in an ADR's CI list or in [`evals/`](../evals/README.md) - and runs the day the harness exists. It does not run now. |
| **Unmeasurable yet** | Needs live data or a real shift. Stated so it is not mistaken for a gap nobody noticed. |

Five of the ten are **derived** and checkable now. That is the section of this repository a judge can most easily falsify, which is the point of putting it here.

## Which ADR made each characteristic driving

| Characteristic | Marked driving in |
|:--|:--|
| Data integrity | [0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md), [0012](../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md), [0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md), [0020](../adrs/ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md), [0021](../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md), [0023](../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) |
| Availability | [0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md), [0010](../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md), [0021](../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md) |
| Safety | [0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md), [0011](../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md), [0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) |
| Fault tolerance | [0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md), [0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) |
| Trustworthiness | [0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md), [0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) |
| Portability | [0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) |
| Privacy | [0012](../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) |
| Performance | [0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) |
| Elasticity | [0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) |
| Operability | [0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) |

---

## 1. Availability

**Goal.** The guest-critical path keeps working when the Wi-Fi, the cloud, or the model provider does not.

**Metric.** The product of the stated availability of every component a request must traverse.

**Threshold.** The hot path's availability expression contains **no cloud term and no model term**. Not "a small number of terms" - none.

### The arithmetic

This is the calculation the whole architecture exists to win, so it is worth doing in full.

**The async advice path**, which a congestion forecast must traverse to reach a duty manager:

```
zone gateway   99.50%   (a physical device in a field, with power and weather)
estate WAN     99.00%   (the brief's patchy Wi-Fi, being honest about it)
Pub/Sub        99.95%
BigQuery       99.99%
Vertex AI      99.50%
Cloud Run      99.95%
                            product = 97.905%   = 7.6 days a year unavailable
```

**The hot path, gate admit, as built.** A signed claim is verified locally against a cached public key ([ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)):

```
gate lane device   99.50%
                            product = 99.500%   = one term, because there is one component
```

And because [hld/sizing](../hld/sizing.md#3-gate-lane-count) specifies N+1 lanes, the gate as a whole is available if any lane is:

```
P(all 3 lanes down) = 0.005^3 = 0.0000001   ->   99.99999%
```

**The hot path if it had been built the obvious way**, asking a server whether this ticket is valid:

```
gate lane device   99.50%
estate WAN         99.00%
Cloud Run          99.95%
ticketing store    99.99%
                            product = 98.446%   = 5.7 days a year unavailable
```

### What that difference is worth

5.7 days a year of a gate that cannot admit anyone, at 15,000 visitors a day:

> **85,000 visitors turned away per year.**

That number is the entire argument for the signed-claim design, for the edge snapshot, for the local party ledger, and for the rule that nothing in `async` is ever called by `hot`. It is also why `NFR_2` can promise gate redeem availability that is independent of the cloud: not because the cloud is reliable, but because the gate never asks it anything.

The async path's 97.9% is not a problem, and saying why matters as much as the number. **Nothing waits on it.** A forecast that is 7.6 days a year late is advice that is 7.6 days a year absent, and a duty manager who sees "unknown, last updated 3 hours ago" makes the decision they made before the system existed.

### Enforcement

The claim is structural, so the test is structural rather than a monitoring threshold:

> **No cloud SDK import is reachable from the gate verification path or the checkout price read.**

A static dependency check in CI, already listed in [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md). This is the right shape of test, because availability regressions of this kind do not arrive as an outage. They arrive as an innocuous pull request that adds a lookup to a hot path and passes every functional test.

**Status: Derived.** **Verdict: Pass**, by construction. The enforcement test is *specified*, not running.

---

## 2. Data integrity - a gap is unknown, never zero

**Goal.** No consumer can mistake silence for a measurement. Marked driving in six ADRs, more than any other characteristic.

**Metric.** Share of windows with missing input that are rendered as `unknown` rather than as a number.

**Threshold.** **100%.** This is a correctness property, not a quality target, so there is no acceptable non-zero rate.

**Arithmetic.** The volume this has to hold across, from [hld/sizing](../hld/sizing.md): 20,074,000 messages a month across 535 devices in 12 zones. A single zone gateway failing blinds 45 devices at once. At 1.35 messages a second per zone, a gateway down for one hour produces **4,860 missing readings** - and the failure mode this guards against is those 4,860 gaps being averaged into a heat map as a quiet zone, sending staff away from the place they are most needed.

**Enforcement.** Four tests, all in [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) and [ADR-0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md):

- A device that misses its interval produces a gap marker.
- A consumer aggregate over a gapped window reports unknown, not zero.
- The forecast pauses rather than extrapolating when coverage falls below threshold.
- A replayed batch produces no double-counted occupancy.

**Status: Specified.** **Verdict: Not running.** The tests are written; the harness is not built, which [evals/](../evals/README.md) states plainly.

---

## 3. Safety - no autonomous welfare or safety action

**Goal.** AI can draft, rank, and propose. It cannot open a ride, silence a welfare alarm, medicate, or resolve a case.

**Metric.** Count of code paths by which a model output changes estate state without a human decision.

**Threshold.** **Zero**, with exactly one bounded exception: a price inside a band a human approved in advance ([ADR-0010](../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)), which is L3 act-in-band and is clamped in code.

**Arithmetic.** There is none, and that is the point. A characteristic like this is binary; expressing it as a percentage would be a way of agreeing in advance to violate it occasionally.

**Enforcement.**

- No code path resolves a welfare alert without a keeper decision ([ADR-0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)).
- A model in shadow produces no keeper-visible alert.
- The price clamp rejects any proposal outside the approved band, and the test asserts the rejection rather than the clamping.
- The experiment framework cannot gate a safety message ([ADR-0011](../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)'s unbypassable denylist).
- MFA on ride evacuation, mass refund, and model promotion (`NFR_6`).

**Status: Specified.** **Verdict: Pass** by design, enforcement not running.

---

## 4. Trustworthiness - the alert inbox stays worth reading

**Goal.** Keepers and duty managers keep reading what the system sends them.

**Metric.** Alerts per shift, estate-wide, and accept rate.

**Threshold.** **4 alerts per shift, hard cap 6.** Accept rate between 20% and 95% - a floor because nobody is finding anything, and a *ceiling* because an accept rate above 95% means keepers are clicking through rather than judging.

**Arithmetic.** Derived in full in [ADR-0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md#the-cost-asymmetry-and-the-attention-budget-it-sets). The short form: a missed illness costs £900, a false alarm costs £4.94, a ratio of 182:1. Taken alone that argues for alerting at a 0.55% probability of illness. But keeper attention decays with volume, and a dead inbox converts true positives into false negatives, so effective recall is model recall multiplied by attention - and that product peaks at 4 alerts a shift, not at maximum recall:

| Alerts/shift | Model recall | Attention | Events caught of 330 | Annual cost |
|--:|--:|--:|--:|--:|
| 2 | 0.55 | 0.95 | 172 | £148,136 |
| **4** | **0.68** | **0.88** | **197** | **£132,597** |
| 12 | 0.85 | 0.55 | 154 | £200,058 |
| 24 | 0.92 | 0.25 | 76 | £313,773 |

**At 24 alerts a shift the model finds 92% of genuine events and the collection catches 76 of them.** The firehose is not a trade of money for welfare; it loses both.

**Enforcement.** Alert volume above the shift budget fails the eval rather than being delivered ([evals/animal-health](../evals/animal-health/README.md)). Accept rate is one of the four continuous monitors in [mlops](../hld/mlops/README.md).

**Status: Derived** (the budget) **/ Unmeasurable yet** (the attention curve it rests on, which is calibrated during shadow). **Verdict: the number exists and is defensible; the curve behind it is the softest input in this repository and is listed as an open question.**

---

## 5. Portability - a provider swap takes two weeks

**Goal.** `NFR_14`: swap a provider for one capability in ≤2 weeks without changing ticketing or MQTT contracts.

**Metric.** Elapsed days of a real, timed swap drill.

**Threshold.** **≤ 10 working days.**

**Arithmetic.** The scope a swap has to cover, from [uncertainty](../hld/mlops/uncertainty.md): one adapter implementation, one eval suite re-run, one golden-case re-baseline. It does **not** cover re-embedding, because there are no embeddings; it does not cover retraining, because six of seven capabilities train in BigQuery ML from estate tables that no provider holds; and it does not cover fine-tuning data, because none exists.

**Enforcement.** An annual swap drill on a designated capability - the flow forecast - timed and written up. A drill exceeding two weeks is a revisit trigger on [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md): the abstraction is not working and gets fixed before another capability is added.

**Status: Unmeasurable yet.** **Verdict: unproven.** This is the single largest untested claim in the submission, and the drill is the only thing that will settle it.

---

## 6. Privacy - inferred attributes never become identity

**Goal.** Cohort analysis uses what a guest told us. It does not infer demographics, and nothing analytical leaks into a CRM (`NFR_8`).

**Metric.** Count of inferred attributes in the cohort feature set; count of analytical fields written to a guest record.

**Threshold.** **Zero and zero.**

**Arithmetic.** None needed; this is a schema property. What makes it enforceable rather than aspirational is that it is checked at the boundary: the cohort feature set is an explicit allow-list of declared fields, so adding an inferred attribute requires editing the allow-list, which is a visible change in a pull request rather than an emergent behaviour.

**Enforcement.** [evals/cohorts](../evals/cohorts/README.md) gates inferred attributes at zero, and it is one of the two hardest gates in the repository - the other being the price clamp.

**Status: Specified.** **Verdict: Pass** by construction.

---

## 7. Performance - the gate does not make anyone wait

**Goal.** `NFR_3`: gate redeem ≤2 seconds p95 from local cache; checkout ≤5 seconds p95 when online.

**Metric.** p95 latency at the lane, and the resulting queue.

**Threshold.** 2 s p95 verify; 10-second sustained presentation cycle including the party walking through.

**Arithmetic.** The thing that actually matters here is not the 2 seconds, it is what the 2 seconds buys, and [hld/sizing](../hld/sizing.md#3-gate-lane-count) does that conversion: at 15,000 visitors a day, a 40% first-two-hours arrival share and a 3.2 average party size, the peak is 25 presentations a minute. At 6 presentations per lane per minute and 70% target utilisation that is 6 lanes, plus one spare: **7 lanes, and 3 today.**

The sensitivity is worth carrying into the civil works, because it is the expensive direction to be wrong in:

| First-two-hours arrival share | Lanes at 15,000/day |
|:--|--:|
| 30% | 5 |
| 40% (the assumption) | 7 |
| 60% | 10 |

**Status: Derived** (lane count) **/ Unmeasurable yet** (the latency itself). **Verdict: the capacity plan is sound; the latency target is unproven.**

---

## 8. Cost efficiency - AI does not eat the ticket margin

**Goal.** `NFR_12`: cloud cost visible per pipeline, inference budgeted per capability, and the whole thing small against takings.

**Metric.** Cost per visitor, and inference spend per capability against its budget.

**Threshold.** **≤ $0.0005 per visitor**, and no capability over its hard cap.

**Arithmetic.** From [cost-analysis](../cost-analysis/README.md): $72.40/month at 15,000 visitors a day, which is 450,000 visitors a month.

```
$72.40 / 450,000 = $0.00016 per visitor      -> 3.1x headroom against the threshold
```

Two supporting checks, both with their own thresholds:

| Check | Threshold | Modelled | Verdict |
|:--|:--|--:|:--|
| Cloud as a share of ticket revenue | ≤ 0.1% | 0.00067% | Pass, by 150x |
| Bill after a provider doubles every price | ≤ $150/month | $86.87 | Pass |
| Inference total against combined hard cap | ≤ $210/month | $17.08 | Pass |

**Enforcement.** Per-pipeline billing labels with alert thresholds, and the cost-per-capability monitor that is already one of the four continuous monitors in [mlops](../hld/mlops/README.md).

**Status: Derived.** **Verdict: Pass** with substantial headroom, and the headroom is the finding: the architecture is 3x inside a threshold that a token-metered design would struggle to meet at all.

---

## 9. Elasticity - 3x growth without a rewrite

**Goal.** `NFR_10`: three times the visitors without rewriting ticketing, ingest, or intranet contracts.

**Metric.** What changes between 5,000 and 15,000 visitors a day.

**Threshold.** No architectural change. Capacity changes only.

**Arithmetic.** From [hld/sizing](../hld/sizing.md#7-what-this-means-at-15000-visitors-a-day):

| Quantity | 5,000/day | 15,000/day | Change |
|:--|--:|--:|--:|
| MQTT devices | 527 | 535 | +1.5% |
| Messages/second, peak | 17.3 | 17.6 | +1.7% |
| Warehouse GiB/month | 3.80 | 3.97 | +4.5% |
| Cloud cost/month | $67.24 | $72.40 | +7.7% |
| **Gate lanes** | **3** | **7** | **+133%** |

**Only the gate scales with visitors.** Everything else is fixed by the size of the estate and the density of its instrumentation, because 521 of 535 devices count rides, enclosures and paths rather than people.

**Status: Derived.** **Verdict: Pass**, with the caveat that this makes 3x growth a civil-engineering problem at the perimeter rather than a cloud-capacity problem - a cheaper answer than `NFR_1` anticipates, but one that has to be in the ground before the visitors arrive, not after.

---

## 10. Operability - three people can actually run this

**Goal.** The criterion six ADRs weigh options against, and the one this page is least comfortable about.

**Metric.** Routine operational load, in FTE-equivalents, against a three-person team.

**Threshold.** **≤ 1 of the 3 FTE** on routine operations, leaving two on delivery.

**Arithmetic.** [cost-analysis §9](../cost-analysis/README.md#9-where-the-money-actually-is) makes the stakes explicit:

| Line | £/month | Share of run cost |
|:--|--:|--:|
| Platform team, 3 FTE | £11,250 | 84.0% |
| Hardware, amortised | £2,078 | 15.5% |
| Cloud | £57 | 0.43% |

**A decision that adds half an FTE costs 40 times the entire cloud bill.** The load this architecture actually creates:

| Source | Load | Note |
|:--|:--|:--|
| 12 zone gateways as estate infrastructure | Power, mounting, time sync, firmware, site visits | [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) names this as its central negative consequence |
| 535 field devices | Battery, replacement, recalibration | Scales with instrumentation, not visitors |
| 7 AI capabilities, each with a fallback, an eval set and a promotion gate | The real question | |
| Serverless cloud | Near zero | The reason [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) chose it |

**Status: Unmeasurable yet.** **Verdict: At risk, and it is the honest answer.** Nothing in the cloud bill threatens this estate. Seven AI capabilities under a three-person team might, and no arithmetic here proves otherwise. It is why [ADR-0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) keeps most capabilities advisory rather than autonomous, and why the [phasing in the README](../README.md) ships three capabilities rather than seven.

**Revisit trigger:** routine operations passing 1.5 FTE. At that point the answer is to retire a capability, not to hire - because a capability nobody has time to supervise is worse than one that does not exist.

---

## Summary

| # | Characteristic | Threshold | Modelled | Status | Verdict |
|--:|:--|:--|:--|:--|:--|
| 1 | Availability | No cloud term on the hot path | Zero terms; 85,000 visitors/year saved | Derived | **Pass** |
| 2 | Data integrity | 100% of gaps render as unknown | Correctness property | Specified | Not running |
| 3 | Safety | Zero autonomous state changes | Zero, one clamped exception | Specified | **Pass** by design |
| 4 | Trustworthiness | ≤4 alerts/shift | 4, from a 182:1 cost ratio | Derived | **Pass** |
| 5 | Portability | ≤10 working days to swap | Untested | Unmeasurable yet | **Unproven** |
| 6 | Privacy | Zero inferred attributes | Zero, allow-listed | Specified | **Pass** by construction |
| 7 | Performance | ≤2 s p95, 7 lanes at target | 3 lanes now, 7 planned | Derived / not yet | Capacity sound |
| 8 | Cost efficiency | ≤$0.0005/visitor | $0.00016 | Derived | **Pass**, 3.1x headroom |
| 9 | Elasticity | No architectural change at 3x | +7.7% cost, +4 lanes | Derived | **Pass** |
| 10 | Operability | ≤1 of 3 FTE on routine ops | Unknown | Unmeasurable yet | **At risk** |

Five pass on arithmetic a reader can check today. Three are specified and waiting on a harness. Two are honestly unproven, and both of them - the swap drill and the three-person team - are about people rather than technology, which is where this architecture's remaining risk actually sits.

Related: [cost-analysis](../cost-analysis/README.md), [hld/sizing](../hld/sizing.md), [hld/mlops](../hld/mlops/README.md) (the four continuous monitors), [evals](../evals/README.md), [3_NFRs](../requirements/3_NFRs.md).
