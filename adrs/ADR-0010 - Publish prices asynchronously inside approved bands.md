# ADR-0010 - Prices are published as a snapshot inside human-approved bands, never computed at checkout

## Date

2026-09-14

## Status

Proposed

## Context

The estate must become profitable or the carnivorous plants get sold. Dynamic pricing is the obvious lever: charge more on a sunny Saturday, less on a wet Tuesday, and shape the family-pass mix toward what actually sells.

Two constraints make the obvious implementation wrong.

**The checkout and gate cannot depend on a model.** NFR_3 says pricing and experiment assignment at checkout read a snapshot and do not call a model. The kiosk is on the estate, behind the same patchy Wi-Fi as everything else, and a kiosk that cannot quote a price is a kiosk that cannot sell a ticket.

**Unguarded prices are a business risk, not just a technical one.** A model that locks a family out of a heritage estate, or gives away admission at a price that cannot cover animal care, causes damage that a rollback does not undo. [5_Risks and mitigation](../requirements/5_Risks%20and%20mitigation.md) names this directly.

There is also a cold-start problem nobody can wish away: the estate has **no conversion history**. There is no price-elasticity data because there has never been a system that recorded price against outcome. A model cannot be trained on data that does not exist, so the first version of this capability must be useful while being, in effect, a rules engine that logs well.

This record decides how prices reach the guest. It does not decide the experiment mechanism ([ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)), who is analysed afterwards ([ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)), or the price points themselves, which are commercial decisions.

**Foreclosed here:** any model inference on a guest-facing purchase or admission path, and any published price outside a human-approved band.

## Evaluation criteria

- **No model on the purchase path (driving)** - a kiosk with a dead uplink must still quote and sell. NFR_3 and NFR_16.
- **Bounded blast radius (driving)** - no price can be published outside approved floors and ceilings, whatever the model says. Violations counted, target zero.
- **Works with no history** - useful in phase 1 with manual prices and no trained model ([Appendix C](../requirements/Appendix%20C_%20Future%20scope.md)).
- **Measurable yield effect** - a holdout, so "the model helped" is a measurement rather than a belief.
- **Reversibility** - a bad price list must be replaceable within one publish cycle.

## Options

- **Option A - Real-time price per request**: the checkout calls a pricing service which calls a model, per guest, per basket.
- **Option B - Cached real-time**: as A, with a cache in front.
- **Option C - Async publish inside approved bands (chosen)**: a scheduled job proposes prices, they are clamped to commercial floors and ceilings, published as a versioned snapshot, and pulled by all channels.
- **Option D - Manual pricing only**: commercial sets prices by hand, indefinitely.

| | No model on path (driving) | Bounded blast radius (driving) | Cold start | Measurable | Reversibility |
|---|---|---|---|---|---|
| A Real-time | Fail - the stated constraint | Fail - each request is a chance to be wrong, unclamped | Fail - needs history to be worth the latency | Partial | Partial - a bad model prices live |
| B Cached real-time | Partial - fails on cache miss, which is exactly when the link is down | Partial | Fail | Partial | Partial |
| C Async publish | Pass - snapshot read, no inference | Pass - clamping happens before publication, once | Pass - phase 1 publishes manual prices through the same path | Pass - holdout on the snapshot | Pass - republish or roll back to the previous version |
| D Manual only | Pass | Pass | Pass | Fail - no lever, no lift to measure | Pass |

Not options: surge pricing while a guest is inside the estate (they have already paid to enter; re-pricing captive guests is a trust catastrophe on a family day out); pricing on inferred demographics ([ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) forbids it).

## Decision

**A scheduled job proposes prices; the proposal is clamped to commercial floors and ceilings; the clamped result is published as a versioned snapshot; every channel reads the snapshot.**

```
occupancy + daypart + weather + calendar + conversion history
        v
  proposal job (scheduled)
        v
  CLAMP to commercial floors/ceilings   <-- deterministic code, not the model
        v
  versioned price list snapshot  -->  web / kiosk / gate lane (last known good)
```

Four rules follow.

**The clamp is deterministic code outside the model, and it runs on every proposal.** A model cannot be trusted to respect its own bounds, and a prompt or a feature weight cannot be an enforcement mechanism. The clamp is the safety property, and a proposal that needed clamping is logged as a signal that the model is drifting.

**The snapshot is versioned and immutable, and every channel keeps the last good one.** A kiosk offline for six hours sells at six-hour-old prices. That is correct behaviour, not degradation: the alternative is not selling.

**Phase 1 publishes manually set prices through exactly this path.** The pipeline ships before the model does, so the whole mechanism - publish, pull, clamp, holdout, rollback - is proven with humans in the loop of the numbers themselves. When a model appears it changes only where the proposal comes from.

**A holdout never receives model-proposed prices.** Without it, "yield went up" is indistinguishable from "it was a sunny quarter". The holdout is how OKR 2.2 is measurable at all.

Authority is **L3 Act-in-band after shadow** ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)): the job publishes automatically, but only inside bands commercial set in advance, with a kill switch that reverts to the previous snapshot.

No-model-on-path and bounded blast radius decided it. Real-time (A) fails the first outright and the second by construction - every request is an unclamped opportunity to be wrong. Cached real-time (B) is worse than it looks: its failure mode is a cache miss, and cache misses correlate with exactly the connectivity trouble the estate has. Manual pricing (D) is safe and is what phase 1 does; it loses as a destination because it offers no lever, and the estate needs the margin.

## Key differentiators

- **The guest-facing path has no AI in it at all,** so pricing intelligence cannot take down ticket sales.
- **The safety bound is code, not a model property,** and its violation count is a metric with a target of zero.
- **The same mechanism carries manual and model prices,** so phase 1 de-risks phase 2 instead of being thrown away.
- **Rollback is publishing the previous snapshot** - the fastest remediation available.
- **Offline price staleness is a designed behaviour with a visible age,** not an undiscovered bug.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Availability | **Improved (driving)** | Selling continues during any cloud or model outage, from the last snapshot (NFR_16). |
| Safety of business outcome | **Improved (driving)** | Clamping bounds the worst published price regardless of model behaviour. |
| Performance | **Improved** | Checkout reads a local snapshot; no inference latency on the guest path (NFR_3). |
| Testability | **Improved** | Clamp violations and snapshot integrity are assertable; the holdout gives a real measurement (NFR_13). |
| Recoverability | **Improved** | Roll back by republishing a prior version. |
| Price freshness | **Weakened (deliberate)** | Prices react on a publish cycle, not to the current minute. A sudden heatwave is priced late. |
| Personalisation | **Weakened** | A published list is per-SKU and per-segment, not per-guest. Genuinely individual pricing is unavailable. |
| Consistency across channels | **Weakened** | Channels on different snapshot versions can show different prices at the same moment. |

**Deliberately downplayed: freshness and per-guest personalisation.** Both are what a real-time engine buys, and both are refused. On this estate the guest journey starts with a purchase made at home or at a kiosk on arrival, and a family deciding whether to visit is not a minute-by-minute demand curve - so the value of per-minute pricing is small, while the cost of a kiosk that cannot quote is a lost visit. The channel-consistency weakness is the sharper one: two guests in the same queue can see different prices because their kiosks pulled at different times, and that needs a short publish interval and an honest answer at the counter.

**Fit with the existing architecture.** This is the gate pattern applied to commerce: read a locally held artefact, decide without the network, reconcile later. The price snapshot sits in the same edge snapshot store as the experiment flags, the revocation list, and the SOP cache, and it carries its own age like every other edge artefact ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)). The AI is on the async side of the boundary drawn in [hld/README](../hld/README.md), with the rest of the models.

## Consequences

### Positive

- Ticket sales survive a total AI or cloud failure (driving criterion).
- No published price can fall outside commercial bounds (driving criterion).
- Phase 1 ships a working, measurable pricing pipeline with no model at all.
- Rollback is one publish; the worst case is yesterday's prices.

### Negative

- **Two guests can be quoted different prices in the same queue** because their channels hold different snapshot versions. Staff need an answer for that conversation, and the answer must be honest rather than technical.
- Real demand shocks - an unforecast heatwave, a competitor closing - are priced a cycle late.
- The clamp means the model's upside is capped by commercial's imagination in setting bands. A genuinely better price outside the band is simply not available.
- With no conversion history, the first model version is barely better than the rules it replaces, and it must be honest about that rather than pretending to precision.
- A holdout costs money by construction: some guests are deliberately shown non-optimised prices for as long as the measurement runs.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Ruinous price published | Model proposes far outside sane range | Deterministic clamp before publication; violation count with a target of zero; clamping events alarm as a drift signal |
| Family priced out | Optimisation makes the estate inaccessible | Floors **and** ceilings; family-pass accessibility reviewed by commercial as a guest-satisfaction metric, not only yield |
| Stale snapshot | An offline channel sells at old prices for hours | Age shown on every snapshot; maximum staleness before a channel refuses to sell a discounted variant; short publish interval |
| Channel price mismatch | Two guests, two prices, same queue | Short publish interval; staff guidance and the ability to honour the lower price; mismatch rate monitored |
| Cold start overconfidence | A model trained on thin data quoted as if precise | Shadow mode first; holdout; phase 1 manual prices publish through the same path so the model must beat a real baseline |
| Experiment contamination | Pricing changes mid-experiment, wrecking attribution | Experiment assignment is sticky and snapshot-versioned ([ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)) |
| Yield claimed without proof | Seasonal uplift attributed to the model | Holdout is mandatory for as long as the capability is credited with lift |

## Verification

**Primary metrics**

- **Floor/ceiling violations in published snapshots: target 0.** A single violation is a capability failure.
- Yield per visitor, model cohort against holdout (OKR 2.2).
- Contribution margin per visitor trend (OKR 2.1).
- Share of purchases made from a stale snapshot, and maximum staleness observed.
- Clamping rate - how often the model proposes something needing clamping, as an early drift indicator.

**Tests (CI)** - golden cases in [`evals/pricing/`](../evals/pricing/)

- A proposal above the ceiling or below the floor is clamped, and the clamp is logged.
- No published snapshot contains a price outside its band.
- The checkout price read makes no call to a capability interface or a model.
- A channel with no network sells from its last snapshot.
- A snapshot older than the configured maximum refuses to apply a discount variant and falls back to the base band.
- Rolling back to the previous snapshot version restores prior prices exactly.
- The holdout cohort never receives a model-proposed price.

**Ops check**

- Commercial reviews proposed versus published prices before the first automatic publish leaves shadow.
- Counter staff walk through the two-guests-two-prices conversation before launch.

**Open questions**

- Publish interval - nightly, or more often. Shorter reduces the mismatch window and increases cost. Before launch.
- Maximum snapshot staleness before a channel refuses discount variants - with commercial, before launch.
- Holdout size, balancing measurement power against forgone yield - with commercial, before the model leaves shadow.
- Whether family-pass shapes are priced independently or as a bundle - before [ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md) experiments run on composition.
- Band-setting cadence and who signs off - commercial, before go-live.

**Revisit triggers**

- Clamping becomes frequent - the model is wrong or the bands are, and both need a look before trusting the output.
- Channel price mismatch generates guest complaints - shorten the interval or narrow variant scope.
- A genuine need for intra-day reaction appears (a heatwave pattern costing real money) - revisit publish frequency, not the no-model-on-path rule.
- Two years of conversion history exists - re-evaluate model class; this record's structure does not change.

## Conclusion

Prices are proposed by a scheduled job, clamped by deterministic code to commercial floors and ceilings, and published as a versioned snapshot that every channel reads locally. Chosen because the purchase path must work offline and because an unbounded published price is a business risk that no rollback repairs. The costs are prices that react on a cycle rather than a minute, no per-guest personalisation, and the possibility of two guests in one queue seeing two prices - accepted in exchange for a kiosk that can always sell and a price that can never be ruinous.

Related: [ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md) (how a variant travels with the guest), [ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) (who responded), [ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) (L3 in-band authority), [hld/scenarios/yield](../hld/scenarios/yield/README.md), golden cases in [`evals/pricing/`](../evals/pricing/).
