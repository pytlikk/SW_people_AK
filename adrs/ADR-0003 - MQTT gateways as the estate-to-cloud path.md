# ADR-0003 - Zone MQTT gateways with store-and-forward, and a gap is published as unknown

## Date

2026-09-14

## Status

Proposed

## Context

The kata gives us three facts that together define this record: Wi-Fi coverage is patchy, cloud services are allowed *provided* there is a way of getting information from the estate to the cloud, and there is budget for MQTT-capable hardware throughout the park.

So the transport is settled - MQTT - and the real decision is the **topology and the semantics of missing data**.

What has to travel: zone and ride occupancy counts, ride heartbeats (vibration, motor current, gate sensors, e-stop), enclosure environment readings (water quality, temperature), feeder events, keeper observations entered on handhelds, and gate redemption events from [ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md). On the order of 500-1,000 devices across 40 rides and 55 displays (NFR_1).

The part that makes this interesting is not the happy path. It is that **an animal house is exactly where the Wi-Fi dies** (NFR_17), and that is also where the most consequential data is produced. A design that loses keeper observations during an outage fails the animal-welfare goal outright, and a popularity heat map that renders a disconnected zone as empty will send staff to the wrong place - the precise failure the Countess is paying to avoid.

This record decides how estate data reaches the cloud and what consumers see during an outage. It does not decide which sensors are procured, the warehouse schema, or how popularity is computed from these counts ([ADR-0013](ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)).

**Foreclosed here:** devices calling the cloud directly, and any consumer interpreting absent telemetry as a zero.

## Evaluation criteria

- **Survives multi-hour islanding without losing health or access events (driving)** - NFR_4 states this as a requirement. Popularity counts may be lossy; welfare and access events may not.
- **A gap is distinguishable from a zero (driving)** - the consumer contract must make silence visible. This is what protects every downstream AI capability from confidently wrong input.
- **Operability for a three-person team** - no per-device cloud identity to manage by hand, no broker cluster to babysit.
- **Cost (NFR_12)** - hardware is capital budget; cloud ingest and hot storage are not unbounded.
- **Ordering and duplication tolerance** - replayed buffers will arrive late and more than once.

## Options

- **Option A - Devices connect directly to a cloud MQTT endpoint**: each sensor holds cloud credentials and publishes upstream itself.
- **Option B - One estate-wide broker**: all devices publish to a single on-site broker that forwards to the cloud.
- **Option C - Zone gateways with store-and-forward, bridging to cloud ingest (chosen)**: each zone runs a local broker with disk-backed persistence; zone gateways bridge to Pub/Sub when a link is available; local consumers subscribe locally.
- **Option D - Batch file sync**: devices write to local storage, a courier process uploads periodically.

| | Multi-hour islanding (driving) | Gap vs zero (driving) | Operability | Cost | Dup/ordering |
|---|---|---|---|---|---|
| A Direct to cloud | Fail - a device buffer is small and a patchy radio means constant reconnects | Fail - absence is indistinguishable from a dead device | Fail - 1,000 cloud device identities to provision | Partial | Fail |
| B Single estate broker | Partial - survives a WAN outage but not a WLAN island, and the broker is one fault away from total blindness | Partial - possible but the broker becomes the only place to detect it | Partial | Low | Partial |
| C Zone gateways | Pass - disk-backed buffer per zone; a dead link islands one zone | Pass - the gateway knows its own last-seen per device and emits heartbeats | Pass - tens of gateways carry cloud identity, not thousands of sensors | Pass | Pass - gateway assigns sequence and dedupe keys |
| D Batch file sync | Pass for durability | Partial - but latency breaks the 60-second welfare alert target (NFR_3) | Partial | Low | Partial |

Not options: a proprietary always-on IoT suite with guaranteed bandwidth (the constraints name MQTT-capable hardware as the field bus); cellular per device (recurring cost per sensor against a capital hardware budget, and it re-creates Option A's identity problem).

## Decision

**Every zone runs a gateway that is both a local MQTT broker and a store-and-forward bridge to cloud ingest. The gateway is the unit of islanding, and it publishes its own silence.**

```
device --MQTT--> zone gateway --(disk buffer)--> bridge --> Pub/Sub --> Dataflow --> BigQuery
                      |                                                    |
                      +--> local subscribers (gate lane, keeper app, edge dashboard)
```

Four rules follow.

**Events are classified, and the class decides the guarantee.**

| Class | Examples | QoS | Buffer policy |
|---|---|---|---|
| Access | gate `redeemed`, revocation ack | 1, persisted | Never dropped; oldest-first drain |
| Welfare | feed events, water quality, keeper observations, e-stop, escape alarm | 1, persisted | Never dropped; oldest-first drain |
| Popularity | zone counters, ride cycle counts | 1, aggregated | Downsampled under buffer pressure, with the downsampling itself recorded |

Under sustained buffer pressure the gateway degrades **popularity** fidelity to protect access and welfare. It does not choose silently: the degradation is an event.

**Every gateway emits a heartbeat, and every published reading carries provenance.** A reading carries the gateway id, the device id, the event time, the ingest time, and a sequence number. A consumer can therefore always answer "when did I last hear from this device" - which is what makes the next rule enforceable.

**A gap is published as `unknown`, never as zero.** When a device or gateway misses its expected interval, the pipeline emits an explicit gap marker for that window. Occupancy for a silent zone is unknown occupancy. This is the contract [ADR-0013](ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) and [ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) both depend on, and it is the single most load-bearing sentence in this record.

**Consumers are idempotent, and late data is normal.** Dedupe is on `(device id, sequence)`. A reconnecting gateway replays hours of events out of order relative to live traffic, so every downstream aggregate must be recomputable for a window that has already been reported. Dead-letter with replay for anything that fails validation.

Islanding survival and gap semantics decided it. Option A fails both: small device buffers and no way to tell a quiet zone from a dead sensor. Option B is the tempting middle - one broker is less to run - and it loses because a WLAN island between a sensor and that broker is the common case here, not the rare one, and because a single broker makes both popularity and animal telemetry fail together (a named risk in [5_Risks and mitigation](../requirements/5_Risks%20and%20mitigation.md)). Option D is durable but too slow for a 60-second welfare alert.

## Key differentiators

- **The blast radius of a dead link is one zone.** With a single broker it is the estate.
- **Silence is a first-class signal.** The system reports what it does not know, which is what lets every AI capability downstream refuse to guess.
- **Access and welfare events outrank popularity under pressure,** and the trade is explicit rather than emergent.
- **Keeper handhelds and gate lanes have a local subscriber to talk to** even when the estate is cut off, so the animal house - the worst-connected building - still works.
- **Cloud device identity is held by tens of gateways, not thousands of sensors,** which is the difference between a provisioning script and a provisioning project.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Fault tolerance | **Improved (driving)** | Islanding is contained per zone; disk-backed buffers survive multi-hour outages (NFR_4). |
| Data integrity | **Improved (driving)** | Provenance and gap markers mean no consumer can silently mistake absence for zero. |
| Observability | **Improved** | Gateway lag, buffer depth, drop, and replay are explicit metrics (NFR_11). |
| Scalability | **Improved** | New device classes and new zones onboard without touching guest checkout (NFR_10). |
| Cost efficiency | **Improved** | Aggregation at the edge keeps the hot store from carrying raw counter traffic (NFR_12). |
| Operational complexity | **Weakened (deliberate)** | Tens of gateways are physical estate assets that need power, mounting, time sync, firmware, and someone to visit them. |
| Latency | **Weakened** | An extra hop, and replayed data can arrive hours late, so every aggregate must be restatable. |
| Consistency | **Weakened** | The warehouse is eventually consistent with the field by design; "today's popularity" is provisional until buffers drain. |

**Deliberately downplayed: operational simplicity.** A single broker would be one thing to run instead of tens. We spend that simplicity because the estate's failure mode is a *local* radio hole, not a WAN outage - and a topology whose availability depends on the worst Wi-Fi patch in the park is not a topology, it is a hope. The offsetting cost is real and lands on whoever maintains estate hardware.

**Fit with the existing architecture.** This is the same reconciliation model as the gate: decide and record locally, reconcile in the cloud afterwards, and never let a remote dependency into a path that must keep working ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)). Field writes are append-only events; the cloud merges them (Appendix A 2-6). The gap-as-unknown rule is the data-layer expression of the same honesty the AI records enforce at the output layer.

## Consequences

### Positive

- Multi-hour islanding of any zone loses no access or welfare event (driving criterion).
- Downstream capabilities can distinguish a quiet zone from a broken sensor (driving criterion).
- The keeper app and gate lane work against a local broker in the worst-connected buildings.
- Backfill is bounded and replayable, so an outage produces a cost spike rather than a data hole.

### Negative

- **Tens of gateways become estate infrastructure with a maintenance rota** - power, weatherproofing, time sync, firmware. This is a facilities commitment the Countess is not currently staffed for, and it is a direct consequence of this choice.
- **Every downstream aggregate must be restatable,** because a gateway can deliver yesterday afternoon tomorrow morning. Reports are provisional, and "the number changed since I looked" will need explaining to the Countess.
- Popularity fidelity degrades first under buffer pressure, so the busiest day with the worst connectivity is also the day the popularity data is weakest - exactly when it is most wanted.
- Gap markers make dashboards honest and uglier; a heat map with unknown patches invites "why don't you know?" questions that a fake zero would have hidden.
- Time sync becomes safety-adjacent: skewed gateway clocks corrupt both dwell measurement and the audit trail.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Buffer overflow | Islanding outlasts disk capacity | Class-based shedding (popularity first, never welfare or access); buffer-depth alarm well before the ceiling; documented hours-of-islanding SLO |
| Silent drop | A gateway discards under pressure without saying so | Shedding emits an event; drop counters exported; CI test that a shed decision is observable |
| Gap treated as zero | A consumer averages over unknown windows | Gap markers are part of the published contract; consumer tests assert unknown propagates; forecast pauses on excessive gaps ([ADR-0013](ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)) |
| Duplicate events | Replay after reconnect | Dedupe on `(device id, sequence)`; idempotent consumers; replay is a normal operation, not an incident |
| Clock skew | Dwell and audit corrupted | NTP at the gateway; devices timestamped by the gateway when they cannot be trusted; skew beyond tolerance alarms |
| Rogue or tampered device | Physical access to estate hardware | Per-device credentials at the zone broker; unknown device ids quarantined, not silently accepted; tamper-evident enclosures |
| Gateway as a single point per zone | One dead gateway blinds a zone | Accepted; the zone reports unknown rather than zero, and the duty-manager view shows it; dual gateways only for zones with welfare-critical sensing |
| Cost of hot telemetry | Raw counters in an expensive store | Edge aggregation; tiering hot 30d / warm 1y / cold 3y (NFR_9) |

## Verification

**Primary metrics**

- Hours of islanding survived per zone without loss of a welfare or access event.
- Gateway lag p95 and buffer depth, against the ≤30 second ingest target under healthy links (NFR_3).
- Share of ops screens showing data age - target 100% (OKR 8.2).
- Backfill drain time after a simulated outage, and duplicate rate after replay. Modelled at 17 minutes for one zone and 83 minutes estate-wide after a 72-hour island, inside the 4-hour RTO in `NFR_2` ([hld/sizing](../hld/sizing.md#5-backfill-drain)). The "cost spike rather than a data hole" claim above prices out at **$1.41**.

**Tests (CI)**

- A welfare event published during a simulated outage is delivered exactly once after reconnect.
- Under forced buffer pressure, popularity is shed and welfare and access are not - and the shedding is itself observable.
- A device that misses its interval produces a gap marker, and a consumer aggregate over that window reports unknown rather than zero.
- A replayed batch produces no duplicate rows and no double-counted occupancy.
- An unknown device id is quarantined rather than ingested.
- A malformed payload lands in the dead-letter queue and can be replayed after a schema fix.

**Ops check**

- Physically pull a zone gateway's uplink during a busy period and confirm the duty-manager view shows unknown for that zone, not empty.
- Confirm keeper data entry in an animal house works with the uplink down and syncs afterwards.

**Open questions**

- ~~Hours-of-islanding SLO and therefore disk sizing per gateway - before hardware procurement.~~ **Resolved** in [hld/sizing](../hld/sizing.md#4-gateway-disk-buffer-and-the-islanding-slo): **72-hour SLO, 8 GiB usable buffer per gateway.** The busiest zone fills 7.03 MiB/hour, so 72 hours needs 506 MiB and 8 GiB buys 48 days. The consequence is that class-based shedding will essentially never fire at this sizing - it is insurance against a far denser sensing plan, not a live control.
- Zone boundaries: how many gateways, and which of the 55 displays share one - before installation, and it depends on the site survey rather than on software.
- Which zones carry welfare-critical sensing and therefore justify a second gateway - with the vet and ops, before installation.
- Downsampling ratios for popularity under pressure - before the first peak season.
- Whether keeper handhelds speak MQTT directly or via a local HTTP shim on the gateway - before the keeper app is built.

**Revisit triggers**

- A zone's islanding regularly exceeds its buffer - re-survey the radio path or add a cellular uplink for that zone specifically.
- Gap markers become so common that the heat map is mostly unknown - the sensing plan, not the transport, is wrong.
- Device count grows past a few thousand and gateway provisioning becomes the dominant cost - re-evaluate a managed device-management product ([ADR-0001](ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) revisit trigger).

## Conclusion

Zone gateways are the unit of islanding: a local broker with a disk-backed buffer, a bridge to cloud ingest, and local subscribers for gates and keeper apps. Events are classed so access and welfare are never shed, and every gap is published as unknown rather than zero. The cost is tens of physical gateways to maintain and aggregates that must be restatable when a buffer drains late - both accepted because the estate's characteristic failure is a local radio hole, and a single broker would turn that into estate-wide blindness.

Related: [ADR-0001](ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) (Pub/Sub and the warehouse), [ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) (redemption events as audit, not admit), [ADR-0013](ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) and [ADR-0021](ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md) (the two consumers that depend hardest on gap semantics).
