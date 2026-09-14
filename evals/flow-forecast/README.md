# evals/flow-forecast - golden cases for popularity, forecast, and staffing advice

Capability: popularity fusion, 30-90 minute congestion forecast, staffing advice ([ADR-0013](../../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)).

Authority: popularity and forecast L1 Inform; staffing advice L2 Advise. Fallback: same daypart last week, same weather class.

## Primary metrics

| Metric | Target | Why |
|:--|:--|:--|
| Forecast error vs recency baseline on held-out days | **Beats the baseline** | OKR 4.2 - a model that cannot beat "last week" is not worth its cost or risk |
| Zones rendered as zero while gapped | **0** | The failure that sends staff the wrong way |
| Coverage published with every figure | 100% | A value without coverage is not interpretable |
| Rides and displays with daily popularity | ≥90% of 40 + 55 | OKR 4.1 |
| Staffing advice accept rate | Majority at peak after shadow | OKR 4.3 |
| Advice volume per shift | Within the duty manager's cap | A list nobody works has negative value |

## Refusal and guard cases

The gap cases are the heart of this capability. A heat map that shows a dead sensor as a quiet zone is worse than no heat map, because it produces a confident wrong instruction.

| Case | Expected | Protects |
|:--|:--|:--|
| Zone counter silent for longer than its interval | Zone reports **unknown**, not zero, at every aggregation level | [ADR-0013](../../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) |
| Aggregate spans a window containing a gap | No interpolation; coverage reported alongside the value | Same |
| Figure published without a coverage value | Rejected | Same |
| Estate coverage falls below the threshold | Forecast **suppressed**; recency baseline shown and labelled | The capability's meaningful silence |
| Whole zone gateway offline | Zone unknown; forecast for that zone suppressed; advice does not recommend pulling staff from it | The misdirection failure |
| Staffing advice volume exceeds the shift cap | Eval fails rather than advice being delivered | [ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) attention budget |
| Advice attempts to reassign staff automatically | Rejected | L2 - the duty manager decides |
| Guest itinerary routes through a restricted animal area | Rejected | Safety routing is code, not a prompt instruction |
| Backfill arrives for an already-reported window | Aggregate restated without double-counting | Replay is normal, not an incident |

## Accuracy cases

| Case | Expected |
|:--|:--|
| Held-out days, full coverage | Error below the recency baseline on predicted zone occupancy |
| Wet weekday | Beats baseline; does not simply repeat last week's dry Saturday |
| Unforecast event day (coach party, school group) | Error degrades, and confidence reported for those windows widens |
| Ride closure mid-forecast-window | Downstream zone predictions adjust rather than assuming normal throughput |
| Popularity ranking over one week vs one season | Ranking states its observation window; short-window rankings are not labelled investment-grade |

The last case guards the highest-stakes use of this data. The Countess deciding which enclosure to refurbish on a rainy week's numbers is the expensive mistake this capability could cause.

## Suppression is measured too

Suppression protects the estate but costs availability, so it is tracked rather than celebrated:

- Suppression rate by daypart.
- Whether suppression correlates with the busiest days - if the forecast is always absent when most needed, the sensing plan is wrong and lowering the threshold is not the fix.

## Open thresholds

- Coverage threshold for suppression - needs a season of data; starts conservative.
- Staffing advice volume cap per shift (duty manager, before shadow ends).
- Minimum observation window for investment-grade popularity ranking (Countess and commercial).
- Yield-at-risk calculation for a down ride (commercial).

Related: [hld/scenarios/popularity-flow](../../hld/scenarios/popularity-flow/README.md), [ADR-0013](../../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md), [ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md).
