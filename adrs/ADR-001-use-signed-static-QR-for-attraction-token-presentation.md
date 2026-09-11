# ADR-001 — Use signed static QR for attraction-token presentation

## Date

2026-09-11

## Status

Accepted

## Context

The visitor app sells tokens at home. The visitor must prove a right at an **attraction** while estate Wi-Fi is patchy: MQTT on the intranet, a few Wi-Fi patches as gateways to an internet DB. Kiosks exist for top-up and dead-phone reprint.

Same app, same wallet, same presentation:

| Kind | Count | Checkpoint |
|---|---|---|
| Amusement rides | 40 | ride gate |
| Animal displays / enclosures | 55 | enclosure / display entrance |

5000 visitors/day (locked board). Whether every enclosure is a paid checkpoint, and MQTT placement for up to 40 + 55 devices, are TBD (ops / pytlikk / mroj4n).

This record answers **only** how the visitor presents an already-issued signed token at a checkpoint. It does not decide token pricing, AI popularity, MQTT broker, or animal health telemetry. The phone must not call the internet database at the checkpoint.

## Evaluation criteria

- **Offline at the checkpoint (driving)** — 0 refusals caused by network error for a valid issued token (ride or enclosure).
- **Operability (driving)** — three-person team; no Apple/Google Wallet certification; no 5000-band/day pool in v1.
- **Integrity** — forged tokens rejected; wrong `attractionId` rejected; same-checkpoint replay bounded (window TBD — pytlikk).
- **Throughput** — scan-to-decision is local verify, not a live API (p99 TBD — pytlikk).
- **One app** — rides and displays share wallet, QR, and kiosk reprint.

## Options

- **Signed static QR**: phone or kiosk printout shows a signed payload; checkpoint camera scans the visitor; device verifies locally.
- **Rotating barcode (SafeTix-style)**: payload redraws so screenshots die; often needs a network to refresh.
- **Phone NFC (Apple VAS / Google Smart Tap)**: tap a certified reader; QR still needed as fallback.
- **RFID/NFC wristband at kiosk**: home purchase bound to a band; tap at the checkpoint (RulaBand / MagicBand).
- **QR now, wristband later**: same claims, new reader when ops funds a band pool.
- **BLE / UWB / face**: used in parks for wayfinding, show tracking, or Express anti-transfer — not a trustworthy or lawful admit radio here.
- **MQTT-only wallet update**: app balance changes only when a gateway is up.

## Decision

**Use signed static QR.** The checkpoint camera scans the visitor. The device verifies locally and admits or rejects with no network round-trip.

Offline and operability decide it. Rotating barcodes fight patchy Wi-Fi. NFC is viable but fails operability (certified readers at up to ~95 points). Wristbands are the best zoo/kids UX but need a band logistics shop we do not have. BLE/UWB/face are out as the admit radio.

- Token is attraction-scoped (`attractionId` = a ride **or** a display/enclosure) and single-use at that checkpoint.
- Local seen-token cache is the admit/reject record. MQTT `validated` / `revoked` is for popularity, audit, and extra lanes — not for admit, not for the app wallet.
- **Wallet without Wi-Fi:** when the visitor *shows* the QR, the app moves it to *pending* and decrements the balance. Optional later MQTT confirm; if none arrives, age *pending* → *used* (window TBD — pytlikk/ops). Kiosk is the dispute desk. A post-admit receipt QR on the display is a manual repair only — visitors skip a second scan.
- Same token claims must later fit a wristband or phone-NFC reader (new ADR, same economy).
- Crafterro’s animal-health tracker does not admit visitors. Enclosure scans may feed popularity; they do not replace eating/health/piranha telemetry.

## Key differentiators

- Works with no phone uplink and no estate Wi-Fi at the queue (rides and outdoor enclosures).
- Commodity camera at each checkpoint — no wallet-cert programme, no band inventory.
- One payload type for 40 rides and 55 displays; a ride token does not open an enclosure.
- Phone wallet updates on QR-reveal, so MQTT is not the wallet path.

## Consequences

### Positive

- Gate/enclosure still admits when Wi-Fi is down (offline criterion).
- pytlikk can test scanners before mroj4n’s intranet is live.
- Kiosk reprint is the same QR if the phone is dead.
- Later band/NFC can reuse claims (operability now, reversibility later).

### Negative

- Optical scan is slower and fussier than a tap (glare, dirt, outdoor enclosures).
- Static QR can be photographed until marked used.
- Failed gate read after QR-reveal can leave the app *pending* — kiosk must correct it.
- Up to 40 + 55 scanners if every display is gated.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Screenshot / double-scan | Static QR reusable until used | Single-use + local cache; ride token ≠ enclosure token; MQTT eventual; fraud rate TBD (pytlikk) |
| Optical fuss | Glare, cracked glass, dirty cameras | Brightness checklist; kiosk reprint; revisit NFC/band after first weekend fail-scan rate (ops) |
| Dead phone | Cannot show QR | Kiosk reprint (gateway path) |
| Extra scan skipped | Post-admit display QR would not update the wallet | Burn on QR-reveal; display QR only as repair |
| Device count | Up to 40 + 55 MQTT clients | Same device role; placement is mroj4n; not every enclosure must be gated (TBD ops) |
| Animal-health mix-up | Ticketing confused with Crafterro telemetry | Separate topics; health data does not open checkpoints |
| Later band/NFC | Payload lock-in | Claims opaque except `attractionId` / validity; schema is a follow-up ADR |

## Verification

- CI: checkpoint validation path has no outbound HTTP/TCP to the internet DB or shop.
- CI: valid QR accepted offline; tampered / expired / wrong attraction rejected; replay in local cache rejected; kiosk reprint accepted once; app decrement on QR-reveal needs no network.
- Ops: brightness / dirty-camera / paper reprint at rides **and** enclosure checkpoints. Owner: pytlikk until ops takes the runbook.

Open: screenshot window (pytlikk); *pending* → *used* age-out (pytlikk/ops); kiosk screen vs paper; which of 55 displays are paid (ops/Countess); fail-scan rate after first weekend (ops).

Do not reopen phone→DB at the checkpoint unless the offline criterion is dropped in writing. Revisit rotating barcodes only with a Wallet-cached code that needs no attraction Wi-Fi.

## Conclusion

Signed static QR, checkpoint scans the phone, local verify — the same for 40 rides and 55 animal displays/enclosures. Offline and team size beat NFC and wristbands for v1. MQTT does not update the wallet. Animal health stays a different system.

Related: token claim schema; MQTT broker (mroj4n); token economy / pricing / AI; enclosure paid-vs-free; wristband ADR if funded.
