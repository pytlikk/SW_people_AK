# ADR-001 — Use a home-bought token pool, spend at live park prices

## Date

2026-09-11

## Status

Proposed

## Context

Von Digitalis has 40 amusement rides and 55 animal displays/enclosures — up to 95 paid checkpoints if every display is gated — serving 5000 visitors/day (locked board). Estate Wi-Fi is patchy; MQTT on the intranet with a few Wi-Fi-gateway patches is the assumed backbone until its own ADR exists. Kiosks handle top-up and dead-phone rescue.

The board wants dynamic pricing later (popular attractions cost more) to spread usage and raise profit, a return-visit incentive (leftover value should bring people back), and membership products (annual, daily, path-based daytime). Those products sit **on** a spend mechanism; they are not a second admit model.

This record answers the economy question: what is sold **from home** — a fungible token pool, or a catalogue of ride/display tickets? How a spend is presented at the checkpoint is ADR-002.

Two units must stay distinct:

- **Pool** — fungible tokens in the wallet, bought at home (or topped up at a kiosk). Not bound to an attraction.
- **Claim** — the signed QR in ADR-002. Minted when the visitor spends from the pool at an attraction, at that moment’s token price.

The checkpoint never reads the pool. Shop, wallet, and pricing table change together; the gate only verifies a claim. That is the contract with ADR-002.

This record does **not** decide: token pack sizes or prices (TBD); which of the 55 displays are paid (TBD); the pricing algorithm or AI; membership SKU structure; MQTT broker placement.

**Foreclosed here:** selling per-attraction tickets from home. The gate therefore never has to reprice or version an already-sold ride ticket. A claim is minted at spend time (ADR-002); until then the wallet holds only pool balance.

## Evaluation criteria

- **Repricing agility (driving)** — changing an attraction’s token price mutates 0 already-sold wallets and 0 already-minted claims (CI).
- **Operability / SKU footprint (driving)** — home catalogue has no `attractionId`; adding or dropping a paid checkpoint is a pricing-table row, not a shop redeploy (CI). Three-person team.
- **Guest flexibility** — no home pre-binding of tokens to attractions; spending happens at or after entry. Flexibility UX specifics TBD.
- **Leftover-value path** — token balance persists after the visit; expiry policy, no-refund or credit-only terms TBD, required before launch.
- **Product extensibility** — membership, daily, and path-based daytime products must not fork checkout or the admit path; the checkpoint still consumes an ADR-002 claim for every spend.

## Options

- **Option A — Per-attraction tickets from home:** visitor buys one ticket per attraction (anytime-today or timed slot) before arriving. Catalogue size scales with chargeable checkpoints.
- **Option B — Token pool from home, spend at live prices (chosen):** visitor buys a pool of tokens before visiting. Each attraction has a token price in a server-side table; the app mints an ADR-002 claim when the visitor spends, at the live price.
- **Option C — Hybrid: pool + optional attraction lock-in:** visitor buys a pool and may optionally pre-bind a few must-do attractions at a locked price or slot.
- **Option D — Unlimited day pass:** one flat-price product admits the visitor to all attractions for the day. No per-attraction cost lever.

| | Repricing agility (driving) | SKU operability (driving) | Guest flexibility | Leftover-value path |
|---|---|---|---|---|
| A tickets from home | Fail — price and attraction lock at sale | Fail — catalogue grows with checkpoints | Fail — allocation is on the booking screen | Weak — unused tickets want refunds, not a return visit |
| B token pool | Pass — table update, issued pool unchanged | Pass — one currency; pack sizes are product SKUs of that currency | Pass — spend after arrival | Pass — leftover pool stays in the wallet |
| C hybrid | Partial — locked slice cannot reprice | Worse than B — lock-in SKUs and refunds | Better certainty for must-dos | Same as B for the unlocked remainder |
| D unlimited day | Fail — no per-attraction lever | Pass — one product | Pass — go anywhere | Fail — no leftover unit to bring people back |

Product extensibility (not a table column): B and C keep one admit path; A grows a ticket catalogue; D *is* the day product and removes the per-attraction lever.

## Decision

**Use Option B: home-bought token pool, spent at live park token-prices.**

Repricing agility and SKU operability decide it. The shop sells a currency (and a few pack sizes of that currency). Attraction prices live in a server-side table; ops updates the table without touching wallets already sold. Option A fails both drivers: a ticket locks attraction and price at home, so a later price change means refunds or versioning already-sold rights, and the catalogue scales with 40 rides plus however many of the 55 displays are paid. Option D fails repricing agility (no per-attraction lever) and leftover-value (nothing left to bring people back). Option C is viable but adds lock-in UI, capacity holds, and partial refunds; that cost is not justified until the popularity-meter ADR shows which attractions actually need advance protection.

## Key differentiators

- **One currency covers every paid checkpoint.** Adding or dropping a paid attraction is a pricing-table row, not a new ticket product. Pack sizes (if any) are SKUs of the same currency — TBD, not 40+55 ride tickets.
- **Repricing does not touch wallets already sold.** Pool balance is tokens, not attraction rights. The next spend reads the live table; claims already minted (ADR-002) keep the price they were minted at.
- **Leftover tokens survive the visit.** A later return-incentive ADR can credit the same wallet without a new ticket type.
- **Membership and path-based daytime options layer on the pool.** They pre-load or discount tokens; the checkpoint still consumes an ADR-002 claim. A membership that sets token cost to 0 still mints and burns a signed QR — it does not bypass the gate.
- **Dynamic pricing is a table writer, not a second economy.** When the pricing ADR lands, it writes the same table the shop, app, and kiosks already read.

## Consequences

### Positive

- Repricing an attraction is a table update: no re-issue, no refund of the pool, no shop redeploy (repricing agility).
- Shop catalogue stays a currency plus pack sizes, not one SKU per checkpoint (operability).
- Visitors allocate spend after they arrive (guest flexibility).
- Leftover pool is a return-visit hook rather than a refund queue (leftover-value).
- Membership and path products do not fork checkout or the checkpoint (product extensibility).

### Negative

- Guests who wanted a locked ride at a locked home price do not get that in v1. If dynamic pricing is active, a popular attraction can cost more tokens than they saw at breakfast.
- Balance UX (low-balance warnings, kiosk top-up, “tokens are not a ride ticket”) is work Option A never needed.
- If the app cannot refresh the pricing table, QR-reveal may mint at a cached price. The checkpoint still admits on the signed claim (ADR-002); dispute is a kiosk problem, not a gate outage.
- Families planning a must-do list at home have no reservation hold. Queues are managed by price and popularity later, not by a timed ticket.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Price-table staleness | App shows a cached cost; live table differs at QR-reveal | Max cache age TBD. Show the minted deduction on confirm. Gate admits on the signed claim, not the quoted price. Kiosk is the dispute desk. |
| Guest confusion | Visitors think they bought a ride, then see a different token cost in the park | First-purchase copy: pool ≠ ticket. Show live cost before QR-reveal. Kiosk staff. |
| Pool leftover / refund pressure | Large unused balances become cash-refund demand instead of a return visit | Token expiry / no-refund / credit-only policy TBD before launch; say it at purchase |
| Pricing-table write error | Ops sets cost 0 or prices the wrong enclosure | Two-step confirm, audit log, min-price guard. Cost 0 is allowed only as an explicit membership/path product. |
| Must-do certainty | Some attractions may need a hold so guests do not miss them | Revisit Option C after the popularity-meter ADR has data. No lock-in in v1. |
| Membership vs ADR-002 | A “free with membership” ride must not skip the signed claim | Pricing table can be 0 or discounted; checkpoint still verifies an ADR-002 QR |

## Verification

These tests guard the economy; they do not replace ADR-002’s offline gate tests.

- **Shop does not sell attraction tickets (CI):** catalogue has pool / pack SKUs only — no `attractionId` on a home purchase.
- **QR-reveal deducts live or cached price (CI):** deduction applies the pricing-table amount at QR-reveal time; if the table is unreachable, the last-cached amount applies. Both paths are tested. Gate does not re-check price — it verifies only the signed claim.
- **Wallet immutability on reprice (CI):** changing an attraction’s token price does not alter existing pool balances or already-minted ADR-002 payloads.
- **Leftover persists (CI):** unspent pool remains after a simulated visit; no automatic zero-out unless a later product ADR sets expiry.
- **Table down does not close the gate (CI):** app may show a stale price; checkpoint still verifies the signed claim locally (ADR-002).
- **Scheduled review:** revisit Option C after the popularity-meter ADR has a season of data.

**Open questions**

- Token expiry, no-refund, credit-only policy — before launch.
- Pack sizes and home-purchase prices — before shop build.
- Pricing-table max cache age — before beta.
- Which of 55 displays are paid — before gate installation.

**Revisit triggers**

- Revisit Option C if post-launch visitor feedback shows must-do attraction miss-rate is a retention blocker; threshold agreed before first season review.
- Reverse the pool decision only if a regulatory or payment-scheme ruling requires per-attraction rights at the point of sale (forecloses the "pool = currency" model).

## Conclusion

A home-bought token pool is the least-worst fit for an estate that must reprice attractions without rebuilding tickets, keep the shop operable for a three-person team, leave leftover value for a return visit, and layer membership on one admit path. Per-attraction tickets from home lock price and multiply SKUs. An unlimited day pass removes the usage lever. Hybrid lock-in waits on popularity data.

Related: [ADR-002](ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) (how a spend becomes a gate claim); popularity meter; dynamic pricing; return incentive / AI Guide. Token expiry and refunds are a product decision, not an ADR.
