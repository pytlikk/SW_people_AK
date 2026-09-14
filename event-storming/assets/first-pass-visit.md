# First Pass - Board V: Visit Access / Ticketing (reconstructed)

> **Reconstructed first pass from ADR-001, ADR-002, requirements/. Not a photographed workshop.**
> No physical sticky-note session. This document simulates the messy first pass that
> would emerge from a facilitated event-storming session run against the two committed ADRs
> and the requirements documents. Unordered dump; duplicates expected; corrections and
> questions written in-line as they arose.
>
> Scope: **guest lane - ticketing and access.** The ops storm (Animal Care, Intranet) is
> separate work in ops-backup/. This board reconstructs what ADR-001 and ADR-002 decided.
> Do not reverse those ADRs from here.
>
> Sticky type in [BRACKETS]. Order is approximate within each reading block.

---

## Session dump

---

**Reading ADR-001: what is sold from home?**

[ACTOR] Guest

[CMD] Browse ticket options online (or on kiosk)

[CMD] Select token pack

[QUESTION] Are we selling "token packs" or "ride tickets"?
  - ADR-001 is explicit: pool of fungible tokens, not per-attraction tickets.
    The gate never sees the pool; it only verifies a signed claim (ADR-002).
    Keep "token pack" as the guest-visible label.

[CMD] Select token pack (confirmed label - pool currency SKU)

[CMD] Choose family pass
  - [QUESTION] What is a family pass exactly?
    Appendix A 1-2: encodes party rules (adult/child count, validity window,
    timed-experience entitlements if any). Not a second economy - it is a pool SKU
    with rules attached. SKU shape is TBD (ADR-001 open: pack sizes and prices TBD).

[EVENT] Token pool purchased
[EVENT] Family pass created (party rules encoded)

[DUPLICATE] "Token pool purchased" and "family pass created" - are these the same event?
  CORRECTION: keep both as separate stickies. A family pass IS a pool purchase,
  but it carries party rules the gate can check. Two events, one flow.

[EXTERNAL] Payment Provider / PCI-scoped
  - NFR_6: estate does not store raw card data. Payment is PCI-scoped;
    the estate platform receives a confirmation, not the card details.

[EVENT] Payment confirmed (payment-provider acknowledgement)
[EVENT] Token pool (or family pass) created in wallet

[DUPLICATE] "Payment confirmed" vs "token pool created" - adjacent events from
  different viewpoints (provider vs domain). Keep both in first pass.

[QUESTION] Pack sizes? TBD. ADR-001 open: pack sizes and home-purchase prices TBD before shop build.

[QUESTION] Family pass party rules shape: 2+2, 2+3, weekday vs weekend? TBD.
  ADR-001 says "SKU shape still TBD."

---

**Reading ADR-001 continued: wallet and spending**

[EVENT] Pool balance available in wallet (guest opens app; sees token count)

[CMD] View pool balance

[EXTERNAL] Estate mobile app / web (guest device)

[EXTERNAL] Pricing table (server-side; this is NOT the pricing AI; the pricing
  AI is a "table writer on ADR-001" per adrs/README.md - later ADR)

[CMD] View attraction token price (before spending at ride or display)

[QUESTION] What if the pricing table is unreachable when the guest opens the app?
  - ADR-001 negative consequences: "If the app cannot refresh the pricing table,
    QR-reveal may mint at a cached price." ADR-001 open: max cache age TBD.
  - The gate still admits on the signed claim regardless of price staleness (ADR-002).

[QUESTION] Price-table max cache age? TBD. ADR-001 open question.

[CMD] Tap QR-Reveal at ride / display (the spend moment)
  - App mints a signed claim at the live or cached price. Pool is debited at this
    moment, NOT at the gate scan. ADR-002: "Burn on QR-reveal is the wallet write
    (no gate network)."

[EVENT] Claim minted (signed payload: attractionId, timestamp, signature)
  [QUESTION] Who signs the claim - the app client or the server? Signing key
    distribution is not decided in ADR-002. Open question.

[EVENT] Pool balance decremented (burn-on-reveal)
  CORRECTION: "burn-on-reveal" is the ADR-002 term. Pool debits at QR-reveal time,
  not at gate scan. Gate never touches the pool. ADR-002 driving criterion for this:
  gate must admit with no network round-trip.

[QUESTION] Does the pool debit happen client-side on tap or on server confirm?
  If the app is offline at QR-reveal time, what is the debit path? Not settled
  in ADR-002. Leave as open question.

---

**Reading ADR-002: checkpoint scan and verify**

[ACTOR] Guest at ride gate or enclosure entrance (same actor, different location)

[CMD] Present QR at checkpoint scanner (hold phone under camera; or paper reprint)

[EVENT] QR scanned (checkpoint camera reads the barcode)

[EVENT] Claim verified locally
  - ADR-002 driving criterion: "offline at the checkpoint - 0 refusals caused by
    network error for a valid issued token." No outbound HTTP. No live API call.

[EVENT] Gate / enclosure admitted

[EVENT] Claim burned (written to local seen-token cache)
  CORRECTION: "burned" in context of ADR-002 = the checkpoint writes the claim
  to its local seen-token cache. This is the replay protection. The checkpoint
  device has no live database connection for this step.

[QUESTION] What if the scan fails - glare, dirty camera, outdoor lighting, cracked screen?
  - ADR-002: kiosk reprint is the recovery path. "Failed gate read after QR-reveal
    leaves the app in pending; the visitor must reach a kiosk to correct it - no
    self-service path."

[QUESTION] Which of the 55 displays are paid checkpoints? TBD. ADR-002 open.

[QUESTION] Kiosk output: screen-only or paper reprint? TBD. ADR-002 open.

[QUESTION] Fail-scan rate threshold that triggers NFC/wristband review? TBD.
  ADR-002 says: "revisit triggers" when fail-scan rate exceeds ops-agreed threshold.

---

**MQTT audit and popularity channel**

[EVENT] `validated` MQTT event published (by checkpoint verifier after admit)

[EVENT] `revoked` MQTT event published (on replay detection or rejected claim)

[EXTERNAL] MQTT Broker (topology TBD; assumption, not decided; separate ADR needed)
  NOTE: MQTT is NOT the wallet path. ADR-002 is explicit:
  "Local seen-token cache is the admit/reject record. MQTT validated/revoked is
  for popularity, audit, and extra lanes - not for admit, not for the app wallet."

[EXTERNAL] Popularity Aggregator (Board P - PM-01) - consumer of validated events
  - The `validated` event carries attractionId and timestamp; PM-01 uses these
    to count entries and estimate dwell. See Board P.

[QUESTION] Who monitors `revoked` events for fraud patterns? Kiosk staff? Security
  module? Not decided. Separate concern from admit logic.

[QUESTION] MQTT broker topology? QoS level? Number of brokers on estate? TBD.
  MQTT is a working assumption (budget exists; placement is a future ADR per Assumptions).

---

**Kiosk rescue path**

[ACTOR] Guest (dead phone, empty battery, or low balance)

[CMD] Approach kiosk

[CMD] Request top-up at kiosk (add tokens to pool via kiosk payment)

[CMD] Request reprint at kiosk (phone is dead; same QR not yet burned at checkpoint)

[EVENT] Kiosk top-up completed (pool balance recharged; kiosk issued payment confirmation)

[EVENT] Kiosk reprint issued (same QR payload; once; kiosk checks claim state)
  NOTE: kiosk reprint produces the SAME QR payload - it does not mint a new claim.
  ADR-002: "Kiosk reprint is the same QR if the phone is dead."

[QUESTION] Kiosk output: paper QR printout or screen display only? TBD. ADR-002 open.

[QUESTION] Token expiry / no-refund / credit-only policy? TBD before launch. ADR-001 open:
  "Token expiry, no-refund, credit-only policy - before launch." This is a product
  decision, not an ADR-001 or ADR-002 decision.

[QUESTION] Fraud window between QR-reveal and checkpoint cache write? TBD. ADR-002:
  "The fraud window is the interval between QR-reveal (wallet debit) and the
  checkpoint cache write; length of that window is TBD and is the primary integrity risk."

[QUESTION] Pending->used age-out duration? TBD. ADR-002 open.

---

## Component candidates emerging

As events were laid out, four bounded roles became clear from this board:

**VA-01 Token Pool Service**
  Owns the pool balance, home purchase, pack and family-pass SKUs, and pool-to-claim
  deduction. The shop sells a currency (pool), not a catalogue of per-attraction tickets
  (ADR-001). Family pass party rules live in the SKU, not in a separate access product.

**VA-02 Claim Minter**
  Signs the QR payload (attractionId, timestamp, signature) at spend time. Reads the
  live or cached pricing table. Debits the pool on QR-reveal (burn-on-reveal). Not a
  pricing engine - it reads the table written by the later pricing ADR.

**VA-03 Checkpoint Verifier**
  Local verify only - no outbound HTTP or TCP to the internet DB (ADR-002 driving
  criterion). Maintains the local seen-token cache. Publishes `validated`/`revoked`
  MQTT events as a side effect of admit/reject. Serves both 40 rides and up to 55
  enclosures with the same role.

**VA-04 Kiosk Terminal**
  Top-up (recharges pool via payment), reprint (same QR payload, once), and dispute
  desk for failed gate reads. Recovery path for the dead-phone case.

---

## Boundary decisions confirmed from this board

1. VA-03 has no outbound HTTP/TCP to the internet DB at admit time (ADR-002).
2. MQTT `validated`/`revoked` is OUTPUT of VA-03; it does NOT update the app wallet.
3. VA-01 / VA-02 are NOT the pricing engine; dynamic pricing is a table writer (later ADR).
4. Family pass party rules live in the pool SKU, not in a separate access product.
5. Kiosk reprint reuses the same QR payload; it does not mint a new claim.
6. The `validated` event is the primary input to Board P (popularity meter).

---

## Corrections from this pass

1. "Token pool purchased" and "family pass created" kept as separate events (different
   party-rule encoding, not just a different price).
2. "Burn-on-reveal" clarified: pool debits at QR-reveal, not at gate scan.
3. MQTT is the audit/popularity channel, NOT the wallet path - corrected after
   initially placing a "wallet updated via MQTT" sticky.
4. Kiosk reprint is the same QR payload, not a new minted claim.

---

*Digitised board: [board-visit.svg](board-visit.svg)*
