# Glossary

Two lists. The first is the one worth reading: words this repository uses in a specific way, where guessing from context would get you a slightly wrong answer. The second is the ordinary expansion of every acronym that appears in running text.

## 1. Terms this repository uses precisely

| Term | What it means here |
|:--|:--|
| **Hot path** | Any flow that must complete without the cloud, without Wi-Fi, and without a model: gate admit, entitlement check, the price shown at checkout, keeper data entry. The defining test is not latency, it is **who it is allowed to depend on**. A hot path may be slow; it may not be remote. |
| **Async path** | Everything AI. Runs on a schedule, writes a snapshot or an alert, and is permitted to fail without the estate noticing within the hour. |
| **Advisory** | The capability produces a recommendation a human acts on. It cannot change estate state by itself. Contrast **act-in-band**. |
| **Act-in-band** | The one narrow exception to advisory: a capability may act automatically, but only inside a range a human approved in advance - a price inside an approved band, never a price it chose freely. Defined in [ADR-0005](../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md). |
| **A gap is unknown, never zero** | The single most load-bearing sentence in the repository. When a sensor, zone, or gateway goes silent, the pipeline emits an explicit gap marker and every consumer downstream reports *unknown* for that window. Silence is never averaged, interpolated, or rendered as an empty patch that looks like "nobody is there". |
| **Islanding** | A zone losing its uplink while continuing to operate locally. The **unit of islanding** is the zone, because the estate's characteristic failure is a local radio hole rather than a wide-area outage. |
| **Store-and-forward** | A zone gateway persists events to disk while islanded and replays them on reconnect. Replayed data arrives late and out of order, which is why every aggregate in this architecture must be **restatable**. |
| **Restatable** | An aggregate that can be recomputed for a window already reported, because a buffer may deliver yesterday afternoon tomorrow morning. Reports are provisional until buffers drain. |
| **Snapshot** | A file of last-known-good values - prices, experiment flags, entitlement cache, standard operating procedures - published by the async path and read locally by the edge. The edge never calls the thing that produced it. |
| **Freshness / data age** | How old a snapshot or reading is, displayed to the user rather than hidden. A screen that shows stale data without saying so is treated as a defect, not a degradation. |
| **Capability interface** | An estate-owned, typed contract for one AI capability, with a Vertex adapter behind it, at least one alternative adapter kept compilable, and a named fallback. Metering, budgets, and the kill switch live at this boundary. [ADR-0004](../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md). |
| **Fallback** | What a capability serves when its model is unavailable, drifting, or over budget: rules, last week's average, a published rate card, or a human. Every fallback in this repository is exercised on a schedule rather than merely declared. |
| **Shadow mode** | A model runs on live data and writes its outputs where only the team can see them. It raises no alert and changes no decision. The mandatory first stage for anything safety- or welfare-adjacent. |
| **Promotion** | Moving a capability up the authority ladder: **L0 shadow** (no output), **L1 inform** (visible, not actioned), **L2 advise** (a human acts on it), **L3 act-in-band** (acts automatically within approved limits). Gated by evidence, never by calendar. |
| **Clamp** | A deterministic rule that bounds a model's output before anyone sees it - a price floor and ceiling, for instance. The clamp is not advice; the model physically cannot propose outside it. |
| **Golden case** | A fixed input with a known-correct expected output, stored in `evals/`, used to detect drift after a model or provider changes. Includes **refusal cases**: inputs where the correct answer is "I will not answer this". |
| **Drift** | A model's behaviour changing without its code changing, usually because the world or the provider moved. Detected by re-running golden cases on a schedule. |
| **Fitness function** | An automated, objective test that an architectural characteristic still holds - a measurable threshold with arithmetic behind it, run in CI or on a schedule. See [fitness-functions/](../fitness-functions/README.md). |
| **Driving characteristic** | The small number of architecture characteristics a given decision is actually chosen on. In the options matrices, driving criteria are marked; the rest are tie-breakers. |
| **Swap drill** | A timed, written-up rehearsal of replacing one capability's model provider, run annually. `NFR_14` sets the target at two weeks. |
| **Care subject** | The unit an animal-welfare record attaches to. For most species that is an individual; for a shoaling colony it is the enclosure or the colony, because individuals cannot be identified. [ADR-0020](../adrs/ADR-0020%20-%20Enclosure%20and%20colony%20as%20the%20care%20subject.md). |
| **Census anchor** | A human-counted population figure with a timestamp, against which a model may estimate an interval. As the anchor ages the interval widens; past a maximum age the estimate is withdrawn rather than extrapolated. [ADR-0023](../adrs/ADR-0023%20-%20Anchor%20aquatic%20population%20on%20human%20census.md). |
| **Keeper attention budget** | How many alerts one keeper can meaningfully triage per shift. The constraint that sets every animal-health alerting threshold, derived in [ADR-0022](../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md). |
| **Entitlement** | The right to enter, carried as a signed claim rather than looked up. A gate verifies the signature offline; it does not ask a server for permission. |
| **Party ledger** | The local record of which members of a family pass have already passed through, kept at the gate so a party can be admitted across several presentations without a central lookup. |
| **Sticky assignment** | An experiment assignment that is computed once and does not change, even offline or on a different device, so a guest never sees two different prices for the same thing. [ADR-0011](../adrs/ADR-0011%20-%20Sticky%20offline%20experiment%20assignment.md). |
| **Declared attributes** | Facts a guest told us - ticket type, party shape, postcode district if given. Contrast **inferred demographics**, which this architecture does not compute and which `evals/cohorts` gates at zero. |
| **Backfill** | The flood of buffered events arriving after an island reconnects. Normal operation, not an incident. |
| **Dead letter** | Where an event goes when it fails validation, so it can be replayed after a schema fix instead of being lost. |
| **Zone gateway** | A per-zone device that is simultaneously a local MQTT broker, a disk-backed buffer, a bridge to cloud ingest, and the thing local consumers subscribe to when the estate is cut off. |
| **Deliberate omission** | Something this submission chose not to build, listed with its reason in the [README](../README.md). Distinguishes a considered gap from an oversight. |

## 2. Acronyms and abbreviations

| | |
|:--|:--|
| **AWS** | Amazon Web Services. Evaluated and not chosen in [ADR-0001](../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md). |
| **BFF** | Backend for Frontend. A service that shapes data for one specific UI rather than serving a general-purpose API. |
| **BLE** | Bluetooth Low Energy. |
| **C4** | The C4 model for software architecture diagrams: Context, Container, Component, Code. This repository uses the first two plus sequence diagrams. |
| **CRM** | Customer Relationship Management. Named in `NFR_8` to draw a line: cohort analysis is analytical and must not leak into marketing profiles. |
| **CUPED** | Controlled-experiment Using Pre-Existing Data. A variance-reduction technique that makes an A/B test conclusive on less traffic. |
| **F&B** | Food and beverage. |
| **FTE** | Full-time equivalent. |
| **GCP** | Google Cloud Platform. |
| **GDPR** | General Data Protection Regulation. Treated as the default privacy bar in `NFR_8` because the estate's jurisdiction is undecided. |
| **GPU** | Graphics Processing Unit. |
| **HLD** | High Level Design - the [hld/](../hld/README.md) folder. |
| **HVAC** | Heating, Ventilation and Air Conditioning. |
| **IAM** | Identity and Access Management. |
| **MAPE** | Mean Absolute Percentage Error. The accuracy metric for the flow forecast. |
| **MFA** | Multi-Factor Authentication. Required in `NFR_6` for destructive operations: ride evacuation, mass refund, model promotion. |
| **ML** | Machine Learning. |
| **MLOps** | The practice of getting models into production and keeping them honest: evaluation, promotion, monitoring, rollback. See [hld/mlops/](../hld/mlops/README.md). |
| **MQTT** | Message Queuing Telemetry Transport. The lightweight publish/subscribe protocol the estate's field devices speak. Named in the kata brief as the hardware the estate has budget for. |
| **MSK** | Managed Streaming for Apache Kafka, an AWS service. |
| **MTTD** | Mean Time To Detect. `NFR_11` sets it at ≤5 minutes for ingest and gate failures. |
| **NFC** | Near-Field Communication. The tap-to-pay radio; considered and rejected as the admit mechanism in [ADR-0002](../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md). |
| **NTP** | Network Time Protocol. Clock sync at the gateway, which is safety-adjacent here because skew corrupts both dwell measurement and the audit trail. |
| **p95 / p99** | The 95th and 99th percentile. "Gate redeem ≤2 s p95" means 95 of every 100 redemptions finish inside two seconds. |
| **PCI** | Payment Card Industry, as in PCI DSS, the card-data security standard. `NFR_6` keeps the estate out of scope by tokenising through a provider. |
| **PII** | Personally Identifiable Information. |
| **POS** | Point of Sale. |
| **QoS** | Quality of Service. In MQTT, the delivery guarantee: QoS 0 at most once, QoS 1 at least once, QoS 2 exactly once. Welfare and access events use QoS 1, persisted. |
| **QR** | Quick Response code. The 2D barcode carrying the signed entitlement claim. |
| **RAG** | Retrieval-Augmented Generation. Answering from retrieved documents rather than from model memory; the pattern behind the optional ops copilot. |
| **RFID** | Radio-Frequency Identification. |
| **RPO** | Recovery Point Objective. How much data you can afford to lose, measured in time. `NFR_2` sets ≤15 minutes. |
| **RTO** | Recovery Time Objective. How long you can afford to be down. `NFR_2` sets ≤4 hours for the cloud ops view. |
| **SDK** | Software Development Kit. |
| **SECU** | Streaming Engine Compute Unit, a Dataflow billing unit. Appears only in [cost-analysis](../cost-analysis/README.md). |
| **SKU** | Stock Keeping Unit. A distinct sellable or billable item - a ticket type, or a line on a cloud bill. |
| **SLO** | Service Level Objective. A target the estate holds itself to, as distinct from a contractual SLA. |
| **SOP** | Standard Operating Procedure. The written procedures keepers and ride ops follow, cached offline on handhelds. |
| **SRE** | Site Reliability Engineering. |
| **TLS** | Transport Layer Security. Encryption in transit, required in `NFR_6` "where the radio allows". |
| **UPS** | Uninterruptible Power Supply. |
| **UWB** | Ultra-Wideband. A short-range precise-positioning radio; considered and rejected for gate admit. |
| **VAS** | Value Added Services, Apple's protocol for passes in Wallet. |
| **WAN** | Wide Area Network. The estate's link to the outside world, as distinct from the WLAN inside it. |
| **WCAG** | Web Content Accessibility Guidelines. `NFR_16` requires WCAG 2.1 level AA on web and kiosk. |
| **WLAN** | Wireless Local Area Network. The park Wi-Fi the brief describes as patchy, and the reason for most of this architecture. |

Related: [README](../README.md), [3_NFRs](3_NFRs.md), [diagrams/legend](../diagrams/legend.md) (what the shapes mean, as this page covers what the words mean).
