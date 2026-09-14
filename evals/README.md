# evals - golden cases per AI capability

NFR_13 makes this folder a condition of going live, not optional polish: every production AI capability has golden cases here, a documented metric, shadow mode before it can act, and a fallback.

These are specifications written in a form that can fail. They are deliberately written before the models.

| Capability | Folder | Primary metric | ADR |
|:--|:--|:--|:--|
| Pricing and experiments | [pricing/](pricing/README.md) | Floor/ceiling violations = 0; yield lift vs holdout | [ADR-0010](../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md), [ADR-0011](../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md) |
| Cohort analysis | [cohorts/](cohorts/README.md) | Inferred attributes persisted = 0; groundedness | [ADR-0012](../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) |
| Flow forecast | [flow-forecast/](flow-forecast/README.md) | Error vs recency baseline; correct suppression | [ADR-0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) |
| Animal health | [animal-health/](animal-health/README.md) | Recall vs keeper labels; volume within budget | [ADR-0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) |
| Piranha population | [piranha-population/](piranha-population/README.md) | Interval coverage; absolute error vs census | [ADR-0023](../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) |

## Two kinds of case, and the second is the important one

**Accuracy cases** ask whether the model is good: forecast error against actuals, recall against keeper labels, interval coverage against census.

**Refusal cases** ask whether the system declines to answer when it should. A population estimate without an interval is rejected. A forecast on low coverage is suppressed. A shadow model produces no alert. A price above the ceiling is clamped.

Refusal cases are what make this architecture verifiable. An accuracy number is a claim that degrades quietly; a refusal case fails loudly in CI the moment someone removes the guard. Most of the estate's AI risk is not "the model is a bit wrong" - it is "the model produced a confident answer from input it should have refused".

## Conventions

- A case states its **input**, its **expected output or refusal**, and **why it matters**.
- Cases that encode a constraint from an ADR cite it, so removing the guard breaks a traceable test.
- Golden cases run on a schedule, not only at deploy, because a provider can change behaviour under a pinned version ([ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)).
- A capability with no fallback registered cannot enter shadow, let alone production.

## Status

These are specifications at kata stage: the cases and thresholds are defined, the harness is not built. Several thresholds are deliberately open questions in the ADRs (attention budgets, coverage thresholds, maximum anchor age) because they need keeper, duty-manager, and commercial agreement rather than an architect's guess.

Related: [hld/mlops](../hld/mlops/README.md), [Appendix B verification section](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md).
