# Cost analysis

The kata briefing asks how we would handle our model provider changing prices. This document exists so that question has an arithmetic answer rather than a reassuring one.

The short version: **only $14.47 of a $72 monthly cloud bill is metered by a model provider at all, so doubling every AI price in the world adds twenty dollars a month.** That is not luck. It is the consequence of a decision taken in [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) and [ADR-0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) - batch-scored classical models over the estate's own data, with exactly one generative path - and the cost model is where that decision gets paid off.

Every figure derives from the volumes in [hld/sizing](../hld/sizing.md) and the published rates in the next section. Cloud costs are in USD because Google publishes them in USD; estate economics are in GBP.

This document closes three open questions:

- **Per-capability inference budgets** - [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md), section 6
- **Whether the ops copilot is funded at all** - [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md), section 7
- **Per-pipeline budget ceilings, especially the inference line** - [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md), section 6

## Published rates

All list prices, single-region, no committed-use discount, read September 2026.

| Service | SKU | Rate | Source |
|:--|:--|:--|:--|
| Pub/Sub | Message throughput, publish and delivery | $40/TiB, first 10 GiB/month free, **1 KiB minimum per message** | [pricing](https://cloud.google.com/pubsub/pricing) |
| Pub/Sub | BigQuery export subscription | $50/TiB, no free tier | [pricing](https://cloud.google.com/pubsub/pricing) |
| BigQuery | Active logical storage | $0.02/GiB-month, first 10 GiB free | [pricing](https://cloud.google.com/bigquery/pricing) |
| BigQuery | Long-term logical storage (after 90 days unmodified) | $0.01/GiB-month | [pricing](https://cloud.google.com/bigquery/pricing) |
| BigQuery | On-demand analysis | $6.25/TiB scanned, first 1 TiB/month free | [pricing](https://cloud.google.com/bigquery/pricing) |
| BigQuery ML | ARIMA_PLUS, logistic, linear, k-means model creation | **$312.50/TiB processed** | [pricing](https://cloud.google.com/bigquery/pricing) |
| BigQuery ML | Boosted tree creation (Vertex passthrough) | $6.25/TiB + Vertex node-hours | [pricing](https://cloud.google.com/bigquery/pricing) |
| BigQuery ML | Evaluation, inspection, prediction | $6.25/TiB | [pricing](https://cloud.google.com/bigquery/pricing) |
| Dataflow | Streaming vCPU / memory / Streaming Engine unit | $0.069/vCPU-hr, $0.003557/GiB-hr, $0.089/SECU-hr | [pricing](https://cloud.google.com/dataflow/pricing) |
| Cloud Run | vCPU active / idle, memory, requests | $0.000024 and $0.0000025/vCPU-s, $0.0000025/GiB-s, $0.40/M requests | [pricing](https://cloud.google.com/run/pricing) |
| Cloud Run | Free tier | 180,000 vCPU-s, 360,000 GiB-s, 2 M requests per month | [pricing](https://cloud.google.com/run/pricing) |
| Vertex AI | Gemini 2.5 Flash, **batch** | $0.15/M input, $1.25/M output | [pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing) |
| Vertex AI | Gemini 2.5 Flash, standard / cached input | $0.30 and $2.50/M, cached input $0.03/M | [pricing](https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing) |
| Cloud Storage | Archive class, single region | $0.0012/GiB-month, 365-day minimum | [pricing](https://cloud.google.com/storage/pricing) |

## Assumptions

Continuing the numbering in [hld/sizing](../hld/sizing.md), which supplies A1-A11.

| # | Assumption | Value | Note |
|:--|:--|:--|:--|
| C1 | Exchange rate | $1.27 = £1 | One conversion, applied once, at the roll-up |
| C2 | Adult day ticket | £24 | The brief names no price; this is the assumption every revenue figure below rests on |
| C3 | Child day ticket | £16 | |
| C4 | Family pass, 2 adults + 2 children | £70 | The product [FR#1](../requirements/2_FRs.md) is built around |
| C5 | Blended ticket revenue per visitor | £19 | Tickets only. Food, retail and events are excluded, which understates revenue and so overstates the cloud's share |
| C6 | Warehouse subscriptions per topic | 2 | The landing pipeline and the alerting path |
| C7 | Staff using the intranet | 30 | Duty managers, ride ops, keepers, commercial, the Countess |
| C8 | Platform team | 3 FTE at £45,000 loaded | The three-person team the operability criterion in six ADRs refers to |
| C9 | Hardware amortisation | 5 years, straight line | |
| C10 | Vertex node-hours for the two boosted-tree retrains | $5 and $8/month | Estimated, not derived. The largest unverified figure in this document |

## 1. Ingest: what the 1 KiB floor does

Pub/Sub bills a **1 KiB minimum per message** regardless of payload. At 250 B per reading ([A9](../hld/sizing.md)), the estate pays for 4.1x the bytes it actually sends. **Message count, not payload size, is the cost driver** - which means the lever is aggregation at the edge, exactly what [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) already mandates for a different reason.

```
20,074,000 messages/month x 1 KiB                    = 19.14 GiB published
x 3 (publish + 2 deliveries, C6)                     = 57.43 GiB
- 10 GiB free                                        = 47.43 GiB = 0.0463 TiB
x $40/TiB                                            = $1.85/month
```

The counterfactual is the point. If all 535 devices published once a second, as an unaggregated design would:

```
535 x 86,400 x 30 = 1,386,720,000 messages/month     = $154.59/month
```

**Edge aggregation is an 84x lever on ingest cost**, and it is free because the gateway had to exist anyway to survive islanding. It is rare for the cheap answer and the resilient answer to be the same answer; here they are, and it is worth saying so out loud.

### The streaming pipeline is the single largest line, and it should not be

[ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) draws the path as `Pub/Sub -> Dataflow -> BigQuery`. Priced, a 24/7 streaming Beam job is this:

```
2 vCPU   x 730 h x $0.069     = $100.74
8 GiB    x 730 h x $0.003557  =  $20.77
3 SECU   x 730 h x $0.089     = $194.91
                                -------
                                $316.42/month
```

**That is more than four times the entire rest of the platform, to process 17.6 messages a second.** It also quietly contradicts a claim in [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md), which selected GCP partly because "BigQuery is serverless so the warehouse has no idle cost". The warehouse has none. The pipeline in front of it has a great deal.

The work that job does is validation, dedupe on `(device id, sequence)`, and gap detection. None of it needs to be streaming:

| | Path A: Dataflow streaming | Path B: Pub/Sub BigQuery subscription |
|:--|:--|:--|
| Landing | Beam pipeline, 24/7 worker | Export subscription, no compute |
| Dedupe | Stateful, in-flight | Hourly `MERGE` on the open partition |
| Gap detection | Windowed, in-flight | Scheduled query every 5 minutes on a clustered table |
| Gap marker latency | seconds | up to 5 minutes |
| Cost | **$316.42/month** | **$1.43/month** |

The latency column is where a reader should push back, because [NFR_3](../requirements/3_NFRs.md) demands welfare alerts within 60 seconds. It survives the push: **welfare alerts are raised at the gateway by deterministic rules, not in the cloud** ([ADR-0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)). The cloud path carries the model in shadow and fills the warehouse. Neither is minute-sensitive.

**Recommendation: Path B at launch**, with Dataflow reserved for the batch retrains where it is genuinely the right tool. Every figure below uses Path B. This is offered as a proposed amendment to [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) rather than an override of it - the transport decision that record makes is unaffected, and the interface is identical either way.

**Revisit trigger:** any consumer that needs cloud-side detection inside a minute, or any enrichment that cannot be expressed as SQL over the landed table. Either one buys back the $315.

## 2. Warehouse and analysis

Storage comes straight from [hld/sizing section 6](../hld/sizing.md): **$0.85/month** for all three retention tiers at full accumulation, 195 GiB in total.

Analysis is the larger line, and it is driven by staff, not by sensors:

```
30 staff (C7) x 20 dashboard views/day x 30 days     = 18,000 queries/month
each scanning one day-partition of occupancy         = 0.132 GiB
                                                     = 2,382 GiB
+ scheduled feature queries (96 forecast runs/day)   =    58 GiB
+ eval harness and the four MLOps monitors           =   100 GiB
                                                     = 2.48 TiB
- 1 TiB free, x $6.25/TiB                            = $9.25/month
```

Partitioning by day and clustering by zone is what keeps this at $9.25 rather than $700. An unpartitioned full-table scan per dashboard view would cost 30x more than everything else in this document combined, which is the practical reason the partitioning appears in [hld/data-structure](../hld/data-structure/README.md) rather than being left to an implementer's judgement.

## 3. Model training and scoring

This is the section most likely to surprise, and not in the direction people expect.

**BigQuery ML charges $312.50/TiB for model creation - fifty times the analysis rate.** An iterative model makes up to 50 passes over its training table, so bytes processed is roughly table size x iterations. Training on raw rows is therefore the expensive mistake available here, and it has nothing to do with model providers:

| Pricing model trained on | Table | Passes | Processed | Weekly retrain | Per month |
|:--|--:|--:|--:|--:|--:|
| Raw order rows, 1 year | 513 MB | 50 | 25.7 GB | $7.30 | **$29.17** |
| Aggregated feature table `(day, price band, channel, party size)` | 6.6 MB | 50 | 330 MB | $0.09 | **$0.38** |

**77x, for a change that costs one `GROUP BY`.** Every capability below trains on an aggregated feature table for this reason.

| Capability | Technique | Feature table | Cadence | Rate | $/month |
|:--|:--|--:|:--|:--|--:|
| Popularity and flow forecast | ARIMA_PLUS, multi-series | 42 MB | monthly | $312.50/TiB | 0.60 |
| Pricing and yield | Logistic regression | 6.6 MB | weekly | $312.50/TiB | 0.38 |
| Cohort clustering | k-means, declared attributes only | 3 MB | monthly | $312.50/TiB | 0.04 |
| Piranha population | Regression against a census anchor | 1 MB | weekly | $312.50/TiB | 0.06 |
| Animal health anomaly | Boosted tree per subject type | 120 MB | monthly | node-hours (C10) | 5.00 |
| Ride maintenance ([ADR-0014](../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md)) | Boosted tree on vibration and current | 400 MB | monthly | node-hours (C10) | 8.00 |
| All batch scoring | `ML.PREDICT` | 60 GB/month | continuous | $6.25/TiB | 0.34 |
| | | | | **Subtotal** | **$14.42** |

Two observations a judge should be able to check:

- **The two boosted trees are 90% of the AI bill**, and they are the two capabilities the brief did not ask for directly - animal health inference and ride maintenance. The three things the Countess actually asked about cost $1.02 a month between them.
- **Nothing here is priced per token.** Six of seven capabilities are classical models scored in batch against the estate's own warehouse. That is the structural fact the sensitivity analysis in section 6 turns on.

## 4. The one token-metered path

[ADR-0012](../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) puts a generative narrative on top of the cohort tables. It is the only place in the architecture where a model provider meters us by the token.

```
4 weekly commercial summaries + 1 monthly cohort report + 15 segment narratives
  = 20 generations/month

input   20 x 8,000 tokens = 0.160 M x $0.15/M (batch)   = $0.024
output  20 x 1,200 tokens = 0.024 M x $1.25/M (batch)   = $0.030
                                                          -------
                                                          $0.054/month
```

**Five and a half cents a month.** Batch rather than standard pricing because these are scheduled reports that nobody is waiting on, which halves the rate for free.

### Is the ops copilot funded?

[ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) leaves this open, and correctly identifies why it is the awkward one: *"it is the only capability whose cost scales with staff curiosity rather than with estate size."*

```
30 staff (C7), 20 of them asking 3 questions/day       = 1,800 queries/month
cached SOP corpus  1,800 x 5,000 tokens x $0.03/M      = $0.27
uncached question  1,800 x 1,000 tokens x $0.30/M      = $0.54
output             1,800 x   400 tokens x $2.50/M      = $1.80
                                                         -------
                                                         $2.61/month
```

Without context caching on the shared SOP corpus the same usage costs $5.04, so caching is worth having but is not load-bearing.

**Answer: yes, fund it, with a $30 hard cap.** At the modelled usage it is 3.6% of the bill. The honest risk is not the rate, it is the multiplier: at 100x the assumed usage it costs $261/month and becomes larger than everything else put together. So it is funded with a cap that drops it to non-generative SOP search rather than a cap that stops it - which is the same fallback discipline [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) requires of every other capability.

## 5. Roll-up

| Line | $/month |
|:--|--:|
| Cloud Run: 4 services, 2 kept warm for the 5 s checkout p95 | 24.55 |
| Platform allowance: logging, monitoring, egress, registry, secrets | 20.00 |
| BigQuery analysis: dashboards, features, evals | 9.25 |
| Model training and batch scoring (section 3) | 14.42 |
| Pub/Sub throughput | 1.85 |
| Pub/Sub BigQuery subscription and scheduled dedupe/gap queries | 1.43 |
| BigQuery storage, hot and warm | 0.68 |
| Cloud Storage Archive, cold tier | 0.17 |
| Vertex AI generative: cohort narrative | 0.05 |
| **Total, 15,000 visitors/day** | **$72.40** |
| Optional ops copilot | +2.61 |

Cloud Run deserves a note: **$19.44 of its $24.55 is idle time on two minimum instances.** A cold start would blow the 5-second checkout p95 in `NFR_3`, so the estate is paying twenty dollars a month to never make a guest wait for a container. Stated plainly because it is the one place the architecture buys latency with money rather than with design.

### Cost per visitor

| | 5,000/day (today) | 15,000/day (target) |
|:--|--:|--:|
| Visitors/month | 150,000 | 450,000 |
| Cloud total | $67.24 | $72.40 |
| **Cost per visitor** | **$0.00045** | **$0.00016** |

**Tripling visitors raises the bill by 7.7%, and the cost per visitor falls by two thirds.** That follows directly from [hld/sizing section 7](../hld/sizing.md): only the gate scales with visitors, and gate hardware is capital, not cloud.

### Against the takings of a day

At C2-C5 and 15,000 visitors/day:

```
450,000 visitors x £19 (C5)                = £8,550,000/month ticket revenue
$72.40 / 1.27 (C1)                         =       £57.01/month cloud
```

| Framing | Figure |
|:--|:--|
| Cloud as a share of ticket revenue | 0.00067% |
| Ticket revenue per £1 of cloud spend | £149,800 |
| The whole platform, per month | less than one family pass (C4) |
| **AI**, per visitor | **£0.000025, one four-hundredth of a penny** |

`NFR_12`'s risk line reads: *"Vision models and unbounded GenAI copilots dominate opex before tickets do."* Measured, they are 0.09% of the estate's run cost. That risk was real and the architecture retired it - by rejecting vision for occupancy ([ADR-0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)) and by capping the one copilot.

## 6. Sensitivity

| Scenario | What moves | New total | Change |
|:--|:--|--:|--:|
| **Baseline**, 15,000/day | | **$72.40** | |
| Volume 3x: 5,000 to 15,000/day | Cloud Run leaves the free tier; ticketing rows triple | $72.40 | +7.7% from $67.24 |
| **Model provider doubles every price** | AI-metered lines: $14.47 to $28.94 | **$86.87** | **+20%** |
| Model provider raises prices 10x | AI-metered lines: $14.47 to $144.70 | $202.63 | +180% |
| Instrumentation 3x: 535 to 1,605 devices | Pub/Sub 3x, storage 3x, analysis 3x | $112.20 | +55% |
| Ops copilot funded, as modelled | +$2.61 | $75.01 | +3.6% |
| Ops copilot used 10x the assumption | +$26.10 | $98.50 | +36% |
| Ops copilot used 100x the assumption | +$261.00 | $333.40 | +360% |
| Dataflow streaming retained (Path A) | +$314.99 | $387.39 | +438% |

Read down that column and the conclusion is not subtle. **The two largest risks to this budget are an uncapped copilot and an always-on pipeline. Neither is a model-pricing risk.** A provider doubling its prices is the *fifth* worst thing that could happen to this bill, and it costs fourteen dollars.

That is the answer to the briefing's first uncertainty question, and the reason it is a boring answer is architectural rather than fortunate: [ADR-0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) made AI asynchronous and advisory, [ADR-0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) refused vision in favour of counting, and [ADR-0012](../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) left exactly one generative surface. Each was decided on other grounds. The price insensitivity is the dividend.

## 7. Per-capability inference budgets

This closes the open question in [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md): *"Per-capability inference budgets - with commercial, before go-live."*

`ADR-0004` requires metering at the capability interface with "a budget with a kill switch that drops it to its fallback rather than taking the consumer down". These are those budgets. The alert threshold is roughly 3x modelled, to leave room for growth without hiding a runaway; the hard cap is roughly 10x, at which point the capability stops calling the model and serves its documented fallback.

| Capability | Modelled | Alert at | Hard cap | Behaviour at the cap |
|:--|--:|--:|--:|:--|
| Popularity and flow forecast | $0.85 | $5 | $20 | Last week's same-slot average, flagged stale |
| Pricing and yield | $0.40 | $3 | $15 | Published rate card, rules only, no proposals |
| Cohort analysis and narrative | $0.09 | $2 | $10 | The SQL cohort tables, without the narrative |
| Animal health anomaly | $5.03 | $15 | $50 | Deterministic rules layer only; shadow pauses |
| Piranha population | $0.07 | $2 | $10 | Last census plus the feed-based interval |
| Ride maintenance ([ADR-0014](../adrs/ADR-0014%20-%20Predictive%20ride%20maintenance%20in%20shadow%20behind%20the%20inspection%20schedule.md)) | $8.03 | $25 | $75 | The statutory inspection schedule, unchanged |
| Ops copilot | $2.61 | $10 | $30 | SOP retrieval and search, no generation |
| **Total** | **$17.08** | **$62** | **$210** | |

Every fallback in the right-hand column is a capability the estate already has and already tests, because [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) requires the fallback to be exercised rather than merely declared. Hitting a budget cap is therefore a degradation, not an outage - and for ride maintenance and animal health it degrades to the regime the estate is legally obliged to run anyway.

## 8. Per-pipeline budget ceilings

This closes the open question in [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md): *"Per-pipeline budget ceilings, especially the inference line - before the first AI capability leaves shadow."*

`NFR_12` requires cost visible by pipeline. These are the three pipelines it names, plus the two that the arithmetic says also need watching.

| Pipeline | Billing label | Modelled | Alert at | Ceiling |
|:--|:--|--:|--:|--:|
| Ticketing | `pipeline=ticketing` | $24.55 | $75 | $200 |
| Warehouse and analysis | `pipeline=warehouse` | $10.10 | $40 | $120 |
| AI inference | `pipeline=inference` | $14.47 | $62 | $210 |
| MQTT ingest | `pipeline=ingest` | $3.28 | $15 | $50 |
| Platform | `pipeline=platform` | $20.00 | $50 | $100 |
| **Total** | | **$72.40** | **$242** | **$680** |

`ADR-0001` also sets a revisit trigger: *"BigQuery or Pub/Sub cost exceeds the ticketing line - revisit tiering before revisiting the provider."* Where does that sit? Warehouse is $10.10 against ticketing's $24.55, and warehouse scales with instrumentation and staff. **The trigger fires at roughly 2.5x today's dashboard and sensor load** - which the instrumentation-3x row in section 6 already crosses. So it is a live trigger, not a theoretical one, and the first mitigation is materialised views for the dashboard queries rather than anything to do with GCP.

## 9. Where the money actually is

Cloud is not the estate's cost. Stating only the cloud bill would be the more flattering choice and the less useful one.

**Capital, day one:**

| Item | Qty | Unit | Total |
|:--|--:|--:|--:|
| Field sensors: vibration, current, counters, probes, beacons, feed stations | 485 | £120 | £58,200 |
| Zone gateways: industrial, 8 GiB buffer, PoE, weatherproof, UPS | 12 | £600 | £7,200 |
| Gate lanes: scanner, controller, canopy, power ([3 lanes day one](../hld/sizing.md)) | 3 | £2,500 | £7,500 |
| Keeper handhelds: rugged, gloved use, full-shift battery (`NFR_17`) | 14 | £500 | £7,000 |
| Duty-manager tablets | 8 | £400 | £3,200 |
| Weather stations | 2 | £800 | £1,600 |
| Radio survey, installation, cabling, weatherproofing | | | £40,000 |
| **Total capital** | | | **£124,700** |

**Monthly run cost:**

| Line | £/month | Share |
|:--|--:|--:|
| Platform team, 3 FTE (C8) | £11,250 | 84.0% |
| Hardware, amortised over 5 years (C9) | £2,078 | 15.5% |
| Cloud, everything except AI | £45.62 | 0.34% |
| **AI: training, scoring, generation** | **£11.39** | **0.09%** |
| **Total** | **£13,385** | |

**Eighty-four per cent of the run cost is three people.** The hardware is fifteen. The AI is a rounding error on the rounding error.

Two things follow, and both are architectural rather than financial:

- **Operability is the expensive characteristic here, not compute.** Six ADRs weigh options against "operability for a three-person team". This table is why that was the right criterion: a decision that adds half an FTE costs 40x more than the entire cloud bill, and a decision that doubles the cloud bill costs nothing. [ADR-0003](../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) accepts "tens of gateways with a maintenance rota" as its central negative consequence - this is the table against which that consequence should be judged, and it is a real cost in the 15.5% line and a larger one in the 84%.
- **The Countess's question about AI cost has been answered in the wrong currency all along.** The risk was never the inference bill. It is whether a three-person team can run seven AI capabilities, each with a fallback, an eval set, and a promotion gate. That is a headcount question, and it is why [ADR-0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) keeps most capabilities advisory and why the [phasing in the README](../README.md) ships three, not seven.

## What would change these numbers

| Change | Effect | Where it is tracked |
|:--|:--|:--|
| A jurisdiction ADR forcing a multi-region or a Tier 2 region | Cloud Run up ~25%, storage up ~50% | [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) open question on region |
| Vision for occupancy, currently rejected | Adds a per-frame metered path; this analysis stops holding | [ADR-0013](../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) |
| Any capability moving from batch to per-request inference | Turns a fixed cost into a variable one | [ADR-0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) |
| Ticket prices C2-C5 proving wrong | Only the percentage framings in section 5 move; no cloud figure changes | This document |
| C10, the two Vertex node-hour estimates | Up to $13 of a $72 bill is estimated rather than derived | Section 3 |

Related: [hld/sizing](../hld/sizing.md) (every volume used here), [fitness-functions](../fitness-functions/README.md) (the cost-per-capability fitness function), [hld/mlops](../hld/mlops/README.md) (the cost-per-capability monitor that enforces section 7), [3_NFRs](../requirements/3_NFRs.md) (`NFR_12`, `NFR_14`).
