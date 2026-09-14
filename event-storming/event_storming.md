# Event Storming - Guest Lane (current)

**This is the current derivation-chain work.** Three boards, one session:
visit access (deep), popularity meter (next ADR), AI Guide / return incentive (shallow).

Ops boards (Animal Care, Intranet / Estate OS) are **backup** - a parallel prior session.
They are not the current focus: [ops-backup/event_storming.md](ops-backup/event_storming.md).

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

## Three boards

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

---

## Depth allocation

| Board | Depth | Rationale |
|---|---|---|
| V - Visit Access | Deep | ADR-001 and ADR-002 are already proposed. Board reconstructs decided ADRs, not future ones. |
| P - Popularity Meter | Next-ADR | Third in the ADR plan (adrs/README.md). `validated` events from ADR-002 are the primary input. |
| G - AI Guide | Shallow | Depends on PM-01 for wait-time data. Stays shallow until PM ADR exists. |

Ops boards: Animal Care (CC-01..CC-05) and Intranet / Estate OS (CC-06..CC-14) are in
ops-backup/. Neither ops board is the current derivation-chain work.

---

## Key decisions

| Decision | Existing link | Status |
|---|---|---|
| Token pool economy (pool vs per-attraction tickets) | [ADR-001](../adrs/ADR-001-use-home-bought-token-pool.md) | Proposed |
| Signed static QR; local verify; burn-on-reveal; MQTT as audit channel NOT wallet path | [ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) | Proposed |
| Popularity measurement: gate counts + ride cycles + gaps-as-unknown | [FR#2D](../requirements/2_FRs.md); [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) | Requirements; no ADR yet (PM ADR next) |
| Opt-in itinerary / next-best-experience; no animal-area shortcuts | [FR#2G](../requirements/2_FRs.md) | Requirements; no ADR yet |
| Win-back / next-best-visit; 90-day return metric; caps + opt-out | [FR#2H](../requirements/2_FRs.md) | Requirements; no ADR yet |
| Dynamic pricing: table writer on ADR-001 economy; NOT on this board | [ADR-001](../adrs/ADR-001-use-home-bought-token-pool.md) (Key differentiators) | Planned; see [adrs/README.md](../adrs/README.md) |

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

[ops-backup/event_storming.md](ops-backup/event_storming.md) contains two boards:
**Board A - Animal Care (Keepers)** and **Board B - Intranet / Estate OS (Duty Manager,
Ride Crew)**. Component candidates CC-01 to CC-14 are defined there. CC-12 (Popularity
Aggregator) is described in ops-backup as "ticketing-side"; this session makes the
ownership explicit as PM-01. The intranet heat map (CC-06) consumes PM-01; it does not
own the count.

Links in ops-backup/event_storming.md to ../../adrs/ and ../../requirements/ remain
correct and are not edited.

---

## Iterations

### Iteration 1 - First pass (messy)

| Board | File |
|---|---|
| Visit Access / Ticketing | [assets/first-pass-visit.md](assets/first-pass-visit.md) |
| Popularity Meter | [assets/first-pass-popularity.md](assets/first-pass-popularity.md) |
| Return Incentive / AI Guide | [assets/first-pass-guide.md](assets/first-pass-guide.md) |

Reconstructed first pass from ADRs and requirements/. Unordered sticky dump;
corrections and questions in-line; duplicates preserved.

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

Chronological phases run left to right. Component candidates (green) occupy the
rightmost phase on each board as the session output.
