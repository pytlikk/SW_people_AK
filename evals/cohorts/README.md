# evals/cohorts - golden cases for cohort analysis

Capability: cohort and segment analysis of experiment and visit outcomes ([ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)).

Authority: L1 Inform. Fallback: declared-attribute pivot tables with no generated narrative.

## Primary metrics

| Metric | Target | Why |
|:--|:--|:--|
| Inferred attributes persisted to a guest record | **0, structurally** | NFR_8 - inferred is analytical, never identity |
| Cohort coverage of visits | Tracked | Family-pass and member buyers declare most; plain-admission visitors are thin by design |
| Groundedness of generated narrative | Every claim traceable to a supplied metric | An insight that cannot be traced to a query is a hallucination with good grammar |
| Findings that changed a decision | Tracked | The honest usefulness test for an L1 capability |

## Refusal and guard cases

| Case | Expected | Protects |
|:--|:--|:--|
| Guest record schema is extended with an inferred demographic field | Build fails | [ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) - the central constraint |
| Cohort analysis output attempts a write to a guest or membership record | Rejected | Inferred segments live for one analysis |
| Marketing targeting selects on an inferred segment | Rejected | Cohort-level and consented targeting only |
| Analytics consent used to satisfy a marketing-consent check | Rejected | Consent is separate per purpose |
| Reported segment below the minimum cohort size | Suppressed | Re-identification risk |
| Pricing requests an inferred demographic as an input | Unavailable | No pricing on inferred attributes |
| Generated narrative asserts a figure not in the supplied metrics | Eval fails | Groundedness |
| Generated narrative asserts causation from an observational cohort | Eval fails | "Families with toddlers converted better" is not "toddlers cause conversion" |

## Accuracy and usefulness cases

Cohort analysis has no ground-truth label, so the cases test **reproducibility and honesty** rather than correctness.

| Case | Expected |
|:--|:--|
| Same input period analysed twice | Same cohorts and same metrics; clustering is seeded and reproducible |
| Narrative regenerated from identical metrics | No new claims appear between runs |
| A cohort with a small sample | Reported with its sample size and a stated uncertainty, or suppressed |
| Declared party structure conflicts with gate-admitted count | Flagged as a data-quality case, not silently averaged |

The last one matters: a 2+2 pass that admitted four adults is a misdeclaration, and it is a commercial signal rather than a cohort attribute.

## What is deliberately untestable here

There are no cases for age, gender, or household-type accuracy, because those attributes are never collected. That absence is the point of [ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md): the declared party structure is a guest-asserted fact with no error rate, so there is nothing to validate against.

## Open thresholds

- Minimum reportable cohort size (privacy reviewer, before the first briefing).
- Which membership attributes are worth the signup friction (commercial).

Related: [hld/scenarios/yield](../../hld/scenarios/yield/README.md), [ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md).
