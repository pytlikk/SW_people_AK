# First Pass - Board B: Intranet / Estate OS (Duty Manager, Ride Crew)

> **Live asset.** Promoted from ops-backup on 2026-09-14; see pointer at
> [../ops-backup/event_storming.md](../ops-backup/event_storming.md).
> Parent index: [../event_storming.md](../event_storming.md).
>
> **Reconstructed first pass from `requirements/`, not a photographed workshop.**
> There was no physical sticky-note session. This document simulates the messy
> first pass that would emerge from a facilitated event-storming session run
> against the committed requirements. Unordered dump; duplicates expected;
> corrections and questions written in-line as they arose.
>
> Scope: **ops domain only (intranet / estate OS).** The intranet is the estate
> operating system - one place for staff to see occupancy, alerts, ride status,
> gate-throughput counts, and incidents. It is NOT the system of record for
> payments, the wallet, or the MQTT firehose (Appendix A 2 intro). Ticketing
> (ADR-001, ADR-002) appears as external upstream; decisions are now Proposed
> and load-bearing. Board B must not contradict them. The visit-access storm is
> Board V.
>
> Sticky type in [BRACKETS]. Order is approximate.

---

## Ticketing boundary (ADR-001 / ADR-002) - established upstream constraints

[EXTERNAL] Ticketing / Token System (ADR-001 / ADR-002) - upstream only.

ADR-001 established that the shop sells a currency (a token pool), not attraction
rights. Repricing does not touch issued wallets; claims already minted keep the
price they were minted at. The intranet may PROJECT leftover-pool and redemption
facts for staffing and congestion purposes; it is NOT the system of record for
wallet balance, SKU catalogue, or minting. Membership or path-based daytime
products that set token cost to zero still mint and burn a signed ADR-002 QR at
the checkpoint (ADR-001 Key differentiators). Board B must not invent a parallel
ticket type or an ops-level "free admit" that bypasses the claim.

ADR-002 established that the MQTT `validated` / `revoked` events are for
popularity, audit, and extra lanes - NOT for admit, NOT for the app wallet. The
popularity aggregator PM-01 (Board P) owns the count feed. IO-01 (Estate
Heatmap) and IO-04 (Staffing and Tasking Module) consume PM-01; they do not own
the count or subscribe directly to the raw `validated` event stream.

Characteristics that these ADR decisions bring as upstream constraints (cited
from the ADRs, not invented here; no system-wide funnel exists yet):
- ADR-001 strengthens evolvability / repricing agility and operability; weakens
  predictability, recoverability, and guest certainty.
- ADR-002 strengthens reliability (offline admit) and operability; weakens
  throughput / performance, security / integrity, and recoverability.

Board B inherits these trade-offs. It does not re-litigate them.

[HOTSPOT] Signing-key architecture for claims is an open question in ADR-002
(client-side vs. server-side; owner: TBD). If any intranet view shows
claim-verification status or signing-related metrics, it must point at that
open question. Board B must not assume a resolved key-custody design.

[HOTSPOT] ADR-002 states that a pending-to-used failure after a failed gate read
has no self-service path; the kiosk is the only correction point. Board B must
not grow a "remote refund" or ops-console correction command for gate failures.
These are visitor-lane costs named in ADR-002 (weakened recoverability); they are
not fixable on the intranet board without a new ADR that explicitly allows it -
and no such ADR exists.

[HOTSPOT] Screenshot-able QR and the fraud window between QR-reveal and
checkpoint cache write (ADR-002 weakened security / integrity; window length TBD)
are visitor-lane integrity costs. Board B must not silently "fix" them via an
intranet admin command or a remote revocation path not already described in
ADR-002.

---

## Actors

[ACTOR] Duty Manager - inferred role for "Staff (deploy / invest)"; brief does
not use the title "duty manager" but Appendix A 2-1 ("Duty-manager view") does.
Label as **inferred role**.
[ACTOR] Ride Crew - inferred role for "Staff (rides)"; Appendix A 2-2 is the
source. Label as **inferred**.
[ACTOR] Countess - view-level access; makes invest decisions; does NOT dispatch
field commands directly from the intranet.

---

## External inputs (phase 1)

[EXTERNAL] Ticketing / Token System (ADR-001 / ADR-002) - upstream only. The
intranet subscribes to the event stream produced by Board V / VA-03; it does NOT
write back into ticketing.

NOTE: There is no `[CMD] Consume ticket redemption event`. Consuming an upstream
event stream is an integration concern, not an intranet command. The intranet
receives gate-throughput counts as a downstream consumer of PM-01 (which
aggregates ADR-002 `validated` events). The intranet does not own the count.

[EVENT] Gate throughput measured - NOTE: this event reaches the intranet from
PM-01 (Board P / PM-01 Popularity Aggregator), which aggregates ADR-002
`validated` events from VA-03. Intranet (IO-01 Estate Heatmap) is a projection
consumer; it does NOT subscribe directly to raw `validated` events and does NOT
own the count.

[EXTERNAL] MQTT hardware (budget exists per Assumptions; placement TBD per ADR).
Sensors include vibration, motor current, gate counters, e-stop, zone occupancy.

[EVENT] MQTT zone occupancy received (estate telemetry - FR#2D source data)
[EVENT] MQTT heartbeat received (ride hardware: vibration, motor, gate sensors,
e-stop - Appendix A 2-3)
[EVENT] Ride cycle completed (MQTT count event)

[QUESTION] Which of the 40 rides have MQTT heartbeat devices installed? Not
necessarily all of them - placement is an ADR. Do NOT assume full coverage.

[QUESTION] MQTT gaps: when zone telemetry drops, the gap must show as "unknown"
not as zero (Appendix B - "Gaps in MQTT must show as unknown, not as zero"). How
does the intranet display this? NFR_11 mentions MTTD for P0 failures but no
specific staleness display threshold is given in requirements. TBD in a later
ADR.

[QUESTION] Keeper mobile entries (offline note / feed / task entry) - how do
these reach the intranet integration bus when the keeper is in a low-signal area?
NFR_4 says "field writes are append-only events; the cloud is the reconciler."
The intranet consumes these; the offline sync mechanism is TBD.

---

## Domain events - ops state (phase 2)

[EVENT] Heat map refreshed (Appendix A 2-1: "Live (or last-known) heat map of
the estate with data-age shown") - data comes from PM-01 and MQTT; IO-01 does
not own the source counts
[EVENT] MQTT gap flagged (shows as unknown, not as zero; staleness visible -
NFR_11, Appendix B)
[EVENT] Popularity rank updated (FR#2D: rides, zones, enclosures by daypart) -
produced by PM-01; intranet IO-01 / IO-04 consume it, do not own it
[EVENT] Congestion forecast generated (FR#2E: 30-90 minutes ahead; ML; includes
confidence)
[EVENT] Ride anomaly flagged for inspect (FR#2K: from ML on MQTT heartbeats +
cycle counts)
[EVENT] Staffing recommendation generated (FR#2F: AI advisory; from popularity +
predicted load + downtime cost) - popularity data comes from PM-01

[EXTERNAL] Cloud analytics / data warehouse - NOT the system of record for
payments or MQTT firehose. The intranet publishes operational state; analytical
state belongs in the warehouse (Appendix A 2-6).

---

## Ride ops commands and events (phase 3)

[CMD] Change ride status (human only - duty manager or ride crew; NFR_7: "AI may
draft work orders or suggest closures; it may not open a ride or silence a
welfare alarm")
[EVENT] Ride status changed (open / closed / delayed / evacuated - Appendix A
2-2)

[CMD] Record inspection (ride, enclosure, building)
[EVENT] Inspection recorded (asset id, who, when, checklist, next due - Appendix
A 2-3)

[CMD] Create work order (can be an AI-drafted suggestion from FR#2K; human
decides whether to raise it)
[EVENT] Work order created (fault, severity, parts, time-to-repair - Appendix A
2-3)
[EVENT] Work order closed

[EVENT] Maintenance alert raised (FR#2K - from ML on MQTT sensor data; anomaly
flag + recommended inspect-by time)
[CMD] Accept maintenance alert (duty manager or ride crew)
[CMD] Reject maintenance alert (duty manager or ride crew; false-positive rate is
a first-class metric per FR#2K)

[EXTERNAL] Ride safety regulations (deterministic; NFR_7: "safety flows remain
scripted"; not subject to AI A/B or experiment)

[HOTSPOT] Popular ride downtime: FR#2K says "Popular-ride downtime should surface
yield-at-risk." The work-order event should carry or link to a yield-at-risk
signal (tokens / hour lost). How is this computed? Needs popularity data from
PM-01 (FR#2D). Cross-domain dependency; popularity is PM-01's output.

[QUESTION] Animal-escape incident: Appendix A 2-3 mentions "Incidents: guest
injury, near miss, animal escape (cross-link to animal module)." How does this
cross-link work between Board B and Board A? The animal module (Board A) raises
the alert; the intranet incident manager links to it. Interface TBD.

---

## Staffing, tasking, incidents, copilot (phase 4)

[EVENT] Staffing recommendation generated -- DUPLICATE from phase 2; keep in
first pass
[CMD] Accept staffing recommendation (duty manager)
[CMD] Reject staffing recommendation (duty manager) - both feed back to the ML
model as human signals

[CMD] Assign task (duty manager or ride crew; Appendix A 2-5)
[EVENT] Task assigned (inspect ride, move queue staff, extra keeper round)
[EVENT] Task completed

[CMD] Report incident (guest injury, near miss, animal escape - Appendix A 2-7)
[EVENT] Incident reported (time, zone, actions)
[EVENT] Incident escalated to duty manager (safety flows scripted; NFR_7)

[CMD] Query ops copilot (FR#2L: "Copilot for keepers and duty managers")
[EVENT] Ops copilot query answered (FR#2L: grounded on SOPs, keeper notes, live
state; citations required; display only - "Staff remain accountable" per
Appendix B)

[HOTSPOT] FR#2L says "Copilot for keepers AND duty managers." Board A has a
Keeper Copilot (AC-05). Is this one shared service or two? The grounding docs
differ (SOPs + keeper notes vs. SOPs + live ops state). Open question - do not
resolve here.

[HOTSPOT] Ops copilot must not advise on claim signing, key custody, or gate
admission internals. Signing-key architecture is an open question in ADR-002
(see Ticketing boundary section above). Copilot scope is ops SOPs, ride/staff
state - not ticketing internals.

[EXTERNAL] Estate SOPs / keeper notes / ops documents (grounding corpus for the
copilot RAG; these are not modelled here as a system, just as the document
source)

---

## Component candidates (rough first-pass list, IO- prefix)

These are names, not decisions. CC-06..CC-14 are the legacy IDs from the
ops-backup era; the live IDs below supersede them. CC-12 is PM-01 in the
guest-lane board (see event_storming.md). See the CC-to-IO mapping table in the
parent index.

- IO-01 (was CC-06): maintains the live occupancy heat map with data-age (Estate
  Heatmap); projection consumer of PM-01 and MQTT; does NOT own the count
- IO-02 (was CC-07): tracks ride status, cycle count, queue proxy (Ride
  Operations Manager); status changes are human-only (NFR_7)
- IO-03 (was CC-08): manages inspections, work orders, and asset maintenance
  records (Asset and Maintenance Tracker; consumes FR#2K anomaly flags)
- IO-04 (was CC-09): manages roster, tasks, and staffing suggestions (Staffing
  and Tasking Module; FR#2F); consumes PM-01 popularity output, not raw
  validated events
- IO-05 (was CC-10): manages incidents, escalation, and cross-links (Incident
  Manager; Appendix A 2-7)
- IO-06 (was CC-11): generates 30-90 min congestion predictions (Congestion and
  Flow Forecaster; FR#2E)
- PM-01 (was CC-12): Popularity Aggregator - ownership made explicit on Board P /
  guest-lane session; IO-01 and IO-04 consume it; they do not own it
- IO-07 (was CC-13): routes ticket events, MQTT, keeper writes, and publishes ops
  state (Integration Bus, logical); analytical state belongs in the data warehouse
- IO-08 (was CC-14): serves copilot queries grounded in SOPs and live state (Ops
  Copilot / RAG; FR#2L; may overlap with AC-05 Keeper Copilot - open question)

---

## Corrections and open questions from this pass

1. The intranet is NOT the SoR for payments or MQTT firehose (Appendix A 2
   intro). Do not model it as such.
2. MQTT gaps must display as unknown, not zero (Appendix B). Any heat-map or
   trend component must handle this.
3. Ride status changes are human-only commands (NFR_7). AI can flag and suggest;
   it cannot change status.
4. Popular-ride downtime should surface yield-at-risk (FR#2K). Cross-domain
   signal: PM-01 popularity -> maintenance path.
5. Animal-escape incidents cross-link to the animal module (Board A). Interface
   is TBD.
6. Keeper Copilot (AC-05) vs. Ops Copilot (IO-08): one service or two? Open
   question for a later ADR.
7. Do not invent MQTT device counts, sensor placement, latency thresholds beyond
   what NFRs state.
8. No remote-refund or gate-correction command on this board. ADR-002: kiosk is
   the dispute desk for gate failures; weakened recoverability is a visitor-lane
   cost, not an intranet fixable thing.
9. Popularity rank is PM-01's output; IO-01 and IO-04 are consumers. Do not
   model the intranet as owning or re-deriving the count.
10. Membership / path products that set token cost to zero still consume an
    ADR-002 signed QR at the checkpoint. Board B must not invent a "free admit"
    path that bypasses the claim.

---

*Digitised board: [board-intranet.svg](board-intranet.svg)*
