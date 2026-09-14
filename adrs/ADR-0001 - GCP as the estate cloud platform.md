# ADR-0001 - GCP as the estate cloud platform, with the estate able to run without it

## Date

2026-09-14

## Status

Proposed

## Context

The estate needs somewhere to put telemetry history, run the three AI deep-dives, and produce Countess-level reporting. The kata permits cloud services on one condition: there must be a way of getting information from the estate to the cloud ([4_Assumptions and constraints](../requirements/4_Assumptions%20and%20constraints.md)).

That condition inverts the usual cloud decision. The estate's hot paths - gate admit, checkout price display, keeper data entry - are specified to work while the cloud is unreachable (NFR_2, NFR_4). So the cloud is not the system of record for access. It is the reconciler, the warehouse, and the place models are trained and scored in batch.

What the cloud is actually being bought for:

- **Ingest** of MQTT telemetry from ~500-1000 devices across 40 rides and 55 displays, including multi-hour backfill spikes after a Wi-Fi island reconnects (NFR_1).
- **Three async AI workloads**: overnight pricing proposals, 30-90 minute flow forecasts, animal health anomaly scoring. None of them sit on a request path.
- **Warehouse and reporting**: popularity, yield per visitor, 90-day return, inspection compliance.
- **Eval and drift infrastructure** to satisfy NFR_13.

This record decides the managed-service home for those four things. It does **not** decide the estate-to-cloud transport ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)), how models are served or swapped ([ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)), or anything about gate behaviour ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)).

**Foreclosed here:** a cloud-only architecture. No later record may put the cloud on the gate admit path, the checkout price read, or keeper data entry.

## Evaluation criteria

- **Fit for burst ingest and batch AI (driving)** - a reconnecting gateway dumping hours of buffered telemetry must not need capacity planning, and three scheduled ML jobs must not need a cluster to be babysat by a three-person team.
- **Operability for a three-person team (driving)** - managed over self-hosted everywhere it is offered. We do not have an SRE rota.
- **Portability under vendor uncertainty (NFR_14)** - the two-week capability-swap target must survive this choice.
- **Cost visibility (NFR_12)** - per-pipeline cost attribution for ticketing, ingest, and inference, with killable inference.
- **Privacy posture (NFR_8)** - region pinning available, because the jurisdiction is undecided and GDPR-class is the default bar.

## Options

- **Option A - GCP (chosen)**: Pub/Sub ingest, BigQuery warehouse, Cloud Run services, Vertex AI for models.
- **Option B - AWS**: IoT Core, Kinesis/MSK, Redshift or Athena, SageMaker.
- **Option C - Azure**: IoT Hub, Event Hubs, Synapse, Azure ML.
- **Option D - Self-hosted on estate**: everything on-premise; no cloud dependency at all.

| | Burst ingest + batch AI (driving) | Operability for 3 people (driving) | Portability | Cost visibility | Privacy pinning |
|---|---|---|---|---|---|
| A GCP | Pass - Pub/Sub absorbs backfill without provisioning; BigQuery is serverless so the warehouse has no idle cost | Pass - fewest pieces to run; Cloud Run needs no cluster | Partial - Pub/Sub and BigQuery are proprietary, but the contracts above them are ours | Pass - per-service labels and budget alerts | Pass - region pinning |
| B AWS | Pass - IoT Core has the richest device-management story | Partial - more assembly; MSK or Kinesis shard planning is real work | Partial - same class of lock-in | Pass | Pass |
| C Azure | Pass | Partial - Synapse is heavier than we need for this data volume | Partial | Pass | Pass |
| D Self-hosted | Fail - training, backfill, and the warehouse all land on estate hardware and the estate's patchy power and network | Fail - three people cannot run a broker cluster, a warehouse, and a training rig | Pass - total portability | Partial | Pass |

Not options: multi-cloud from day one (the team is three people and the kata is one estate); serverless-only with no warehouse (popularity and 90-day return both need history).

## Decision

**Use GCP as the managed home for ingest, warehouse, async AI, and reporting - and keep the estate operable without it.**

| Concern | Service | Why this one |
|---|---|---|
| Event backbone | Pub/Sub | Absorbs reconnect backfill without shard math; the one property that matters most for a patchy-Wi-Fi estate |
| Stream/batch processing | Dataflow | Dedupe, gap detection, enrichment on the ingest path ([ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)) |
| Warehouse | BigQuery | Serverless, so 55 displays' worth of telemetry has no idle cost; the tiering in NFR_9 maps to native lifecycle |
| Services | Cloud Run | Ticketing, entitlement issue, intranet BFF; scale-to-zero suits a 5,000/day estate growing to 15,000 |
| Models | Vertex AI | Training, batch scoring, eval - behind our own interface per [ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) |
| Identity (staff) | Identity Platform | Staff intranet roles; guest identity is optional by design (Appendix A 1-4) |

Two rules follow.

**The cloud is the reconciler, never the authority on access.** Entitlement issue happens in the cloud; entitlement *verification* happens at the gate with no network ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)). Field writes are append-only events the cloud merges after the fact.

**The portable contracts are ours, and they are the ones at the edges.** MQTT topics and payloads, entitlement claim format, event schemas on Pub/Sub, and the capability interfaces in front of models. Anything GCP-specific lives behind those. This is what makes NFR_14's two-week swap arithmetic rather than archaeology.

Burst ingest and operability decided it. AWS IoT Core is the better device-management product and would win an estate with tens of thousands of devices; at 500-1000 devices the ingest burst and the size of the team matter more, and Pub/Sub plus BigQuery is the smaller thing to run. Self-hosting is the honest alternative given the offline posture, and it loses because the estate's own power and network are the least reliable part of this architecture - putting the warehouse behind them is the opposite of the intent.

## Key differentiators

- **Backfill is absorbed, not planned for.** The characteristic failure of this estate is a gateway reconnecting with hours of buffered telemetry. Pub/Sub plus BigQuery makes that a cost line rather than an incident.
- **No idle cost for a seasonal estate.** Scale-to-zero services and a serverless warehouse fit a business with wet Tuesdays.
- **Async AI needs no always-on inference tier.** All three deep-dives are scheduled jobs, so the cheap batch path is also the correct one.
- **The lock-in is confined to the middle.** Devices speak MQTT, gates verify signed claims, and models sit behind interfaces - so the replaceable parts are the ones most likely to need replacing.

## Architecture characteristics

| Characteristic | Effect | Why |
|---|---|---|
| Elasticity | **Improved (driving)** | Reconnect spikes and 3x visitor growth are absorbed by managed ingest rather than by capacity planning (NFR_1). |
| Operability | **Improved (driving)** | Managed services mean a three-person team runs product, not infrastructure. |
| Cost efficiency | **Improved** | Serverless warehouse and scale-to-zero services suit variable visitation (NFR_12). |
| Availability of analytics | **Improved** | Cloud RTO of 4 hours for the intranet view is acceptable precisely because the gate does not depend on it (NFR_2). |
| Portability | **Weakened** | Pub/Sub, BigQuery, and Dataflow are proprietary. Migration means rewriting the ingest and warehouse layers, even though the contracts survive. |
| Simplicity | **Weakened (deliberate)** | Edge plus cloud is two deployment targets and two failure models, versus one. This is the price of the offline requirement, not of GCP. |
| Cost predictability | **Weakened** | BigQuery scan and Pub/Sub volume are usage-priced, so a bad query or a chatty device shows up on the bill. |

**Deliberately downplayed: portability of the data platform.** We keep the interfaces portable and accept that the plumbing is not. The kata's stated uncertainty is about *AI models and providers* (NFR_14), which is where the abstraction is spent. Abstracting the warehouse as well would buy insurance against a risk nobody has named, at the price of the thing we actually need - a small team shipping three verifiable AI capabilities.

**Fit with the existing architecture.** The cloud sits behind the same event backbone the rest of the estate uses: MQTT into Pub/Sub, append-only field events, reconciliation after the fact. The AI capabilities read from the warehouse and publish snapshots back to the edge; none of them introduce a new identity system or a new network dependency, which is the NFR_15 bar.

## Consequences

### Positive

- Reconnect backfill is a managed concern (burst ingest).
- One cloud account, one IAM model, one billing view for a three-person team (operability).
- Async pricing, flow forecast, and health scoring all run as scheduled jobs with no serving tier to keep warm.
- Region pinning is available the day a jurisdiction ADR lands.

### Negative

- **Two deployment targets forever.** Every capability now has an edge question: what does this do when the cloud is gone? That question is the point, but it is also permanent overhead on every future feature.
- Warehouse and ingest migration would be a rewrite, not a reconfiguration.
- Usage pricing means a single unbounded query or a misconfigured device can move the bill before anyone notices - NFR_12's per-pipeline visibility is a requirement, not a nicety.
- GCP skills are less common than AWS in the general hiring pool, which matters for a volunteer-scale team.

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| Cloud treated as the gate's authority | The easiest bug in this architecture is a developer adding a cloud call to the admit path | CI assertion that the checkpoint validation path makes no outbound calls ([ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)) |
| Cost spiral | Usage-priced warehouse plus vision or GenAI inference exceeds ticket margin | Per-pipeline labels, budget alerts, inference budget per capability, data tiering hot 30d / warm 1y / cold 3y (NFR_9) |
| Vendor lock-in | Ingest and warehouse are proprietary | Portable contracts at MQTT, claim format, event schema, and model interface; ingest and warehouse accepted as the lock-in surface |
| Single-region outage | Estate analytics and reporting unavailable | Edge keeps operating; ops sees last-known-good with a freshness banner; RTO 4h is acceptable by NFR_2 |
| Jurisdiction undecided | Region choice may be wrong for the eventual legal home | Region pinning is configuration; no data-residency-dependent design choices until the jurisdiction ADR |
| Model provider change | Vertex model deprecated or repriced | Capability interface and pinned versions ([ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)) |

## Verification

**Primary metrics**

- Backfill drain time after a simulated multi-hour island, and whether any health or access event was lost.
- Cloud cost per pipeline (ticketing, ingest, inference) reported separately (NFR_12).
- Ingest lag p95 under healthy links, target ≤30 seconds (NFR_3).

**Tests (CI)**

- No cloud SDK import is reachable from the gate verification path or the checkout price read.
- Event schemas validate against the published contract; an unknown field does not drop a message.
- A replayed duplicate telemetry batch produces no duplicate rows downstream (idempotent consumers).

**Ops check**

- Freshness of each edge cache and MQTT gateway lag visible on the duty-manager view (NFR_11).
- Budget alert fires in a staging project before it matters in production.

**Open questions**

- Region and eventual jurisdiction - before any guest PII is stored beyond a ticket.
- Whether the ticketing store is Cloud SQL or Firestore - before ticketing implementation; both satisfy this record.
- ~~Per-pipeline budget ceilings, especially the inference line - before the first AI capability leaves shadow.~~ **Resolved** in [cost-analysis](../cost-analysis/README.md#8-per-pipeline-budget-ceilings): five labelled pipelines totalling $72.40/month modelled, $242 alert, $680 ceiling. The inference line is $14.47.

**Revisit triggers**

- Device count grows past a few thousand, or device management (firmware, rotation, provisioning at scale) becomes the dominant operational cost - re-evaluate AWS IoT Core.
- BigQuery or Pub/Sub cost exceeds the ticketing line - revisit tiering before revisiting the provider. Modelled, warehouse is $10.10 against ticketing's $24.55, so this trigger fires at roughly 2.5x today's sensor and dashboard load ([cost-analysis](../cost-analysis/README.md#8-per-pipeline-budget-ceilings)). It is a live trigger, not a theoretical one, and the first mitigation is materialised views rather than anything to do with the provider.
- A jurisdiction ADR lands that GCP cannot satisfy in-region.

## Conclusion

GCP hosts ingest, the warehouse, async AI, and reporting, chosen on burst-ingest fit and on being the smallest thing a three-person team can operate. The estate is designed to keep admitting guests and recording keeper observations when that cloud is unreachable, so the lock-in we accept is in the data platform, not in the paths that must never fail.

Related: [ADR-0002](ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md) (offline gate), [ADR-0003](ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md) (the ingest path this record assumes), [ADR-0004](ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) (model portability).
