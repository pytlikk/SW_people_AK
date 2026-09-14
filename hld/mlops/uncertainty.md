# Dealing with uncertainty in AI

The kata briefing poses three questions directly. This page answers them in order, under their own headings, with the arithmetic and the named alternatives rather than a posture.

The answers are short because the architecture did the work in advance. Stated once, up front:

> **Six of the seven AI capabilities are classical models trained and scored in BigQuery over the estate's own data. One is a prompt. None is fine-tuned. There is no vector store and there are no embeddings in production.**
>
> A model provider is therefore not a dependency this estate has much of. That was decided in [ADR-0001](../../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md), [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) and [ADR-0013](../../adrs/ADR-0013%20-%20Popularity%20and%20flow%20as%20advisory%20signals%20that%20degrade%20to%20unknown.md) on other grounds - offline-first operation, cost, and privacy. The provider-independence is the dividend.

## "The best models or providers today might not be the best tomorrow"

**Every capability sits behind an estate-owned interface**, not a vendor SDK ([ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)). The contract is typed and ours; the Vertex adapter is an implementation detail behind it; a second adapter is kept compilable for every capability. `NFR_14` sets the migration target at **two weeks to swap a provider for one capability without touching ticketing or MQTT contracts**.

Can a model be swapped without changing application code? Yes, and it is checked rather than assumed:

| Control | How it works |
|:--|:--|
| **Swap drill** | Annually, one real capability is migrated to its alternative adapter, timed, and written up. The flow forecast is the designated subject: it is ours, it is the least safety-adjacent, and its fallback is well understood. A drill that exceeds two weeks is a `ADR-0004` revisit trigger - the abstraction is not doing its job and gets fixed before another capability is added. |
| **Version pinning** | Model versions are pinned. Nothing auto-upgrades on a safety-adjacent path (`NFR_14`). A provider shipping a better model is a change we schedule, not a change that happens to us overnight. |
| **Prompts are code** | The one prompt in production lives in this repository as a versioned file. Changing it is a pull request, and the pull request must pass the eval suite in [evals/cohorts](../../evals/cohorts/README.md) before it merges. A prompt change is a release. |
| **Golden cases detect the silent case** | The failure this question is really about is not a provider disappearing, it is a provider quietly changing behaviour underneath a pinned name. The four monitors in [mlops](README.md) re-run golden cases on a schedule for exactly that, and a drift alarm drops the capability to its fallback without a human in the loop. |

The part of this question that most submissions miss is that **"better" is not a property of the model alone.** A model that scores higher on a public benchmark but refuses fewer bad inputs is worse here, because [evals](../../evals/README.md) gates several capabilities on refusal behaviour - zero inferred attributes in cohorts, zero prices outside the clamp. The eval suite is what defines "better" for this estate, and it is ours.

## "How would you handle your model provider changing prices on you?"

There is a full cost model in [cost-analysis](../../cost-analysis/README.md). The answer in one line:

> **$14.47 of a $72.40 monthly cloud bill is metered by a model provider. Doubling every AI price adds $14.47, taking the bill to $86.87 - a 20% rise on a bill that is 0.00067% of ticket revenue.**

| Scenario | New monthly total | Change |
|:--|--:|--:|
| Baseline | $72.40 | |
| **Provider doubles every price** | **$86.87** | **+20%** |
| Provider raises prices 10x | $202.63 | +180% |
| For comparison: the ops copilot used 100x the modelled rate | $333.40 | +360% |
| For comparison: keeping a 24/7 streaming pipeline | $387.39 | +438% |

**A provider doubling its prices is the fifth most expensive thing that could happen to this budget.** That is the honest answer, and it is more useful than a mitigation plan would be.

### The cost controls, named

| Control | Status here |
|:--|:--|
| **Batching** | Default, not an optimisation. Six of seven capabilities are batch-scored on a schedule; the seventh, the cohort narrative, uses batch inference pricing at half the standard rate because nobody is waiting on a monthly report. |
| **Caching** | Context caching on the SOP corpus for the ops copilot: $0.03/M against $0.30/M on the cached portion, taking it from $5.04 to $2.61/month. |
| **Token budgets** | Seven per-capability budgets with alert thresholds at ~3x modelled and hard caps at ~10x ([cost-analysis §7](../../cost-analysis/README.md#7-per-capability-inference-budgets)). |
| **Rate limits** | The copilot is the only capability whose cost scales with human curiosity rather than estate size, so it carries a hard cap that drops it to non-generative retrieval. |
| **Spend alerts** | Five labelled pipelines with alert and ceiling thresholds ([cost-analysis §8](../../cost-analysis/README.md#8-per-pipeline-budget-ceilings)). |
| **Cost per business transaction** | Cost per visitor, $0.00016 at 15,000/day, tracked as an operational metric under `NFR_12` and as a [fitness function](../../fitness-functions/README.md). |
| **Tiered model routing** | **Deliberately not built.** [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) forbids a routing layer "until a second provider actually exists", because a router with one route is a liability with a config file. The trigger that makes it worth building is a second provider genuinely in production for the same capability. |

The kill switch is the control that matters most and it is the least glamorous: **every capability has a documented fallback that the estate already runs and already tests.** Hitting a budget cap degrades the capability - to the published rate card, to last week's average, to the rules layer, to the statutory inspection schedule - rather than taking a consumer down. For ride maintenance and animal health, the fallback is the regime the estate is legally obliged to operate anyway.

## "What might happen if the provider you used suddenly shut down?"

### The named alternatives

`ADR-0004` previously said only that "at least one alternative adapter is kept compilable". Naming them:

| Failure | Alternative | What changes | What degrades |
|:--|:--|:--|:--|
| **Gemini 2.5 Flash deprecated or repriced** | **Anthropic Claude, via Vertex AI Model Garden** | An adapter and a re-run of the eval suite. Same project, same region, same IAM, same billing. | Nothing structural. The cohort narrative's tone changes and its golden cases are re-baselined. |
| **Google exits generative AI entirely** | **Mistral Small, self-hosted on Cloud Run with an NVIDIA L4 GPU** | A container and a scale-to-zero service. | Cost rises from $2.66 to roughly $40/month at $0.672/GPU-hour, which is bounded and knowable. Quality drops; the eval suite says by how much. |
| **Google Cloud shuts down** | Any managed warehouse plus any scikit-learn runtime | A migration, measured in weeks, of ingest and warehouse - the lock-in surface [ADR-0001](../../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md) accepted explicitly. | **The six classical capabilities do not change at all.** ARIMA, logistic regression, k-means and gradient-boosted trees are textbook algorithms. |

That last row is the real answer. **The model is not the asset. The labelled estate data is.** A forecast of zone occupancy is not a capability Google owns; it is a capability the estate owns, expressed as a hundred lines of SQL over its own counters.

### Where the assets live

| Asset | Where it lives | Vendor-held? |
|:--|:--|:--|
| Prompts | This repository, versioned, PR-gated | No |
| Evaluation sets and golden cases | [evals/](../../evals/README.md), in this repository | No |
| Training data | BigQuery, in the estate's own project | No |
| Trained model artefacts | BigQuery ML models in the estate's dataset | No |
| Fine-tuning data | **None exists.** No capability is fine-tuned. | N/A |
| Embeddings and vector indexes | **None exist.** See below. | N/A |
| Feature definitions | SQL in this repository | No |

### The re-embedding trap, and why it does not apply

A provider change usually hurts most in a place that is easy to miss: **embeddings are coupled to the model that produced them.** Swap the embedding model and every vector in the store is meaningless, so a provider migration silently becomes a re-embed-everything migration, priced per token across the whole corpus and repeated for every index.

**This architecture does not have that problem, and it is worth being precise about why rather than claiming it as a virtue.**

- Six capabilities are classical models over structured warehouse tables. There is nothing to embed.
- The cohort narrative is a single prompt over SQL result tables. It performs no retrieval, so there is no index.
- The optional ops copilot is the only place retrieval appears at all, over a SOP corpus of a few hundred documents.

And for a corpus that small, **retrieval does not need embeddings.** The copilot is specified to use lexical search over the SOP corpus in BigQuery rather than a vector index, which means a provider change re-generates nothing. That is not cleverness; it is corpus size. A few hundred procedures is a scale at which keyword search is both cheaper and more auditable - and auditability matters more than recall when the answer is a safety procedure a keeper is about to follow.

**The honest limit:** if the SOP corpus grew by two orders of magnitude, lexical retrieval would stop being sufficient and the trap would become real. Even then it would be small here - re-embedding a 10,000-document corpus is roughly 20 million tokens, which at generation input rates as a deliberate over-estimate is a few dollars, once. The trap is expensive at web scale. This is an estate with 40 rides.

**Revisit trigger:** the SOP corpus passing a few thousand documents, or a capability appearing that genuinely needs semantic similarity. At that point a vector store is a real decision and gets its own ADR, including the re-embedding cost of ever changing the embedding model.

### The degraded mode

Every AI provider on earth going dark at once is the easiest of these scenarios to answer, because the architecture already runs that way whenever the Wi-Fi dies:

- Gates admit, because they verify signed claims locally ([ADR-0002](../../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)).
- Checkout shows a price, because it reads a snapshot rather than calling a model ([ADR-0010](../../adrs/ADR-0010%20-%20Publish%20prices%20asynchronously%20inside%20approved%20bands.md)).
- Keepers record observations, because the handheld writes to a local broker ([ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md)).
- Welfare alarms fire, because they are deterministic rules at the gateway, not a model ([ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)).
- The heat map reports **unknown** rather than empty, which is the contract in [ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md).

What is lost is advice: the congestion forecast, the price proposals, the anomaly ranking, the narrative. The estate runs the way it ran before any of it existed, and every screen says so.

**The availability arithmetic behind that claim** - why the hot path carries no provider availability term at all, because it makes no call - is worked through in [fitness-functions](../../fitness-functions/README.md#2-availability).

Related: [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) (the interface these answers rest on), [cost-analysis](../../cost-analysis/README.md), [mlops](README.md) (promotion and the four monitors), [evals](../../evals/README.md), [3_NFRs](../../requirements/3_NFRs.md) (`NFR_14`).
