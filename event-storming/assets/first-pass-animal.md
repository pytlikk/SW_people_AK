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

[HOTSPOT] What is the census interval? Brief does not say. Do NOT invent a
number.

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

---

## AI / ML phase (FR#2I, FR#2J, FR#2L)

[EVENT] Animal health anomaly detected (FR#2I - ML + optional Vision; triggered
by feed refusal, aggression, env drift, keeper notes)
[EVENT] Feed refusal spike detected (sub-event of the above)
[EVENT] Water quality anomaly detected (enclosure environment drift)

[DUPLICATE] Animal health anomaly detected - yes, already listed. Keep both in
first pass; merge in digitised.

[EVENT] Piranha population estimate generated (FR#2J - Vision / ML; colony-level;
error band required; NOT a single magic number)
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
brief-specific checks). Need to document this trade-off in a later ADR, not
here.

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

- AC-01 (was CC-01): candidate SoR for health + feeding + environment per display
  (Animal Care Record Service)
- AC-02 (was CC-02): manages the alert inbox and keeper accept / reject workflow
  (Alert Inbox)
- AC-03 (was CC-03): runs the anomaly detection (FR#2I) - not named, not a model
  choice
- AC-04 (was CC-04): runs the piranha population estimator (FR#2J) - needs an
  eval harness
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

---

*Digitised board: [board-animal.svg](board-animal.svg)*
