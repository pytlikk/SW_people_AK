# ADR-021 - Keep keeper observations off the telemetry path, in a device-local append-only log

## Date

2026-09-14

## Status

Proposed

## Context

Keepers work across 55 displays on a sprawling estate with patchy Wi-Fi. Animal houses are the worst case: thick walls, aquaria, and indoor spaces where signal dies. Keeper observation is the **only** data source present at every display - sensor coverage is partial and is not decided here ([animal-care-scope.md](../docs/animal-care-scope.md), section 7).

ADR-020 fixed *what* a record is about: a care subject. This record decides *how a keeper's observation gets out of the field*.

[Appendix A](../requirements/Appendix%20A_%20Core%20functionality.md) 2-6 already requires that field writes are append-only and the cloud reconciles. That is a requirement, not a decision. The decision is which path carries the write, and it turns on a distinction the requirements do not draw:

| | Sensor telemetry | Keeper observation |
|---|---|---|
| Author | Machine | Human |
| Volume | High, continuous | Low, a few per subject per day |
| A lost record is | A gap, flagged as unknown | A welfare failure **and a missing FR#2I training label** |
| Payload | A number | Text, structured feed data, sometimes a photo |
| Recoverable | Next reading arrives in minutes | Never - the animal was seen once, at that moment |

A dropped water-temperature reading is a gap flag ([Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md): gaps must show as unknown, not as zero). A dropped note about a lethargic venomous snake is unrecoverable, and because [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) makes keeper accept/reject the training signal for FR#2I, silent loss does not merely lose data - it **biases the model** toward whatever keepers happened to record in good coverage.

MQTT is the assumed field bus, but broker topology and gateway placement are unowned assumptions in this repository ([ADR-001](ADR-001-use-home-bought-token-pool.md), [ADR-002](ADR-002-use-signed-static-QR-for-attraction-token-presentation.md), scope section 2). Any capture path that depends on those decisions is blocked on an ADR nobody is writing.

This record does **not** decide: which enclosures are instrumented or with what device classes; the anomaly-detection approach (ADR-022); population estimation (ADR-023); field-level schema, which follows in the data contract; or any staff-facing screen, which belongs to the intranet workstream.

**Foreclosed here:** publishing human-authored welfare records on the MQTT telemetry path. Telemetry carries machine readings only.

## Evaluation criteria

- **No lost human observation (driving)** - an acknowledged observation survives multi-hour islanding, app restart, and battery death. Target: 0 acknowledged observations lost (CI).
- **Label integrity for FR#2I (driving)** - every observation and every keeper accept/reject reaches the training store exactly once. Duplicates and silent drops both corrupt the labels.
- **Backbone independence** - capture must not block on the undecided broker and gateway ADRs.
- **Shift battery** - capture works for a full shift with no held connection (NFR_17).
- **Freshness honesty** - keeper and intranet can both see what is unsynced and how old it is (NFR_4, NFR_11).
- **Entry cost** - gloved, outdoors, offline, with the subject resolved per ADR-020 (NFR_17).

## Options

- **Option A - Keeper app is an MQTT client**: observations publish to the same bus as telemetry, QoS 1, buffered on the device.
- **Option B - Device-local append-only log, batch sync (chosen)**: the device is the durable first system of record. Observations are append-only events with client-generated idempotent ids, synced opportunistically as a batch to any reachable estate endpoint, at-least-once.
- **Option C - Fixed terminals at each enclosure**: keepers enter observations at wall-mounted terminals with better connectivity.
- **Option D - Cloud-first with an offline cache**: a standard offline-capable web app queues writes in browser storage.
- **Option E - Paper, transcribed later**: the clipboard status quo.

| | No lost observation (driving) | Label integrity (driving) | Backbone independence | Shift battery | Freshness honesty |
|---|---|---|---|---|---|
| A MQTT client | Partial - buffer is on the device anyway | Partial - still needs idempotent ids | Fail - blocked on the broker ADR | Weak - held connection or reconnect storms | OK |
| B device-local log | Pass | Pass - idempotent ids, at-least-once, dedupe on ingest | Pass - syncs to any endpoint | Pass - no held connection | Pass |
| C fixed terminals | Pass at the terminal | Pass | Partial - wiring to 55 points | Not applicable | OK |
| D cloud-first cache | Fail - browser storage is not a durable system of record | Fail - eviction loses labels silently | Pass | OK | Weak |
| E paper | Pass - paper does not crash | Fail - transcription delay, loss, no reliable timestamps | Pass | Not applicable | Fail |

Not options: synchronous cloud writes at the point of entry, which is the failure mode [1_1_Business challenges.md](../requirements/1_1_Business%20challenges.md) names explicitly; and last-write-wins reconciliation, which would let a late sync silently overwrite a keeper's earlier account of the same animal.

## Decision

**Keeper observations are captured in a device-local append-only log and synced as idempotent batches. They do not travel on the MQTT telemetry path.**

- The device is the first system of record. An observation is durable the moment it is acknowledged to the keeper, with no network.
- Every event carries a client-generated idempotent id, so retry is safe and at-least-once delivery is sufficient - the only delivery guarantee that survives a patchy link.
- Corrections are append-only amendments referencing the original. Nothing is overwritten, consistent with ADR-020's wrong-subject mitigation.
- **Attachments sync separately and later.** An observation and any alert it triggers are never blocked waiting for a photo to cross a thin link.
- MQTT carries sensor telemetry only, and telemetry is joined to a subject through placement-at-time (ADR-020).

No-lost-observation and label integrity decide it. Option D fails both: browser storage can be evicted, and an evicted training label fails silently, which is the worst possible failure for FR#2I. Option A fails backbone independence - it makes our capture path hostage to a broker decision no workstream owns - and buys nothing, because the buffer still sits on the device either way. Option E is honest about durability and hopeless about timestamps and delay. Option C is genuinely sound at the terminal, but wiring 55 points is a cost the estate has not agreed, and keepers will not walk back to a wall to record a refusal they observed three enclosures ago.

## Key differentiators

- **Human-authored records get a durability guarantee telemetry does not need.** The distinction is drawn once, in the architecture, instead of being discovered after the first lost welfare note.
- **Not blocked on an unowned decision.** Capture works whatever the broker topology turns out to be, and can adopt it later if it offers a better guarantee.
- **Retry is safe by construction.** Client-generated ids mean the sync protocol can be dumb and still not corrupt the FR#2I label set.
- **A photo never delays an alert.** Deferring attachments separates evidence-gathering from welfare signalling.
- **The keeper's original account is preserved.** Amendments layer, they do not overwrite - which matters both for training labels and for a newly public poisonous collection where paper logs will not defend a claim.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Availability | **Improved (driving)** | Recording works with zero connectivity. The estate's hardest constraint is met at the point of data entry, not worked around downstream. |
| Reliability | **Improved (driving)** | An observation is durable the moment it is acknowledged to the keeper, not the moment a server hears about it. |
| Data integrity | **Improved** | Client-generated idempotent ids plus dedupe on ingest turn at-least-once delivery into exactly-one training labels. |
| Auditability | **Improved** | Amendments layer over the original, so what the keeper first said remains attributable. |
| Evolvability | **Improved** | Capture does not depend on the unowned broker decision, so this workstream is not blocked by an ADR nobody is writing. |
| Recoverability | **Weakened** | The device is a system of record. Lose it before sync and the unsynced work is gone. No mitigation closes that window. |
| Consistency | **Weakened** | Ordering holds within a device only. Cross-keeper ordering under clock skew is unresolved. |
| Simplicity | **Weakened** | Two paths to build and operate - observation sync and MQTT telemetry - plus a device fleet as an operational concern. |
| Interoperability | **Weakened** | At-least-once imposes idempotency on every downstream consumer, including other workstreams' code. |

**Deliberately downplayed: recoverability of a single device.** The alternative that would protect it is a synchronous cloud write, which fails availability outright in an animal house with no signal. Availability at the point of entry is the driving characteristic, and device loss is the price.

**Fit with the existing architecture.** This is the same trade ADR-002 makes at the checkpoint - do the work locally, reconcile later, never block a human on a network round-trip. The difference is what is being protected: ADR-002 protects admission, this protects a record that cannot be recreated.

## Consequences

### Positive

- Observations survive islanding, restart, and battery death (no lost observation).
- Idempotent ids and dedupe on ingest give exactly-one labels from at-least-once delivery (label integrity).
- The workstream proceeds without the broker ADR (backbone independence).
- No held connection, so the battery lasts a shift (shift battery).
- Unsynced count and oldest-unsynced age are first-class, publishable signals (freshness honesty).

### Negative

- **The keeper device is now a durable system of record, so losing or breaking it loses unsynced work.** Mitigations reduce the window; they do not close it. This is the price of not writing to the cloud synchronously, and it is a real residual risk.
- Device clocks drift. Ordering is reliable **within** a device, not across devices; cross-keeper ordering after skew is TBD.
- At-least-once delivery imposes idempotency on every downstream consumer, including the AI feature pipeline. We are imposing a constraint on other people's code.
- Two paths to build and operate - observation sync plus MQTT telemetry - instead of one.
- An alert can be raised while its photo is still pending, so the intranet must render "evidence pending" rather than "no evidence". More UX work handed to that workstream.
- On-device storage grows during long islanding, and the eviction policy must be written carefully enough that it can never drop an unsynced event.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Device loss with backlog | A phone is dropped in a tank with a shift of unsynced observations | Sync opportunistically at any reachable endpoint, not only at base; prominent unsynced counter; end-of-shift sync prompt. Residual risk accepted and recorded. |
| Clock skew | Device times drift, so cross-device ordering is wrong | Per-device monotonic sequence plus endpoint receipt stamp; order within a device by sequence; cross-device rule TBD |
| Duplicate on retry | The same observation syncs twice and becomes two training labels | Client-generated idempotent event id; dedupe on ingest; CI replay test |
| Silent eviction | Storage pressure drops unsynced events | Unsynced events are never evictable; the app blocks new entry before discarding one - fail loud, not quiet |
| Path confusion | An observation is published to MQTT by a well-meaning implementer | Stated as foreclosed; CI asserts no MQTT client in the observation write path |
| Adoption | Entry is slow, so keepers revert to paper and the labels never arrive | Subject auto-resolves where an enclosure holds one (ADR-020); gloved-use targets (NFR_17); adoption is measured, not assumed |
| Attachment flood | Photos saturate a thin link and starve observation sync | Observations sync first, attachments deferred; size cap TBD |

## Verification

**Tests (CI)**

- An observation recorded with no network survives app restart and battery death, then syncs; 0 acknowledged observations lost.
- Replaying a sync batch twice yields exactly one record and one training label per event.
- An amendment never overwrites the original; both remain retrievable, with the original attributable.
- Attachment upload failure blocks neither the observation nor an alert derived from it.
- The observation write path contains no MQTT client and no synchronous call to a cloud endpoint.
- Unsynced events cannot be evicted by storage pressure; the app refuses new entry first.

**Ops check**

- A keeper completes a feed entry with gloves, outdoors, offline, in an enclosure holding more than one subject (extends the ADR-020 ops check).
- Unsynced count and oldest-unsynced age are visible to the keeper and published to the intranet.

**Open questions**

- Cross-device event ordering under clock skew - before beta.
- On-device retention and eviction policy for already-synced events - before beta.
- Attachment size cap and whether attachments sync on any link or only a strong one - before device procurement.
- Device class: personal phone or issued rugged device. This changes the loss risk materially - before rollout.
- Whether fixed terminals should complement the app at high-traffic enclosures - after first season.

**Revisit triggers**

- Oldest-unsynced age routinely exceeds one shift -> revisit gateway placement, or add fixed terminals as a complement (Option C).
- Device loss with unrecovered data exceeds an ops-agreed threshold -> revisit issued-device policy.
- A backbone ADR lands offering a durable store-and-forward guarantee for human-authored records -> revisit whether the observation path can merge onto it.

## Conclusion

Keeper observations are unrecoverable and are the FR#2I training labels, so they get a durability guarantee that sensor telemetry neither needs nor provides: a device-local append-only log, idempotent ids, at-least-once batch sync, deferred attachments, and no dependence on the MQTT broker decision nobody owns.

Related: ADR-020 (what an observation is about), ADR-022 (what the labels train), ADR-023 (census entry follows this same path). Field-level schema follows in the data contract; boundaries in [animal-care-scope.md](../docs/animal-care-scope.md).
