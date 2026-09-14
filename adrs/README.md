# Architecture decision records

Template: [ADR-000-template.md](ADR-000-template.md) - Kata Log shape (Five Nines, Pragmatic, BluzBrothers, CELUS Ceals).

A requirement is not a decision. Tech choice, MQTT bus, pricing algorithm, and AI placement wait for their own ADR.

| # | Title | Status | Date |
|---|---|---|---|
| [001](ADR-001-use-home-bought-token-pool.md) | Use home-bought token pool, spend at live park prices | Proposed | 2026-09-11 |
| [002](ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) | Use signed static QR for attraction-token presentation | Proposed | 2026-09-11 |
| [020](ADR-020-use-care-subject-as-unit-of-record.md) | Use a care subject, distinct from its enclosure, as the unit of record | Proposed | 2026-09-14 |
| [021](ADR-021-keep-keeper-observations-off-the-telemetry-path.md) | Keep keeper observations off the telemetry path, in a device-local append-only log | Proposed | 2026-09-14 |
| [022](ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md) | Detect welfare anomalies against each subject's own baseline before any trained model | Proposed | 2026-09-14 |
| [023](ADR-023-anchor-piranha-population-on-human-census.md) | Publish a piranha population interval anchored on human census, never a count | Proposed | 2026-09-14 |

## Number blocks

Workstreams run in parallel, so numbers are allocated in blocks rather than first-come.

| Block | Workstream | Boundaries |
|---|---|---|
| 001-019 | Ticketing and access; popularity; pricing; return incentive | - |
| 020-029 | Animal care | [animal-care-scope.md](../docs/animal-care-scope.md) |

## Plan (which ADR next)

| Order | Topic | Needs | Not this ADR |
|---|---|---|---|
| 1 | Token pool: home-bought pool vs per-attraction tickets | ADR-001 (this log) | Prices, rates, refund policy |
| 2 | Ticketing QR at rides + enclosures | ADR-002; assumes ADR-001 | - |
| 3 | Popularity meter (checkpoint events → usable counts) | ADR-002 `validated` events | Not the pricing model |
| 4 | Dynamic pricing (AI) | Popularity + shop data | Not gate admission |
| 5 | Return incentive: AI Guide + engaging games in the app | Tokens + popularity | Not animal health |

Animal care runs in parallel, not behind the list above:

| Order | Topic | Needs | Not this ADR |
|---|---|---|---|
| 020 | Care subject as the unit of record | - (this log) | Sensors, algorithms, field schema |
| 021 | Keeper observation capture and offline reconciliation | ADR-020 | Which enclosures are instrumented |
| 022 | Health and feeding anomaly detection (FR#2I) | ADR-020, ADR-021 | Population estimation |
| 023 | Jumping-piranha population estimation (FR#2J) | ADR-020 | Individual animal health |

MQTT broker, kiosks, and gateway placement are still assumptions until someone writes those ADRs. Animal care depends on that backbone and names it as an assumption rather than designing it ([animal-care-scope.md](../docs/animal-care-scope.md), section 2).

Ride and asset predictive maintenance (FR#2K) has **no owner**. It is specified in requirements but not claimed by any workstream, and it is not an explicit requirement in the kata brief.
