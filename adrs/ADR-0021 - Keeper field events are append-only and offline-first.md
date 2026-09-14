# ADR-0021 - Keeper entries are append-only events captured offline, and a keeper-confirmed fact is never revised by a model

## Date

2026-09-14

## Status

Proposed

## Context

The animal houses are where the Wi-Fi dies. NFR_17 says so explicitly, and it is the ordinary condition of a sprawling estate with thick-walled heritage buildings and water everywhere.

They are also where the most valuable data on the estate is produced. Feed amounts, refusals, leftovers, aggression, unusual behaviour, census counts, and mortalities are the ground truth for the entire animal-care capability. [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) is explicit that keeper accept/reject is the training signal, and [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) depends on census and carcass entries being trustworthy.

So the system has a keeper on a handheld, in a building with no signal, wearing gloves, holding a bucket, needing to record something that a model will later be trained on. If data capture fails there, every capability downstream is built on paper notes typed up later - which is the practice the estate is replacing.

Two design problems follow, and the second is the one usually missed.

**Capture must not require connectivity.** Obvious, and solved by local write plus sync.

**Keeper judgement must outrank model output permanently.** A keeper who confirms a dead animal has established a fact. A model that later scores that colony's consumption as "consistent with no loss" must not be allowed to soften, average, or re-estimate around it. The sync mechanism therefore needs to distinguish *facts a human asserted* from *measurements a device produced*, because they have different authority.

This record decides how keeper data reaches the platform. It does not decide the care subject model ([ADR-0020](ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md)), anomaly scoring ([ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)), or the population method ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)).

**Foreclosed here:** a keeper workflow that requires connectivity at the point of entry, mutable field records, and any model output that overrides a keeper-confirmed fact.

## Evaluation criteria

- **Entry works with no connectivity (driving)** - a keeper in an animal house must record feed and observations without a network, for a full shift.
- **Keeper-confirmed facts are authoritative and immutable (driving)** - a confirmed mortality or census is a deterministic input, never a model input to be smoothed.
- **Audit defensibility** - a hazardous collection newly open to the public needs a record of who knew what, when (NFR_7).
- **Usable in the field** - gloves, water, poor light, full-shift battery (NFR_17).
- **Ordering tolerance** - entries will sync hours late and out of order relative to sensor data.

## Options

- **Option A - Online forms**: keeper app posts to the cloud API on submit.
- **Option B - Local cache with last-write-wins sync**: edit records locally, sync the current state, latest write wins.
- **Option C - Append-only event log with typed authority (chosen)**: every entry is an immutable event carrying its authority level; the cloud derives state; conflicts are resolved by authority and time, never by overwrite.
- **Option D - Paper, transcribed later**: the current practice.

| | Offline entry (driving) | Confirmed facts authoritative (driving) | Audit | Field usability | Ordering |
|---|---|---|---|---|---|
| A Online forms | Fail - the animal houses are the problem | Partial | Partial | Fail - a spinner in a snake house | n/a |
| B Last-write-wins | Pass | Fail - a later sync can silently overwrite a confirmed mortality | Fail - history destroyed by overwrite | Pass | Fail - "last" is by arrival, not by truth |
| C Append-only + authority | Pass | Pass - authority is a property of the event | Pass - nothing is ever destroyed | Pass | Pass - event time plus authority resolves |
| D Paper | Pass | Partial - trustworthy but slow | Partial | Pass | Fail - no timeliness at all |

Not options: voice capture as the primary input ([Appendix C](../requirements/Appendix%20C_%20Future%20scope.md) defers a voice copilot for gloved, noisy houses); photo-only records with no structured fields (unqueryable, and the anomaly capability needs structure).

## Decision

**Every keeper entry is an immutable, append-only event with a declared authority level. The cloud derives current state; it never overwrites history.**

| Authority | Event types | Behaviour |
|---|---|---|
| **Confirmed fact** | Mortality, census count, treatment administered | Deterministic. Applies exactly. No model may adjust it. |
| **Keeper observation** | Refusal, aggression, unusual behaviour, structured flags plus free text | Strong signal. Feeds scoring and labels; a keeper's judgement is not overridden by a score. |
| **Measurement** | Offered amount, leftover amount, environment reading entered by hand | Data. Subject to normal validation. |

Four rules follow.

**The handheld writes to the zone gateway, not to the cloud.** The local broker is reachable inside the building; the cloud is not ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)). Entries are welfare-class events, so they are never shed under buffer pressure.

**Corrections are new events, not edits.** A keeper who mistyped a leftover amount appends a correction referencing the original. Both exist forever. This costs storage and gains a defensible record for a poisonous collection open to the public, where "who knew what when" is the question that matters after an incident.

**A confirmed fact is applied deterministically and permanently.** A recovered carcass decrements a colony's cardinality by exactly one ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)), and no subsequent model estimate may smooth that away. This is the rule that makes the entire population capability honest, and it is why authority is a field on the event rather than a convention in a service.

**Offered and leftover are separate required fields.** Not a total, not "fed normally". [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) shows why: feeding to appetite makes per-capita intake unobservable and destroys the population signal entirely. The data model enforces the protocol so it cannot quietly drift back to habit.

Offline capture and fact authority decided it. Online forms (A) fail in the buildings that matter most. Last-write-wins (B) is the dangerous one, and worth naming precisely: a late-syncing handheld could overwrite a confirmed mortality with a stale "all normal" round, silently, and nothing in the system would notice. Paper (D) is the baseline being replaced, and its real defect is not accuracy but latency - a refusal noticed on Monday and typed up on Thursday cannot trigger an alert.

## Key differentiators

- **The keeper never waits for a network,** so capture happens at the moment of observation rather than from memory at the end of a shift.
- **Authority is a property of the event,** so human-confirmed facts are structurally protected from model output.
- **Nothing is ever destroyed,** which is the audit posture a newly public hazardous collection requires.
- **The feeding protocol is enforced by the schema,** keeping the population signal observable.
- **Corrections are visible,** so a mistake and its fix are both part of the record.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Availability | **Improved (driving)** | Full keeper workflow with no connectivity, for a whole shift (NFR_4, NFR_17). |
| Data integrity | **Improved (driving)** | Immutable events; confirmed facts cannot be overwritten or model-adjusted. |
| Auditability | **Improved** | Complete history including corrections; defensible after an incident (NFR_7). |
| Testability | **Improved** | Authority precedence and deterministic application are assertable. |
| Model quality | **Improved** | Keeper accept/reject and observations are clean, timely labels ([ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)). |
| Storage cost | **Weakened** | Every correction is retained alongside the original, for the life of the animal plus a legal period (NFR_9). |
| Query complexity | **Weakened** | Current state is derived rather than stored, so every read path needs the derivation and the ordering rules. |
| Keeper effort | **Weakened (deliberate)** | Separate offered and leftover measurement is more work per round than "fed, seemed fine". |

**Deliberately downplayed: keeper convenience.** A fixed-offer protocol with measured leftovers is genuinely more work than feeding to appetite, and this is an architectural choice imposing a change on someone else's daily practice. It is spent because without it the population capability has no between-census signal at all and cannibalism becomes completely invisible. It also needs keeper agreement before instrumentation, not after - if keepers do not accept it, [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) loses its primary input and should be revisited rather than quietly degraded.

**Fit with the existing architecture.** This is the gate pattern in the animal house: decide and record locally, sync as append-only events, let the cloud reconcile ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md), Appendix A 2-6). Authority precedence is the same principle as a valid entitlement admitting without a model vote - the human-established fact wins.

## Consequences

### Positive

- Capture happens in the building, at the moment of observation (driving criterion).
- A confirmed mortality or census can never be softened by a model (driving criterion).
- The audit trail survives corrections, which is what a hazardous public collection needs.
- Timely refusal and aggression data makes early detection possible at all.

### Negative

- **The feeding protocol change lands on keepers, and it may be refused.** Fixed offer with measured leftover is more work every round. Keeper agreement is a prerequisite, and without it the population signal collapses - this is a dependency on a human decision that architecture cannot enforce.
- Storage grows with corrections and is retained for the life of the animal plus a legal period, which for long-lived species is a long time.
- Derived state means every read path carries ordering and authority logic, and a bug there is subtle rather than loud.
- A keeper cannot simply fix a mistake; they append a correction, which is conceptually heavier and needs training.
- Late-syncing handhelds mean the animal-care view is provisional, so "the alert appeared this morning for something that happened yesterday" will be a normal occurrence.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Protocol not followed | Keepers feed to appetite despite the schema | Keeper agreement before instrumentation; missing leftover flagged as incomplete; consumption-based inference disabled for a subject whose protocol compliance lapses |
| Model overrides a fact | An estimate smooths away a confirmed mortality | Authority is a field; deterministic application tested in CI; the population contract requires it ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)) |
| Multi-day islanding | A handheld out of range longer than its local storage | Local storage sized for multi-shift; welfare events never shed by the gateway; [Appendix C](../requirements/Appendix%20C_%20Future%20scope.md) defers deeper multi-day protocols |
| Device loss or damage | A handheld drops in a tank | Events sync to the gateway per entry, not per shift, so loss costs minutes; paper fallback for a broken device |
| Clock skew | Field entries timestamped wrongly, corrupting ordering | Gateway stamps receipt time alongside keeper-entered event time; skew beyond tolerance alarms |
| Correction abuse | Corrections used to rewrite an inconvenient record | Both versions retained and visible; correction author and time recorded |
| Free-text dependency | Important signal buried in prose a model cannot use | Structured flags for the known cases; free text is supplementary; flag taxonomy designed with keepers |

## Verification

**Primary metrics**

- Share of keeper rounds recorded in the field rather than retrospectively.
- Protocol compliance: share of feed events with both offered and leftover recorded.
- Sync latency from entry to cloud availability, and maximum observed islanding per handheld.
- Correction rate, as a usability signal about the entry UI.

**Tests (CI)**

- A keeper entry succeeds with the network interface disabled and syncs on reconnect.
- A confirmed mortality decrements colony cardinality by exactly one and is not altered by any subsequent model estimate.
- A late-arriving observation cannot overwrite a confirmed fact recorded earlier.
- A correction event preserves the original; neither is deleted.
- A feed event missing `leftover` is flagged incomplete and excluded from consumption inference.
- Derived current state is identical regardless of the order events arrive in.
- Welfare-class events are never shed under simulated gateway buffer pressure.

**Ops check**

- A keeper completes a full round in an animal house with the uplink down, wearing gloves, and confirms battery lasts the shift.
- Keepers review the flag taxonomy and confirm it covers what they actually see.

**Open questions**

- **Feeding protocol change to fixed offer with measured leftover - needs keeper agreement, before instrumentation.** This is the highest-risk open question in the animal deep-dive.
- Structured flag taxonomy - with keepers, before [ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) leaves shadow.
- Handheld local storage sizing and the islanding duration it must cover - before procurement.
- Whether handhelds speak MQTT directly or via a local HTTP shim on the gateway - before the app is built.
- How treatment events are recorded against a colony subject - with the vet ([ADR-0020](ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md) open question).

**Revisit triggers**

- Keepers reject the fixed-offer protocol - revisit [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)'s primary signal rather than degrading it silently.
- Correction rate is high - the entry UI is wrong, not the keepers.
- Free-text notes routinely carry signal the flags miss - extend the taxonomy.
- Handhelds regularly exceed their local storage - the radio survey or the storage sizing is wrong.

## Conclusion

Keeper entries are immutable append-only events written to the local zone gateway, carrying an authority level that makes a confirmed mortality or census deterministic and permanently safe from model adjustment. Chosen because the animal houses have no signal and because the entire animal-care capability rests on keeper judgement outranking model output. The costs are more work per feeding round, growing storage, and derived-state complexity - with the feeding protocol change being a genuine dependency on keeper agreement rather than something the architecture can impose.

Related: [ADR-0020](ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md) (what entries attach to), [ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) (what consumes them), [ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md) (why offered and leftover are separate), [ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) (welfare-class transport).
