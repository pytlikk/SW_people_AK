# Risks & Mitigation Strategies

> This is a consolidated risk registry for the overall solution.
> Detailed risk trade-offs and mitigation strategies for specific technology decisions can be found in the corresponding ADRs.

| Risk Area | Description | Mitigation |
|-----------|-------------|------------|
| **Patchy connectivity / data loss** | Wi-Fi islands drop MQTT and keeper updates. Popularity, animal health, and even entitlements can go stale or vanish if buffers are too small or QoS is fire-and-forget. | Edge gateways with store-and-forward, MQTT QoS 1+ for health/access events, dead-letter and replay, freshness banners on every ops view, gate redeem from local entitlement cache. |
| **Split-brain access** | Online purchase vs offline gate, or two gateways, admit or refuse inconsistently after a partition. | Entitlements issued as signed, offline-verifiable tokens; cloud reconciles after the fact; append-only event log; explicit conflict rules (admit if token valid and unspent locally). |
| **3× scale shock** | Ingest, heat maps, and gate queues designed for 5,000/day collapse at 15,000/day. | Size NFRs to the 3-year target now. Load-test ingest backfill after outages. Separate hot telemetry store from warehouse. |
| **MQTT / IoT reliability** | Duplicate, delayed, or missing sensor data drives false animal alerts or blank popularity maps. | Idempotent consumers, device identity, time sync, gap detection, degrade to last-good + “degraded” flag rather than interpolating silently. |
| **AI false negatives (animals / rides)** | A sick animal or failing popular ride is missed; cost and harm are high. | High-recall thresholds, keeper/engineer confirm, periodic human census (piranha) and inspections as ground truth, never autonomous welfare or ride-open decisions. |
| **AI false positives / trust** | Noisy alerts or bad itineraries train staff and guests to ignore the system. | Shadow mode, precision tracking, suppress duplicate alerts, show evidence and confidence, budget a human-review SLA. |
| **Unverified AI in production** | GenAI and models drift or a provider silently changes behaviour; judges (and the Countess) cannot tell if it still works. | Golden cases in `evals/` per capability, scheduled eval jobs, drift alarms, pinned model versions on safety-adjacent paths, rollback to rules/last-week average. |
| **Provider / model uncertainty** | Best model today is expensive, worse, or gone tomorrow (explicit kata concern). | Vendor abstraction, two-week swap target, cost alerts, documented fallback model or non-AI path for each capability. |
| **Vendor lock-in (cloud)** | Ingest, identity, and warehouses glued to one cloud AI studio. | Open contracts at MQTT, HTTPS APIs, and event schemas. Keep ticketing/entitlements portable. Reassess portability in ADRs. |
| **Privacy & cameras** | Occupancy or “demographics” becomes unlawful tracking; inferred segments leak into CRM. | Count-based popularity by default; consent for trails; inferred ≠ identity; short camera retention; privacy ADR before vision roll-out. |
| **Experiment ethics** | A/B tests on price or routes accidentally include safety, accessibility, or welfare. | Hard denylist: safety closures, evacuation, welfare thresholds, exit accessibility, legal T&Cs. Experiment service cannot override those flags. |
| **Dynamic pricing harm** | Unguarded model prices lock families out or give away the estate. | Human-approved floors/ceilings, async publish, holdout groups, edge last-known-good, commercial review of lift vs guest-satisfaction. |
| **Cost spiral** | Vision + unconstrained GenAI inference exceed ticket margin. | Per-capability inference budget, prefer ML/code on telemetry, GenAI only where generation is needed (copilot, narrative insights), data tiering. |
| **Safety / heritage incidents** | Historic rides or poisonous animals cause injury; logs cannot show who knew what when. | Audited work orders, inspection due dates, deterministic alarm paths, AI suggestions cannot clear an alarm. |
| **Intranet as the wrong system of record** | Putting tickets or MQTT firehose in the staff app’s database. | Intranet is a BFF + UX on top of ticketing, telemetry, and animal services. Tickets stay in the ticketing store; telemetry in a time-series/event path. |
| **Team / ops overhead** | Edge + MQTT + AI evals are more moving parts than a single cloud app. | Runbooks, SLO dashboards (especially freshness), start with three AI deep-dives rather than a shopping list, human fallback on day one. |
