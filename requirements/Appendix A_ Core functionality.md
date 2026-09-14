# Core Guest, Operations (Intranet) & Commercial Features

> Note: Guest surfaces stay small and reliable (especially at the gate). Operations is the estate operating system - not a document portal. Commercial features exist to grow visitation and profit without putting models on the access path.

## 1. Core Guest Features

- **1-1  Discover and buy tickets**
  - Individual admission and **family passes** (kata must-have).
  - Channels: web and on-site kiosk at minimum; mobile optional.
  - Show price from the published list (including active experiment variant), not from a live model call.
  - Payment via a PCI-scoped provider; confirmation yields an entitlement (QR/barcode or equivalent) the gate can verify offline.

- **1-2  Entitlements and family passes**
  - SKU ≠ access rights. A family pass encodes party rules (e.g. adult/child counts), validity window, and any included timed experiences.
  - Concurrent entry rules are enforced at the gate (N of M in the party).
  - Optional: timed slots for animal talks / piranha feeding as entitlements, not as hope.

- **1-3  Gate access**
  - Redeem entitlements at **estate perimeter gates**. Zone gates (e.g. venomous house as a timed entitlement) are a later phase - see [Appendix C](Appendix%20C_%20Future%20scope.md).
  - v1 scope: rides and enclosures are **counted, not gated**. A ride or display scan (where fitted) feeds popularity (FR#2D); it does not admit or charge. Admission is bought once at the perimeter, not per attraction.
  - **Must work when cloud/Wi-Fi is down**, from a local snapshot of unspent entitlements and revocation list.
  - Offline redemptions enqueue for cloud reconciliation (no double-spend within the local gate cluster; residual cross-gate risk documented in [ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)).
  - Clear fail states: already used, expired, wrong day, party count exceeded, ride/estate closed.

- **1-4  Optional identity / membership**
  - First visit can be anonymous (ticket only).
  - Incentivize email/QR membership so return visits, win-back, and cohort analysis have a key.
  - Consent separate from purchase: analytics, marketing, optional on-site trail.

- **1-5  On-estate guest aids (non-blocking)**
  - Published “start here” / wait-time boards that degrade to static content.
  - Opt-in itinerary or checkpoint QR trail for guests who want it (feeds FR#2G).
  - Never required to complete a visit.

- **1-6  Data privacy & rights**
  - Access, rectify, erase, port for identified guests.
  - Occupancy counts are not identity.
  - Camera clips purpose-limited (see NFRs).

## 2. Operations Features (Intranet)

The intranet is the **estate operating system**: one place for staff to see occupancy, queues, animal alerts, ride status, tickets-in-play, and incidents. It is not the system of record for payments or the MQTT firehose.

- **2-1  Duty-manager view**
  - Live (or last-known) heat map of the estate with data-age shown.
  - Open incidents, closed rides, animal alerts, gate throughput.

- **2-2  Ride operations**
  - Status: open / closed / delayed / evacuated per ride.
  - Cycle count vs capacity, current queue proxy if instrumented.
  - Dispatch and guest-comms hooks when a popular ride goes down.

- **2-3  Asset & maintenance**
  - Asset identity: ride/enclosure/building id, zone, criticality.
  - MQTT heartbeat where fitted: vibration, motor current, gate sensors, e-stop.
  - Inspections: who, when, checklist, next due.
  - Work orders: fault, severity, parts, time-to-repair.
  - Incidents: guest injury, near miss, animal escape (cross-link to animal module).

- **2-4  Animal care**
  - 55 displays: species/collection, enclosure environment (water params, HVAC, barriers).
  - Health: keeper observations + sensor context.
  - Feeding: amount, refusal, leftover, aggression (“how much / how well”).
  - Population: especially jumping piranha (counts, breeding, losses).
  - Alert inbox: AI suggestions with accept/reject (FR#2I, FR#2J).

- **2-5  Staffing & tasking**
  - Roster vs predicted load (FR#2F).
  - Assign tasks (inspect ride, move queue staff, extra keeper round).
  - Keeper/ride-crew mobile entry with offline sync.

- **2-6  Integration bus (logical)**
  - Consume ticket events, MQTT telemetry, keeper apps, maintenance, AI outputs.
  - Publish operational state for UIs; analytical state belongs in the warehouse.
  - Conflict rule: field writes are append-only events; cloud reconciles.

- **2-7  Support & incidents**
  - Guest-facing incident log (lost child, medical, animal-area breach) with time, zone, actions.
  - Escalation to duty manager; safety flows remain scripted.

## 3. Commercial & Growth Features

- **3-1  Price list & experiment snapshots**
  - Commercial publishes (or approves) price bands and active experiments.
  - POS/web/kiosk pull snapshots on a schedule and keep last-known-good.
  - Assignment key sticky for the visit (party / device / membership).

- **3-2  Packages and family-pass configuration**
  - Configure family-pass shapes (e.g. 2+2 vs 2+3, weekday vs weekend) as data, so A/B tests (FR#2B) do not require a release.

- **3-3  Popularity & yield reporting**
  - Which parts of the estate are used, by daypart and ticket mix.
  - Yield per visitor, ancillary attach if collected (F&B, photos, tours).
  - Countess / exec view: profit, repeat visit, “where to invest” (FR#2F).

- **3-4  Returning-visitor loop**
  - Membership file, offer history, 90-day return metric.
  - Win-back campaigns (FR#2H) honour caps and opt-out.

## 4. Data the core platform must collect

Enough to answer the kata without waiting for AI:

| Domain | Minimum fields |
|:--|:--|
| Ticket | SKU, channel, timestamp, price charged, experiment id + variant, party size / family structure (declared), entitlement id |
| Redemption | Gate id, timestamp, success/fail reason, offline? |
| Occupancy | Zone/ride/enclosure id, count or event, ts, device id, gap flag |
| Ride asset | Status, cycle count, inspection due, work orders, incidents |
| Animal | Display id, health notes, feed events, environment readings, piranha census |
| Staff | Task assigned/done, zone, ts |
| Experiment | Assignment key, variant, outcome metrics (conversion, yield, 90-day return) |
