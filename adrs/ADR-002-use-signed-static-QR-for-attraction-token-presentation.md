# ADR-002 — Use signed static QR for attraction-token presentation

## Date

2026-09-11

## Status

Proposed

## Context

The visitor app sells a token pool at home (ADR-001). The visitor must prove a right at an **attraction** while estate Wi-Fi is patchy: MQTT on the intranet, a few Wi-Fi patches as gateways to an internet DB. Kiosks exist for top-up and dead-phone reprint.

Same app, same wallet, same presentation:

| Kind | Count | Checkpoint |
|---|---|---|
| Amusement rides | 40 | ride gate |
| Animal displays / enclosures | 55 | enclosure / display entrance |

5000 visitors/day (locked board). Kiosks, MQTT-on-intranet, and “few Wi-Fi patches as gateways” are **working assumptions** until their own ADRs exist. Whether every enclosure is a paid checkpoint, and MQTT placement for up to 40 + 55 devices, are TBD.

This record answers **only** how the visitor presents an already-issued signed token at a checkpoint. It does not decide token pricing, AI popularity, MQTT broker, or animal health telemetry. The phone must not call the internet database at the checkpoint. The signed claim is the only contract the checkpoint has with ADR-001; pool balance lives in the wallet, not at the gate.

## Evaluation criteria

- **Offline at the checkpoint (driving)** — 0 refusals caused by network error for a valid issued token (ride or enclosure).
- **Operability (driving)** — three-person team; no Apple/Google Wallet certification; no 5000-band/day pool in v1.
- **Integrity** — forged tokens rejected; wrong `attractionId` rejected; same-checkpoint replay bounded (window TBD).
- **Throughput** — scan-to-decision is local verify, not a live API (p99 TBD).
- **One app** — rides and displays share wallet, QR, and kiosk reprint.

## Options

- **Option A — Signed static QR (chosen)**: phone or kiosk printout shows a signed payload; checkpoint camera scans; device verifies locally with no network.
- **Option B — Rotating barcode (SafeTix-style)**: payload redraws each interval so screenshots cannot replay; needs a refresh signal to reach the phone.
- **Option C — Phone NFC (Apple VAS / Google Smart Tap)**: tap a certified reader; no screenshot risk; QR still needed as fallback.
- **Option D — RFID/NFC wristband issued at kiosk**: home purchase bound to a wristband at kiosk check-in; tap at checkpoint; best zoo-scale UX.

Not options: BLE/UWB/face are excluded as the admit radio (no reliable identity binding at throughput scale, legal risk); a wristband roll-out later reuses the same signed claims (follow-on ADR, not a competing choice today); MQTT is the audit channel, not the wallet path.

| | Offline at checkpoint (driving) | Operability (driving) | Integrity | Throughput | Cost / complexity |
|---|---|---|---|---|---|
| A Signed static QR | ✓ local verify; no network | ✓ commodity camera; no cert programme | Single-use cache; screenshot risk until burned | p99 TBD; optical slower than tap | Lowest |
| B Rotating barcode | ✗ needs phone refresh; fights patchy Wi-Fi | ✓ no cert | Strong screenshot defence | Similar to A | Medium; refresh protocol |
| C Phone NFC | ✓ if key pre-loaded | ✗ certified readers at ~95 checkpoints | Strong; no screenshot | Fastest; tap-only | High; reader procurement + cert |
| D RFID/NFC wristband | ✓ | ✗ band stock + kiosk binding; no band shop today | Strong; no screenshot | Fastest; tap | High; band inventory + logistics |

## Decision

**Use signed static QR.** The checkpoint camera scans the visitor. The device verifies locally and admits or rejects with no network round-trip.

Offline and operability decide it. Rotating barcodes fight patchy Wi-Fi. NFC is viable but fails operability (certified readers at up to ~95 points). Wristbands are the best zoo/kids UX but need a band logistics shop we do not have.

- Token is attraction-scoped (`attractionId` = a ride **or** a display/enclosure) and single-use at that checkpoint.
- Local seen-token cache is the admit/reject record. MQTT `validated` / `revoked` is for popularity, audit, and extra lanes — not for admit, not for the app wallet.
- Burn on QR-reveal is the wallet write (no gate network). A post-admit display QR is repair only. Pending→used age-out and kiosk dispute: TBD.
- Same token claims must fit a wristband or phone-NFC reader in a later roll-out (follow-on ADR, same economy).
- Animal-health telemetry is a separate system; enclosure scans may feed popularity counts but do not open checkpoints.

## Key differentiators

- Works with no phone uplink and no estate Wi-Fi at the queue (rides and outdoor enclosures).
- Commodity camera at each checkpoint — no wallet-cert programme, no band inventory.
- One payload type for 40 rides and 55 displays; a ride token does not open an enclosure.
- Phone wallet updates on QR-reveal, so MQTT is not the wallet path.

## Consequences

### Positive

- Gate/enclosure still admits when Wi-Fi is down (offline criterion).
- Scanners can be tested before the intranet is live.
- Kiosk reprint is the same QR if the phone is dead.
- Later band/NFC can reuse claims (operability now, reversibility later).

### Negative

- Scan-to-decision throughput at outdoor enclosures is lower than a tap; p99 scan time TBD — glare and dirty cameras are the likely failure mode, not software.
- The fraud window is the interval between QR-reveal (wallet debit) and the checkpoint cache write; length of that window is TBD and is the primary integrity risk.
- Failed gate read after QR-reveal leaves the app in *pending*; the visitor must reach a kiosk to correct it — no self-service path.
- Up to 95 cameras if every ride and display is gated; outdoor mounting and maintenance at that scale must be planned before installation.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Screenshot / double-scan | Static QR reusable until used | Single-use + local cache; ride token ≠ enclosure token; MQTT eventual; fraud rate TBD |
| Optical fuss | Glare, cracked glass, dirty cameras | Brightness checklist; kiosk reprint; revisit NFC/band after first weekend fail-scan rate |
| Dead phone | Cannot show QR | Kiosk reprint (gateway path) |
| Extra scan skipped | Visitors will not scan a second QR after the gate | Burn on reveal; display QR is repair only |
| Device count | Up to 40 + 55 MQTT clients | Same device role; placement is a later ADR; not every enclosure must be gated (TBD) |
| Animal-health mix-up | Ticketing confused with animal-health telemetry | Separate topics; health data does not open checkpoints |
| Later band/NFC | Payload lock-in | Claims opaque except `attractionId` / validity; schema is a follow-up ADR |

## Verification

**Tests (CI)**

- Checkpoint validation path has no outbound HTTP/TCP to the internet DB or shop.
- Valid QR accepted offline; tampered, expired, or wrong-attraction QR rejected; replay in local cache rejected; kiosk reprint accepted once; app decrement on QR-reveal needs no network.

**Ops check**

- Brightness, dirty-camera, paper reprint at ride and enclosure checkpoints.

**Open questions**

- Fraud window between QR-reveal and checkpoint cache write — before launch.
- Pending→used age-out duration — before beta.
- Kiosk output: screen-only or paper reprint — before installation.
- Which of 55 displays are paid — before gate installation.
- Fail-scan rate threshold that triggers NFC/wristband review — agreed before first weekend.

**Revisit triggers**

- Offline criterion dropped in writing → re-evaluate phone-to-DB at the checkpoint.
- Fail-scan rate exceeds the ops-agreed threshold after first weekend → evaluate rotating barcode (Wallet-cached, no attraction Wi-Fi needed) or NFC.

## Conclusion

Signed static QR, checkpoint scans the phone, local verify — the same for 40 rides and 55 animal displays/enclosures. Offline and team size beat NFC and wristbands for v1. MQTT does not update the wallet. Animal health stays a different system.

Related: [ADR-001](ADR-001-use-home-bought-token-pool.md) (why a pool, not home-booked attraction tickets); MQTT broker; pricing / AI; enclosure paid-vs-free; wristband ADR if funded.
