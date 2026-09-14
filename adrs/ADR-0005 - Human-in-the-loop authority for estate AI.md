# ADR-0005 - Authority is assigned per capability, and no model holds authority over a safety or welfare outcome

## Date

2026-09-14

## Status

Proposed

## Context

The estate has poisonous animals and 18th-century amusement rides open to the public for the first time. Several NFRs already draw the line: ride status, e-stop, escape, and evacuation are deterministic and human-authored (NFR_7); animal and ride decisions need human confirmation (NFR_15); AI may draft a work order but may not open a ride or silence a welfare alarm.

What is missing is the *mechanism*. "Human in the loop" said generally produces one of two failures, and both are worse than no AI:

- **Rubber-stamping.** A queue of low-value confirmations trains the duty manager to click accept without reading. The human is nominally in the loop and functionally absent - and now there is an audit trail implying review that did not happen.
- **Alert fatigue.** High-recall animal alerts (FR#2I demands recall first) mean false positives by design. Staff who learn to ignore the inbox will ignore the true positive too. This is already a named risk in [5_Risks and mitigation](../requirements/5_Risks%20and%20mitigation.md).

So the design problem is not whether humans are involved. It is **how much human attention each capability is allowed to spend, and who owns the outcome when it is wrong**.

This record defines authority levels and the promotion path between them, for every AI capability in FR#2. It does not decide the models ([ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)), thresholds (capability ADRs), or intranet layout.

**Foreclosed here:** any capability acting autonomously on a welfare, safety, or access outcome, at any confidence, ever. No later ADR may promote its way past this.

## Evaluation criteria

- **No autonomous safety or welfare action (driving)** - a hard constraint from NFR_7 and the hazardous-operations constraint, not a tunable.
- **Attention is budgeted (driving)** - review volume per role per shift must be small enough that review is real. A keeper has animals to look after.
- **Accountability is unambiguous** - for every AI-influenced outcome, exactly one human role owns it.
- **Feedback is captured** - accept and reject are the training signal (FR#2I) and the drift detector.
- **Promotion is earned** - autonomy inside a band is reachable, but only through shadow and measured precision.

## Options

- **Option A - Uniform confidence bands across all capabilities**: one threshold policy, auto above, review in the middle, suppress below.
- **Option B - Review everything**: no AI output acts without a human, in every case.
- **Option C - Authority assigned per capability, with a promotion path (chosen)**: each capability is placed at an authority level based on what a wrong answer costs; promotion requires shadow evidence; safety and welfare are capped by policy.
- **Option D - Autonomy with post-hoc audit**: act, log, let humans correct afterwards.

| | No autonomous safety action (driving) | Attention budgeted (driving) | Accountability | Feedback | Promotion |
|---|---|---|---|---|---|
| A Uniform bands | Fail - a threshold high enough for price is not a defence for welfare | Fail - the same band produces a trickle for pricing and a flood for animals | Partial | Pass | Partial |
| B Review everything | Pass | Fail - guarantees rubber-stamping, which is the failure mode dressed as rigour | Pass | Pass | Fail - nothing can improve |
| C Per-capability authority | Pass - policy cap, not a threshold | Pass - volume is a design input per role | Pass - one owner per outcome | Pass | Pass - shadow-gated |
| D Post-hoc audit | Fail | Pass | Fail - nobody owns it until after harm | Partial | n/a |

Not options: AI clearing a safety hold or closing a welfare alarm ([Appendix C](../requirements/Appendix%20C_%20Future%20scope.md) defers automatic ride return-to-service, and this record forecloses it for welfare permanently); experimenting on safety closures, evacuation copy, or welfare thresholds (denylisted by constraint).

## Decision

**Each capability is assigned one of four authority levels. The level is set by the cost of a wrong answer, not by model confidence, and welfare, safety, and access are capped at Advisory or below by policy.**

| Level | What the system may do | Human role |
|---|---|---|
| **L0 Observe** | Record and score only. Nothing is shown as a recommendation. | Nobody acts. This is shadow mode. |
| **L1 Inform** | Display a signal with confidence, evidence, and freshness. No suggested action. | Reader interprets. |
| **L2 Advise** | Propose a specific action with evidence. Nothing happens until a human accepts. | Named role accepts or rejects, with a reason on reject. |
| **L3 Act-in-band** | Act automatically inside limits a human set in advance, with a kill switch. | Human sets and reviews the band; does not see each instance. |

Assignment:

| Capability | Level | Owner | Ceiling |
|---|---|---|---|
| Gate admit / deny | **Not AI** | Gate policy; staff break-glass is audited | Deterministic forever ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)) |
| Ride open / closed / evacuate | **Not AI** | Ride ops, duty manager | Deterministic forever |
| Animal health anomaly | L2 Advise | Keeper, escalating to vet | **L2 max** - never promotes ([ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)) |
| Piranha population estimate | L1 Inform | Keeper | **L1 max** - it is a number with an interval, not an action ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)) |
| Predictive ride maintenance | L2 Advise | Ride engineer | **L2 max** - may draft a work order, never change ride status |
| Staffing recommendation | L2 Advise | Duty manager | L2; volume capped per shift ([ADR-0013](ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md)) |
| Flow / congestion forecast | L1 Inform | Duty manager | L1 |
| Price proposal | L3 Act-in-band after shadow | Commercial sets floors and ceilings | L3 inside approved bands only ([ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)) |
| Experiment assignment | L3 | Commercial designs; denylist is code | L3, denylist never overridable ([ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)) |
| Cohort analysis | L1 Inform | Commercial, Countess | L1 ([ADR-0012](ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)) |
| Guest itinerary suggestion | L3 | Guest experience; safety routing is code | L3; no routes through restricted areas |
| Win-back offer | L3 Act-in-band | Commercial | L3 inside frequency caps and opt-out |
| Ops copilot (optional) | L1 Inform | Staff remain accountable | L1; display only, citations required |

Four rules follow.

**Pricing reaches L3 and animal health never does - and the reason is the cost of being wrong.** A bad price inside an approved band costs margin on some tickets and is reversible by republishing. A missed sick animal is an animal suffering, and an over-trusted health alarm that auto-resolves is a keeper not visiting. Confidence cannot buy authority here, because the failure is not probabilistic in cost.

**Every review has an attention budget, and exceeding it is a capability defect.** Each L2 capability declares a maximum items per role per shift. Above that, the capability tunes its threshold or suppresses duplicates - it does not spend more of the keeper's day. A capability that cannot stay inside its budget is not ready to leave shadow, because a queue nobody can work is worse than no queue.

**Reject requires a reason code; both accept and reject are training data.** This is the FR#2I signal and the drift detector. An L2 capability whose accept rate collapses has already failed, whether or not its offline metrics look fine.

**Promotion is L0 to L1 to L2, evidence-gated, and capped.** A capability enters at L0, and moves up only with golden cases passing, a measured precision at its operating threshold, and a review volume inside budget. The ceiling in the table above is policy: no amount of evidence promotes animal health past L2.

Attention budgeting and the hard safety cap decided it. Uniform bands (A) fail because one threshold cannot serve both a reversible price and an irreversible welfare outcome, and because the same band produces a trickle in one capability and a flood in another. Reviewing everything (B) sounds safest and is the trap: it manufactures rubber-stamping and then records it as oversight.

## Key differentiators

- **Authority is set by consequence, not by confidence.** A 99%-confident welfare alert still does not get to act, because the cost of the 1% is an animal.
- **Human attention is a budgeted, first-class resource.** Review volume is a design constraint, so the loop stays real.
- **Exactly one role owns each outcome,** so nothing lands in the gap between keeper and duty manager.
- **Rejections are captured as signal,** making the loop a training and drift mechanism rather than a gate.
- **The ceiling is policy, not configuration.** No future threshold tuning can quietly promote a welfare capability.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Safety | **Improved (driving)** | No model holds authority over an animal, a ride, or an admission (NFR_7). |
| Auditability | **Improved (driving)** | Every AI-influenced outcome has a role, a timestamp, a model version, and a reason on reject. |
| Accuracy over time | **Improved** | Accept and reject are labels; the loop improves the model it gates (FR#2I). |
| Trust | **Improved** | Volume caps and duplicate suppression keep the inbox worth reading (NFR_11). |
| Observability | **Improved** | Accept rate per capability is a leading indicator of drift, visible before offline metrics move. |
| Latency to action | **Weakened (deliberate)** | An L2 alert waits for a human. In an animal emergency that delay is real, and it is why deterministic alarm paths exist separately. |
| Operating cost | **Weakened** | Review is staff time. A capability that cannot justify its attention budget should not exist. |
| Capability ceiling | **Weakened** | Welfare and safety capabilities can never be fully automated, so their efficiency gain is bounded by human throughput. |

**Deliberately downplayed: speed of automated response on welfare and safety.** A fully autonomous welfare responder would act in seconds rather than minutes. We refuse it, and the mitigation is that genuine emergencies - e-stop, escape, evacuation - travel deterministic alarm paths that do not involve a model at all. AI here is for the slow, expensive failure the brief actually describes: an animal quietly eating less for three days. That problem does not need seconds.

**Fit with the existing architecture.** This is the gate's posture applied to models: the authoritative decision sits with the party closest to the consequence, remote inference is allowed to be wrong, and confirmed facts outrank estimates. A keeper-confirmed carcass decrements deterministically ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)) for the same reason a valid entitlement admits without a model vote. The review queue lives in the ops intranet on the same offline-capable surfaces as the rest of the keeper workflow (NFR_17) - not in a separate AI product.

## Consequences

### Positive

- No AI capability can act on a welfare, safety, or access outcome (driving criterion).
- Review volume is bounded per role, so the loop stays meaningful (driving criterion).
- Accept and reject rates give early drift warning per capability.
- Pricing and offers still get real automation, inside bands a human set.

### Negative

- **Welfare response is gated on human availability,** including at 3am. The estate needs an on-call keeper path, or overnight alerts wait - a staffing consequence of an architectural choice.
- **Attention budgets will force suppression of true positives.** A capability at its volume cap raises its threshold, which means some real anomalies are not surfaced. This is a deliberate trade of recall for a queue that gets read, and it must be measured, not assumed.
- Review is a permanent operating cost that grows with the estate, and it caps how much the AI can ever save.
- Four authority levels are more concept than a small team wants; without the promotion checklist being enforced, everything drifts to L2 by default.
- Reason codes need a taxonomy that keepers will actually use, or the training signal degrades to "rejected: other".

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Rubber-stamping | Accept becomes a reflex | Volume caps per role per shift; duplicate suppression; accept-rate monitoring; periodic golden-case injection to check reviewers are discriminating |
| Alert fatigue | High-recall welfare alerts flood the keeper inbox | Attention budget is a promotion gate; suppression of repeat alerts on the same subject; evidence and confidence shown so triage is quick |
| Suppressed true positive | Threshold raised to meet the budget hides a real case | Recall against keeper-labelled events tracked per period; a recall drop is a capability failure, not an acceptable saving |
| Overnight gap | L2 alerts wait for a human who is asleep | On-call escalation for welfare-critical subjects; deterministic alarms remain separate and immediate |
| Authority creep | A capability quietly promoted | Ceilings in this table are policy; CI asserts declared level against the policy table; promotion needs a written eval record |
| Reason-code decay | Everything rejected as "other" | Small taxonomy designed with keepers; "other" rate monitored as a data-quality metric |
| Accountability gap | An outcome nobody owned | Every capability names one owning role here; unassigned capabilities cannot leave L0 |

## Verification

**Primary metrics**

- Review items per role per shift, per capability, against its declared budget.
- Accept rate per capability, with a drop treated as a drift signal (OKR 4.3 for staffing suggestions).
- Recall against human-labelled events for L2 capabilities, tracked so budget-driven threshold raises are visible.
- Time-to-acknowledge for welfare alerts, including overnight.
- "Other" share of reject reason codes.

**Tests (CI)**

- A capability declaring an authority level above its policy ceiling fails the build.
- No code path lets an AI output change ride status, silence a welfare alarm, or admit at a gate.
- An L2 output cannot produce a side effect without a recorded human decision.
- Reject without a reason code is rejected.
- Every AI-influenced outcome record contains role, timestamp, model version, confidence, and evidence reference.
- An experiment assignment cannot target a denylisted factor (safety closure, evacuation copy, welfare threshold, exit accessibility).

**Ops check**

- Keepers and the duty manager confirm the inbox is workable in a real shift, including in an animal house with the uplink down.
- Injected golden cases confirm reviewers still discriminate rather than accept by reflex.

**Open questions**

- Attention budget numbers per role per shift - with keepers and the duty manager, before any capability leaves shadow. These are the numbers this whole record depends on and we do not have them yet.
- Overnight welfare escalation path, including whether an on-call keeper exists - with the Countess, before the animal collection opens to the public.
- Reason-code taxonomy - with keepers, before [ADR-0022](ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md) leaves shadow.
- Break-glass procedure and audit for a gate admit override - before launch.
- Minimum shadow duration and precision bar for L1 to L2 promotion - before the first promotion.

**Revisit triggers**

- Accept rate above roughly 95% with no rejections - either the capability is perfect or nobody is reading; investigate before trusting it.
- A capability persistently over its attention budget - retune or retire it; do not raise the budget by default.
- Recall falls while the queue stays inside budget - the threshold is being used to hide the problem.
- Any proposal to promote a welfare or safety capability past its ceiling - requires superseding this record, deliberately.

## Conclusion

Authority is assigned per capability by the cost of a wrong answer: pricing and offers may act inside human-set bands, forecasts and cohorts inform, animal health and ride maintenance advise and never act, and gate admit and ride status are not AI decisions at all. Review volume is budgeted per role so the loop stays real rather than ceremonial. The costs are a permanent human-throughput ceiling on welfare capabilities and the honest admission that attention budgets will sometimes suppress a true positive - both preferable to an inbox nobody reads or a model that can silence an alarm.

Related: [ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) (confidence and evidence in the contract this record relies on), [ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) (why admit is not an AI decision), [Appendix B](../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) (the human-in-the-loop table this record implements), [hld/mlops](../hld/mlops/README.md) (promotion pipeline).
