# ADR-0002 - Admit on a server-signed entitlement verified locally at the perimeter gate

## Date

2026-09-14

## Status

Proposed

## Context

Guests buy admission - individual or family pass - on the web before they travel, or at an on-site kiosk (Appendix A 1-1). The gate then has to admit them across a sprawling estate with patchy Wi-Fi, at a volume rising from 5,000 to 15,000 visitors a day (NFR_1).

A family pass is not a ticket count. It grants **access rights**: a party shape such as 2 adults + 2 children, a validity window, and possibly timed experiences (Appendix A 1-2). Admission is enforced as "N of M of this party have entered", not as "this SKU was scanned".

Two constraints collide here and they decide the record:

- **A paid, unexpired entitlement must admit the guest without a model vote and without a network round trip.** That is stated as a constraint, not a preference ([4_Assumptions and constraints](../requirements/4_Assumptions%20and%20constraints.md)), and NFR_2 targets p99 gate redeem success whenever the local gateway is up, independent of cloud.
- **The gate is the one place a queue of 15,000 people a day physically forms.** Throughput failure here is the failure the Countess will see.

Scope in v1 is the **estate perimeter**. Rides and enclosures are counted, not gated (Appendix A 1-3): a scan at ride 12 or the piranha house feeds popularity (FR#2D) and never decides admission. Zone gating - the venomous house as a timed entitlement - is deferred to [Appendix C](../requirements/Appendix%20C_%20Future%20scope.md). That keeps the number of admit-critical devices at a handful of perimeter lanes instead of 95 outdoor checkpoints.

This record decides **how a guest presents an already-purchased entitlement, and how the gate decides**. It does not decide pricing ([ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)), what the MQTT transport looks like ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)), or the guest identity/membership model.

**Foreclosed here:** any admit decision that requires the cloud, the estate WLAN, or the guest's phone to have signal at the gate.

## Evaluation criteria

- **Offline admit (driving)** - zero refusals caused by a network error for a validly issued entitlement. This is the NFR_2 commitment.
- **Throughput at 15,000/day (driving)** - the perimeter must clear a morning arrival peak. Scan-to-decision must be a local verification, not a request.
- **Operability for a three-person team** - no wallet-certification programme, no inventory to manage, commodity hardware.
- **Integrity** - forged entitlements rejected; replay bounded; party counts not exceeded.
- **Family-pass semantics** - a party arriving together, or in two groups, must both work.

## Options

- **Option A - Server-signed static QR (chosen)**: the purchase service signs an entitlement claim; the guest shows it on a phone screen or a kiosk printout; the gate verifies the signature locally and decrements the party count in a local ledger.
- **Option B - Rotating barcode (SafeTix-style)**: the payload redraws on an interval so a screenshot cannot be reused; the phone needs a refresh signal.
- **Option C - Phone NFC (Apple VAS / Google Smart Tap)**: tap a certified reader.
- **Option D - RFID/NFC wristband bound at a kiosk**: tap at the gate; the best family and children's UX.
- **Option E - Cloud lookup at the gate**: scan an opaque reference, ask the ticketing API.

| | Offline admit (driving) | Throughput (driving) | Operability | Integrity | Cost |
|---|---|---|---|---|---|
| A Signed static QR | Pass - local signature verify, no network | Partial - optical scan is slower than a tap, but perimeter lanes can be added | Pass - commodity cameras, no certification | Partial - screenshot shareable until the party count is spent | Lowest |
| B Rotating barcode | Fail - needs a refresh to reach the phone, which is exactly what patchy Wi-Fi denies | Partial - same as A | Pass | Pass - strong screenshot defence | Medium |
| C Phone NFC | Pass - if keys are pre-loaded | Pass - fastest | Fail - certified reader programme for a three-person team | Pass | High |
| D Wristband | Pass | Pass | Fail - band stock, kiosk binding, and a logistics function we do not have | Pass | High |
| E Cloud lookup | Fail - the stated constraint | Fail - a queue that stops when the link does | Pass | Pass | Low |

Not options: biometric or face entry as the admit radio (identity binding at this throughput is unreliable and the privacy posture in NFR_8 prefers counts over biometrics); BLE/UWB proximity admit.

## Decision

**The purchase service signs an entitlement claim at purchase time. The gate verifies it locally and admits without a network call.**

The claim is minted where connectivity already exists - at purchase, on the web or at a kiosk - which is the property that makes everything else work.

| Field | Meaning |
|---|---|
| `entitlementId` | The access right, not the SKU |
| `partyShape` | Admissions granted, e.g. 2 adult + 2 child |
| `validFrom` / `validTo` | The window; a day pass is a day |
| `scope` | `estate` in v1; the field exists so zone gating later needs no new payload type |
| `signature` | Estate signing key; verified offline against a pre-distributed public key |

Four rules follow.

**The local gate cluster is the admit authority, and it holds a party ledger.** A family pass is not single-use. It is decremented: three of four admitted now, the fourth an hour later. The ledger, not the cloud, answers "how many of this party are already in".

**Redemptions are published to MQTT after the decision, never before it.** The `redeemed` event is audit and popularity input ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md), FR#2D). Admission never waits for it.

**The private key never leaves the cloud.** Guests receive a signed claim; they cannot mint one. Gates hold only the public key, so a stolen or tampered gate can be made to admit wrongly but cannot forge entitlements estate-wide.

**Cross-gate double-spend is accepted and bounded, not solved.** Within a gate cluster the ledger is authoritative. Across partitioned clusters, the same family pass could over-admit until reconciliation. The bound is the party size - a 2+2 pass can over-admit by at most four people - and the estate would rather admit a paying family twice than refuse them once at the gate. The cloud reconciles and flags repeat patterns for commercial review.

Offline admit and throughput decided it. Rotating barcodes (B) fail the driving criterion outright: they defend against screenshots by requiring the connectivity this estate does not have. NFC (C) and wristbands (D) are better at the gate and lose on operability - certified readers or band inventory are programmes, and there are three of us. Cloud lookup (E) is what the constraint forbids.

## Key differentiators

- **Nothing has to work at the gate except a camera and a signature check.** No phone signal, no estate WLAN, no cloud.
- **The party ledger makes a family pass behave the way families behave** - arriving in two groups is normal, not an error state.
- **No key material in the guest's hands.** The old failure mode of client-side minting does not exist because the claim is signed where the payment already happened.
- **Kiosk reprint is the same artefact.** A dead phone is a paper problem, not a support case.
- **The payload survives a later wristband or NFC roll-out.** `scope` and `partyShape` are what a reader would carry too, so the economy does not get renegotiated.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Availability | **Improved (driving)** | The admit path has no remote dependency. A cloud or WLAN outage does not close the estate (NFR_2). |
| Performance | **Improved (driving)** | Scan-to-decision is a local signature verification and a ledger read - microseconds of compute against a p95 budget of 2 seconds (NFR_3). |
| Operability | **Improved** | Commodity cameras, no certification programme, no band inventory. |
| Testability | **Improved** | The gate can be tested end to end before any cloud or MQTT infrastructure exists. |
| Evolvability | **Improved** | `scope` in the claim means zone gating is a policy change, not a new credential system. |
| Consistency | **Weakened (deliberate)** | Partitioned gate clusters can over-admit a party. Chosen over refusing a paying family, and bounded by party size. |
| Security | **Weakened** | A static QR can be photographed and shared until the party count is spent. The signature is unforgeable; the *screenshot* is the attack. |
| Throughput ceiling | **Weakened** | Optical scanning is slower than a tap. Headroom comes from adding lanes, which is a cost and a physical-space problem at 15,000/day. |

**Deliberately downplayed: screenshot resistance.** Every option that defends against a shared screenshot does it with connectivity (B) or with hardware we cannot operate (C, D). The exposure is bounded by the party count, and a family pass shared with a neighbour costs the estate one admission - while a gate that refuses a valid pass because the link is down costs the estate its opening day. If measured fraud exceeds the ops-agreed threshold, the revisit trigger below is the answer, not a weaker offline story.

**Fit with the existing architecture.** This is the same posture the rest of the estate takes: the authoritative decision happens as close to the guest as possible, the cloud reconciles afterwards, and confirmed facts outrank inference. The gate does not consult a model, exactly as the AI records refuse to let a model open a ride ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)) or publish a population without an interval ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)).

## Consequences

### Positive

- The estate admits guests during a total cloud outage (offline admit).
- Perimeter-only gating in v1 keeps admit-critical hardware to a handful of lanes rather than 95 outdoor checkpoints.
- Guests can buy at home and walk straight in; no collection step.
- Popularity counting reuses the same scan infrastructure without making counters admit-critical.

### Negative

- **A shared screenshot is admitted until the party count is spent.** This is the accepted cost and it needs a measured fraud rate to stay accepted.
- **Optical scanning sets the perimeter throughput ceiling.** At 15,000/day, lane count and queue layout become a physical design problem that software cannot fix - it must be planned before the growth arrives, not after.
- Glare, dirty lenses, and cracked phone screens are the realistic failure mode; this is an ops and cleaning commitment, not a code path.
- Cross-cluster over-admission is possible during a partition and will produce reconciliation exceptions someone has to look at.
- Revocation is eventually consistent: a refunded entitlement may still admit until the revocation list reaches the gate.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Screenshot sharing | A family pass QR forwarded to another family | Party ledger caps the loss at the party size; duplicate-use patterns flagged for commercial review; measured fraud rate against an agreed threshold |
| Cross-gate over-admission | Partitioned clusters each admit the same party | Bounded by party size; cloud reconciliation flags repeats; acceptance recorded here deliberately |
| Stale revocation | Refunded or charged-back entitlement still admits | Revocation list distributed with the price/flag snapshot; short validity windows; loss bounded by one admission |
| Optical failure | Glare, dirt, cracked screens at the lane | Brightness and cleaning checklist; kiosk reprint; fail-scan rate tracked from the first weekend |
| Gate tampering | Physical access to a lane device | Public key only on gates; signed firmware; tamper-evident mounting; a compromised gate cannot forge claims for other gates |
| Clock skew | An expired pass admitted, or a valid one refused | Gateway time sync; generous window at the day boundary; skew beyond tolerance raises an ops alert rather than silently failing |
| Throughput at 3x | Morning peak exceeds lane capacity | Lane count sized against the 15,000/day target now (NFR_1); timed-entry experiment available as a demand lever ([ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)) |

## Verification

**Primary metrics**

- Gate redeem success rate for valid entitlements while the local gateway is up - target ≥99.5% p99 (OKR 1.3).
- Scan-to-decision p95, against the 2-second budget in NFR_3.
- Fail-scan rate per lane, reviewed after the first weekend.
- Reconciliation exceptions per day, split into over-admission and stale-revocation.

**Tests (CI)** - golden cases in [`evals/`](../evals/) are for AI capabilities; these are conventional tests

- The gate verification path makes no outbound network call. This assertion is the architecture.
- A valid claim is admitted with the network interface disabled.
- Tampered signature, expired window, and wrong `scope` are each rejected with a distinct reason code.
- A 2+2 family pass admits four and refuses the fifth.
- A partial admission (three of four) persists across a gate device restart.
- A revoked entitlement is refused once the revocation list has arrived, and the decision is recorded either way.

**Ops check**

- Brightness, lens cleanliness, and paper reprint tested at each lane before opening.
- Queue timing at a simulated morning peak with the cloud link pulled.

**Open questions**

- Lane count and physical queue layout for 15,000/day - before the growth phase, and it is a site-design question as much as a technical one.
- Validity window length and day-boundary tolerance - before launch.
- Fail-scan rate threshold that triggers an NFC or wristband review - agreed with ops before the first weekend.
- Signing key rotation and how a rotated public key reaches offline gates - before launch.
- Whether timed experiences (animal talks, piranha feeding) are carried in `partyShape` or as separate claims - before those products are sold.

**Revisit triggers**

- Measured fraud from shared QRs exceeds the agreed threshold - evaluate wristbands for family passes specifically, since that is where both the fraud and the UX pressure concentrate.
- Fail-scan rate stays above the ops threshold after lens and lighting fixes - evaluate NFC.
- Zone gating is funded - this payload already carries `scope`, but the residual double-spend analysis must be redone for many small clusters.
- The offline requirement is dropped in writing - then Option E becomes the simplest answer and this record should be superseded.

## Conclusion

Entitlements are signed by the purchase service and verified locally at perimeter gates, with a party ledger that lets a family pass be spent across several arrivals. Offline admit and peak throughput decided it against rotating barcodes, NFC, and wristbands. The accepted costs are a shareable screenshot bounded by party size and a cross-cluster over-admission window bounded by the same number - both cheaper than a gate that stops when the Wi-Fi does.

Related: [ADR-0001](ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) (cloud as reconciler), [ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) (how `redeemed` events travel), [ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md) (why the gate reads no price), [ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md) (how a variant rides in the entitlement).
