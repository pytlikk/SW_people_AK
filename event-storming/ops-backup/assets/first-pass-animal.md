# First Pass - Board A: Animal Care (Keepers)

> **Reconstructed first pass from `requirements/`, not a photographed workshop.**
> There was no physical sticky-note session. This document simulates the messy
> first pass that would emerge from a facilitated event-storming session run
> against the committed requirements. Unordered dump; duplicates expected;
> corrections and questions written in-line as they arose.
>
> Scope: **ops domain only.** Ticketing (ADR-001, ADR-002) appears as an external
> upstream system. The visit-access storm is a later session.
>
> Sticky type in [BRACKETS]. Order is approximate - not strictly chronological.

---

## Session dump

[ACTOR] Keeper / Staff (animals)
[ACTOR] Countess - reads trends, makes invest decisions; does NOT give field commands
[QUESTION] Is there a veterinary actor? The brief does NOT name a vet org. NFR_15 says "does not replace veterinary authority" and Assumptions say "software recommends; it does not replace veterinary or safety authority." Label as **inferred** if referenced - not a named actor.

[CMD] Record health observation (keeper enters into system)
[CMD] Record feeding observation - Appendix A 2-4 says "amount, refusal, leftover, aggression" - these are the "how much / how well" signals from the brief

[EVENT] Health observation recorded
[EVENT] Feeding observation recorded

[CMD] Add keeper note (text, tagged to display or animal)
[EVENT] Keeper note added

[CMD] Read enclosure environment (triggered on schedule or by sensor event)
[EVENT] Enclosure environment read (water params, HVAC, barriers - Appendix A 2-4)

[EVENT] MQTT feeder event received -- wait, this is an external input not a command. Move.
[EXTERNAL] MQTT feeder device fires event (amount dispensed; timestamp; enclosure id) - same hardware budget as the rest of the estate; placement is TBD per ADR (Assumptions)

[CMD] Conduct piranha census (manual, by keeper) - brief specifically calls out jumping-piranha population
[EVENT] Piranha population counted (keeper census is the ground truth - FR#2J)

[HOTSPOT] What is the census interval? Brief does not say. Do NOT invent a number.

[EVENT] Enclosure scan received - STOP. ADR-002 says: "Animal-health telemetry is a separate system; enclosure scans may feed popularity counts but do not open checkpoints." Enclosure scans (token validation events from ADR-002) feed the popularity meter, not health records. Separate the event paths.

[EVENT] Enclosure scan received (from Ticketing / Token System, ADR-002 validated event - feeds popularity meter; NOT health record)

[EXTERNAL] Ticketing / Token System (ADR-001 / ADR-002) - upstream only; enclosure scan events are the only interface here

[QUESTION] Which of the 55 displays have MQTT sensors installed? Not all of them necessarily. Placement is an ADR. MQTT hardware budget exists (Assumptions) but sensor count and type are TBD.

[QUESTION] Individual animal ID tracking or enclosure-level? Assumptions say: "Enclosure- or colony-level tracking is sufficient except where individual IDs already exist; piranha are colony-level with periodic census."

---

## AI / ML phase (FR#2I, FR#2J, FR#2L)

[EVENT] Animal health anomaly detected (FR#2I - ML + optional Vision; triggered by feed refusal, aggression, env drift, keeper notes)
[EVENT] Feed refusal spike detected (sub-event of the above)
[EVENT] Water quality anomaly detected (enclosure environment drift)

[DUPLICATE] Animal health anomaly detected - yes, already listed. Keep both in first pass; merge in digitised.

[EVENT] Piranha population estimate generated (FR#2J - Vision / ML; colony-level; error band required; NOT a single magic number)
[EVENT] Population trend flagged (breeding, loss - FR#2J)

[QUESTION] Is there camera or sonar hardware at the piranha enclosure? Optional per FR#2J ("camera/sonar if installed"). Placement is an ADR.

[EVENT] Keeper copilot query answered (FR#2L - RAG; grounded on estate SOPs, keeper notes, live state; citations required; fails closed to "ask the vet / duty manager" - NOT a named vet actor)

[HOTSPOT] FR#2L explicitly says "no invented vet doses or safety overrides." This is a hard guardrail, not a nice-to-have. How is it enforced? ADR needed.

[EXTERNAL] Vet / welfare authority (INFERRED - not named in the brief; inferred from NFR_15 "human confirm" and Assumptions "does not replace veterinary authority"; keeper is the interface, not a vet system)

---

## Alert handling

[EVENT] Alert raised (with confidence score and evidence - NFR_15)
[EVENT] Alert inbox updated
[CMD] Accept alert (keeper action)
[CMD] Reject alert (keeper action)
[EVENT] Alert accepted (this is the keeper training signal for the ML model - FR#2I)
[EVENT] Alert rejected (also a training signal)

[HOTSPOT] "High recall first" (FR#2I) - this means we will get false positives. The cost of missed sick animal vs. unnecessary check: NOT symmetric (brief + brief-specific checks). Need to document this trade-off in a later ADR, not here.

[CMD] Query keeper copilot (FR#2L)

---

## Countess view

[EVENT] Health trend visible to Countess (aggregate; read-only; not operational)
[QUESTION] Does Countess need per-animal detail or just summary flags? Brief says she "decide[s] where to invest" - suggesting summary is enough. TBD.

---

## Component candidates (rough first-pass list)

These are names, not decisions. They become candidates for later ADRs.

- Something that is a SoR for health + feeding + environment per display (Animal Care Record Service - candidate)
- Something that manages the alert inbox and keeper accept / reject workflow (Alert Inbox - candidate)
- Something that runs the anomaly detection (FR#2I) - not named, not a model choice
- Something that runs the piranha population estimator (FR#2J) - needs an eval harness
- Something that handles keeper copilot RAG (FR#2L) - could be shared with ops copilot (Board B question)

---

## Corrections and open questions from this pass

1. Enclosure scan events go to the popularity meter, NOT to the health record. Keep these event flows separate (ADR-002 is explicit).
2. Vet authority is inferred, not a named actor or system in the brief. Do not invent a vet organisation.
3. Do not invent: sensor counts, census intervals, anomaly thresholds, population numbers.
4. "Jumping piranha" is the brief's specific phrase; use it when referencing the brief.
5. AI outputs are advisory. No autonomous welfare decisions (FR#2I, Appendix B human-in-the-loop table).

---

*Digitised board: [board-animal.svg](board-animal.svg)*
