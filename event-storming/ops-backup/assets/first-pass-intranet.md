# First Pass - Board B: Intranet / Estate OS (Duty Manager, Ride Crew)

> **Reconstructed first pass from `requirements/`, not a photographed workshop.**
> There was no physical sticky-note session. This document simulates the messy
> first pass that would emerge from a facilitated event-storming session run
> against the committed requirements. Unordered dump; duplicates expected;
> corrections and questions written in-line as they arose.
>
> Scope: **ops domain only (intranet / estate OS).** The intranet is the estate
> operating system - one place for staff to see occupancy, alerts, ride status,
> tickets-in-play, and incidents. It is NOT the system of record for payments or
> the MQTT firehose (Appendix A 2 intro). Ticketing (ADR-001, ADR-002) appears
> as external upstream. The visit-access storm is a later session.
>
> Sticky type in [BRACKETS]. Order is approximate.

---

## Actors

[ACTOR] Duty Manager - inferred role for "Staff (deploy / invest)"; brief does not use the title "duty manager" but Appendix A 2-1 ("Duty-manager view") does. Label as **inferred role**.
[ACTOR] Ride Crew - inferred role for "Staff (rides)"; Appendix A 2-2 is the source. Label as **inferred**.
[ACTOR] Countess - view-level access; makes invest decisions; does NOT dispatch field commands directly from the intranet.

---

## External inputs (phase 1)

[EXTERNAL] Ticketing / Token System (ADR-001 / ADR-002) - upstream only. The intranet consumes redemption events (gate throughput, occupancy from enclosure / ride scans). It does NOT write back into ticketing.

[CMD] Consume ticket redemption event (this is more of a subscription / integration concern; noted here for domain boundary clarity)

[EVENT] Gate throughput measured (derived from ticket redemption events from ADR-002)

[EXTERNAL] MQTT hardware (budget exists per Assumptions; placement TBD per ADR). Sensors include vibration, motor current, gate counters, e-stop, zone occupancy.

[EVENT] MQTT zone occupancy received (estate telemetry - FR#2D source data)
[EVENT] MQTT heartbeat received (ride hardware: vibration, motor, gate sensors, e-stop - Appendix A 2-3)
[EVENT] Ride cycle completed (MQTT count event)

[QUESTION] Which of the 40 rides have MQTT heartbeat devices installed? Not necessarily all of them - placement is an ADR. Do NOT assume full coverage.

[QUESTION] MQTT gaps: when zone telemetry drops, the gap must show as "unknown" not as zero (Appendix B - "Gaps in MQTT must show as unknown, not as zero"). How does the intranet display this? NFR_11 mentions MTTD for P0 failures but no specific staleness display threshold is given in requirements. TBD in a later ADR.

[QUESTION] Keeper mobile entries (offline note / feed / task entry) - how do these reach the intranet integration bus when the keeper is in a low-signal area? NFR_4 says "field writes are append-only events; the cloud is the reconciler." The intranet consumes these; the offline sync mechanism is TBD.

---

## Domain events - ops state (phase 2)

[EVENT] Heat map refreshed (Appendix A 2-1: "Live (or last-known) heat map of the estate with data-age shown")
[EVENT] MQTT gap flagged (shows as unknown, not as zero; staleness visible - NFR_11, Appendix B)
[EVENT] Popularity rank updated (FR#2D: rides, zones, enclosures by daypart)
[EVENT] Congestion forecast generated (FR#2E: 30-90 minutes ahead; ML; includes confidence)
[EVENT] Ride anomaly flagged for inspect (FR#2K: from ML on MQTT heartbeats + cycle counts)
[EVENT] Staffing recommendation generated (FR#2F: AI advisory; from popularity + predicted load + downtime cost)

[EXTERNAL] Cloud analytics / data warehouse - NOT the system of record for payments or MQTT firehose. The intranet publishes operational state; analytical state belongs in the warehouse (Appendix A 2-6).

---

## Ride ops commands and events (phase 3)

[CMD] Change ride status (human only - duty manager or ride crew; NFR_7: "AI may draft work orders or suggest closures; it may not open a ride or silence a welfare alarm")
[EVENT] Ride status changed (open / closed / delayed / evacuated - Appendix A 2-2)

[CMD] Record inspection (ride, enclosure, building)
[EVENT] Inspection recorded (asset id, who, when, checklist, next due - Appendix A 2-3)

[CMD] Create work order (can be an AI-drafted suggestion from FR#2K; human decides whether to raise it)
[EVENT] Work order created (fault, severity, parts, time-to-repair - Appendix A 2-3)
[EVENT] Work order closed

[EVENT] Maintenance alert raised (FR#2K - from ML on MQTT sensor data; anomaly flag + recommended inspect-by time)
[CMD] Accept maintenance alert (duty manager or ride crew)
[CMD] Reject maintenance alert (duty manager or ride crew; false-positive rate is a first-class metric per FR#2K)

[EXTERNAL] Ride safety regulations (deterministic; NFR_7: "safety flows remain scripted"; not subject to AI A/B or experiment)

[HOTSPOT] Popular ride downtime: FR#2K says "Popular-ride downtime should surface yield-at-risk." The work-order event should carry or link to a yield-at-risk signal (tokens / hour lost). How is this computed? Needs popularity data from FR#2D. Cross-domain dependency.

[QUESTION] Animal-escape incident: Appendix A 2-3 mentions "Incidents: guest injury, near miss, animal escape (cross-link to animal module)." How does this cross-link work between Board B and Board A? The animal module (Board A) raises the alert; the intranet incident manager links to it. Interface TBD.

---

## Staffing, tasking, incidents, copilot (phase 4)

[EVENT] Staffing recommendation generated -- DUPLICATE from phase 2; keep in first pass
[CMD] Accept staffing recommendation (duty manager)
[CMD] Reject staffing recommendation (duty manager) - both feed back to the ML model as human signals

[CMD] Assign task (duty manager or ride crew; Appendix A 2-5)
[EVENT] Task assigned (inspect ride, move queue staff, extra keeper round)
[EVENT] Task completed

[CMD] Report incident (guest injury, near miss, animal escape - Appendix A 2-7)
[EVENT] Incident reported (time, zone, actions)
[EVENT] Incident escalated to duty manager (safety flows scripted; NFR_7)

[CMD] Query ops copilot (FR#2L: "Copilot for keepers and duty managers")
[EVENT] Ops copilot query answered (FR#2L: grounded on SOPs, keeper notes, live state; citations required; display only - "Staff remain accountable" per Appendix B)

[HOTSPOT] FR#2L says "Copilot for keepers AND duty managers." Board A has a Keeper Copilot. Is this one shared service or two? The grounding docs differ (SOPs + keeper notes vs. SOPs + live ops state). Open question - do not resolve here.

[EXTERNAL] Estate SOPs / keeper notes / ops documents (grounding corpus for the copilot RAG; these are not modelled here as a system, just as the document source)

---

## Component candidates (rough first-pass list)

- Something that maintains the live occupancy heat map with data-age (Estate Heatmap - candidate)
- Something that tracks ride status, cycle count, queue proxy (Ride Operations Manager - candidate)
- Something that manages inspections, work orders, and asset maintenance records (Asset & Maintenance Tracker - candidate; consumes FR#2K anomaly flags)
- Something that manages roster, tasks, and staffing suggestions (Staffing & Tasking Module - candidate; FR#2F)
- Something that manages incidents, escalation, and cross-links (Incident Manager - candidate; Appendix A 2-7)
- Something that generates 30-90 min congestion predictions (Congestion & Flow Forecaster - candidate; FR#2E)
- Something that aggregates occupancy, ride cycles, and gate counts into popularity ranks (Popularity Aggregator - candidate; FR#2D)
- Something that routes ticket events, MQTT, keeper writes, and publishes ops state (Integration Bus, logical - candidate; Appendix A 2-6)
- Something that serves copilot queries grounded in SOPs and live state (Ops Copilot / RAG - candidate; FR#2L; may overlap with Board A Keeper Copilot)

---

## Corrections and open questions from this pass

1. The intranet is NOT the SoR for payments or MQTT firehose (Appendix A 2 intro). Do not model it as such.
2. MQTT gaps must display as unknown, not zero (Appendix B). Any heat-map or trend component must handle this.
3. Ride status changes are human-only commands (NFR_7). AI can flag and suggest; it cannot change status.
4. Popular-ride downtime should surface yield-at-risk (FR#2K). Cross-domain signal: popularity -> maintenance path.
5. Animal-escape incidents cross-link to the animal module (Board A). Interface is TBD.
6. Keeper Copilot vs. Ops Copilot: one service or two? Open question for a later ADR.
7. Do not invent MQTT device counts, sensor placement, latency thresholds beyond what NFRs state.

---

*Digitised board: [board-intranet.svg](board-intranet.svg)*
