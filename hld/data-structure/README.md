# Data structures and sourcing

Sketches, not schemas. Enough to show the contracts hold together and that the estate can answer its own questions before any model exists. Field-level schemas are a follow-up per [ADR-0002](../../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) and [ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md).

Source of truth for required fields is [Appendix A section 4](../../requirements/Appendix%20A_%20Core%20functionality.md).

## Three properties every field-sourced record carries

This is the part worth reading. It is what makes gap-as-unknown enforceable rather than aspirational.

| Property | Why |
|:--|:--|
| **Provenance** - device id, gateway id, sequence number | Dedupe on `(device id, sequence)` after a buffer replay ([ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)) |
| **Two timestamps** - event time and ingest time | A reconnecting gateway delivers yesterday today; aggregates must be restatable by event time |
| **Gap flag** - or an explicit gap marker record | The consumer must be able to tell silence from zero |

A record without provenance cannot be deduplicated, and a series without gap markers cannot be honest. Both are structural, not conventions.

## Entitlement claim

The signed payload a guest presents. The only contract the gate has with the cloud.

```
entitlementId    opaque id
scope            "estate"          # zone gating later, same payload type
partyShape       { adult: 2, child: 2 }
validFrom        timestamp
validTo          timestamp
experimentRef    variant id or null  # sticky offline assignment (ADR-0011)
signature        estate signing key
```

Deliberately absent: guest name, price paid, SKU. The gate needs access rights, not commercial or personal data - a family pass is an access right, and the less the claim carries, the less a photographed QR leaks (NFR_8).

The party ledger is **not** in the claim. It lives on the gate cluster, because a claim the guest holds cannot be trusted to count itself down.

## Ticket and redemption

| Record | Fields | Notes |
|:--|:--|:--|
| Ticket | SKU, channel, timestamp, price charged, experiment id + variant, declared party structure, entitlement id | Declared party structure is CRM truth; nothing inferred is written here ([ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)) |
| Redemption | Gate id, timestamp, success or fail reason, offline flag, party count admitted | The offline flag is what makes reconciliation exceptions explainable |

`price charged` is recorded rather than derived. The price list changes; the historical fact must not, or yield analysis silently rewrites the past.

## Occupancy and popularity

```
zoneOrAssetId    ride, zone, or display id
count            integer, or null when unknown
eventTime        when it happened at the sensor
ingestTime       when the cloud saw it
deviceId         provenance
gatewayId        provenance
sequence         dedupe key
gapFlag          boolean
```

`count` is nullable and that is the design. A silent counter produces `null` plus a gap marker, never `0`. Every aggregate over a window containing a gap reports unknown coverage alongside its value ([ADR-0013](../../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)).

## Ride asset

| Record | Fields |
|:--|:--|
| Asset | Ride id, zone, criticality, heritage constraints |
| Status | Open / closed / delayed / evacuated, set by, timestamp - **human-authored** |
| Cycle count | Asset id, count, window, provenance |
| Inspection | Who, when, checklist result, next due |
| Work order | Fault, severity, parts, time to repair, originating alert if any |
| Incident | Type, zone, timestamp, actions, escalation |

`originating alert` links a work order back to the AI suggestion that prompted it, if one did. That link is how precision gets measured against real outcomes rather than against labels we made up.

## Animal care

Subject is the enclosure or colony, not always the individual ([ADR-0020](../../adrs/ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md)).

| Record | Fields | Notes |
|:--|:--|:--|
| Care subject | Subject id, display id, species or collection, subject kind (individual / enclosure group / colony), cardinality certainty | Certainty is a property of the subject, not a footnote |
| Environment reading | Subject id, parameter (water quality, temperature), value, provenance, gap flag | Sensor path, same contract as occupancy |
| Feed event | Subject id, offered amount, leftover amount, refusal, aggression observed, keeper id, event time | Offered **and** leftover - a fixed-offer protocol is what makes per-capita intake observable ([ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)) |
| Observation | Subject id, keeper id, structured flags + free text, event time, entered-offline flag | Keeper judgement is deterministic input, not a model input to be smoothed |
| Census | Subject id, counted value, method, keeper id, timestamp | **Ground truth.** Resets the population anchor |
| Mortality | Subject id, confirmed, keeper id, timestamp | Decrements deterministically by exactly one |
| Health alert | Subject id, model version, confidence, evidence refs, keeper decision, reason code | Decision and reason are the training signal (FR#2I) |

Two fields here are load-bearing and easy to lose:

**`offered` and `leftover` separately.** Feeding to appetite makes per-capita intake unobservable, which collapses the entire between-census population signal. The data model forces the distinction so the operational protocol cannot quietly drift back.

**`method` on census.** Netting and visual estimate are not the same ground truth, and pretending they are would corrupt the interval calibration that [ADR-0023](../../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) is built on.

## Population estimate output

```
subjectId
estimate         point value
intervalLow      required
intervalHigh     required
anchorDate       last census
anchorAgeDays
usable           false past maximum anchor age
trend            rising / stable / falling / unknown
flags            breeding suspected, loss suspected (keeper confirmation pending)
```

`intervalLow` and `intervalHigh` are not optional, and `usable` can be false. CI rejects a point estimate published without an interval. This is the one output in the estate that is allowed to refuse to answer.

## Experiment

| Record | Fields |
|:--|:--|
| Assignment | Assignment key (party / device / membership), experiment id, variant, assigned at, snapshot version |
| Outcome | Assignment key, conversion, yield, 90-day return flag |

Assignment is captured at the moment of assignment, including the snapshot version, so an analysis can reconstruct what the guest was actually shown rather than what the current configuration says they should have been ([ADR-0011](../../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)).

## Retention

Per NFR_9, enforced as warehouse lifecycle policy rather than as a convention.

| Class | Retention |
|:--|:--|
| Financial / ticket | 7 years (plan for tax rules) |
| Occupancy / telemetry | Hot 30 days, warm 1 year, cold 3 years |
| Guest photos / camera clips | Shortest of purpose or 90 days, unless incident hold |
| Keeper and veterinary records | Life of the animal plus a defined legal period |
| Experiment assignments | 90 days plus audit - long enough to evaluate return visits |

## Data sourcing

| Source | Mode | Path |
|:--|:--|:--|
| Web and kiosk purchases | Transactional | Ticketing service to warehouse |
| Gate redemptions | Near-real-time, buffered | Lane to gateway to Pub/Sub |
| Occupancy and ride heartbeats | Streaming, lossy-tolerant | MQTT to gateway to Pub/Sub |
| Enclosure environment | Streaming, welfare class | MQTT to gateway to Pub/Sub, never shed |
| Keeper entries | Append-only, offline-first | Handheld to gateway to Pub/Sub |
| Weather and event calendar | Batch pull | External API to warehouse |
| Census and mortality | Manual, ground truth | Keeper app, deterministic |

Related: [core functionality](../core-func/README.md), [ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md), [ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md).
