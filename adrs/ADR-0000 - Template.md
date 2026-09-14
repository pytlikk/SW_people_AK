# ADR-00XX - Title

> Title states the decision, not the topic. "Publish prices asynchronously", not "Pricing".

## Date

YYYY-MM-DD

## Status

Proposed | Accepted | Superseded by ADR-00YY

## Context

What forces this decision now. Which FR/NFR it serves, what is already decided upstream, and what makes the obvious answer wrong.

State explicitly what this record does **not** decide, so the next ADR has room.

**Foreclosed here:** the option this record takes off the table for everyone downstream.

## Evaluation criteria

The criteria the options are judged against. Mark the one or two that actually decide it as **(driving)** - if everything is equally important, nothing is.

- **Criterion (driving)** - how it is measured or observed.
- **Criterion** - why it matters but does not decide.

## Options

- **Option A - name (chosen)**: one line.
- **Option B - name**: one line.

| | Driving criterion | Driving criterion | Other criterion | Cost / complexity |
|---|---|---|---|---|
| A | | | | |
| B | | | | |

Not options: what was excluded before comparison, and why (legal, safety, no team capacity, decided in an earlier ADR).

## Decision

The decision in one bold sentence, then the rules that follow from it.

Name the criteria that decided it and say why the runner-up lost. "Both are fine" is not a decision record.

## Key differentiators

- What this buys that the alternatives do not.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| | **Improved (driving)** | |
| | **Weakened** | |

**Deliberately downplayed:** the characteristic knowingly sacrificed, and the argument for spending it.

**Fit with the existing architecture.** How this matches the offline story, event backbone, and identity model the rest of the estate uses (NFR_15) - AI additions are not a sidecar product.

## Consequences

### Positive

- Tied back to the driving criteria.

### Negative

- Including the consequence that lands on someone else's team or on ops practice.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| | | |

## Verification

**Primary metrics** - what proves this works, and what would show it degrading (NFR_13).

**Tests (CI)** - golden cases in [`evals/<capability>/`](../evals/); assertions that can fail.

**Ops check** - what a human confirms in the field.

**Open questions** - with a deadline and an owner, not a wish list.

**Revisit triggers** - the observation that reopens this record.

## Conclusion

Two or three sentences. What was decided, on which criteria, and what it costs.

Related: ADR links, `evals/` path.
