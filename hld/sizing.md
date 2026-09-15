# Sizing and volumetrics

The brief gives five numbers - 5,000 visitors a day growing to 15,000, 40 rides, 200+ animals, 55 displays - and the rest of this repository quotes them. This document turns them into the figures the architecture is actually built against: how many devices, how many messages a second, how many gate lanes, how much disk in a gateway, how long a backfill takes, and how many gibibytes land in the warehouse.

Every number below is derived. Where a derivation needs an input the brief does not give, the assumption is numbered and stated, so a reader who disagrees can change one line and recompute the rest.

This document closes two open questions that other records left deliberately open:

- **Gate lane count** for 15,000/day - [ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)
- **Hours-of-islanding SLO and therefore disk sizing per gateway** - [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md), which flags it as blocking hardware procurement

## Assumptions

| # | Assumption | Value | Why this value |
|:--|:--|:--|:--|
| A1 | Operating hours | 10 h/day (10:00-20:00) | A seasonal estate opening; welfare sensing continues for the other 14 h |
| A2 | Share of daily visitors arriving in the first two hours | 40% | Parks with a fixed opening time are front-loaded; this is the figure the lane count is most sensitive to |
| A3 | Busiest 15-minute window versus the two-hour mean | 1.6x | Arrivals inside the opening surge are themselves uneven |
| A4 | Average party size | 3.2 people | Family passes are the product ([FR#1](../requirements/2_FRs.md)); one presentation admits the whole party ([ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)) |
| A5 | Gate presentation cycle, sustained | 10 s | Present, scan, party walks through. The 2 s p95 in `NFR_3` is the verify; this is the human |
| A6 | Target utilisation at peak | 70% | `NFR_1` sets autoscaling at ~70%; a queue at 100% utilisation is unbounded |
| A7 | Aquatic share of the 55 displays | 18 | The brief says "a mix of aquatic and land-based"; the piranha colony is one of these |
| A8 | Estate divided into zones | 12 | One gateway per zone ([ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)); final boundaries are a site-survey question |
| A9 | Telemetry payload on the wire | 250 B | A reading plus the provenance `ADR-0003` mandates: gateway id, device id, event time, ingest time, sequence |
| A10 | Warehouse row, logical | 200 B | Payload after parsing, before BigQuery compression |
| A11 | Per-zone uplink available for backfill | 2 Mbit/s | Consumer-grade radio backhaul across a large estate, sharing the link with live traffic |

## 1. Device count

`NFR_1` asserts "on the order of 500-1,000 MQTT devices". Here is where that number comes from.

| Class | Devices | Count | Publishes |
|:--|:--|--:|:--|
| **Rides (40)** | Vibration, drive and structure | 80 | Every 10 s |
| | Motor current | 40 | Every 10 s |
| | Ride cycle counter | 40 | Per cycle |
| | E-stop and status beacon | 40 | Every 60 s |
| | Queue-entry counter | 40 | Aggregated every 60 s |
| **Displays and enclosures (55)** | Approach and dwell counter | 55 | Aggregated every 60 s |
| | Air temperature and humidity | 55 | Every 300 s |
| | Water quality probe (aquatic only, A7) | 18 | Every 300 s |
| | Water temperature (aquatic only, A7) | 18 | Every 60 s |
| | Feed station | 55 | Per feed event |
| **Zones and paths (12, A8)** | Path counter at zone boundaries | 44 | Aggregated every 60 s |
| | Zone gateway | 12 | Heartbeat every 30 s |
| **Perimeter and staff** | Gate lane scanner and controller (2 x 7 lanes, section 3) | 14 | Per redemption |
| | Keeper handheld | 14 | On sync |
| | Duty-manager tablet | 8 | On sync |
| | Weather station | 2 | Every 300 s |
| | **Total** | **535** | |

**535 devices at full day-one instrumentation**, which lands in the lower half of the asserted range with headroom to roughly 800 as path counters are densified and more enclosures gain feed instrumentation.

The important structural fact: **only 14 of 535 devices scale with visitor count.** Everything else scales with the estate. Tripling visitors does not triple telemetry, which is the opposite of the intuition and it is why section 6 shows ingest cost flat against growth.

## 2. Message rate

| Class | Count | Interval | msg/s |
|:--|--:|:--|--:|
| Vibration | 80 | 10 s | 8.00 |
| Motor current | 40 | 10 s | 4.00 |
| Approach and dwell counter | 55 | 60 s | 0.92 |
| Path counter | 44 | 60 s | 0.73 |
| E-stop and status beacon | 40 | 60 s | 0.67 |
| Queue-entry counter | 40 | 60 s | 0.67 |
| Zone gateway heartbeat | 12 | 30 s | 0.40 |
| Water temperature | 18 | 60 s | 0.30 |
| Ride cycle counter | 40 | ~180 s at peak | 0.22 |
| Air temperature and humidity | 55 | 300 s | 0.18 |
| Water quality | 18 | 300 s | 0.06 |
| Keeper handheld sync | 14 | bursty | 0.05 |
| Feed station | 55 | ~4/day | 0.003 |
| | | **Operating steady** | **16.2** |
| Gate redemption (section 3) | 7 lanes | peak 25 presentations/min | 0.42 |
| Higher ride cycling and counter activity at peak | | | ~1.0 |
| | | **Peak** | **17.6** |

Outside operating hours the ride sensors idle and welfare sensing continues: air temperature, water quality, water temperature, e-stop beacons and gateway heartbeats total **1.61 msg/s**.

```
daily  = 10 h x 16.2 msg/s x 3,600  +  14 h x 1.61 msg/s x 3,600  +  4,688 gate presentations
       = 583,200 + 81,144 + 4,688
       = 669,032 messages/day
```

**Roughly 670,000 messages a day, 20 million a month.** Two readings follow from that:

- The two 10-second sensor classes are **74% of all traffic** (12 of 16.2 msg/s). Predictive ride maintenance is the entire telemetry budget; everything the brief actually asked for is the rounding error. If ingest ever needs to shrink, that is the only place to look.
- 17.6 msg/s is a rate a single broker on a Raspberry Pi handles without noticing. **Nothing in this estate is throughput-constrained.** The constraints are radio coverage, keeper attention, and gate lanes - which is why the rest of this document spends its time there.

## 3. Gate lane count

This closes the open question in [ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md): *"Lane count and physical queue layout for 15,000/day."*

```
peak arrivals   = 15,000 x 40% (A2) / 120 min          =  50 people/min
busiest 15 min  = 50 x 1.6 (A3)                        =  80 people/min
presentations   = 80 / 3.2 (A4)                        =  25 presentations/min
per lane        = 60 s / 10 s (A5)                     =   6 presentations/min
lanes at 100%   = 25 / 6                               = 4.2 lanes
lanes at 70%    = 4.2 / 0.7 (A6)                       = 6.0 lanes
plus one spare for a lane failure                      =   7 lanes
```

| Visitors/day | Peak presentations/min | Lanes at 70% | With one spare |
|:--|--:|--:|--:|
| 5,000 (today) | 8.3 | 2 | **3** |
| 10,000 | 16.7 | 4 | **5** |
| 15,000 (target) | 25.0 | 6 | **7** |

**Build three lanes now and civil works for seven.** The lanes are the expensive, slow part - they need physical space, a canopy, power and a queue approach - and `ADR-0002` already names throughput as the characteristic this design weakens. Retrofitting four more lanes into a perimeter that was poured for three is the failure mode this table exists to prevent.

Sensitivity, because A2 is the assumption doing the most work here:

| If first-two-hours share is | Lanes at 15,000/day |
|:--|--:|
| 30% | 5 |
| 40% (A2) | 7 |
| 50% | 8 |
| 60% | 10 |

Timed entry slots would flatten A2 and remove lanes from the capital plan. That is a commercial decision, not an architectural one, and it is listed in [Appendix C](../requirements/Appendix%20C_%20Future%20scope.md).

## 4. Gateway disk buffer and the islanding SLO

This closes the open question in [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md): *"Hours-of-islanding SLO and therefore disk sizing per gateway - before hardware procurement."*

The mean zone carries 16.2 / 12 = 1.35 msg/s. Zones are not equal - the ride zones carry the vibration sensors - so size every gateway for the busiest, at roughly 3x the mean:

```
busiest zone    = 1.35 x 3                             =  4 msg/s
persisted size  = 250 B payload (A9) + broker overhead = 512 B/message
buffer fill     = 4 x 512                              = 2,048 B/s = 7.03 MiB/hour
```

| Islanding survived | Buffer needed, busiest zone |
|:--|--:|
| 24 h | 169 MiB |
| **72 h (the SLO)** | **506 MiB** |
| 7 days | 1.18 GiB |
| 48 days | 8 GiB |

**The SLO is 72 hours and the specification is 8 GiB of usable buffer.**

72 hours is chosen because it covers a Friday-evening failure in a far corner of the estate that nobody can reach until Monday. 8 GiB is chosen because it is the smallest sensible industrial eMMC and it buys 48 days rather than 3 - the part costs nothing and the difference between "survives a long weekend" and "survives a hard winter" is a rounding error on the bill of materials.

That has a consequence worth stating plainly. `ADR-0003` designs class-based shedding, where popularity is downsampled under buffer pressure to protect welfare and access. **At this sizing that shedding will essentially never fire.** It is insurance against a much denser sensing plan or a multi-week outage, not against the estate as designed. Keeping it is right; claiming it as a live control would not be.

## 5. Backfill drain

When an islanded gateway reconnects it replays everything it buffered, and `ADR-0003` promises that "an outage produces a cost spike rather than a data hole". Here is the size of both.

```
one zone, 72 h  = 72 x 3,600 x 4 msg/s                 = 1,036,800 messages
on the wire     = 1,036,800 x 250 B (A9)               = 259 MB
drain time      = 259 MB / (2 Mbit/s, A11)             = 17 minutes
```

An estate-wide outage islands all twelve zones at once and they share one uplink. Assuming half the estate link is left for live traffic:

```
all zones       = 12 x 259 MB                          = 3.11 GB
drain time      = 3.11 GB / (10 Mbit/s x 50%)          = 83 minutes
```

**83 minutes, inside the 4-hour RTO in `NFR_2`.** The 15-minute RPO in the same requirement is a normal-flow target and is met at 17.6 msg/s with room to spare; after a three-day estate-wide island the warehouse is restated within an hour and a half, and every aggregate over that window is recomputed because `ADR-0003` requires aggregates to be restatable.

The cost of that spike, using the section 6 rates:

```
12 zones x 1,036,800 messages x 1 KiB billing minimum x 3 (publish + 2 deliveries)
  = 36 GiB = 0.035 TiB x $40/TiB = $1.41
```

**A three-day estate-wide blackout costs about a pound and a half to drain.** That is the whole of the "cost spike rather than a data hole" claim, quantified.

## 6. Warehouse volume per retention tier

At the 15,000/day design point:

| Stream | Rows/month | Row size (A10) | GiB/month |
|:--|--:|--:|--:|
| Telemetry | 20,074,000 | 200 B | 3.74 |
| Ticketing: orders and redemptions | 591,000 | 400 B | 0.22 |
| Keeper field events | 10,000 | 500 B | 0.005 |
| | | **Total** | **3.97** |

Against the tiers `NFR_9` already fixes, at steady state once each tier is full:

| Tier | Retention | Accumulated | Rate | Cost/month |
|:--|:--|--:|:--|--:|
| Hot | 30 days, BigQuery active | 3.97 GiB | $0.02/GiB-month | $0.08 |
| Warm | 1 year, BigQuery, long-term after 90 days | 47.6 GiB | $0.02 then $0.01/GiB-month | $0.60 |
| Cold | 3 years, Cloud Storage Archive | 142.9 GiB | ~$0.0012/GiB-month | $0.17 |
| | | **195 GiB** | | **$0.85** |

The whole estate's data history, three years of it, is **under 200 gibibytes**. It would fit on a laptop. This is the number that makes the cost analysis come out the way it does, and it is worth stating before anyone specifies a data platform sized for a problem this estate does not have.

## 7. What this means at 15,000 visitors a day

| Quantity | At 5,000/day | At 15,000/day | Change | Driver |
|:--|--:|--:|--:|:--|
| MQTT devices | 527 | 535 | +1.5% | Estate, not visitors (8 more gate devices) |
| Messages/second, peak | 17.3 | 17.6 | +1.7% | Estate, not visitors |
| Messages/month | 19.98 M | 20.07 M | +0.5% | Estate, not visitors |
| Warehouse GiB/month | 3.80 | 3.97 | +4.5% | Mostly estate; ticketing rows triple |
| Gateway buffer | 8 GiB | 8 GiB | none | Unchanged |
| **Gate lanes** | **3** | **7** | **+133%** | **Visitors** |

**Only the gate scales with the brief's growth figure.** Everything else is fixed by the size of the estate and the density of its instrumentation. The 3x growth question is therefore a civil-engineering question about the perimeter, not a cloud-capacity question - which is a much cheaper answer than the one a reader of `NFR_1` would expect, and it is the reason `NFR_10` can promise 3x growth "without a rewrite".

Related: [cost-analysis](../cost-analysis/README.md) (what these volumes cost), [hld/deployment](deployment.md) (where the counts are physically placed), [ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) and [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) (the two records whose open questions this closes), [3_NFRs](../requirements/3_NFRs.md).
