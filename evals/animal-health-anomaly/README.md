# Eval - animal health and feeding anomaly (FR#2I)

Golden cases for the detector decided in [ADR-022](../../adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md).

> **Status: specification, not a running harness.** These files state what correct behaviour is. There is no runner in this repository yet ([evals/TEMPLATE.md](../TEMPLATE.md): store evaluation placeholders only). What would execute them is named below.

## What is being verified

| | |
|---|---|
| Primary metric | Recall against keeper- and vet-confirmed welfare events, by collection |
| Precision proxy | Accept rate on raised alerts |
| Guardrail metrics | Alert budget burn; stale-baseline count; suppressions by declared expected state |
| Fallback | Tier 1 baselines; tier 0 safety rules. Both survive the loss of every model and vendor |

**Recall needs a denominator the system does not own.** A keeper must be able to open a welfare event with no alert prompting it, and those unprompted confirmed events are what recall is measured against. Without that path the detector can only measure its own opinion of itself. This is an architectural requirement, not a reporting detail - see ADR-022, Verification.

## How these cases are meant to work

Tiers 0 and 1 are arithmetic, so they are **exactly reproducible**: a golden case returns the identical decision every run, or the code is wrong. That is the reason the tier ladder in ADR-022 puts deterministic detection first - most of the detector can be tested like ordinary software.

Tier 3 is a language model and is not reproducible. Its cases assert **constraints** - what must never appear, and that every claim cites an evidence field that exists on the alert - rather than an expected string.

Cases assert **behaviour** (alert raised, suppressed, unknown, reduced confidence), not specific scores. Thresholds, warm-up windows, and alert budgets are open questions in ADR-022 and are configuration, not fixtures. A case that hard-coded a score would break the moment ops tuned a threshold, and would be testing the configuration rather than the logic.

Fixture values are illustrative. They are shaped to be plausible for the collection described, not drawn from real estate data, which does not exist.

## Case index

| Case | Tier | Asserts |
|---|---|---|
| [golden-001](golden-001-intake-decline-individual.json) | 1 | A sustained intake decline against a subject's own history raises a ranked alert with evidence |
| [golden-002](golden-002-stale-baseline-unknown.json) | 1 | A subject with no recent observations publishes as unknown, never as normal |
| [golden-003](golden-003-expected-state-suppression.json) | 1 | A keeper-declared seasonal state suppresses scoring and writes an audit record |
| [golden-004](golden-004-tier0-edge-offline.json) | 0 | A hard environment threshold fires at the edge with no cloud, not gated by any model |
| [golden-005](golden-005-narrative-forbidden-advice.json) | 3 | The narrative states no diagnosis, dose, or treatment, and cites only evidence present on the alert |
| [golden-006](golden-006-new-arrival-warmup.json) | 1 | A subject inside its warm-up window uses a collection prior and declares reduced confidence |

## Schema

The five fields from [`golden-case-template.json`](../golden-case-template.json) are present in every case. Three are added: `tier` and `adr` for traceability, and `asserts` so a reader knows what a case is defending without reverse-engineering it.

## Not yet written

Named so the gap is visible rather than inferred:

- **Kill switch** - disabling tiers 2 and 3 leaves tiers 0 and 1 fully functional.
- **Label weighting** - an investigated reject and a dismissed reject must not carry equal weight into tier 2 training (ADR-022, label-bias risk).
- **Tier 2 promotion gate** - a shadow model must beat tier 1 recall at equal alert budget on held-out confirmed events before promotion. Needs tier 2 to exist.
- **Placement-at-time join** - environment readings from a subject's previous enclosure must not appear in its features after a move (ADR-020).
