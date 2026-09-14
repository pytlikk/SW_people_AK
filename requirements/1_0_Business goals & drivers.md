# Business Goals & Drivers

> Note: This document explains how _business drivers_ (factors, resources, and processes that have a major impact on a business's performance and success) affect _business goals_ (specific, measurable, and time-bound objectives that a business aims to achieve) which are implemented through the project _solutions_:
> **Business Drivers → Business Goals → Capabilities / Solutions**
>


## Business Drivers

- **Estate Profitability**: The previous business (highly explosive garden gnomes) is no longer viable. The sprawling estates must generate a replacement income stream from ticketing, on-site experiences, and related spend, or the family faces forced asset sales (including the carnivorous plant collection).
- **Visitor Growth**: Average visitation is ~5,000 per day and must reach at least 15,000 per day within three years. Growth must not collapse guest experience, ride safety, or animal welfare.
- **Operational Visibility**: The estate currently has no reliable view of which rides, zones, and animal displays are popular. Without that, investment and staff deployment are guesswork.
- **Animal Welfare Cost**: Care for 200+ exotic and poisonous animals across 55 displays is already expensive; illness multiplies cost. Healthy animals are both an ethical duty and a profit lever.
- **Guest Loyalty**: First-time ticket sales alone will not hit the growth and profit targets. Returning visitors are required, but the estate does not yet know how to earn them.
- **Safety & Heritage Duty**: Forty 18th-century amusement rides recently passed inspection (after asbestos, broken glass, and garden gnomes were removed). Ongoing evidence of safety and uptime is required while the previously private animal collection opens to the public.

## Business Goals

| Business goal in 3 years | Business strategy |
|:--|:--|
| Grow average daily visitors from ~5,000 to ≥15,000. | Sell tickets (including family passes) at scale. Instrument the estate so capacity, queues, and offers can absorb 3× load. Use demand-aware pricing and packages without blocking access when connectivity is poor. |
| Increase returning-visitor share of daily attendance (suggested target: from unmeasured to ≥30% of visits). | Optional identity / membership after first visit, post-visit offers, personalized itineraries, experiments on what actually brings people back. |
| Make investment and staffing decisions from measured popularity, not anecdote. | Zone / ride / enclosure telemetry, dwell and throughput, predicted congestion, staff recommendations on the ops intranet. |
| Reduce cost and incidence of animal illness; keep animals healthy and the jumping-piranha population under control. | Track health, feeding quality, and (for piranha) population. Detect anomalies early; keepers confirm or reject AI alerts. |
| Keep critical guest access and safety operations available despite patchy Wi-Fi. | Offline-capable gates and kiosks; MQTT store-and-forward to the cloud; last-known-good prices and entitlements at the edge. |

## Drivers -> Goals -> Solutions

| Drivers | Goals | Solutions |
|:--|:--|:--|
| Estate Profitability | positive contribution margin | Ticketing & entitlements (individual + family passes). Async dynamic pricing and package experiments. Cost visibility for animal care and ride downtime. |
| Visitor Growth | ≥15,000 visitors/day within 3 years | Scalable ticketing and access. Demand forecasting. Crowd/flow prediction so 3× visitation remains operable. Guest-facing itinerary / queue advice. |
| Operational Visibility | Invest and deploy staff where it matters | MQTT occupancy + ride/enclosure popularity. Ops intranet heat map. AI staffing and investment recommendations with measured vs predicted evals. |
| Animal Welfare Cost | Healthy animals; controlled piranha population; lower sickness cost | Animal-care records + sensors. Anomaly detection on health/feed/environment. Piranha population estimate with periodic human census as ground truth. |
| Guest Loyalty | Returning visitors ≥30% of visits | Membership / identity loop. Next-best-visit and win-back offers. A/B tests on routes, prices, and packages; AI cohort analysis of *who* returned. |
| Safety & Heritage Duty | Rides stay inspectable and available; hazardous collection safe for public | Ride/asset maintenance with sensor heartbeats and work orders. Predictive maintenance on popular rides. Safety closures never subject to A/B tests. |

## Scale context

| Asset | Now | Pressure |
|:--|:--|:--|
| Visitors | ~5,000 / day | ≥15,000 / day within 3 years |
| Rides | 40 historic 18th-century rides | Popularity, queues, staffing, ongoing safety evidence |
| Animals | 200+ animals, 55 displays/enclosures, aquatic + land, including jumping piranha | Health, feeding, population; care is costly if they get sick |
| Estate | Large, sprawling, patchy Wi-Fi | MQTT hardware budget; cloud allowed if data can leave the estate |
