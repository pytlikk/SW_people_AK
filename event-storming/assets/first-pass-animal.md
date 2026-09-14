# First Pass - Board A: Animal Care (Keepers)

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
> **ADR alignment note (added 2026-09-14 post-merge):** ADR-020..023 landed in
> the repo after this first pass was written against requirements alone. Sticky
> entries below that contradict those ADRs are annotated inline with
> [ADR-020], [ADR-021], [ADR-022], or [ADR-023] markers and a correction.
> Component IDs AC-01..AC-05 are NOT renumbered; only their descriptions are
> updated. See also the Board A section of the parent index.
>
> Scope: **ops domain only.** Ticketing (ADR-001, ADR-002) appears as an external
> upstream system whose decisions are now Proposed and load-bearing. Board A must
> not contradict them. The visit-access storm is Board V.
>
> Sticky type in [BRACKETS]. Order is approximate - not strictly chronological.

---

## Ticketing boundary (ADR-001 / ADR-002) - established upstream constraints

[EXTERNAL] Ticketing / Token System (ADR-001 / ADR-002) - upstream only.

ADR-001 established that the shop sells a currency (a token pool), not attraction
rights. Repricing does not touch issued wallets; claims already minted keep the
price at which they were minted. Animal Care does NOT own wallet balance, the SKU
catalogue, or claim minting. Membership or path-based daytime products that set
token cost to zero still mint and burn a signed ADR-002 QR at the checkpoint -
the gate still admits on a signed claim (ADR-001 Key differentiators). This board
does not invent a parallel admit path.

ADR-002 established that enclosure / attraction scans (the `validated` /
`revoked` MQTT events) feed the popularity aggregator (PM-01 / Board P) for audit
and popularity counts. They do NOT open or update the health record. ADR-002
Conclusion is explicit: "Animal-health telemetry is a separate system; enclosure
scans may feed popularity counts but do not open checkpoints."

Characteristics that these ADR decisions bring as upstream constraints (cited from
the ADRs, not invented here; no system-wide funnel exists yet):
- ADR-001 strengthens evolvability / repricing agility and operability; weakens
  predictability, recoverability, and guest certainty.
- ADR-002 strengthens reliability (offline admit) and operability; weakens
  throughput / performance, security / integrity, and recoverability.

Board A inherits these trade-offs. It does not re-litigate them.

[HOTSPOT] Signing-key architecture for claims is an open question in ADR-002
(client-side vs. server-side; owner: TBD). If any Animal Care view or keeper
copilot output references claim verification, it must point at that open question
and must not assume a resolved key-custody design.

[HOTSPOT] ADR-002 states that a pending-to-used failure after a failed gate read
has no self-service path; the kiosk is the only correction point. Board A must
not introduce an ops-console "remote refund" or correction command for gate
failures. That is a visitor-lane concern, not a keeper-domain concern.

---

## Session dump

[ACTOR] Keeper / Staff (animals)
[ACTOR] Countess - reads trends, makes invest decisions; does NOT give field
commands

[QUESTION] Is there a veterinary actor? The brief does NOT name a vet org. NFR_15
says "does not replace veterinary authority" and Assumptions say "software
recommends; it does not replace veterinary or safety authority." Label as
**inferred** if referenced - not a named actor.

[CMD] Record health observation (keeper enters into system)
[CMD] Record feeding observation - Appendix A 2-4 says "amount, refusal,
leftover, aggression" - these are the "how much / how well" signals from the
brief

[ADR-021 PATH] These keeper commands do NOT travel on the MQTT telemetry path.
Each observation is a human-authored record captured in a device-local
append-only log on the keeper's device and synced as an idempotent batch to any
reachable estate endpoint. A lost sensor reading is a gap flag; a lost keeper
note is unrecoverable and a missing FR#2I training label. See
[ADR-021](../../adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md).
MQTT carries sensor telemetry only (see [EXTERNAL] MQTT feeder device below).

[EVENT] Health observation recorded
[EVENT] Feeding observation recorded

[CMD] Add keeper note (text, tagged to display or animal)
[EVENT] Keeper note added

[CMD] Read enclosure environment (triggered on schedule or by sensor event)
[EVENT] Enclosure environment read (water params, HVAC, barriers - Appendix A
2-4)

[EVENT] MQTT feeder event received -- wait, this is an external input not a
command. Move.
[EXTERNAL] MQTT feeder device fires event (amount dispensed; timestamp; enclosure
id) - same hardware budget as the rest of the estate; placement is TBD per ADR
(Assumptions)

[CMD] Conduct piranha census (manual, by keeper) - brief specifically calls out
jumping-piranha population
[EVENT] Piranha population counted (keeper census is the ground truth - FR#2J)

[ADR-023 NOTE] The census is the ANCHOR for the population interval, not a
standalone count. Between anchors, feed consumption tracks direction and
magnitude. The interval widens monotonically away from the census; when it
exceeds a threshold the system requests a new census rather than publishing a
bare number. A recovered carcass is a deterministic decrement of exactly one and
is NOT a model input. Vision, if funded, is scoped to surface-feeding frames and
runs in shadow first - it cannot be the sole basis for any published figure.
See [ADR-023](../../adrs/ADR-023-anchor-piranha-population-on-human-census.md).

[HOTSPOT] What is the census interval? Brief does not say. Do NOT invent a
number. ADR-023 answers: census is scheduled by uncertainty (interval width),
not by calendar habit. The maximum anchor age before the estimate becomes
unusable is an open question (to be set before the first season).

[EVENT] Enclosure scan received - STOP. Recheck ADR-002 boundary.
ADR-002 Conclusion: "Animal-health telemetry is a separate system; enclosure
scans may feed popularity counts but do not open checkpoints." Confirmed: this
event goes to PM-01 (Board P), NOT to AC-01 / the health record.

[EVENT] Enclosure scan received (from Ticketing / Token System, ADR-002
`validated` event - feeds PM-01 / Board P popularity aggregator only; NOT a
health record input; NOT a command to the health system)

[EXTERNAL] Ticketing / Token System (ADR-001 / ADR-002) - upstream only;
enclosure scan events are the only interface here; Animal Care does not write
back into ticketing and does not consume wallet balance

[QUESTION] Which of the 55 displays have MQTT sensors installed? Not all of
them necessarily. Placement is an ADR. MQTT hardware budget exists (Assumptions)
but sensor count and type are TBD.

[QUESTION] Individual animal ID tracking or enclosure-level? Assumptions say:
"Enclosure- or colony-level tracking is sufficient except where individual IDs
already exist; piranha are colony-level with periodic census."

[ADR-020 CORRECTION] This question is now answered. ADR-020 forecloses
enclosure as the welfare identity: "A tank cannot be sick." The unit of record
is the care SUBJECT - individual, group, or colony. The enclosure is a separate,
dated placement. Three subject types cover all 55 displays: individual (e.g. a
solitary venomous snake), group (e.g. a troop fed collectively), colony (e.g.
piranha). Welfare history attaches to the subject; environment readings attach to
the enclosure and are joined through placement-at-time-of-observation. The
working assumption from requirements (enclosure-level) is superseded.
See [ADR-020](../../adrs/ADR-020-use-care-subject-as-unit-of-record.md).

---

## AI / ML phase (FR#2I, FR#2J, FR#2L)

[EVENT] Animal health anomaly detected (FR#2I - tiered detector per ADR-022;
triggered by feed refusal, aggression, env drift, keeper notes)

[ADR-022 CORRECTION] The first-pass label "ML + optional Vision" is superseded.
ADR-022 defines a four-tier detector:
- Tier 0: deterministic safety thresholds (water temp, dissolved oxygen,
  containment failures) - runs at EDGE, never gated by a model, fires during
  islanding.
- Tier 1: per-subject baseline scoring (intake rate, refusal rate, feed interval,
  environment drift, each compared against THAT SUBJECT's own recent history) -
  the day-one detector; no labels required.
- Tier 2: supervised model trained on accumulated keeper accept/reject labels -
  NOT at launch; promoted per collection after shadow beats Tier 1 on held-out
  confirmed events.
- Tier 3: GenAI narrative - turns Tier 0-2 evidence into readable explanation;
  NEVER produces a score or decision.
Vision is excluded from v1 health detection; any addition requires its own ADR
with a stated cost ceiling. See
[ADR-022](../../adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md).
[EVENT] Feed refusal spike detected (sub-event of the above)
[EVENT] Water quality anomaly detected (enclosure environment drift)

[DUPLICATE] Animal health anomaly detected - yes, already listed. Keep both in
first pass; merge in digitised.

[EVENT] Piranha population estimate generated (FR#2J - census-anchored interval
per ADR-023; colony-level; error band required; NOT a single magic number)

[ADR-023 CORRECTION] "Vision / ML" as primary path is superseded. The estimate
is an INTERVAL anchored on human census:
- Human census: anchors the absolute value; resets accumulated drift.
- Feed consumption: tracks direction and rough magnitude of change between
  anchors. Requires a fixed-offer / measured-leftover feeding protocol; ad-hoc
  feeding-to-appetite makes per-capita intake unobservable.
- Keeper observations: deterministic events (recovered carcass = -1 exactly;
  observed fry = pending confirmation).
- Vision (optional, one tank): third estimator, shadow first; cannot be the
  sole basis for any published figure; needs its own ADR with a cost ceiling
  before any camera is specified.
The interval widens monotonically with time since the last census. Past the
maximum anchor age the estimate publishes as UNUSABLE, not as a wide-band
number. See [ADR-023](../../adrs/ADR-023-anchor-piranha-population-on-human-census.md).
[EVENT] Population trend flagged (breeding, loss - FR#2J)

[QUESTION] Is there camera or sonar hardware at the piranha enclosure? Optional
per FR#2J ("camera/sonar if installed"). Placement is an ADR.

[EVENT] Keeper copilot query answered (FR#2L - RAG; grounded on estate SOPs,
keeper notes, live state; citations required; fails closed to "ask the vet /
duty manager" - NOT a named vet actor)

[HOTSPOT] FR#2L explicitly says "no invented vet doses or safety overrides." This
is a hard guardrail, not a nice-to-have. How is it enforced? ADR needed.

[HOTSPOT] Keeper copilot must not advise on claim signing, key custody, or gate
admission logic. Signing-key architecture is an open question in ADR-002 (see
Ticketing boundary section above). Copilot scope is health, feeding, SOPs - not
ticketing internals.

[EXTERNAL] Vet / welfare authority (INFERRED - not named in the brief; inferred
from NFR_15 "human confirm" and Assumptions "does not replace veterinary
authority"; keeper is the interface, not a vet system)

---

## Alert handling

[EVENT] Alert raised (with confidence score and evidence - NFR_15)
[EVENT] Alert inbox updated
[CMD] Accept alert (keeper action)
[CMD] Reject alert (keeper action)
[EVENT] Alert accepted (this is the keeper training signal for the ML model -
FR#2I)
[EVENT] Alert rejected (also a training signal)

[HOTSPOT] "High recall first" (FR#2I) - this means we will get false positives.
The cost of missed sick animal vs. unnecessary check: NOT symmetric (brief +
brief-specific checks). Trade-off documented in
[ADR-022](../../adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md):
Tier 1 baseline scoring stays sensitive; keeper attention is protected by a
ranked, budgeted alert inbox rather than by raising the detection threshold.
Accept rate is an ops metric; a sustained drop below the ops-agreed floor
triggers inbox tuning before adding any model.

[CMD] Query keeper copilot (FR#2L)

---

## Countess view

[EVENT] Health trend visible to Countess (aggregate; read-only; not operational)
[QUESTION] Does Countess need per-animal detail or just summary flags? Brief
says she "decide[s] where to invest" - suggesting summary is enough. TBD.

---

## Component candidates (rough first-pass list, AC- prefix)

These are names, not decisions. CC-01..CC-05 are the legacy IDs from the
ops-backup era; the live IDs below supersede them. See the CC-to-AC mapping
table in the parent index.

- AC-01 (was CC-01): System of Record for health + feeding observations, keyed
  to CARE SUBJECT (individual, group, or colony per ADR-020) - NOT per display.
  The enclosure is a dated placement joined to the subject. Name: Animal Care
  Record Service.
- AC-02 (was CC-02): manages the alert inbox and keeper accept / reject workflow
  (Alert Inbox); accept/reject is the FR#2I training signal for Tier 2 (ADR-022).
- AC-03 (was CC-03): runs the anomaly detection (FR#2I) - four-tier detector per
  ADR-022; Tier 0 and 1 are deterministic arithmetic; not a model choice.
- AC-04 (was CC-04): runs the piranha population estimator (FR#2J) - census-
  anchored interval per ADR-023; needs an eval harness; vision is optional/shadow.
- AC-05 (was CC-05): handles keeper copilot RAG (FR#2L) - could be shared with
  IO-08 / Ops Copilot (Board B open question)

---

## Corrections and open questions from this pass

1. Enclosure scan events go to PM-01 / Board P, NOT to AC-01 / the health
   record. ADR-002 Conclusion is explicit. Keep these event flows separate.
2. Vet authority is inferred, not a named actor or system in the brief. Do not
   invent a vet organisation.
3. Do not invent: sensor counts, census intervals, anomaly thresholds, population
   numbers.
4. "Jumping piranha" is the brief's specific phrase; use it when referencing the
   brief.
5. AI outputs are advisory. No autonomous welfare decisions (FR#2I, Appendix B
   human-in-the-loop table).
6. No remote-refund or gate-correction command on this board. ADR-002: kiosk is
   the dispute desk for gate failures.
7. No assumption about signing-key custody. ADR-002 open question; owner TBD.

**Post-merge corrections against ADR-020..023 (annotated inline above):**

8. The unit of record is the CARE SUBJECT, not the enclosure. AC-01 is NOT a
   per-display SoR. Enclosure is a dated placement joined to the subject.
   ADR-020 forecloses enclosure-as-welfare-identity. The requirements-era
   working assumption ("enclosure- or colony-level tracking is sufficient") is
   superseded.
9. Keeper observation commands ([CMD] Record health / feeding observation) travel
   via the ADR-021 device-local log, NOT on the MQTT bus. MQTT carries machine-
   generated sensor telemetry only. The two paths must not be conflated.
10. Anomaly detection is a four-tier detector (ADR-022): Tier 0 deterministic
    safety thresholds at edge; Tier 1 per-subject baselines as the day-one
    detector; Tier 2 supervised model earned from labels; Tier 3 GenAI narrative
    only. "ML + optional Vision" as the primary detector is superseded.
11. Piranha population is a census-anchored interval (ADR-023). Vision is
    optional/shadow-only, NOT the primary estimator. A bare population count is
    not a valid output. Feed-derived consumption tracks change between anchors
    but only under a fixed-offer / measured-leftover feeding protocol.
12. The high-recall trade-off is documented in ADR-022. The alert inbox is the
    labelling machine; keeper accept/reject is the training signal for Tier 2.
    Alert budget and ranked inbox protect keeper attention without raising the
    detection threshold.

---

*Digitised board: [board-animal.svg](board-animal.svg)*
