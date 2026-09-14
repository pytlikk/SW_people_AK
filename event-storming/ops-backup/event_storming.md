# Event Storming - ops backup (animal care + intranet)

**This folder is backup.** It is not the current derivation-chain work.
Current boards (visit access, popularity meter, AI Guide) live in
[../event_storming.md](../event_storming.md).

**Scope of this session:** two ops domains only - Animal Care (Board A) and
Intranet / Estate OS (Board B). The visit-access storm (guest QR scan, token
pool spending at a live checkpoint) is a separate later session. Ticketing
(ADR-001, ADR-002) appears on these boards as an external upstream system only.

**Honesty note:** there was no physical sticky-note workshop. Both iterations
below are reconstructed from the committed requirements documents. The first-pass
files look like a raw session transcript with corrections and questions in-line.
The digitised SVG boards show the same material in chronological lanes with a
colour key. Judges are asked to read them as two passes over the same domain,
not as evidence of a photographed session.

---

## Why this technique

Event storming (Brandolini) puts domain events - things that happened that the
business cares about - on the board first, before commands, actors, or components.
That ordering forces us to name what the estate actually produces (an alert, a
work order, a population estimate) before naming who or what causes it. For a
brief where AI is advisory and humans confirm every significant action, this is
the right discipline: the domain events are real regardless of whether a model
or a keeper triggers them.

For this estate, the technique also clarifies two boundaries that prose
requirements blur: (1) enclosure scans (from ADR-002 ticket validation) feed
a popularity meter, not the health record; (2) the intranet is an operational
projection, not the system of record for payments or MQTT telemetry
(Appendix A, section 2 intro). Both boundaries are hotspots on the boards and
become constraints on later component decisions.

---

## Goals

1. Name the domain events that keepers, duty managers, and ride crews actually
   care about, and distinguish them from system internals.
2. Identify the commands that humans issue (and the ones that are AI-only
   suggestions to be accepted or rejected).
3. Locate the external systems that feed these domains without being designed
   in this session (ticketing, MQTT hardware, cloud analytics).
4. Surface uncertainties that will need their own ADRs before a component is
   built.
5. Produce a candidate component list that the architecture characteristics
   funnel and later C4 work can start from.

---

## Outcomes by sticky type

### Actors (yellow)

| Board | Actor | Source |
|---|---|---|
| A | Staff (animals) / Keeper | Brief; `requirements/1_3_Actors and actions.md` |
| A, B | Countess | Brief; `requirements/1_3_Actors and actions.md`; view / invest only, no field commands |
| B | Duty Manager | Inferred role for "Staff (deploy / invest)"; label: inferred; source: Appendix A 2-1 |
| B | Ride Crew | Inferred role for "Staff (rides)"; label: inferred; source: Appendix A 2-2 |

Veterinary authority is NOT modelled as an actor. The brief does not name a vet
role or organisation. NFR_15 states "human confirm" and the Assumptions state
"software recommends; it does not replace veterinary or safety authority." Any
reference to welfare authority on Board A is labelled as inferred from those
sources, and is a policy constraint, not a named system or actor.

### Commands (blue)

Animal care commands: Record Health Observation, Record Feeding Observation,
Add Keeper Note, Conduct Piranha Census, Read Enclosure Environment, Accept
Alert, Reject Alert, Query Keeper Copilot.

Intranet commands: Consume Ticket Redemption Event, Change Ride Status (human
only; NFR_7), Record Inspection, Create Work Order (AI may draft; FR#2K),
Accept / Reject Maintenance Alert, Assign Task, Report Incident, Accept / Reject
Staffing Recommendation, Query Ops Copilot.

Note on AI-related commands: the AI scenarios (FR#2I, FR#2J, FR#2F, FR#2K,
FR#2L) are represented as commands that humans issue after seeing an advisory
output - "Accept Alert", "Reject Alert", "Accept Maintenance Alert" - and not
as model names or technology choices.

### Domain events (orange)

Alphabetical sample from Board A: Alert Accepted (training signal), Alert
Inbox Updated, Alert Raised (with confidence and evidence), Alert Rejected
(training signal), Animal Health Anomaly Detected, Copilot Query Answered,
Enclosure Environment Read, Enclosure Scan Received (popularity only; not
health - ADR-002), Feeder Event Received (MQTT), Feed Refusal Spike Detected,
Feeding Observation Recorded, Health Observation Recorded, Health Trend Visible
to Countess, Keeper Note Added, MQTT Feeder Event Received, Piranha Population
Counted (census), Piranha Population Estimate Generated, Population Trend
Flagged.

Alphabetical sample from Board B: Congestion Forecast Generated, Gate
Throughput Measured, Heat Map Refreshed (data age shown), Incident Escalated,
Incident Reported, Inspection Recorded, Maintenance Alert Raised, MQTT Gap
Flagged (shows as unknown, not zero), MQTT Heartbeat Received, MQTT Zone
Occupancy Received, Ops Copilot Query Answered (display only), Popularity Rank
Updated, Ride Anomaly Flagged for Inspect, Ride Cycle Completed, Ride Status
Changed, Staffing Recommendation Generated, Task Assigned, Task Completed,
Work Order Closed, Work Order Created.

### External systems / policies (magenta)

| External | Boards | Notes |
|---|---|---|
| Ticketing / Token System (ADR-001, ADR-002) | A, B | Upstream only; enclosure scans may feed popularity; do not open health records or write into health records |
| MQTT hardware (estate-wide) | A, B | Budget exists (Assumptions); device count and placement are TBD - an ADR, not a constraint here |
| Cloud analytics / data warehouse | B | Not SoR for payments or MQTT firehose (Appendix A 2); receives analytical state, not operational state |
| Ride safety regulations | B | Deterministic; safety flows scripted; NFR_7; never AI-overridable |
| Welfare authority (inferred) | A | Not named in brief; inferred from NFR_15 and Assumptions; modelled as a policy sticky, not an actor |
| Estate SOPs / keeper notes | B | Grounding corpus for Ops Copilot RAG (FR#2L); not a system, a document collection |

### Uncertainties, questions, and open decisions (red)

These are genuine questions, not fake answers. Each needs an ADR or at
minimum a numbered assumption before a component is built.

1. Which of the 55 displays are fitted with MQTT sensors? Placement and device
   type are TBD (Assumptions say budget exists; sensor ADR needed).
2. Which of the 40 rides have MQTT heartbeat devices? Same as above.
3. Individual animal ID tracking or enclosure-level? Assumptions say
   enclosure-level suffices except where IDs already exist. Not settled by ADR.
4. What is the anomaly threshold for health and feeding alerts (FR#2I)? Not
   stated in the brief. An ADR must specify the evaluation metric and threshold
   before going live (NFR_13).
5. Is there camera or sonar hardware at the piranha enclosure? FR#2J says
   "optional." Placement is TBD.
6. What is the keeper copilot vet-dose guardrail - specifically, how is
   "fail closed to ask the vet / duty manager" (FR#2L) enforced technically?
   ADR needed.
7. What is the staleness banner threshold for the intranet heat map? NFR_11
   sets MTTD for P0 failures; a specific "data is stale" display threshold is
   not in the brief. Leave TBD in the ops ADR; do not invent a number here.
8. Does the keeper mobile app write directly to the integration bus, or via a
   local gateway, when offline (NFR_4)? The sync path is TBD.
9. Are the Keeper Copilot (Board A) and Ops Copilot (Board B) one service or
   two? The grounding corpora differ; the boundary is an ADR question.
10. Which of the 55 displays are paid checkpoints? ADR-002 leaves this TBD;
    enclosure scans exist only where there are gates.

---

## Process

The session was run in two passes over each board:

**First pass (messy dump):** events were named in any order as the requirements
were read, then corrected in-line. Duplicates were preserved rather than deleted,
so the history of corrections is visible. The first-pass files look like a
facilitator's whiteboard transcript. Key boundary corrections: enclosure scan
events were initially placed in the health record flow, then moved to the
popularity meter path when ADR-002 was re-read.

**Second pass (digitised):** events were sorted into five chronological phases
per board and placed in coloured stickies with a key. Aggregate / component
candidates (green) appear only in the digitised boards, in the rightmost lane,
so they are visibly outputs of the session rather than inputs to it.

The boards are read left to right. Actors and external inputs on the left;
domain events that record what happened; AI / ML processing that raises further
events; human alert and decision commands; component candidates on the right.

---

## Key decisions from the session

The following decisions were either confirmed by existing ADRs or identified as
needing future ADRs. No new ADR is created for the event-storming technique
itself.

| Decision | Existing link | Status |
|---|---|---|
| Ticketing appears as external upstream only; enclosure scans feed popularity, not health records | [ADR-002](../../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) (Conclusion section) | Existing ADR; confirmed here |
| Token pool is the upstream economy; this session does not re-storm ticketing | [ADR-001](../../adrs/ADR-001-use-home-bought-token-pool.md) | Existing ADR; confirmed here |
| AI outputs are advisory; Accept / Reject commands are the human-in-the-loop interface for alerts and recommendations | [requirements/2_FRs.md](../../requirements/2_FRs.md) (FR#2I, FR#2J, FR#2F, FR#2K); [requirements/Appendix B](../../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) | Requirements constraint; no ADR yet |
| MQTT gaps show as unknown, not zero, on the intranet heat map | [requirements/Appendix B](../../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) | Requirements constraint; design detail TBD |
| Ride status changes and animal welfare decisions are human-only commands; AI may draft or suggest but not execute | [requirements/3_NFRs.md](../../requirements/3_NFRs.md) (NFR_7, NFR_15) | Requirements constraint; no ADR yet |
| Intranet is not the SoR for payments or MQTT firehose | [requirements/Appendix A](../../requirements/Appendix%20A_%20Core%20functionality.md) section 2 intro | Requirements constraint |
| Staff (keeper, ride crew, duty manager) remain in the loop; keeper mobile entry must work offline | [requirements/3_NFRs.md](../../requirements/3_NFRs.md) (NFR_4, NFR_17) | Requirements constraint |

---

## Component candidates

These are candidates produced by the session. They are named and bounded, but
not yet designed, and no ADR is written for any of them here. Phrasing is
deliberate: "candidate SoR", "candidate projection store", because these are
inputs to later ADRs, not outputs.

| # | Candidate | Board | Bounded context | Key FRs / source | Notes |
|---|---|---|---|---|---|
| CC-01 | Animal Care Record Service | A | Animal care | FR#1; Appendix A 2-4 | Candidate SoR for health observations, feeding events, enclosure environment readings, keeper notes; per display; NOT the gate or popularity system |
| CC-02 | Alert Inbox | A | Animal care | FR#2I, FR#2J | Receives AI-generated suggestions with confidence scores and evidence; keeper accept / reject is the training signal; advisory only |
| CC-03 | Anomaly Detection Engine | A | Animal care | FR#2I | ML + optional Vision; high-recall tuning (FR#2I: "High recall first"); human confirm before any welfare action; needs golden-case eval harness (NFR_13) |
| CC-04 | Piranha Population Estimator | A | Animal care | FR#2J | Colony-level Vision / ML estimate with confidence interval; periodic keeper census is the ground truth; must have error band, not a single point estimate |
| CC-05 | Keeper Copilot / RAG | A | Animal care | FR#2L | Grounded retrieval on estate SOPs, keeper notes, live state; citations required; fails closed to "ask the vet / duty manager"; may be shared with CC-14 - open question |
| CC-06 | Estate Heatmap | B | Intranet ops | FR#1; FR#2D; NFR_11 | Candidate **projection** of occupancy and freshness; data age shown; MQTT gaps displayed as unknown, not zero. Does not own the count. |
| CC-07 | Ride Operations Manager | B | Intranet ops | FR#1; Appendix A 2-2 | Status per ride (open / closed / delayed / evacuated), cycle count, queue proxy if instrumented; status changes are human commands only (NFR_7) |
| CC-08 | Asset and Maintenance Tracker | B | Intranet ops | FR#1; Appendix A 2-3; FR#2K | Inspections, work orders, incidents; consumes anomaly flags from FR#2K (predictive maintenance); yield-at-risk signal from popular-ride downtime |
| CC-09 | Staffing and Tasking Module | B | Intranet ops | FR#2F; Appendix A 2-5 | Roster vs predicted load; task assignment; AI staffing suggestions that duty manager accepts or rejects; includes keeper / ride-crew mobile entry with offline sync |
| CC-10 | Incident Manager | B | Intranet ops | FR#1; Appendix A 2-7 | Guest injury, near miss, animal escape; escalation to duty manager; safety flows scripted; cross-links to animal module (Board A) on animal-escape events |
| CC-11 | Congestion and Flow Forecaster | B | Intranet ops (displays); visit-access / popularity (produces) | FR#2E | 30-90 minute predicted occupancy (the window is already in FR#2E). Intranet shows it; the model is not the OS. Degrades to recency baseline when ingest is unhealthy (Appendix B). |
| CC-12 | Popularity Aggregator | Shared / visit-access | Popularity meter (not intranet SoR) | FR#2D; ADR-002 `validated` events | Fuses gate counts, ride cycles, MQTT zone occupancy into rank / dwell / throughput. **Intranet (CC-06, CC-09) consumes this; it does not invent it.** Planned next ticketing-side ADR in `adrs/README.md`. |
| CC-13 | Integration Bus (logical) | B | Platform | FR#1; Appendix A 2-6 | Consumes ticket events (ADR-001, ADR-002), MQTT telemetry, keeper app writes, maintenance writes, AI outputs; publishes operational state for UIs; analytical state belongs in the data warehouse. Broker placement is still an assumption. |
| CC-14 | Ops Copilot / RAG | B | Intranet ops | FR#2L | Grounded on estate SOPs, keeper notes, live ops state; citations required; display only - staff remain accountable (Appendix B); may be shared with CC-05 - open question |

**Depth allocation (what to ADR first vs leave shallow):**

- **Next Proposed ADRs from these boards:** CC-01 (keeper records as SoR), CC-02 (alert inbox, human confirm), CC-06 plus freshness (intranet is a projection), CC-13 as a logical bus only until the MQTT ADR exists.
- **Leave shallow until those exist:** CC-03, CC-04, CC-05, CC-11, CC-14 (AI engines and copilots). Naming them here is not a decision to build them first.
- **CC-12 belongs with the popularity-meter ADR**, not with intranet UI. Board B needs the feed; it does not own the meter.
- **Shared later:** CC-05 vs CC-14 (one RAG or two); MQTT device placement.

**Natural groupings visible from the board:**

- **Animal care context (CC-01 to CC-05):** keeper domain; first ADR is the candidate SoR plus accept/reject inbox.
- **Intranet ops context (CC-06 to CC-10, CC-14):** estate OS as a candidate projection; consumes CC-12 and CC-13.
- **Platform (CC-13):** logical boundary with ticketing, MQTT, and AI pipelines. Not a broker topology.

---

## Iterations

### Iteration 1 - First pass (messy)

| Board | File |
|---|---|
| Animal Care | [assets/first-pass-animal.md](./assets/first-pass-animal.md) |
| Intranet / Estate OS | [assets/first-pass-intranet.md](./assets/first-pass-intranet.md) |

Reconstructed first pass from `requirements/`. Unordered sticky dump, corrections
and questions written in-line as they arose, duplicates preserved. Not a
photographed workshop.

### Iteration 2 - Digitised

Colour key for both boards:

| Colour | Sticky type |
|---|---|
| Yellow | Actor |
| Blue | Command |
| Orange | Domain Event |
| Magenta / pink | External / Policy |
| Red | Question / Hotspot |
| Green | Component Candidate (digitised board only) |

Each SVG is self-contained with the colour key inline.

| Board | SVG |
|---|---|
| Animal Care | [assets/board-animal.svg](./assets/board-animal.svg) |
| Intranet / Estate OS | [assets/board-intranet.svg](./assets/board-intranet.svg) |

Chronological lanes run left to right. Phase headers appear above each lane.
Component candidates (green) occupy the rightmost lane on each board as the
session output.
