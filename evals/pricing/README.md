# evals/pricing - golden cases for async pricing and experiments

Capability: price proposal and publication ([ADR-0010](../../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)) and experiment assignment ([ADR-0011](../../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)).

Authority: L3 Act-in-band after shadow. Fallback: last published snapshot, then the commercially approved default band.

## Primary metrics

| Metric | Target | Why |
|:--|:--|:--|
| Floor/ceiling violations in a published snapshot | **0** | A single violation is a capability failure, not a tuning issue |
| Yield per visitor, model cohort vs holdout | Positive lift, no guardrail regression | OKR 2.2 - without the holdout, a sunny quarter looks like a good model |
| Clamping rate | Tracked; a rise is a drift signal | The model proposing out-of-band is an early warning |
| Purchases from a stale snapshot | Works; staleness bounded and observed | Offline selling is required, not tolerated |
| Assignment stickiness violations | **0** | One party, one variant, whole visit |

## Refusal and guard cases

These encode constraints. Each cites the ADR it protects, so deleting the guard breaks a traceable test.

| Case | Expected | Protects |
|:--|:--|:--|
| Model proposes a price above the approved ceiling | Clamped to the ceiling; clamping event logged | [ADR-0010](../../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md) - the clamp is deterministic code outside the model |
| Model proposes below the floor | Clamped to the floor; event logged | Same |
| Snapshot containing an out-of-band price is submitted for publication | Publication rejected | No unbounded price ever reaches a guest |
| Checkout reads a price | No call to a capability interface or model occurs | NFR_3 - no model on the purchase path |
| Channel has no network | Sells from its last snapshot | NFR_16 - a kiosk that cannot quote cannot sell |
| Snapshot older than the configured maximum | Discount variant refused; base band applied | Bounded staleness |
| Rollback to previous snapshot version | Prior prices restored exactly | Reversibility is the remediation |
| Holdout cohort requests a price | Never receives a model-proposed price | Lift must stay measurable |
| Experiment definition names a safety closure | Fails to register | [ADR-0011](../../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md) denylist |
| Experiment definition names evacuation copy, a welfare threshold, exit accessibility, or legal terms | Fails to register | Same - each category is a separate case |
| Same assignment key assigned twice | Same variant both times | Stickiness |
| Gate lane needs a variant | Reads it from the claim; makes no assignment call | Offline stickiness |
| Assignment without a recorded snapshot version | Rejected | Analyses must be reconstructable |

## Accuracy cases

Meaningful only once conversion history exists. Phase 1 publishes manual prices through the same pipeline, so these are the phase 2 gate.

| Case | Expected |
|:--|:--|
| Backtest on held-out periods | Proposals beat the manual-price baseline on yield per visitor |
| Wet weekday, low occupancy | Proposal moves toward the floor, not through it |
| Sunny Saturday, high occupancy | Proposal moves toward the ceiling, not through it |
| Family-pass composition variants | Lift measured on the pre-registered primary metric only |

## Guardrails, never objectives

Monitored for regression on every experiment. A guardrail regression invalidates a win rather than being traded against it.

- Family-pass accessibility - the estate must not optimise families out.
- Complaint rate and opt-out rate.
- Any safety or welfare indicator - these are never optimisable.

## Open thresholds

From the ADRs, pending human agreement rather than analysis:

- Publish interval, and the maximum snapshot staleness before discount variants are refused (commercial, before launch).
- Holdout size, balancing measurement power against forgone yield (commercial).
- Minimum sample and duration before an experiment result may be acted on.

Related: [hld/scenarios/yield](../../hld/scenarios/yield/README.md), [ADR-0010](../../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md), [ADR-0011](../../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md).
