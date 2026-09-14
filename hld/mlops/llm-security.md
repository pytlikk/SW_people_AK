# LLM security

The estate has [seven AI capabilities](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md#the-canonical-inventory). Six are classical models trained in BigQuery ML on estate-owned tables, and only **two paths** send text to a language model: the prose layer on top of cohort analysis, and the optional ops copilot. This page assesses those two against the [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/).

The headline, because it shapes everything below:

> **Six of the ten OWASP risks are already reduced to near-nothing here, and not one of them was reduced on purpose.** They fall out of decisions taken for other reasons - advisory-only authority, no embeddings, batch scoring, prompts in git. Getting security for free is pleasant; not noticing you got it, and later trading it away for a feature, is how it gets lost. So each one is written down with the decision that bought it.

## The two paths

| Path | Shape | Who can reach it | What it can do |
|:--|:--|:--|:--|
| **Cohort narrative** ([ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)) | Generative, **scheduled batch**, ~30 reports/month. Input is a BigQuery aggregate row set. | Nobody, at request time. It is a cron job. | Write a paragraph into a report. Display only. |
| **Ops copilot** ([ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)) | Grounded retrieval over estate SOPs plus live state, with citations. Optional; capped at $30/month. | Authenticated staff on the intranet. | Answer a question on a screen. No tools, no writes. |

**The cohort narrative has no interactive surface at all.** There is no box a human types into, which removes direct prompt injection as a category rather than mitigating it. What remains is indirect injection through the data, and that is a real vector worth taking seriously.

## The ten risks

| # | Risk | Applies here | Why, and what holds it |
|:--|:--|:--|:--|
| **LLM01** | Prompt Injection | **Yes - the main one** | Direct injection needs a prompt box; only the copilot has one, behind staff authentication. The realistic attack is **indirect**: a keeper types something into a free-text observation, or a device is named something hostile, and that string is later retrieved into the copilot's context. Controls below. |
| **LLM02** | Sensitive Information Disclosure | **Yes - copilot** | The copilot searches documents and live state; a keeper, a duty manager and the Countess are not entitled to the same things. Retrieval runs **as the asking user**, not as a service account. The narrative sees aggregates, never rows, and a segment below the minimum cohort size is suppressed before the prompt is built. |
| **LLM03** | Supply Chain | **Partially** | One third-party foundation model; six models trained by us on our own data. [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) already pins versions and forbids auto-upgrade for safety-adjacent capabilities. The model version is recorded on every output. |
| **LLM04** | Data and Model Poisoning | **Yes - and more for the classical models than the LLM** | Nothing is fine-tuned and nothing is embedded, so the LLM path has no poisonable store. The six BigQuery ML models train on estate telemetry, which arrives from **535 devices in fields and animal houses that a determined person can walk up to** - the physical-access risk `NFR_6` already names. Device identity, gap markers, restatable aggregates and the eval suite as a tripwire. |
| **LLM05** | Improper Output Handling | **Yes - both** | Model output is rendered as **text**, never as HTML, never as a link the browser will follow, never as SQL, never as a filename, never as a cache key. There is no path from a model's characters to an executed instruction. |
| **LLM06** | Excessive Agency | **Structurally closed** | Neither path has a tool, a function call, or a write scope. [ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md) caps both at L1/L2: inform and advise. The only capability in the estate that acts without a human is price clamping, which is deterministic code and not a model. |
| **LLM07** | System Prompt Leakage | **Low** | Prompts live in this repository, versioned alongside their eval sets ([uncertainty](uncertainty.md)). They contain no credentials, no keys, and no business rule that is not also published. **A leaked prompt costs us nothing because it is already public to anyone with repository access** - which is the only durable defence against this risk. |
| **LLM08** | Vector and Embedding Weaknesses | **Does not apply** | There is no vector store and there are no embeddings anywhere in the estate. Retrieval is lexical over a small SOP corpus. The same fact answers the re-embedding question in [uncertainty](uncertainty.md#what-might-happen-if-the-provider-you-used-suddenly-shut-down); here it removes embedding inversion, cross-tenant leakage through a shared index, and poisoned-chunk retrieval in one go. |
| **LLM09** | Misinformation | **Yes - the largest real risk on this estate** | [Appendix B](../../requirements/Appendix%20B_%20AI%20scenarios%20explained.md) names it exactly: *"invented vet doses are a failure, not a cleverness."* A confident wrong answer about a feeding SOP is worth more harm than any injection attack in this table, because it needs no attacker. Controls below. |
| **LLM10** | Unbounded Consumption | **Yes - copilot only** | The copilot is the one capability whose cost scales with **staff curiosity rather than estate size** - the reason [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md) questioned funding it at all. Per-user rate limit, $10 alert, $30 hard cap, degrading to lexical search rather than switching off ([cost-analysis §7](../../cost-analysis/README.md#7-per-capability-inference-budgets)). The narrative is a batch job of known size and cannot run away. |

### Where the six came from

| Risk reduced | The decision that did it | Taken because |
|:--|:--|:--|
| LLM06 Excessive Agency | Advisory-only authority levels ([ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md)) | A model must not be able to silence a welfare alarm |
| LLM08 Vector and Embedding | Classical models on estate data; lexical retrieval ([ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md)) | Portability, and a small corpus did not need embeddings |
| LLM10 Unbounded Consumption | Batch scoring, no always-on endpoint ([ADR-0001](../../adrs/ADR-0001%20-%20GCP%20as%20the%20estate%20cloud%20platform.md)) | Cost, and nothing is on a guest's critical path |
| LLM07 System Prompt Leakage | Prompts versioned in the repository | Reproducibility and provider swaps |
| LLM05 Improper Output Handling | Output is advice on a screen, never an instruction | Human-in-the-loop |
| LLM02 Sensitive Disclosure (narrative) | Declared attributes only, aggregates only ([ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md)) | `NFR_8`, and the risk that cohort AI leaks into marketing profiles |

**Every row is a security benefit that arrived as a side effect.** That makes them fragile in a specific way: the day someone gives the copilot a tool so it can open a work order, LLM06 comes back, and the person doing it will be thinking about convenience, not about this table. Hence the CI assertions at the bottom.

## Guardrails: cohort narrative

The threat model is short because the attack surface is. There is no user, no session, and no request.

1. **Column allow-list into the prompt.** The narrative receives a fixed set of aggregate columns - counts, rates, deltas, and cohort labels drawn from a closed vocabulary. **No guest-authored string is in that set.** A party name, a promo code, a feedback comment and a free-text field cannot reach the prompt, because the prompt is built from a schema rather than from a row.
2. **Minimum cohort size, applied before the prompt exists.** Suppression is a SQL predicate on the aggregate, not an instruction to the model. The number is the open question ADR-0012 holds for a privacy reviewer; the placement is not open.
3. **No identifiers, ever.** Not hashed, not tokenised, not pseudonymous. The narrative's inputs cannot be re-joined to a person because they never contained a person.
4. **Output is display-only and labelled generated.** It reaches a report the Countess and marketing read. `NFR_8` names the failure mode - *cohort AI leaking into marketing profiles* - so there is no write path from this capability to a guest record, and CI asserts it.
5. **Numbers in the narrative are not the model's.** Figures are computed in SQL and substituted; the model writes prose around them. A number that appears in a narrative and not in the underlying table is an eval failure ([evals/cohorts](../../evals/cohorts/README.md)).

Point 5 is the load-bearing one. **The most likely harm from this capability is not a leaked cohort - it is a plausible sentence containing a wrong figure, read by the Countess, and acted on.** Keeping arithmetic out of the model removes that whole class.

## Guardrails: ops copilot

Here there is a user, and there is retrieved content the estate does not fully control.

**Untrusted content in the corpus.** SOPs are estate-authored and reviewed. Live state is not: enclosure and device names, alert text, incident notes, and above all **keeper free-text observations**, which [ADR-0021](../../adrs/ADR-0021%20-%20Keeper%20field%20events%20are%20append-only%20and%20offline-first.md) makes append-only and which arrive from a handheld in a field. That is the injection vector, and it is a realistic one.

1. **Retrieved content is fenced and labelled as data.** It arrives in the context delimited and marked untrusted, with the system prompt stating that retrieved text is evidence to quote, never instruction to follow. This is a mitigation, not a guarantee - which is why it is first in the list and last in importance.
2. **Retrieval is scoped to the asking user's role.** The copilot cannot retrieve what the user could not open directly. An injection therefore cannot widen access; at most it can distort an answer over documents the reader was already entitled to.
3. **No tools. No writes. No outbound network.** A successful injection produces **a wrong paragraph on a screen**, and there is no second step. This ceiling is what makes the residual risk acceptable, and it is the property that CI protects.
4. **Citations are mandatory.** An answer with no retrieved source is not rendered; the copilot says it does not know and offers the search results. Groundedness and citation correctness are evaluated, per [Appendix B](../../requirements/Appendix%20B_%20AI%20scenarios%20explained.md).
5. **Forbidden-advice classes are refused deterministically, on the way in and on the way out.** Veterinary dosing and treatment, ride safety clearance, evacuation and emergency procedure, and anything legal or HR. A pre-filter on the question and a post-filter on the answer, both **ordinary code** - the model is not asked to decide whether it should refuse, because a model that can be talked into answering can be talked into refusing to refuse. These route to the named human instead: the vet, the ride engineer, the duty manager.
6. **Live state is quoted with its freshness.** The copilot answers *"zone D is amber as of 14 minutes ago"*, never *"zone D is amber"*. It inherits the estate-wide rule that an unknown is published as unknown, so it says **"I cannot see zone D"** rather than reporting the last value it happened to find.
7. **Rate limit per user, budget cap per month.** LLM10, and also the honest observation that the most likely cause of a runaway bill here is an enthusiastic staff member rather than an attacker.
8. **Every query and answer is logged with its citations and model version.** Retained for audit. A staff member who acted on a copilot answer can show what it said.

## Residual risk

| Risk | Why it remains | Why it is accepted |
|:--|:--|:--|
| Indirect injection via a keeper note distorts a copilot answer | Fencing untrusted content is best-effort against a determined attacker | The attacker must already be staff with handheld access, and the payoff is one wrong paragraph with no second step |
| A grounded answer is still wrong | Retrieval can surface the right document and the model can still misread it | Citations let the reader check in one click; forbidden classes never reach the model at all |
| Telemetry poisoning degrades a classical model | Devices are physically reachable | Shadow mode and evals catch a drifting model before promotion ([ADR-0022](../../adrs/ADR-0022%20-%20Shadow%20before%20promote%20for%20animal%20health%20anomaly%20detection.md)); no capability acts autonomously |
| A future feature reintroduces a closed risk | Six of the ten are closed by side effect | The CI assertions below, and this page as the thing a reviewer points at |

## Asserted in CI

- No capability interface declares a tool, a function-call schema, or a write scope (LLM06).
- No model output reaches an HTML renderer, a SQL string, a shell, or a file path (LLM05).
- The cohort narrative's input schema contains no free-text column and no identifier (LLM01, LLM02).
- No path exists from cohort output to a guest record (`NFR_8`).
- Copilot retrieval carries the caller's identity; no service-account retrieval path compiles (LLM02).
- Forbidden-advice filters are deterministic code, not prompt instructions (LLM09).
- Every prompt in the repository has an eval set beside it (LLM03, LLM09).
- No embedding model or vector store appears in any dependency manifest (LLM08). **If this assertion ever fails, this page is out of date** and the embedding risks need real answers.

Related: [MLOps](README.md), [uncertainty](uncertainty.md), [evals](../../evals/README.md), [ADR-0004](../../adrs/ADR-0004%20-%20Vertex%20AI%20behind%20a%20capability%20interface.md), [ADR-0005](../../adrs/ADR-0005%20-%20Human-in-the-loop%20authority%20for%20estate%20AI.md), [ADR-0012](../../adrs/ADR-0012%20-%20Cohort%20analysis%20on%20declared%20attributes%20only.md), [3_NFRs](../../requirements/3_NFRs.md).
