# How we used AI on the animal collection

> Note: This is the animal-care section of the kata's first deliverable - "a short narrative describing how the team used AI to solve the problems of the Von Digitalis Estates". It covers **FR#2I** and **FR#2J** only. Ticketing, popularity, pricing, loyalty, and the ops intranet are other workstreams, and a combined overview does not exist yet.
>
> Boundaries: [animal-care-scope.md](animal-care-scope.md). Fields: [data contract](animal-care-data-contract.md). Decisions: [ADR-020 to ADR-023](../adrs/README.md). Diagrams: [legend and index](../diagrams/00-legend.md).

## What the Countess asked for

Two sentences in the brief, and they are among the few things it asks for outright:

> "The exotic animal collection also requires careful monitoring - we want ways of tracking animal health, how much/well they are eating, and in the case of the jumping pirana collection, we also need to check population levels."

Behind it sits the business reason: 200+ animals across 55 displays are expensive to keep healthy and very much worse when they are not. Illness caught late is the cost curve the estate cannot afford while it is trying to triple visitation.

## What we found when we looked

Four facts about this estate shaped every decision, and none of them favour reaching for a model.

**There are no labels.** The collection was private and the platform is greenfield. Nobody has ever recorded a labelled illness event here. On day one, supervised learning has nothing to learn from.

**"Normal" is not shared.** A python fed once a fortnight and a piranha colony fed daily do not belong in the same distribution. A refusal is routine for one animal and an emergency for another. Across 55 displays there is no population-level notion of eating well.

**The data is thin.** A keeper produces a few observations per subject per day. After a full year one animal has a few hundred feeding records. That is arithmetic territory, not deep learning territory.

**The animal houses have no Wi-Fi.** Thick walls, aquaria, indoor spaces. Keeper observation is the only data source present at every display, and it is captured exactly where the network is worst.

A fifth fact applies only to the piranha: **measuring them harms them.** A census means netting or draining a tank of piranha, which is stressful for the colony and hazardous for keepers. Any design that assumes frequent census has quietly bought a number with an animal-welfare cost.

## Four decisions

### Decide what a record is about before deciding how to analyse it

The obvious model attaches everything to an enclosure - 55 displays, 55 rows, done. It fails on the day it matters most. Move a sick animal to quarantine and its history stays behind in the old room, at precisely the moment a vet needs it.

So the unit of record is a **care subject** - an individual, a group, or a colony - and the enclosure is a dated placement ([ADR-020](../adrs/ADR-020-use-care-subject-as-unit-of-record.md)). Identity survives re-housing. It also makes the piranha ordinary rather than exceptional: a colony is simply a subject whose population is an estimate, so `FR#2J` needs no special case. And it means keepers are never asked to invent an identity for a fish that cannot be told apart from its neighbours - fabricated identity would poison the training data at source.

### Treat a keeper's note differently from a sensor reading

They look like the same event stream and they are not. A dropped water-temperature reading is a gap, and another arrives in minutes. A dropped note about a lethargic venomous snake is unrecoverable - the animal was seen once, at that moment. Worse, since keeper decisions are the training signal, a silently lost note does not merely lose data. It **biases the detector toward whatever keepers happened to record in good coverage.**

So keeper observations stay off the MQTT telemetry path ([ADR-021](../adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md)). The device is a durable first system of record with an append-only local log, idempotent event ids, and batch sync whenever a link appears. The keeper is released the moment the device acknowledges, never when a server does. Photos sync separately and later, so a slow upload never delays a welfare alert.

### Compare each animal to itself, and let the ops tool build the training set

With no labels, 55 incompatible notions of normal, and a few hundred records per animal per year, the honest detector is not a model. It is arithmetic ([ADR-022](../adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md)).

The detector is a ladder, and **the tier in use is set by label availability rather than by what is fashionable**:

| Tier | What it is | At launch |
|---|---|---|
| 0 | Hard environment thresholds, evaluated at the edge | Live, never model-gated |
| 1 | Each subject scored against its own recent history | Live - the day-one detector |
| 2 | Supervised model trained on accumulated keeper decisions | Not yet. Shadow until it beats tier 1 |
| 3 | GenAI narrative over evidence that already exists | Presentation only |

The part we think is the real idea: **tier 1's second job is to manufacture the dataset tier 2 needs.** The alert inbox, with its accept and reject, is a labelling machine disguised as an ops tool. The system earns its way up the ladder instead of waiting for a data-collection project nobody would fund.

Two rules keep it safe. Tier 0 runs at the edge because an oxygen crash during a four-hour outage cannot wait for the cloud. And the suggested action on every alert is a **lookup from a keeper- and vet-authored playbook, never generated** - an invented veterinary dose is a failure, not a cleverness.

### Count the piranha by refusing to count them

They occlude each other, so any optical count undercounts by an amount that scales with density - which is the very quantity being measured. And nobody acts on the level anyway. The Countess does not deploy staff because there are 417 piranha; she acts when the colony is losing individuals or breeding out of control.

So the published answer is an interval anchored on human census, never a number ([ADR-023](../adrs/ADR-023-anchor-piranha-population-on-human-census.md)). Each signal does exactly one job: census anchors the absolute value, feed consumption tracks change between anchors, a recovered carcass decrements by exactly one, and vision - if it is ever funded - is a third opinion running in shadow.

**The interval widens as the anchor ages**, so the system asks for a census when it needs one. Measurement is scheduled by uncertainty rather than by calendar habit, which is the only way to keep netting a tank of piranha down to the occasions when it buys something. Past a maximum age the estimate publishes as *unusable* rather than as a number with a very wide band, because a wide interval still reads as an answer while "unusable" reads as a request.

## How we will know it is working

The brief asks how we will confirm the AI is actually working and detect it misbehaving in production. Three answers.

**Most of the detector is ordinary software.** Tiers 0 and 1 are arithmetic and exactly reproducible, so a golden case either passes or the code is wrong. Eleven cases are written across [`evals/animal-health-anomaly/`](../evals/animal-health-anomaly/) and [`evals/piranha-population/`](../evals/piranha-population/), each naming the behaviour it defends and what must never happen. Two of them are deliberately the same fixture differing only in a declared seasonal state, so a reviewer can see that the human declaration and nothing else changed the outcome.

**The non-deterministic tier is tested by constraint, not by expected output.** The GenAI narrative case asserts that it names no disease, states no dose, recommends nothing beyond the playbook text, and cites only fields present on the alert.

**Recall needs a denominator the system does not own.** This one has an architectural consequence we had to design for: a keeper must be able to open a welfare event with **no alert prompting them**, and those unprompted confirmed events are what recall is measured against. Without that path the detector can only ever measure its own opinion of itself. For the piranha the equivalent is interval coverage - if the interval claims 90 percent, the census should land inside it about 90 percent of the time. Under-coverage means overconfidence; over-coverage means the interval carries no information.

## When the model vendor changes, raises prices, or disappears

The brief asks this directly. Our answer is structural rather than a promise about abstraction layers.

**Only tier 3 touches a third party.** It writes prose over evidence that already exists. It produces no score, makes no decision, and cannot raise, suppress, or downrank an alert. If that provider triples its price or shuts down tomorrow, welfare detection is entirely unaffected and alerts fall back to template text assembled from the same fields - they simply read like a spreadsheet.

That is the whole of this capability's exposure. No vendor stands between an animal and a keeper. You can see it in the [targeted AI view](../diagrams/ai-animal-health-anomaly.md): the vendor box connects to exactly one thing.

## What we deliberately did not build

- **No vision by default.** It is the known opex risk, it needs labels too, and for the piranha it carries the occlusion bias described above. If funded, it is scoped to surface-feeding frames, where jumping behaviour makes individuals separable, and it starts in shadow.
- **No language model as a detector.** Non-deterministic, no calibrated confidence on rare events, drifts silently on a provider update, and bills continuously across 55 displays.
- **No individual identification of fish**, and no tagging of a colony of this size and temperament.
- **No autonomous action of any kind.** Keepers and vets decide; safety thresholds answer to no model.

## What is unfinished

- Nothing executes the golden cases. They are specifications of correct behaviour, not a harness.
- Tier 2 does not exist, and for small collections it may never accumulate enough confirmed positives to be worth training. Tier 1 being the permanent answer is an accepted outcome.
- Cannibalism is close to invisible in the piranha tank, and it is the most likely loss mode. A fish eaten entirely leaves no carcass, and under feed-to-appetite the tank consumes the same total regardless. Our answer is a fixed-offer feeding protocol, which is a demand on how keepers work and has not been agreed with anyone.
- Vocabularies for subject lifecycle, placement reason, expected state, and census method are open, and so is the retention period NFR_9 leaves undefined.
- The MQTT broker and gateway placement this workstream depends on are named as assumptions because **no workstream owns that decision**.
