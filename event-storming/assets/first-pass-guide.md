# First Pass - Board G: Return Incentive / AI Guide (reconstructed)

> **Reconstructed first pass from FR#2G, FR#2H, Appendix B, ADR-001 leftover-pool hook.**
> **Shallow board.** The goal is to make the leftover-pool hook visible and name the
> in-park trail and win-back flows. This is not a designed AI platform. In-app games
> are a shallow hotspot here, not a game platform.
>
> This board stays shallow until the popularity-meter ADR (PM board) produces the wait-time
> data that the itinerary engine needs.
>
> Sticky type in [BRACKETS]. Duplicates preserved.

---

## Session dump

---

**Post-visit: the leftover pool (ADR-001 hook)**

[ACTOR] Guest (identified OR anonymous; identity determines which downstream flows apply)

[EVENT] Visit ended (guest exits the estate)

[EVENT] Leftover pool persisted (ADR-001: "leftover pool is a return-visit hook rather
  than a refund queue"; unspent tokens remain in the wallet after the visit)

[EXTERNAL] Token Pool Service (VA-01, Board V) - source of the pool balance
  NOTE: the pool balance is the raw input to both the win-back engine and the
  itinerary engine. ADR-001 explicitly names "leftover-value path" as a driving
  criterion for the pool economy.

[QUESTION] Token expiry / no-refund / credit-only policy? TBD before launch.
  ADR-001 open: "Token expiry, no-refund, credit-only policy - before launch."
  This is the same question as Board V. Highlighted again here because it is the
  direct dependency for making the pool a return-visit hook.

[EVENT] Pool balance snapshot taken at visit end (reference point for win-back engine)

---

**In-park trail (opt-in only)**

[ACTOR] Guest who has opted in to trail / itinerary (separate from anonymous guest)
  NOTE: "Opt-in itinerary or checkpoint QR trail for guests who want it" (Appendix A 1-5).
  FR#2G: "Opt-in location or checkpoint scans." Never required to complete a visit.

[CMD] Opt in to itinerary / trail (at gate kiosk or in app on entry)

[EVENT] Guest opted in to trail

[CMD] Scan checkpoint trail QR (at a display or zone marker during the visit)
  [QUESTION] Is the trail QR the same as the access QR?
    ADR-002 burns the access QR on reveal. The trail QR is a different, separate
    concept - it is not an access token. It identifies the guest's trail progress.
    Leave as open design question.

[EVENT] Trail checkpoint scan recorded
  NOTE: consent-gated. NOT the same event as the `validated` access scan (Board V).
  This is a separate, opt-in record.

[EVENT] Itinerary generated (AI suggests walking order; inputs: wait times from PM-01,
  trail progress so far)
  NOTE: advisory only. No unsafe routes through animal areas (FR#2G).

[EXTERNAL] Popularity Aggregator (PM-01, Board P) - source of wait-time and rank data
  NOTE: PM-01 must exist before AG-01 can make useful suggestions. This is the
  sequencing dependency: PM board before a production AG-01.

[EVENT] Next-best-experience suggested (in-park nudge: "skip ride 7 - long queue,
  go to piranha house - short wait")
  NOTE: FR#2G. "Works as static 'start here' cards when connectivity is poor."

[EVENT] Suggestion accepted / dismissed (feedback signal for AI improvement)

[QUESTION] Delivery channel for in-park suggestions: push notification, kiosk screen,
  or in-app card? TBD.

[QUESTION] When offline, itinerary falls back to printed / kiosk cards (FR#2G).
  How is the static fallback generated - last-generated plan or pre-printed card?
  Design decision TBD.

[HOTSPOT] No unsafe animal-area shortcuts (FR#2G constraint).
  Enforcement mechanism: route graph exclusion list? Human-curated zone map?
  TBD. This is a safety constraint, not a nice-to-have.

[HOTSPOT] In-app games: the ADR plan (adrs/README.md) mentions "engaging games in
  the app" under the return-incentive step. Keep as a SHALLOW hotspot only.
  This is NOT a designed game platform on this board. If games are included,
  they are a future shallow ADR, not designed here.

---

**Win-back trigger (post-visit, async)**

[ACTOR] Returning guest (identified; has opted in to marketing; has a membership key
  or email)
  NOTE: anonymous first-visit guests CANNOT receive win-back offers. Identity
  (opt-in) is the hard prerequisite.

[ACTOR] Commercial / Countess (sets win-back parameters, caps, and campaign configs)
  NOTE: not a field actor; configures the engine, does not receive offers.

[CMD] Configure win-back campaign (Commercial actor; sets offer type, channel, caps,
  experiment variant)

[EVENT] Win-back trigger evaluated (post-visit; async ML job)
  - Inputs: leftover pool balance, visit history, experiment cohort (FR#2H + FR#2C)
  - Timing: after the visit, not inline with the gate or the app

[EVENT] Win-back offer generated (ML output: next-best-visit reason, channel, send-time)
  [QUESTION] Win-back offer type: "new feeding slot, weekday family pass, membership"
    (Appendix B examples). Which of these is available in v1? TBD.

---

**Offer channel and return measurement**

[EVENT] Win-back offer sent (email or push; frequency caps applied)

[EXTERNAL] Comms channel (email provider / push notification service)
  [QUESTION] Which comms provider? TBD. Not an estate-platform decision today.

[EVENT] Win-back offer accepted (guest returns; opens the offer link / uses voucher)

[EVENT] Win-back offer declined / unsubscribed

[EVENT] 90-day return measured (outcome metric; FR#2H verification)
  NOTE: "Measure 90-day return" is the stated verification method in Appendix B
  and FR#2H. The >=30% returning-visitor figure in the business-goals file is
  a TEAM HYPOTHESIS, not a figure from the kata brief. Do not cite it as a
  brief number.

[QUESTION] Win-back frequency cap and opt-out mechanism? Not in the brief.
  TBD before launch. FR#2H: "Frequency caps and opt-out."

[QUESTION] Is the itinerary engine (in-park) the same service as the win-back engine
  (post-visit)? Different data, different timing. Probably separate. TBD.

---

## Component candidates emerging

**AG-01 Itinerary / Next-Best-Experience Engine**
  Opt-in; suggests walking order and next attraction using PM-01 wait-time data and
  prior trail checkpoint scans. Advisory only. Degrades to static "start here"
  cards when offline or when PM-01 is unavailable. No unsafe animal-area shortcuts
  (FR#2G). Not a required part of a visit.

**AG-02 Win-Back Engine**
  Post-visit async ML. Inputs: leftover pool balance (VA-01), visit history, cohort
  data (FR#2C). Generates next-best-visit offer. Honours frequency caps and opt-out.
  Measures 90-day return as the primary outcome metric. Requires identity (opt-in).

**AG-03 Trail Checkpoint Log**
  Records opt-in checkpoint trail scans for personalisation and win-back signal.
  Consent-gated. SEPARATE from VA-03 (access checkpoint verifier) - these trail
  scans are advisory and voluntary, not required for admission. Not in the gate path.

---

## Boundary decisions from this board

1. AG-03 is NOT VA-03. Trail scans are opt-in and advisory; access scans are mandatory.
2. AG-01 and AG-02 are advisory; a guest can complete a full visit without either.
3. In-app games are a shallow hotspot; no game platform is designed here.
4. Win-back requires identity (opt-in); anonymous first-visit guests are out of scope.
5. 90-day return is the measurement metric (FR#2H). The >=30% figure is NOT the brief.
6. AG-01 depends on PM-01. Board P must be ADR-ed before AG-01 is built.

---

## Corrections from this pass

1. Initial "trail scan" was conflated with the access checkpoint scan. Corrected:
   trail scans are a separate, consent-gated, non-access record (AG-03 vs VA-03).
2. Win-back trigger was initially placed in-park during the visit. Corrected:
   win-back is post-visit and async. In-park suggestions stay in AG-01.
3. "No animal-area shortcuts" elevated from a note to a hotspot (safety constraint).

---

*Digitised board: [board-guide.svg](board-guide.svg)*
