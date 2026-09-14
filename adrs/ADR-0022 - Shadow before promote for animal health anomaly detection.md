# ADR-0022 - Rules first, model in shadow, and promotion is earned per subject type

## Date

2026-09-14

## Status

Proposed

## Context

Looking after 200+ exotic and poisonous animals is expensive, and much worse when they get sick. The brief asks for ways of tracking animal health and how much and how well they are eating. FR#2I wants anomalies detected across 55 displays, with high recall first, keeper accept/reject as the training signal, and never an autonomous welfare decision.

The hard part is not the model. It is that **there is no training data and no labels**. Nobody has ever recorded this collection's feeding behaviour in a structured way, so on day one there is no history, no notion of normal, and no examples of illness. A supervised model has nothing to learn from.

The species diversity makes it worse. A reticulated python eats once a fortnight; a colony of piranha eats daily; an aquatic invertebrate's baseline is different again. "Ate less than usual" is not one distribution - it is dozens, most with a handful of observations. A single model across 55 displays would be learning an average of incompatible behaviours.

And the cost of errors is asymmetric in a specific way. A missed sick animal is suffering and expense - so recall matters. But a flood of false alarms trains keepers to ignore the inbox, and then the true positive is missed too, with the system having consumed attention on the way. Recall bought with noise is not recall.

This record decides how anomaly detection is built and promoted. It does not decide the subject model ([ADR-0020](ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md)), the keeper data path ([ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md)), or population estimation ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)).

**Foreclosed here:** a model alerting keepers before it has run in shadow against that subject type, and any autonomous welfare action at any confidence.

## Evaluation criteria

- **Useful on day one with no labels (driving)** - the collection opens to the public before any model can be trained.
- **Keeper trust preserved (driving)** - alert volume must stay inside the attention budget, because a queue nobody reads is worse than no queue.
- **Recall on genuine illness** - measured against keeper-labelled events, and it is the reason the capability exists.
- **Handles species heterogeneity** - a fortnightly feeder and a daily colony cannot share one threshold.
- **Cost (NFR_12)** - no vision by default; this runs on feed and environment data the platform already collects.

## Options

- **Option A - Deterministic rules only**: keeper-set thresholds per subject, no model ever.
- **Option B - Model from the start**: train on whatever accumulates and alert from launch.
- **Option C - Rules in production, model in shadow, promotion per subject type (chosen)**: rules alert from day one; the model scores in parallel without alerting; it is promoted only for subject types where it demonstrably beats the rules inside the attention budget.
- **Option D - Vision-based behavioural monitoring**: cameras on enclosures, model watches behaviour.

| | Useful day one (driving) | Keeper trust (driving) | Recall | Heterogeneity | Cost |
|---|---|---|---|---|---|
| A Rules only | Pass | Pass - predictable, keeper-authored | Partial - misses subtle multi-signal patterns | Pass - per subject by construction | Low |
| B Model from start | Fail - no labels, no baseline | Fail - unvalidated volume destroys the inbox immediately | Unknown | Fail - one model, many distributions | Medium |
| C Rules + shadow model | Pass - rules are the day-one product | Pass - promotion is gated on volume | Pass - measured before it can alert | Pass - promotion is per subject type | Low |
| D Vision | Fail - no labels, needs funding | Partial | Partial | Partial | High |

Not options: autonomous intervention of any kind (no auto-medicate, no auto-cull, no auto-resolve - foreclosed here and capped at L2 in [ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)); veterinary imaging models per species ([Appendix C](../requirements/Appendix%20C_%20Future%20scope.md)).

## Decision

**Deterministic rules alert from day one. The model scores in shadow beside them and is promoted only for subject types where it beats the rules while staying inside the keeper attention budget.**

```
feed events (offered, leftover, refusal, aggression)
environment readings (water quality, temperature)
keeper observations (structured flags)
        |
        +--> RULES (keeper-authored thresholds per subject)  --> alert inbox  [live day one]
        |
        +--> model scoring (same inputs)                      --> shadow log  [no alert]
                                                                    |
                                        promotion per subject type, gated on:
                                        - recall >= rules on keeper-labelled events
                                        - alert volume within attention budget
                                        - minimum shadow duration
                                        - golden cases passing
```

Four rules follow.

**Rules are the product, not a placeholder.** Keepers already know that this python eats fortnightly and that water temperature outside a band is a problem. Encoding that is high-value, immediately, and needs no data science. It also generates the labels the model will eventually need, which is the only way out of the cold start.

**Promotion is per subject type, and partial promotion is the expected end state.** The model may beat the rules for daily-feeding colonies where there is signal density, and never beat them for a fortnightly feeder where each data point is a fortnight apart. That is a legitimate outcome: some subject types stay on rules forever. Forcing uniform coverage would mean promoting a model somewhere it has not earned it.

**Every alert carries evidence, confidence, and the specific signals that fired.** A keeper triaging an inbox needs to see "leftover 60% above the trailing norm for three consecutive feeds, water temperature within band" in order to decide in seconds. An alert that says only "anomaly detected, 0.83" costs more attention than it saves.

**Duplicate alerts on the same subject are suppressed into one open case.** Three days of refusal is one case with a worsening trend, not three alerts. This is where most of the attention budget is won or lost.

Authority is **L2 Advise, capped permanently** ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)). Keepers confirm or reject with a reason code; escalation to the vet is a keeper decision; no amount of accuracy promotes this past L2.

Day-one usefulness and trust preservation decided it. Model-from-start (B) fails both: there is nothing to train on, and an unvalidated alert stream destroys the inbox in its first week - after which no later improvement recovers the keepers' attention. Rules-only (A) is genuinely defensible and is what phase 1 ships; it loses as a destination because it cannot combine weak multi-signal evidence, which is exactly where early illness hides. Vision (D) needs labels it cannot have and money that is better spent elsewhere.

## Key differentiators

- **The capability is useful before any model exists,** and the rules generate the labels the model needs.
- **Promotion is earned per subject type,** so a model is never trusted for a species where it has no evidence.
- **Alert volume is a promotion gate, not an afterthought,** which protects the only scarce resource in the animal house - keeper attention.
- **Case suppression means a deteriorating animal is one worsening case,** matching how keepers actually think.
- **Keeper rejects are structured signal,** so the loop improves the thing that gates it.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Safety | **Improved (driving)** | No autonomous welfare action; keeper and vet retain authority (NFR_7, NFR_15). |
| Trustworthiness | **Improved (driving)** | Volume caps and evidence-rich alerts keep the inbox worth reading (NFR_11). |
| Testability | **Improved** | Rules are deterministic and directly testable; the model is measured against them before it can alert (NFR_13). |
| Fault tolerance | **Improved** | Rules are the named fallback and always available ([ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)). |
| Cost efficiency | **Improved** | Runs on already-collected feed and environment data; no vision. |
| Detection latency | **Weakened (deliberate)** | An L2 alert waits for a keeper. Overnight anomalies wait for the morning round unless escalated. |
| Recall in the rules era | **Weakened** | Thresholds miss subtle multi-signal patterns, which is the gap the model is meant to close. |
| Coverage uniformity | **Weakened** | Some subject types stay on rules indefinitely, so capability quality varies across the collection. |

**Deliberately downplayed: recall, in two places.** First, rules-era recall is lower than a good model's would be. Second, and more uncomfortable: when a capability hits its attention budget it raises its threshold, which suppresses some true positives. We accept both because the alternative - a high-recall firehose - produces an inbox that is ignored, at which point measured recall is irrelevant because nobody is reading. The mitigation is that recall is tracked as a first-class metric, so a budget-driven threshold rise is visible as a recall cost rather than an invisible saving.

**Fit with the existing architecture.** The scoring job reads from the warehouse and writes to an alert inbox; it holds no animal state and changes none, exactly like every other capability in [hld/README](../hld/README.md). Keeper-confirmed facts outrank model output ([ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md)), which is the same precedence the gate uses for a valid entitlement. The alert inbox lives on the offline-capable keeper surfaces rather than in a separate AI product (NFR_15).

## Consequences

### Positive

- The collection can open with anomaly detection running from day one (driving criterion).
- Keeper attention is protected by an enforced volume cap (driving criterion).
- Rules produce the labelled history the model needs, so the cold start has an exit.
- The fallback is always live, because it is the day-one product.

### Negative

- **Overnight anomalies wait for a human.** An L2 capability cannot act, so a refusal detected at 2am surfaces on the morning round unless an on-call escalation path exists - which is a staffing question the estate has not answered ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) open question).
- **Capability quality will be uneven across the collection,** and keepers working across species will find the system smarter about some animals than others. That is honest but confusing, and it needs explaining rather than hiding.
- Threshold raises to meet the attention budget suppress real anomalies; the metric makes it visible but does not make it costless.
- Rules are keeper-authored, so their quality depends on keeper time during setup - a real demand on people who have animals to look after.
- Shadow evaluation needs months of labelled events before promotion is possible for any subject type, so the model's value arrives late.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Missed illness | False negative on a genuinely sick animal | Rules err toward sensitivity for high-risk subjects; recall tracked against keeper-labelled events; keeper rounds remain the primary safeguard, not the system |
| Alert fatigue | Volume erodes trust | Attention budget is a promotion gate; case suppression; evidence shown for fast triage; accept rate monitored as drift |
| Suppressed true positive | Threshold raised to fit the budget | Recall reported per period; a recall drop is a capability failure requiring a fix, not an accepted saving |
| Confounded signals | Temperature, season, or breeding move consumption without illness | Environment as covariates; seasonal baselines per subject; consumption alone never triggers a population claim ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)) |
| Protocol non-compliance | Missing leftover data degrades the signal | Consumption inference disabled for subjects whose protocol lapses ([ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md)) |
| Premature promotion | Model promoted on thin evidence | Minimum shadow duration and label count per subject type; written eval record required |
| Drift after promotion | Seasonal or collection change degrades the model | Scheduled golden-case runs; accept-rate monitoring; automatic demotion to rules on a drift alarm |
| Rules never authored | Setup demand on keeper time is not met | Starter thresholds from published husbandry guidance, reviewed rather than authored from scratch |

## Verification

**Primary metrics**

- **Recall against keeper-labelled health events**, tracked per period and per subject type (OKR 5.1).
- **Alert volume per keeper per shift**, against the attention budget.
- Accept rate and reject reason distribution - the drift indicator.
- Time-to-detect a keeper-confirmed anomaly, versus the paper-round baseline (OKR 5.1).
- Shadow model recall and volume versus rules, per subject type - the promotion evidence.

**Tests (CI)** - golden cases in [`evals/animal-health/`](../evals/animal-health/)

- A model in shadow never produces a keeper-visible alert.
- Promotion requires recorded shadow duration, label count, recall versus rules, and volume within budget; a missing element blocks promotion.
- Alert volume exceeding the shift budget fails the eval rather than being delivered.
- Repeat anomalies on one subject collapse into a single open case with a trend.
- Every alert carries confidence, evidence, contributing signals, and input freshness.
- No code path lets an alert be resolved without a keeper decision.
- A drift alarm demotes the subject type to rules automatically.
- A subject with incomplete feed protocol data is excluded from consumption-based scoring.

**Ops check**

- Keepers confirm the inbox is workable in a real shift, including in an animal house with the uplink down.
- Injected golden cases confirm keepers are still discriminating rather than accepting by reflex.

**Open questions**

- Attention budget per keeper per shift - with keepers, before anything leaves shadow. This is the number the whole record depends on.
- Starter thresholds per subject type - with keepers and the vet, before the collection opens.
- Minimum shadow duration and label count for promotion - before the first promotion.
- Overnight escalation path for high-risk subjects - with the Countess, before the collection opens.
- Reject reason taxonomy - with keepers ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) open question).

**Revisit triggers**

- Accept rate above roughly 95% with no rejections - nobody is reading; investigate before trusting.
- Recall falls while volume stays inside budget - the threshold is hiding the problem.
- The model fails to beat rules for any subject type after a full season - retire the model and keep the rules, which is a legitimate outcome.
- A high-value subject type needs faster-than-human response - that is an argument for a deterministic alarm, not for promoting this capability past L2.

## Conclusion

Keeper-authored deterministic rules alert from day one and remain the permanent fallback; the model scores in shadow and is promoted only for subject types where it beats those rules while staying inside the keeper attention budget. Chosen because there are no labels on day one and because an unvalidated alert stream would destroy keeper trust before any model could earn it. The accepted costs are lower rules-era recall, uneven capability quality across a very mixed collection, and welfare response that waits for a human - the last being deliberate, since genuine emergencies travel deterministic alarm paths instead.

Related: [ADR-0020](ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md), [ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md), [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) (reuses this promotion pattern), [ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) (L2 cap), golden cases in [`evals/animal-health/`](../evals/animal-health/).
