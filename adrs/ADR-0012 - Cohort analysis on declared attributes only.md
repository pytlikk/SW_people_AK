# ADR-0012 - Cohorts are built from declared and consented attributes, and inferred segments never become records

## Date

2026-09-14

## Status

Proposed

## Context

"Variant B won by 3%" is not an investment decision. The Countess needs to know *who* responded - families with toddlers on wet Tuesdays, or locals, or coach parties - and where they went, because that is what turns an experiment result into a decision about which enclosure to refurbish (FR#2C).

The obvious way to get rich demographics is to infer them: cameras at the gate estimating age and group composition, or behavioural clustering that assigns a household type. The estate is about to install sensors anyway, so the marginal cost looks small.

It is not small. NFR_8 says occupancy and popularity prefer counts and opted-in trails over biometric identification, that inferred demographics are analytical and not CRM, and that camera use is purpose-limited. [5_Risks and mitigation](../requirements/5_Risks%20and%20mitigation.md) names the failure directly: cameras sold as "popularity" that become unlawful surveillance, and cohort AI leaking into marketing profiles. The jurisdiction is undecided, so GDPR-class is the default bar.

There is also a quality argument that usually gets forgotten. Inferred demographics are *guesses with an error rate*, and grouping guests by a guess means every downstream conclusion inherits that error invisibly. A declared "2 adults + 2 children" from a family-pass purchase is a fact the guest asserted. An inferred "family with young children" from a camera is a model output that will be wrong for some proportion of grandparents, school groups, and tall teenagers - and nothing downstream will ever know which.

This record decides what cohorts may be built from and where the output may go. It does not decide the experiments ([ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md)) or the pricing mechanism ([ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)).

**Foreclosed here:** inferred attributes written to a guest record, used to set a price, or used to target an individual offer. Permanently, at any confidence.

## Evaluation criteria

- **Inferred attributes never become identity (driving)** - the NFR_8 bar, and the difference between analytics and surveillance.
- **Actually answers "who and what next" (driving)** - a cohort analysis that cannot inform an investment decision is not worth building.
- **Lawful under an undecided jurisdiction** - defensible on the GDPR-class default.
- **Works with mostly anonymous guests** - identity is optional on first visit, and the majority will stay anonymous.
- **Cost (NFR_12)** - vision and unbounded GenAI narrative are the named opex risks.

## Options

- **Option A - Declared and consented attributes only (chosen)**: ticket type, declared party structure, opted-in membership attributes, plus zone and ride events and spend.
- **Option B - Vision-inferred demographics at the gate**: estimate age and group composition from cameras.
- **Option C - Behavioural clustering promoted to guest profiles**: cluster on movement and spend, then store the assigned segment on the guest record.
- **Option D - Third-party data enrichment**: buy household attributes against a postcode or email.

| | Inferred never identity (driving) | Answers who/what next (driving) | Lawful on GDPR-class default | Anonymous-friendly | Cost |
|---|---|---|---|---|---|
| A Declared + consented | Pass - nothing inferred is stored | Pass - declared party structure is the strongest signal available and it is free | Pass | Pass - declared at purchase, no identity needed | Low |
| B Vision demographics | Fail - creates a biometric-adjacent inference per guest | Partial - richer, but with an unmeasurable error rate | Fail on the default bar | Pass | High |
| C Clustering to profiles | Fail - that is exactly the leak NFR_8 forbids | Pass | Fail | Partial | Medium |
| D Data enrichment | Fail | Partial | Fail without a lawful basis nobody has established | Fail - needs identity | Medium |

Not options: gate cameras for people-counting as a default sensing method ([Appendix C](../requirements/Appendix%20C_%20Future%20scope.md) defers full vision people-tracking; piranha is the named exception); face recognition in any form.

## Decision

**Cohorts are built only from what the guest declared, what they consented to, and what happened. Inferred segments exist only inside an analysis and are never written to a guest record, a price, or an individual offer.**

| Input | Status | Source |
|---|---|---|
| Ticket type and channel | Declared | Purchase |
| Declared party structure (2 adults + 2 children) | Declared | Family-pass purchase - the guest asserted it |
| Membership attributes | Consented | Opt-in, separate from purchase |
| Zone, ride, and enclosure events | Observed, count-level | MQTT and scans ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)) |
| Spend and ancillary attach | Observed | Ticketing, POS when integrated |
| Experiment variant | Assigned | [ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md) |
| Individual guest location trail | Consent required | Opt-in only |
| Inferred age, gender, household type | **Never collected** | - |

Three rules follow.

**The declared party structure is the cohort backbone, and it is better than what inference would have produced.** A family pass purchase states the party shape as a condition of the product. That is a guest-asserted fact with no error rate, available for every family pass sold, requiring no camera and no consent negotiation. The privacy-preserving option is also the higher-quality one here, which is unusual and worth saying out loud.

**Inferred segments are analysis artefacts with a lifetime of one analysis.** A clustering run may discover "wet-weekday small-party visitors convert on bundles". That finding informs the next experiment or the next refurbishment. It does not become a field on anybody's record, it does not select who receives an offer, and it does not enter a price.

**GenAI writes the narrative, never the numbers.** The metrics come from SQL over the warehouse. A model may turn them into readable prose for the Countess, with the underlying figures shown alongside. A cohort insight that cannot be traced to a query is a hallucination with good grammar.

Authority is **L1 Inform** ([ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)). The output is a briefing; commercial and the Countess decide.

Non-identity and genuine usefulness decided it. Vision demographics (B) fail the driving criterion and the default lawful bar simultaneously, and buy resolution with an error rate nobody can quantify. Clustering promoted to profiles (C) is the specific leak NFR_8 names. Enrichment (D) needs identity the estate deliberately does not require.

## Key differentiators

- **The best demographic signal on this estate is already declared and already lawful,** because a family pass has to state its party shape to be a family pass.
- **The privacy constraint improves data quality** rather than degrading it: asserted facts beat inferred guesses.
- **Anonymous guests are fully analysable** at cohort level, which matters because most of them are.
- **No camera is needed for the commercial question,** which removes both the opex and the surveillance risk.
- **Narrative and numbers are separated,** so an insight is always traceable to a query.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Privacy | **Improved (driving)** | No inferred attribute exists to leak, be subpoenaed, or be repurposed (NFR_8). |
| Data integrity | **Improved (driving)** | Cohorts rest on asserted facts and counted events, not on model output with an unmeasured error rate. |
| Cost efficiency | **Improved** | No vision pipeline; GenAI confined to narrative generation over computed metrics (NFR_12). |
| Auditability | **Improved** | Every insight traces to a query over declared and observed data. |
| Legal defensibility | **Improved** | Defensible before a jurisdiction is even chosen. |
| Analytical resolution | **Weakened (deliberate)** | No age, gender, or household inference. Cohorts are coarser than a marketing team would want. |
| Coverage | **Weakened** | Individual admission tickets declare little, so the richest cohorts are family-pass and member buyers. |
| Personalisation ceiling | **Weakened** | Offers target cohorts and consented members, never an inferred individual profile. |

**Deliberately downplayed: analytical resolution.** Inference would give finer segments, and finer segments would give better-targeted offers. We refuse it because a wrong inference is invisible to everyone downstream while a declared attribute is not, because the estate's jurisdiction is undecided, and because the cost of being the heritage estate that put demographic cameras on children is not a data-protection fine - it is the visitor numbers. The practical loss is that solo and pair visitors on plain admission tickets remain a thin cohort, and the answer is to make membership worth joining rather than to start guessing.

**Fit with the existing architecture.** The same rule the whole estate runs on: confirmed facts outrank estimates, and an estimate never gets promoted into a record. A keeper-confirmed carcass decrements deterministically ([ADR-0023](ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md)), a valid entitlement admits without a model vote ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)), and an inferred cohort never becomes a guest attribute. Same posture, three different domains.

## Consequences

### Positive

- Nothing inferred exists to leak into marketing or CRM (driving criterion).
- Cohort analysis works today, on data the platform already collects for other reasons.
- No camera, no vision opex, no surveillance exposure for the commercial use case.
- Every insight is reproducible from a query.

### Negative

- **Plain-admission solo and pair visitors are analytically thin.** We know when they came and where they went, not who they are. If they turn out to be the growth segment, this record limits how much can be learned about them without changing the product to encourage membership.
- Cohorts are coarse, so some real effects will be invisible - a genuine difference between grandparent groups and parent groups is not detectable here.
- Offer targeting is cohort-level, which is weaker than individual targeting and will show up as lower win-back conversion than a data-rich competitor would achieve.
- Declared party structure can be misdeclared by guests to get a cheaper pass, so the backbone attribute has a fraud channel rather than an error rate.
- GenAI narrative still costs money per report and still needs groundedness evaluation.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Inferred segment leaks to CRM | A clustering label written to a guest record | No inferred field exists in the guest schema; CI asserts the schema; analysis outputs are reports, not writes |
| Scope creep to cameras | "Just for demographics" camera proposal | Foreclosed here; any vision for people needs a privacy ADR first (NFR_8), and the piranha exception is purpose-limited |
| Consent conflation | Analytics consent treated as marketing consent | Consent is separate from purchase and separate per purpose (Appendix A 1-4) |
| Hallucinated insight | GenAI narrative invents a finding | Metrics computed in SQL and displayed alongside; groundedness evals; narrative is display-only at L1 |
| Re-identification | Coarse cohorts combined until an individual is identifiable | Minimum cohort size for any reported segment; no cross-joining of trail data with small declared segments |
| Misdeclared party | Guests understate party size or age mix | Gate party-count enforcement catches over-admission; declared structure is treated as a commercial signal, not as verified identity |
| Thin anonymous cohorts | The largest guest group is the least understood | Membership incentives (OKR 3.2); accepted as the cost of anonymity by design |

## Verification

**Primary metrics**

- Count of inferred attributes persisted to a guest record - **target zero, structurally**.
- Cohort coverage: share of visits attributable to a declared or consented cohort.
- Whether cohort findings changed a decision - the honest usefulness test for an L1 capability.
- Groundedness score on generated narrative; forbidden-claim cases pass.

**Tests (CI)** - golden cases in [`evals/cohorts/`](../evals/cohorts/)

- The guest record schema contains no inferred demographic field; adding one fails the build.
- A cohort analysis output cannot be written back to a guest or membership record.
- A reported segment below the minimum cohort size is suppressed.
- Generated narrative claims are checkable against the computed metrics supplied to it; an unsupported claim fails.
- Marketing targeting selects on declared or consented attributes only; an inferred-segment selector fails.
- Analytics consent does not satisfy a marketing-consent check.

**Ops check**

- Commercial confirms a cohort briefing is actionable, not just interesting - the L1 usefulness bar.
- Privacy review of the first real briefing before it is circulated.

**Open questions**

- Minimum reportable cohort size - with a privacy reviewer, before the first briefing.
- Which membership attributes are worth asking for, balanced against signup friction - with commercial, before membership launches.
- Whether ancillary spend (F&B, photos, tours) is integrated, since it changes yield-per-visitor materially - deferred in [Appendix C](../requirements/Appendix%20C_%20Future%20scope.md).
- Jurisdiction, which sets the actual privacy bar rather than our assumed default.

**Revisit triggers**

- Anonymous visitors turn out to be the growth segment and cohort thinness blocks a real decision - the answer is a better membership offer, not inference; revisit only if that fails.
- A jurisdiction ADR establishes a lawful basis and a proportionality case for some inference - revisit deliberately, with a privacy ADR, not by extending this one.
- GenAI narrative cost exceeds its value - drop to metrics-only, which is the fallback in [ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md).

## Conclusion

Cohorts are built from declared party structure, consented membership attributes, and observed events; inferred segments live inside one analysis and never become a stored attribute, a price, or an individual target. Chosen because inferred demographics fail the privacy bar and because the declared party structure is a higher-quality signal than inference would produce. The cost is coarse cohorts and analytically thin anonymous visitors - accepted, with membership incentives as the intended remedy.

Related: [ADR-0010](ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md), [ADR-0011](ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md), [ADR-0005](ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) (L1 Inform), [hld/scenarios/yield](../hld/scenarios/yield/README.md), golden cases in [`evals/cohorts/`](../evals/cohorts/).
