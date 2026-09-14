# First Pass - Board P: Popularity Meter (reconstructed)

> **Reconstructed first pass from ADR-002, FR#2D, Appendix B. Not a photographed workshop.**
> Next-ADR depth: this board feeds the proposed popularity-meter ADR (third in the ADR
> plan per adrs/README.md). The goal is to name what events count, what gaps look like,
> and what "a usable count" means without pretending these are decided.
>
> Do not name a pricing model on this board. Do not name the intranet heat-map UI.
> Both are consumers of this feed; this board owns the feed.
>
> Sticky type in [BRACKETS]. Duplicates preserved.

---

## Session dump

---

**What generates a signal worth counting?**

Reading ADR-002: `validated` MQTT events are published by VA-03 after each checkpoint
admit. Each event carries attractionId and timestamp. This is the primary signal.

[EVENT] `validated` MQTT event received (from VA-03 Checkpoint Verifier)
  - attractionId + timestamp = minimum for a count. FR#2D: entries, throughput.
  - Same event type for 40 rides and up to 55 enclosures.

[EXTERNAL] Checkpoint Verifier (VA-03, Board V) - source of validated events

[EVENT] Ride cycle completed (MQTT signal from ride hardware - ride has turned over)
  - FR#2D says "ride cycle counts" are an input.
  - This is a separate MQTT signal, not from the gate camera.
  [QUESTION] Where exactly does the ride-cycle sensor publish? To the same topic as
    validated events, or a different one? Not decided. Broker placement TBD.

[EXTERNAL] MQTT ride cycle sensor (hardware; count and placement TBD)

[EVENT] Zone occupancy reading received (optional MQTT people-counter)
  - FR#2D: "MQTT occupancy" is listed. Appendix B: "Fuse MQTT counts, ride cycles,
    and ticket scans into heat maps and ranks."
  [QUESTION] Are there MQTT zone counters at all zones / displays? Device placement
    is TBD. Not all 55 displays necessarily have a sensor. Add as question.

[EXTERNAL] MQTT zone occupancy device (optional; placement TBD; budget exists per Assumptions)

[EXTERNAL] MQTT Broker (topology TBD; working assumption shared with Board V)

[QUESTION] Zone occupancy device placement and count? TBD.

---

**Gap detection: what happens when the signal is absent?**

[QUESTION] What if a device goes silent?
  - FR#2D: "Degrade gracefully when MQTT is gapped."
  - Appendix B: "Gaps in MQTT must show as unknown, not as zero."
  CORRECTION: zero is the dangerous value. An absent device should NOT produce a
  count of zero - that looks like an empty exhibit. It must produce an "unknown" flag.
  This is a first-class system requirement, not a nice-to-have.

[EVENT] MQTT gap detected (device silent beyond threshold)

[EVENT] Attraction count flagged as unknown (NOT zero)

[DUPLICATE] "MQTT gap detected" and "attraction count flagged as unknown" - these
  are the same event from different angles. Keep both in first pass; merge in digitised.

[QUESTION] Gap threshold duration: how long of silence before the bucket becomes "unknown"?
  Not in the brief. Do NOT invent a number. Leave TBD.

[QUESTION] Who is responsible for gap detection - the MQTT broker, the aggregator, or
  a dedicated monitor? The question is whether gap detection belongs in the main
  aggregation service or in a separate watchdog. Leave as open design question.

---

**Aggregation and counting**

[EVENT] Attraction count incremented (per `validated` MQTT event received)

[EVENT] Ride cycle counted (per MQTT ride cycle event)

[EVENT] Dwell estimate updated
  [QUESTION] How is dwell estimated? Three approaches came up:
    (a) checkpoint entry + exit pair (needs exit event - is there one?);
    (b) zone occupancy time window (requires zone sensors);
    (c) approximate from throughput rate (no exit event needed but imprecise).
    None of these is decided. Leave as open design question.

[DUPLICATE] "Attraction count incremented" appears both under validated events and
  under the aggregation step. CORRECTION: "incremented" is the aggregation action;
  "validated MQTT event received" is the input. Both in digitised board; merged here.

[EVENT] Popularity rank recalculated
  [QUESTION] Rank recalculation: event-driven (on every count update) or periodic batch?
    The intranet heat map (CC-06) needs freshness; exact lag is TBD.

---

**Publication: who consumes the feed?**

[EVENT] Popularity snapshot published

[EXTERNAL] Estate Heatmap / Intranet (CC-06 from ops-backup) - primary consumer
  NOTE: CC-06 projects this feed; it does NOT own the counts. The popularity meter
  (PM-01 on this board) owns the feed. Intranet is a consumer.

[EXTERNAL] Dynamic Pricing (later ADR) - table writer that reads popularity signal
  NOTE: dynamic pricing is a "table writer on ADR-001" per adrs/README.md. It reads
  this feed. It is not on this board.

[EXTERNAL] Congestion Forecaster (CC-11 from ops-backup) - consumes popularity + load

[EXTERNAL] Board G AI Guide (AG-01) - consumes wait-time proxy from this feed

[EXTERNAL / POLICY] Privacy constraint - occupancy counts are anonymous aggregate.
  Individual dwell never infers identity. NFR_8: "occupancy and popularity prefer
  counts and opted-in trails over biometric identification."

[QUESTION] Snapshot frequency: real-time stream, per-minute batch, or hourly?
  The intranet heat map needs freshness (NFR_11); exact lag TBD.

---

## Component candidates emerging

**PM-01 Popularity Aggregator** (= CC-12 from ops-backup, now owned here)
  Fuses `validated` MQTT events (per attractionId), ride cycle counts, and optional
  zone occupancy readings into ranked counts + dwell estimates. Gaps show as unknown,
  not zero. Publishes the popularity feed consumed by the intranet, pricing, and guide.
  NOTE: CC-12 was described in ops-backup as "ticketing-side." This board claims
  ownership. CC-06 (intranet heatmap) is a consumer, not the source.

**PM-02 Gap Monitor**
  Watches MQTT device heartbeats per attractionId. Flags the count bucket as unknown
  when device silence exceeds a threshold (threshold TBD). Prevents zero-count misread
  of a silent device. May be a module within PM-01 or a separate watchdog.

---

## Boundary decisions from this board

1. PM-01 owns the popularity feed. CC-06 (intranet heatmap) is a consumer.
2. Gap bucket is explicit: silent device = unknown, NOT zero (Appendix B, FR#2D).
3. Privacy: counts are anonymous aggregate. No individual dwell identity (NFR_8).
4. Dynamic pricing is a TABLE WRITER on this feed; it is not on this board.
5. PM-01 = CC-12 from ops-backup; no functional change, ownership made explicit.

---

## Corrections from this pass

1. Initial "dwell" sticky placed gap-flagging in the aggregation step. Moved to
   a dedicated Gap Detection phase after re-reading Appendix B.
2. Zero-count from a silent device identified as an active failure mode, not a corner
   case. Flagging as unknown is a requirement, not a nice-to-have.
3. Privacy constraint added after "dwell" raised the question of tracking individuals.

---

*Digitised board: [board-popularity.svg](board-popularity.svg)*
