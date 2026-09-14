# ADR-0011 - Assign the experiment variant at purchase and carry it in the entitlement

## Date

2026-09-14

## Status

Proposed

## Context

The Countess wants more returning visitors and does not know what would produce them. That is an experimentation problem, not a prediction problem: nobody can forecast whether a 2+3 family pass beats a 2+2 with a return voucher, because the estate has never run either.

So FR#2B needs real experiments on family-pass composition, bundles, daypart surcharges, and return voucher versus membership. And the requirement that makes it architecturally interesting is this: **assignment must be sticky for the visit and available offline** - either baked into the ticket or read from a local flag snapshot.

Stickiness is not a nicety. A guest who is quoted a bundle price on the web and then sees a different offer at a kiosk has been shown two variants, and the experiment measuring the difference is now measuring noise. Worse, the guest has experienced the estate as unreliable about money.

The estate also has a category of things that must never be experimented on: safety closures, evacuation copy, welfare thresholds, exit accessibility, and legal terms. That is a constraint, and an experiment framework that *can* target them will eventually target them by accident.

This record decides how a guest is assigned to a variant and how that assignment survives. It does not decide which experiments to run (commercial) or how results are interpreted ([ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)).

**Foreclosed here:** runtime assignment lookups at a gate or kiosk, and any experiment factor that touches safety, welfare, accessibility, or legal copy.

## Evaluation criteria

- **Sticky for the whole visit, including offline (driving)** - one guest, one variant, across web, kiosk, gate, and any later touchpoint, with no network required to know which.
- **Safety denylist unbypassable (driving)** - the framework must be structurally incapable of assigning a denylisted factor, not merely configured not to.
- **Reconstructable after the fact** - an analysis must be able to say what the guest was actually shown, not what today's config says they should have seen.
- **Works without guest identity** - first visits are anonymous by design (Appendix A 1-4).
- **Cheap to run** - three-person team, no experimentation platform to operate.

## Options

- **Option A - Server-side assignment at each request**: call an assignment service whenever a variant is needed.
- **Option B - Client-side hashing**: each surface hashes a local key to derive the variant deterministically.
- **Option C - Assign at purchase, carry in the entitlement (chosen)**: the variant is decided once, when the order is created, and travels inside the signed claim and the ticket record.
- **Option D - Assign at the gate on first scan**: the lane decides on entry.

| | Sticky offline (driving) | Denylist unbypassable (driving) | Reconstructable | Anonymous-friendly | Cost |
|---|---|---|---|---|---|
| A Per-request | Fail - needs the network exactly where there is none | Partial - config-dependent | Pass | Pass | Medium |
| B Client hashing | Partial - deterministic, but a flag-snapshot skew changes the variant mid-visit | Fail - each client would need to hold the denylist | Partial - depends on snapshot version being recorded | Pass | Low |
| C Assign at purchase | Pass - the variant is in the artefact the guest carries | Pass - one code path, denylist enforced at assignment | Pass - snapshot version recorded with the assignment | Pass - the entitlement is the key | Low |
| D Assign at gate | Fail - too late; the purchase decision, the main thing being tested, has already happened | Pass | Partial | Pass | Low |

Not options: a third-party experimentation SaaS (another vendor, another offline story to solve, and the assignment would still need to reach a disconnected gate); experimenting on prices for guests already inside the estate ([ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)).

## Decision

**The variant is assigned once, at order creation, and carried in the signed entitlement and the ticket record. Nothing downstream ever looks it up.**

```
purchase (web or kiosk)
   |
   +-- assignment service: hash(assignment key) against the active experiment snapshot
   |      |
   |      +-- DENYLIST CHECK: factor must not be safety / welfare / accessibility / legal
   |
   +-- variant written to: ticket record  (analysis)
                           signed claim   (offline surfaces, ADR-0002)
```

Four rules follow.

**The assignment key is the party, not the person.** A family buying one pass is one experimental unit. Using a device or a browser would split a family across variants and measure nothing; using identity would exclude the anonymous first-time guests who are most of the population.

**The denylist is code, and it is checked at assignment.** Not a configuration flag, not a review step. An experiment definition naming a denylisted factor fails to register, so no snapshot containing one can ever be published. This is the only defence that survives a tired person at 5pm on a Friday.

**The assignment record captures the snapshot version.** Experiment definitions change. An analysis six weeks later must reconstruct what this guest was shown, and that is only possible if the version that produced the assignment travelled with it.

**Offline surfaces read the variant, never derive it.** A gate lane or kiosk holding a signed claim already has the variant in it. The experiment flag snapshot in the edge store exists for surfaces that need to know what a *variant means* - which bundle, which voucher copy - not to decide who is in it.

Stickiness and the denylist decided it. Per-request assignment (A) fails on an estate whose defining property is that the network is not there. Client hashing (B) is seductively cheap and fails on the denylist: every surface would have to hold and honour the safety rules, which multiplies the number of places the most important constraint can be got wrong. Gate assignment (D) is too late to test the purchase decision, which is the decision that matters for growth.

Authority is **L3** ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)): assignment happens automatically from a snapshot; commercial designs the experiments; the denylist is never overridable by anyone.

## Key differentiators

- **The guest carries their own variant,** so stickiness needs no network, no session, and no identity.
- **The denylist cannot be configured around,** because it is enforced where experiments are defined rather than where they are read.
- **Anonymous guests are first-class,** which matters because they are the majority and the ones whose return behaviour we most need to learn about.
- **The unit is the family,** matching both how tickets are bought and how visit decisions are actually made.
- **Analyses are reconstructable** because the assignment carries its own snapshot version.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Consistency of guest experience | **Improved (driving)** | One variant per party for the whole visit, regardless of surface or connectivity. |
| Safety | **Improved (driving)** | Structural denylist; no experiment can reach a safety, welfare, accessibility, or legal factor (NFR_7). |
| Availability | **Improved** | Offline surfaces need no assignment service (NFR_4). |
| Auditability | **Improved** | Assignment, variant, and snapshot version are recorded at the moment of assignment. |
| Simplicity | **Improved** | No experimentation platform; the entitlement already travels everywhere. |
| Flexibility | **Weakened (deliberate)** | A variant cannot be changed after purchase. No mid-visit reassignment, no ramping a guest into a new variant. |
| Experiment velocity | **Weakened** | An experiment's population accumulates only from new purchases, so a test on a low-traffic segment is slow. |
| Claim size and coupling | **Weakened** | The signed claim carries a commercial concern, which slightly couples the access credential to the experiment system. |

**Deliberately downplayed: post-purchase flexibility.** A runtime assignment service could reassign, ramp, or kill a variant for guests mid-visit. We give that up because the alternative costs stickiness offline, and a guest who sees two different offers has both broken the experiment and lost trust in the estate's pricing. The practical consequence is that killing a bad experiment stops *new* assignments and does not rescue guests already holding a variant - so experiment definitions need care before publication rather than agility afterwards.

**Fit with the existing architecture.** Same pattern as everything else here: decide once where connectivity exists, put the result in an artefact the edge already holds, and never ask the network at the moment of use. The variant rides in the same signed claim as the party shape ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)) and the flag snapshot sits in the same edge store as the price list ([ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)).

## Consequences

### Positive

- Stickiness holds across web, kiosk, and gate with no network (driving criterion).
- Denylisted factors are structurally unreachable (driving criterion).
- Anonymous first-time guests can be experimented on and measured for 90-day return.
- No experimentation vendor, no additional offline story to solve.

### Negative

- **A bad experiment cannot be recalled from guests who already hold it.** Killing it stops new assignments only. Every experiment definition therefore needs review before publication, which is slower than a platform with a kill switch per guest.
- Low-traffic segments accumulate sample slowly, so some questions take a season to answer.
- The experiment factor is fixed at purchase, so anything about *in-visit* behaviour can only be tested through the itinerary surface, not through the pass itself.
- The signed claim now carries a commercial field, so a change to experiment structure touches the access credential's payload - a coupling worth watching.
- Measuring 90-day return requires retaining assignments for at least that long (NFR_9), which is a privacy surface for otherwise anonymous guests.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Denylist bypass | Someone registers a safety-adjacent factor | Enforced in code at definition time; CI test per denylisted category; snapshot publication rejects a non-conforming definition |
| Unrecallable variant | A harmful experiment reaches guests | Pre-publication review; small initial exposure; short validity windows limit how long a variant is in the field |
| Assignment key collision | Two families share a key and a variant | Key derived per order; collisions bounded and detectable in the assignment record |
| Skewed snapshots | Channels on different experiment snapshot versions assign differently | Assignment happens only at purchase, and the version is recorded, so skew is observable rather than silent |
| Peeking | Calling a winner early on a partial sample | Pre-registered primary metric; minimum sample and duration before any read is acted on |
| Safety metric treated as optimisable | An experiment "wins" while accessibility or welfare degrades | Safety and welfare metrics are guardrails, never objectives; a guardrail regression invalidates the result |
| Anonymous retention | Assignments held 90 days for return measurement | Assignment key is not identity; retention bounded by NFR_9; no join to inferred attributes ([ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)) |

## Verification

**Primary metrics**

- Assignment stickiness violations - one party observed under two variants. Target zero.
- Pre-registered primary metric per experiment: yield per visitor and/or 90-day return (OKR 2.2, 3.1).
- Guardrail metrics per experiment: family-pass accessibility, complaint rate, welfare and safety indicators - monitored for regression, never optimised.
- Share of assignments whose snapshot version is recorded - must be 100%, or the analysis is unreconstructable.

**Tests (CI)** - golden cases in [`evals/pricing/`](../evals/pricing/) alongside the pricing cases, since experiments and prices are analysed together

- An experiment definition naming a safety closure, evacuation copy, a welfare threshold, exit accessibility, or legal terms fails to register.
- A snapshot containing a denylisted factor cannot be published.
- The same assignment key resolves to the same variant across repeated assignment attempts.
- A gate lane reads the variant from the claim and makes no assignment call.
- An assignment without a recorded snapshot version is rejected.
- A guest holding variant B is never served variant A content by an offline surface.

**Ops check**

- Commercial dry-runs an experiment definition and confirms the denylist rejection is comprehensible, not a cryptic failure.
- Confirm a kiosk with no uplink honours the variant already in a presented claim.

**Open questions**

- Assignment key derivation: order id, or a party identifier that survives a repeat purchase - before the first experiment, and it matters for measuring return visits.
- Minimum sample and duration policy per experiment class - with commercial, before the first read.
- Whether return vouchers are a variant of the pass or a separate post-visit artefact - before the loyalty experiments in [Appendix C](../requirements/Appendix%20C_%20Future%20scope.md) phase 2.
- Denylist review cadence and who may add to it - before go-live. Adding to it must be easy; removing from it must not.

**Revisit triggers**

- An experiment needs mid-visit reassignment to answer a real commercial question - revisit, knowing the offline cost.
- Experiment velocity becomes the bottleneck on growth decisions - consider parallel non-overlapping experiments before considering runtime assignment.
- A denylist rejection is overridden by anyone, by any mechanism - stop and re-examine this record.

## Conclusion

The variant is assigned once at purchase, from a versioned experiment snapshot, with a code-level denylist on safety, welfare, accessibility, and legal factors, and it travels inside the signed entitlement so every offline surface reads rather than derives it. Chosen on offline stickiness and on making the denylist structurally unbypassable. The cost is that a variant cannot be recalled from a guest who already holds one, which makes pre-publication review the control rather than post-hoc agility.

Related: [ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) (the claim that carries the variant), [ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md) (the snapshot mechanism), [ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md) (analysing who responded), [hld/scenarios/yield](../hld/scenarios/yield/README.md).
