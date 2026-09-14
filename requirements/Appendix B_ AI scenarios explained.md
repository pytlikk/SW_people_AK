# Appendix B: AI Scenarios Explanation

> Note: This document is supplementary to FRs about AI/ML usage and explains application scenarios in more detail.
>
> Design rule: each capability has a purpose, inputs, outputs, **human-in-the-loop**, **eval / golden cases**, and a **failure mode** when the model is wrong. AI is advisory + experimental; ticketing and access stay deterministic.

| AI Functionality | Area | Scenario Explanation |
|:--|:--|:--|
| Async dynamic pricing | Commercial | Overnight (or periodic) job proposes prices and packages inside approved floors/ceilings from occupancy, daypart, weather, and past conversion. POS and web consume a published list. If the cloud is unreachable, last-known-good prices stand. Improves yield without putting a model in the gate. |
| Offer / package A/B | Commercial | Test family-pass shapes, bundles (“rides + venomous-house tour”), daypart surcharges, return voucher vs membership. Sticky assignment for the visit, including **offline** (baked into ticket or local flags). Safety, welfare, and legal copy are not experiment factors. |
| Cohort / demographic analysis | Commercial | After an experiment or a busy day, explain *who* converted or returned - families with toddlers on wet Tuesdays vs locals - using declared ticket attributes and opted-in membership, plus where they went. Inferred segments stay analytical. Turns “variant B won 3%” into an investment decision. |
| Popularity & dwell | Ops | Fuse MQTT counts, ride cycles, and ticket scans into heat maps and ranks. Directly attacks “we have no idea what is popular”. Gaps in MQTT must show as unknown, not as zero. |
| Flow / congestion forecast | Ops | 30–90 min predicted queues and zone load so staff move before the piranha house overflows. Evaluated against actual counts; degrades to recency baseline when ingest is unhealthy. |
| Staffing & investment advice | Ops / Countess | Combine popularity, predicted load, and downtime of *popular* rides (lost tickets/hour) into “send three hosts to zone C” and “this enclosure is expensive and empty”. Duty manager / Countess accept or reject. |
| Itinerary / next-best-experience | Guest | Opt-in suggested walking order and “skip ride 7 queue, start at piranha”. A/B the suggestions. No routes through restricted animal areas. When offline, fall back to printed/kiosk cards. Serves both guest experience and flow. |
| Win-back / next-best-visit | Guest / loyalty | After the visit, propose a reason to return (new feeding slot, weekday family pass, membership). Caps, opt-out, measure 90-day return. This is the answer to “we want returning visitors and aren’t sure how”. |
| Animal health & feeding anomalies | Keepers | Score unusual feed refusal, leftover, aggression, water/temp drift, and keeper notes. Alert with evidence. Keepers confirm/reject → training signal. High recall; never auto-medicate or cull. Lowers the cost of late-discovered illness. |
| Piranha population estimate | Keepers | Colony-level estimate from feeder events and optional camera/sonar, checked against scheduled human census. Kata-specific; must have golden cases and an error band, not a single magic number. |
| Predictive ride maintenance | Ride ops | Anomaly on sensor streams + usage → “inspect ride 12 before Saturday peak”. Draft work order. Engineer decides. False-alarm rate is a first-class metric so historic kit is not taken down every afternoon. |
| Ops copilot (RAG) | Intranet | Staff ask “what is the piranha feeding SOP?” or “why is zone D amber?”. Answers grounded in estate documents and live state, with citations. Invented vet doses are a failure, not a cleverness. |
| MLOps / eval harness | Platform | Version models, run golden cases, detect drift, shadow-deploy, record human overrides. Makes AI *verifiable* in production when providers change behaviour. |

## Human-in-the-loop by default

| Capability | Can act automatically? | Human role |
|:--|:--|:--|
| Gate admit / deny | No (deterministic entitlements) | Staff override only via audited break-glass |
| Price within approved band after shadow | Eventually, with kill switch | Commercial sets bands; reviews lift |
| Experiment assignment | Yes (flag snapshot) | Commercial designs experiments; denylist is code |
| Staffing suggestion | No | Duty manager accepts/rejects |
| Animal health / population | No | Keeper / vet |
| Ride closure | No | Ride ops / duty manager |
| Copilot answers | Display only | Staff remain accountable |

## Verification (minimum)

- Pricing: holdout + floor/ceiling violations = 0.
- Experiments: pre-registered primary metric (yield per visitor and/or 90-day return); no peeking on safety metrics as “optimizable”.
- Flow forecast: predicted vs actual occupancy; pause recommendations when MQTT gap exceeds a threshold.
- Animal alerts: recall vs keeper-labelled events; piranha estimate vs census.
- Maintenance: precision/recall vs work orders and missed failures.
- Copilot: groundedness / citation evals; forbidden-advice golden cases.
