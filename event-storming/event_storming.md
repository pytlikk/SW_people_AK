# Event Storming - Guest Lane (current)

**This is the current derivation-chain work.** Five boards across two sessions:
guest-lane (visit access deep, popularity meter next-ADR, AI Guide shallow) and
ops (Animal Care parallel-shallower, Intranet / Estate OS parallel-shallower).

Animal Care (Board A) and Intranet / Estate OS (Board B) were parked in
ops-backup/ and are now promoted to this index. A one-paragraph pointer remains
at [ops-backup/event_storming.md](ops-backup/event_storming.md). Both ops boards
are explicitly shallower than Board V; see depth allocation below.

**Honesty note:** there was no physical sticky-note workshop. All three passes below are
reconstructed from the committed ADRs and requirements documents. The first-pass files
look like a raw session transcript with corrections and questions in-line. The digitised
SVG boards show the same material in chronological lanes with a colour key. Judges are
asked to read them as two passes over the same domain, not as evidence of a photographed
session.

---

## Why event storming for this session

Event storming (Brandolini) names what the estate produces - a `validated` MQTT event,
a minted claim, a popularity count - before naming who or what causes it. For this
board set, the technique forces two boundaries that prose requirements blur:

1. MQTT `validated`/`revoked` events are the **audit and popularity channel**, not the
   wallet update path. ADR-002 is explicit; the board makes it visible.
2. The popularity aggregator (PM-01) **owns** the count feed. The intranet heat map
   (CC-06) is a consumer. Not the source.

---

## Goals for this session

1. Reconstruct the guest-lane events that ADR-001 and ADR-002 decided, as a board.
2. Name what feeds a popularity count, what a gap looks like, and why zero is wrong.
3. Make the leftover-pool return-incentive hook visible before the AI Guide ADR.
4. Produce a candidate component list with new prefixes (VA-, PM-, AG-) that does not
   reuse CC-01 to CC-14 (defined in ops-backup).

---

## Five boards

### Board V - Visit Access / Ticketing (deep)

Reconstructs ADR-001 (token pool economy) and ADR-002 (signed static QR, checkpoint
verify) as a domain event board. Go deep here because both ADRs are already proposed.

**Scope:** Guest buys pool (including family pass as a pool SKU with party rules);
app wallet holds pool balance; QR-reveal mints a signed claim at live or cached price;
checkpoint camera scans and verifies locally (no network round-trip); claim burns into
the local seen-token cache; `validated`/`revoked` MQTT events are published as the
audit and popularity channel. Kiosk path handles top-up and dead-phone reprint.

**What this board does NOT decide:**
- MQTT broker topology and QoS (TBD; separate ADR)
- Kiosk output: paper reprint or screen-only (TBD; ADR-002 open)
- Which of the 55 displays are paid checkpoints (TBD; ADR-002 open)
- Token expiry / no-refund / credit-only policy (TBD before launch; ADR-001 open)
- Dynamic pricing (table writer; later ADR; not a second economy)

First pass: [assets/first-pass-visit.md](assets/first-pass-visit.md)
Digitised SVG: [assets/board-visit.svg](assets/board-visit.svg)

---

### Board P - Popularity Meter (next-ADR depth)

This board feeds the proposed popularity-meter ADR, which is third in the plan at
[../adrs/README.md](../adrs/README.md). ADR-002 `validated` events are the primary
signal; ride cycles and optional MQTT zone occupancy are secondary.

**Scope:** Event ingestion from VA-03 + MQTT hardware; gap detection (silence = unknown,
NOT zero, per FR#2D and Appendix B); count and dwell aggregation; popularity snapshot
publication to downstream consumers (intranet CC-06, later pricing ADR, congestion
forecaster CC-11).

**What this board does NOT decide:**
- Dwell estimation method (entry+exit pair, zone window, or rate approximation; TBD)
- Gap threshold duration (how long is "silent"? TBD; do not invent a number)
- Snapshot frequency (real-time stream vs batch; TBD)
- Whether PM-02 is a module inside PM-01 or a separate watchdog (TBD)
- Dynamic pricing AI (table writer on this feed; later ADR; not on this board)

First pass: [assets/first-pass-popularity.md](assets/first-pass-popularity.md)
Digitised SVG: [assets/board-popularity.svg](assets/board-popularity.svg)

---

### Board G - Return Incentive / AI Guide (shallow)

Stays shallow until the popularity-meter ADR exists. The goal is to make the
leftover-pool hook (ADR-001) and the FR#2G/FR#2H flows visible for the judge.

**Scope:** Post-visit leftover pool persists (ADR-001 driving criterion); opt-in trail
(FR#2G) generates itinerary using PM-01 wait-time data; win-back trigger evaluates
leftover pool and visit history (FR#2H); win-back offer sent with frequency caps and
opt-out; 90-day return is the outcome metric. In-app games are a shallow hotspot, not
a designed platform.

**What this board does NOT decide:**
- Delivery channel for in-park suggestions (push / kiosk / app; TBD)
- Win-back comms provider (TBD)
- Win-back frequency cap and opt-out mechanism (TBD before launch)
- Whether AG-01 and AG-02 are one service or two (TBD)
- In-app games: shallow hotspot, not designed here

First pass: [assets/first-pass-guide.md](assets/first-pass-guide.md)
Digitised SVG: [assets/board-guide.svg](assets/board-guide.svg)

---

### Board A - Animal Care (Keepers) (parallel, shallower than Board V)

Promoted from ops-backup/ on 2026-09-14. Shallower than Board V by design - visit
access is the hard problem; Animal Care is a parallel workstream that must now be
consistent with the ticketing decisions ADR-001 and ADR-002.

**Scope:** Keeper health and feeding observations, enclosure environment readings,
MQTT feeder events, keeper notes, AI-generated anomaly alerts (FR#2I), piranha
population estimates (FR#2J), keeper copilot RAG (FR#2L). Countess reads aggregate
health trends; she does not issue field commands. Component candidates use AC- prefix.

**What this board does NOT decide:**
- Ticketing, wallet balance, QR claims, or gate admission logic - those belong to
  ADR-001 / ADR-002 and Board V.
- Enclosure scans as health-record inputs - ADR-002 Conclusion is explicit: enclosure
  scans (the `validated` / `revoked` MQTT events) may feed popularity counts
  (Board P / PM-01), not the health record. AC-01 does not receive ADR-002 events.
- MQTT sensor placement and device count - TBD per separate ADR; budget exists
  (Assumptions).
- Anomaly alert thresholds and evaluation metrics - TBD; an ADR must specify before
  go-live (NFR_13).
- Individual vs. enclosure-level animal ID - enclosure-level is the working assumption
  except where IDs already exist (Assumptions).
- Keeper / Ops Copilot: one service or two - open question; grounding corpora differ.
- Signing-key architecture for QR claims - open question in ADR-002 (client-side vs.
  server-side; owner TBD). Board A must not assume a resolved key-custody design.
- Remote-refund or ops-correction commands for gate failures - ADR-002 names kiosk
  as the only correction point; weakened recoverability is a visitor-lane cost.

**Ticketing decisions as upstream constraints (ADR-001 / ADR-002):**
Animal Care does not own wallet balance, SKU catalogue, or claim minting. Membership
or path-based products that set token cost to zero still mint and burn a signed
ADR-002 QR - no parallel admit path. Why ticketing looks like this: ADR-001
strengthens evolvability / repricing agility and operability; weakens predictability,
recoverability, and guest certainty. ADR-002 strengthens reliability (offline admit)
and operability; weakens throughput / performance, security / integrity, and
recoverability. Board A inherits these as given, not re-litigated.

First pass: [assets/first-pass-animal.md](assets/first-pass-animal.md)
Digitised SVG: [assets/board-animal.svg](assets/board-animal.svg)

---

### Board B - Intranet / Estate OS (Duty Manager, Ride Crew) (parallel, shallower than Board V)

Promoted from ops-backup/ on 2026-09-14. Shallower than Board V by design - the
intranet consumes ticketing outputs; it does not decide them.

**Scope:** Live estate heat map (occupancy + data-age), ride status and cycle counts,
asset inspections and maintenance work orders, staffing and task assignment, incident
management, congestion forecasting (FR#2E), ops copilot (FR#2L). Duty Manager and
Ride Crew are the operators; Countess reads investment views. Component candidates
use IO- prefix.

**What this board does NOT decide:**
- Wallet balance, SKU catalogue, or claim minting - owned by ADR-001 / ADR-002 /
  Board V.
- The popularity count - PM-01 (Board P) owns the count; IO-01 (Estate Heatmap) and
  IO-04 (Staffing and Tasking Module) are projection consumers of PM-01.
- MQTT broker topology and QoS - TBD per separate ADR.
- Remote-refund or ops-correction commands for gate failures - ADR-002 names kiosk
  as the only correction point; Board B must not grow a "remote refund" command.
  Weakened recoverability is a visitor-lane cost in ADR-002.
- Signing-key architecture for QR claims - open question in ADR-002 (client-side vs.
  server-side; owner TBD). If any intranet view shows claim-verification status, it
  must point at that open question, not assume a resolved design.
- Screenshot-able QR and fraud window length (ADR-002 weakened security / integrity;
  window TBD) - visitor-lane integrity costs; not fixable on this board without a
  new ADR.
- Popularity subscriptions: IO-01 and IO-04 consume PM-01 output; they do not
  subscribe directly to raw ADR-002 `validated` events and do not own the count.

**Ticketing decisions as upstream constraints (ADR-001 / ADR-002):**
The intranet may project leftover-pool and redemption facts for staffing and
congestion; it is not the system of record for wallet balance or payment events.
Membership or path-based products still consume an ADR-002 claim at the checkpoint -
Board B must not invent a parallel ticket type or a "free admit" path. Why ticketing
looks like this: ADR-001 strengthens evolvability / repricing agility and operability;
weakens predictability, recoverability, and guest certainty. ADR-002 strengthens
reliability (offline admit) and operability; weakens throughput / performance,
security / integrity, and recoverability. Board B inherits these as given.

First pass: [assets/first-pass-intranet.md](assets/first-pass-intranet.md)
Digitised SVG: [assets/board-intranet.svg](assets/board-intranet.svg)

---

## Component candidates

New prefixes only. CC-01 to CC-14 are defined in ops-backup and remain unchanged.

### VA - Visit Access

| ID | Candidate | Bounded context | Key FRs / source | Notes |
|---|---|---|---|---|
| VA-01 | Token Pool Service | Visit access / ticketing | FR#1; Appendix A 1-1, 1-2; ADR-001 | Candidate SoR for pool balance, purchase, pack and family-pass SKUs. NOT a per-attraction ticket catalogue. Family-pass party rules live in the SKU. |
| VA-02 | Claim Minter | Visit access / ticketing | FR#1; ADR-002 | Signs QR payload (attractionId, sig, ts) at spend time. Reads live or cached pricing table. Burns pool on QR-reveal. NOT the pricing engine. |
| VA-03 | Checkpoint Verifier | Visit access / ticketing | FR#1; Appendix A 1-3; ADR-002; NFR_2, NFR_3 | Local verify only; no outbound HTTP at admit time. Seen-token cache. Publishes `validated`/`revoked` MQTT. Same role for 40 rides and up to 55 enclosures. |
| VA-04 | Kiosk Terminal | Visit access / ticketing | FR#1; Appendix A 1-3; ADR-002 | Top-up (pool recharge), dead-phone reprint (same QR payload), dispute desk. Does not mint new claims. |

### PM - Popularity Meter

| ID | Candidate | Bounded context | Key FRs / source | Notes |
|---|---|---|---|---|
| PM-01 | Popularity Aggregator | Popularity meter | FR#2D; ADR-002 `validated` events; NFR_8 | Owns the popularity feed (= CC-12 from ops-backup, ownership made explicit here). Fuses `validated` events + ride cycles + optional zone occupancy. Gaps show as unknown, NOT zero. Published to CC-06, pricing ADR, CC-11, AG-01. |
| PM-02 | Gap Monitor | Popularity meter | FR#2D; Appendix B | Detects MQTT device silence beyond threshold; flags count bucket as unknown. Prevents zero-count misread. May be a module inside PM-01. |

### AG - AI Guide / Return Incentive

| ID | Candidate | Bounded context | Key FRs / source | Notes |
|---|---|---|---|---|
| AG-01 | Itinerary / Next-Best-Experience Engine | Guest experience / AI | FR#2G; Appendix B | Opt-in. Suggests route using PM-01 wait-times and trail history. Advisory only. Degrades to static cards offline. No unsafe animal-area shortcuts (FR#2G; enforcement TBD). NOT a required part of a visit. |
| AG-02 | Win-Back Engine | Guest loyalty / AI | FR#2H; FR#2C; ADR-001 leftover pool | Post-visit async ML. Uses leftover pool balance (VA-01), visit history, cohort data. Generates next-best-visit offer with caps and opt-out. Measures 90-day return (primary metric). Requires guest identity (opt-in). |
| AG-03 | Trail Checkpoint Log | Guest experience | FR#2G; NFR_8 | Consent-gated. Records opt-in trail checkpoint scans for itinerary personalisation. NOT VA-03 (access checkpoint verifier). Not required for a visit. |

### AC - Animal Care (Board A)

New prefixes. CC-01..CC-05 are the legacy IDs from the ops-backup era; they are
retired below. See the CC->AC/IO mapping table.

| ID | Candidate | Bounded context | Key FRs / source | Notes |
|---|---|---|---|---|
| AC-01 | Animal Care Record Service | Animal care | FR#1; Appendix A 2-4 | Candidate SoR for health observations, feeding events, enclosure environment readings, keeper notes; per display. NOT the gate or popularity system. Does NOT receive ADR-002 `validated` events. |
| AC-02 | Alert Inbox | Animal care | FR#2I, FR#2J | Receives AI-generated suggestions with confidence scores and evidence; keeper accept / reject is the training signal; advisory only. |
| AC-03 | Anomaly Detection Engine | Animal care | FR#2I | ML + optional Vision; high-recall tuning (FR#2I: "High recall first"); human confirm before any welfare action; needs golden-case eval harness (NFR_13). |
| AC-04 | Piranha Population Estimator | Animal care | FR#2J | Colony-level Vision / ML estimate with confidence interval; periodic keeper census is the ground truth; must have error band, not a single point estimate. |
| AC-05 | Keeper Copilot / RAG | Animal care | FR#2L | Grounded retrieval on estate SOPs, keeper notes, live state; citations required; fails closed to "ask the vet / duty manager"; may be shared with IO-08 - open question. Must not advise on claim signing or key custody (ADR-002 open question). |

### IO - Intranet / Estate OS (Board B)

New prefixes. CC-06..CC-11 and CC-13..CC-14 are the legacy IDs from the ops-backup
era; they are retired below. CC-12 was already promoted to PM-01 in the guest-lane
session and is listed there.

| ID | Candidate | Bounded context | Key FRs / source | Notes |
|---|---|---|---|---|
| IO-01 | Estate Heatmap | Intranet ops | FR#1; FR#2D; NFR_11 | Candidate **projection** of occupancy and freshness; data age shown; MQTT gaps displayed as unknown, not zero. Consumes PM-01 output; does NOT own the count and does NOT subscribe directly to ADR-002 `validated` events. |
| IO-02 | Ride Operations Manager | Intranet ops | FR#1; Appendix A 2-2 | Status per ride (open / closed / delayed / evacuated), cycle count, queue proxy if instrumented; status changes are human commands only (NFR_7). |
| IO-03 | Asset and Maintenance Tracker | Intranet ops | FR#1; Appendix A 2-3; FR#2K | Inspections, work orders, incidents; consumes anomaly flags from FR#2K (predictive maintenance); yield-at-risk signal from popular-ride downtime uses PM-01 popularity data. |
| IO-04 | Staffing and Tasking Module | Intranet ops | FR#2F; Appendix A 2-5 | Roster vs predicted load; task assignment; AI staffing suggestions that duty manager accepts or rejects; consumes PM-01 popularity output (not raw ADR-002 events); includes keeper / ride-crew mobile entry with offline sync. |
| IO-05 | Incident Manager | Intranet ops | FR#1; Appendix A 2-7 | Guest injury, near miss, animal escape; escalation to duty manager; safety flows scripted; cross-links to animal module (Board A) on animal-escape events. |
| IO-06 | Congestion and Flow Forecaster | Intranet ops / popularity | FR#2E | 30-90 minute predicted occupancy (window already in FR#2E). Intranet shows it; the model is not the estate OS. Degrades to recency baseline when ingest is unhealthy (Appendix B). |
| IO-07 | Integration Bus (logical) | Platform | FR#1; Appendix A 2-6 | Consumes ticket events (ADR-001, ADR-002 via VA-03), MQTT telemetry, keeper app writes, maintenance writes, AI outputs; publishes operational state for UIs; analytical state belongs in the data warehouse. Broker placement is still an assumption. |
| IO-08 | Ops Copilot / RAG | Intranet ops | FR#2L | Grounded on estate SOPs, keeper notes, live ops state; citations required; display only - staff remain accountable (Appendix B); may be shared with AC-05 - open question. Must not advise on claim signing or key custody (ADR-002 open question). |

### CC-to-new-ID mapping (ops-backup era -> live IDs)

Inbound links from the ops-backup era that use CC-01..CC-14 refer to the
candidates below. Numbers are never reused for a different thing.

| Old ID | New ID | Candidate name |
|---|---|---|
| CC-01 | AC-01 | Animal Care Record Service |
| CC-02 | AC-02 | Alert Inbox |
| CC-03 | AC-03 | Anomaly Detection Engine |
| CC-04 | AC-04 | Piranha Population Estimator |
| CC-05 | AC-05 | Keeper Copilot / RAG |
| CC-06 | IO-01 | Estate Heatmap |
| CC-07 | IO-02 | Ride Operations Manager |
| CC-08 | IO-03 | Asset and Maintenance Tracker |
| CC-09 | IO-04 | Staffing and Tasking Module |
| CC-10 | IO-05 | Incident Manager |
| CC-11 | IO-06 | Congestion and Flow Forecaster |
| CC-12 | PM-01 | Popularity Aggregator (promoted to guest-lane session; see PM section above) |
| CC-13 | IO-07 | Integration Bus (logical) |
| CC-14 | IO-08 | Ops Copilot / RAG |

---

## Depth allocation

| Board | Depth | Rationale |
|---|---|---|
| V - Visit Access | Deep | ADR-001 and ADR-002 are already proposed. Board reconstructs decided ADRs, not future ones. This is the hard problem. |
| P - Popularity Meter | Next-ADR | Third in the ADR plan (adrs/README.md). `validated` events from ADR-002 are the primary input. |
| G - AI Guide | Shallow | Depends on PM-01 for wait-time data. Stays shallow until PM ADR exists. |
| A - Animal Care | Parallel, shallower than V | No dedicated ADR. Parallel workstream. Consumes ADR-001 / ADR-002 as upstream constraints; does not decide ticketing. Session reconstructed from requirements/. Next: keeper-records SoR ADR and alert-inbox ADR. |
| B - Intranet / Estate OS | Parallel, shallower than V | No dedicated ADR. Parallel workstream. Consumes ADR-001 / ADR-002 as upstream constraints; popularity data consumed from PM-01, not owned. Session reconstructed from requirements/. Next: same as A, plus MQTT ADR. |

Visit access is the hard problem. Boards A and B are parallel workstreams that
must now be consistent with the ticketing decisions. They are not pretending to
be as deep as Board V.

---

## Key decisions

| Decision | Existing link | Status | Boards |
|---|---|---|---|
| Token pool economy (pool vs per-attraction tickets) | [ADR-001](../adrs/ADR-001-use-home-bought-token-pool.md) | Proposed | V (deep); A, B consume as upstream constraint |
| Signed static QR; local verify; burn-on-reveal; MQTT as audit channel NOT wallet path | [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) | Proposed | V (deep); A, B consume as upstream constraint |
| Enclosure scans (ADR-002 validated events) feed PM-01 / Board P, NOT health records | [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) Conclusion | Existing ADR; confirmed on Board A | A |
| Intranet is a projection consumer of PM-01; it does NOT own popularity count or subscribe to raw validated events | [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md); [FR#2D](../requirements/2_FRs.md) | Requirements constraint + ADR-002 boundary | B |
| No remote-refund command on ops boards; kiosk is the gate-failure correction point | [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) (Weakened recoverability) | ADR-002 constraint; confirmed on Boards A, B | A, B |
| Signing-key architecture (client vs. server) is open; ops boards must not assume resolved key-custody | [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) Open questions | Open question | A, B |
| Popularity measurement: gate counts + ride cycles + gaps-as-unknown | [FR#2D](../requirements/2_FRs.md); [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) | Requirements; no ADR yet (PM ADR next) | P |
| Opt-in itinerary / next-best-experience; no animal-area shortcuts | [FR#2G](../requirements/2_FRs.md) | Requirements; no ADR yet | G |
| Win-back / next-best-visit; 90-day return metric; caps + opt-out | [FR#2H](../requirements/2_FRs.md) | Requirements; no ADR yet | G |
| Dynamic pricing: table writer on ADR-001 economy; NOT on this board | [ADR-001](../adrs/ADR-001-use-home-bought-token-pool.md) (Key differentiators) | Planned; see [adrs/README.md](../adrs/README.md) | - |

---

## Open questions (genuine TBDs)

These are questions from the boards, not fake answers. Each needs an ADR or a numbered
assumption before the corresponding component is built.

1. **MQTT broker topology and QoS level.** How many brokers on the estate? Where are
   the gateways? What QoS is needed for `validated` events? TBD; separate ADR.
2. **Kiosk output: paper reprint or screen-only?** ADR-002 open question. Before
   installation.
3. **Which of the 55 displays are paid checkpoints?** ADR-002 open. Before gate
   installation.
4. **Token expiry / no-refund / credit-only policy.** ADR-001 open. Before launch.
5. **Pack sizes and home-purchase prices.** ADR-001 open. Before shop build.
6. **Price-table max cache age.** ADR-001 open. Before beta.
7. **Fraud window between QR-reveal and checkpoint cache write.** ADR-002 open.
   Before launch.
8. **Dwell estimation method.** Entry+exit pair? Zone window? Rate approximation?
   Not decided. PM ADR must answer this.
9. **Gap threshold duration** (how long of silence before "unknown"?). PM ADR.
   Do not invent a number.
10. **Signing key distribution for claims.** Who signs? App client or server? TBD.
11. **Win-back frequency cap and comms provider.** FR#2H; TBD before launch.
12. **No-animal-area-shortcut enforcement mechanism** for AG-01. Route graph exclusion
    list? Human-curated map? TBD.
13. **Token expiry interaction with win-back.** If a guest's leftover pool expires before
    the win-back offer arrives, the hook is broken. Policy needed before AG-02 is built.

---

## Ops-backup note

[ops-backup/event_storming.md](ops-backup/event_storming.md) is now a one-paragraph
pointer. Board A and Board B are live in this index; their first-pass files and
digitised SVGs are in `assets/`. Duplicate files have been removed from ops-backup/.

CC-12 (Popularity Aggregator) was described in ops-backup as "ticketing-side"; the
guest-lane session made ownership explicit as PM-01. See the CC-to-new-ID mapping
table above.

---

## Iterations

### Iteration 1 - First pass (messy)

| Board | File |
|---|---|
| Visit Access / Ticketing | [assets/first-pass-visit.md](assets/first-pass-visit.md) |
| Popularity Meter | [assets/first-pass-popularity.md](assets/first-pass-popularity.md) |
| Return Incentive / AI Guide | [assets/first-pass-guide.md](assets/first-pass-guide.md) |
| Animal Care (Board A) | [assets/first-pass-animal.md](assets/first-pass-animal.md) |
| Intranet / Estate OS (Board B) | [assets/first-pass-intranet.md](assets/first-pass-intranet.md) |

Reconstructed first pass from ADRs and requirements/. Unordered sticky dump;
corrections and questions in-line; duplicates preserved. Boards A and B are
promoted from ops-backup/; they carry ADR-001 / ADR-002 alignment edits not
present in the original parked copies.

### Iteration 2 - Digitised

Colour key for all three boards:

| Colour | Sticky type |
|---|---|
| Yellow | Actor |
| Blue | Command |
| Orange | Domain Event |
| Magenta / purple | External / Policy |
| Red | Question / Hotspot |
| Green | Component Candidate (digitised boards only) |

Each SVG is self-contained with the colour key inline.

| Board | SVG |
|---|---|
| Visit Access / Ticketing | [assets/board-visit.svg](assets/board-visit.svg) |
| Popularity Meter | [assets/board-popularity.svg](assets/board-popularity.svg) |
| Return Incentive / AI Guide | [assets/board-guide.svg](assets/board-guide.svg) |
| Animal Care (Board A) | [assets/board-animal.svg](assets/board-animal.svg) |
| Intranet / Estate OS (Board B) | [assets/board-intranet.svg](assets/board-intranet.svg) |

Chronological phases run left to right. Component candidates (green) occupy the
rightmost phase on each board as the session output.
