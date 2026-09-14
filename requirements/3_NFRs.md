# Non-Functional Requirements

## 1. Core System Qualities

**NFR_1. Scalability:** Design for ≥15,000 visitors/day (3× current), not for today’s 5,000. Support peak concurrent on-estate guests in the 5,000–8,000 range, 40 rides, 55 animal displays, and on the order of 500–1,000 MQTT devices. Autoscaling of cloud analytics/AI paths at ~70% capacity. Edge components (gates, kiosks, zone gateways) must be sized for local peak without waiting for cloud scale-out.
**Risk:** Queue and telemetry volume jump faster than visitor count (more devices, more events per guest). Cross-estate backfill after a Wi-Fi outage creates ingest spikes.

**NFR_2. Reliability:** Critical guest path (purchase confirmation already issued, gate redeem, entitlement check, ride-open/closed status used by ops) available even when park Wi-Fi and/or cloud are degraded. Target: gate redeem p99 success when the local gateway is up, independent of cloud. Cloud analytics RPO ≤15 min under normal MQTT flow; RTO for ops intranet cloud view ≤4 hours with last-known-good edge dashboards in the meantime.
**Risk:** Treating the cloud as the gate’s system of record. Split-brain entitlements after prolonged partition.

**NFR_3. Performance:** Gate redeem ≤2 seconds (p95) from local cache. Web/kiosk checkout ≤5 seconds (p95) when online. MQTT telemetry from gateway to cloud ingest ≤30 seconds under healthy links; store-and-forward when not. Animal-health and ride-anomaly alerts to keeper/duty-manager UI ≤60 seconds from event after they have reached a gateway. Pricing and experiment assignment at checkout: read a snapshot, do not call a model.
**Risk:** Model inference on the hot path. Chatty cloud APIs from keeper devices in low-signal zones.

## 2. Connectivity, Edge & Data Movement

**NFR_4. Offline / patchy Wi-Fi:** Gates, kiosks, and keeper apps operate on last-known-good data (prices, experiment flags, entitlements, SOP cache). Field writes are append-only events; the cloud is the reconciler. MQTT QoS and gateway buffers must survive multi-hour islanding without silent drop of health or access events.
**Risk:** Buffer overflow, clock skew, duplicate events, “offline” UIs that look live but are stale (must show freshness).

**NFR_5. Estate-to-cloud path:** Cloud services are allowed. There must be an explicit, retryable path (MQTT gateways / brokers, dead-letter, replay) from estate devices to cloud warehouses and AI jobs. No AI training or Countess-level reporting may assume a device can call the cloud directly.
**Risk:** One poorly placed broker becomes a single point of failure for both popularity and animal telemetry.

## 3. Security, Safety & Compliance

**NFR_6. Security:** TLS in transit where the radio allows; secrets not in MQTT payloads; least-privilege staff roles on the intranet; MFA for destructive ops (ride evacuation command, mass refund, model promotion). Payment via a PCI-scoped provider (tokenization); the estate platform does not store raw card data.
**Risk:** Physical access to MQTT hardware, rogue devices, kiosk tampering.

**NFR_7. Safety integrity:** Ride-open/closed, emergency stop, animal-escape, and evacuation flows are deterministic and human-authored. AI may draft work orders or suggest closures; it may not open a ride or silence a welfare alarm. All safety actions are audited.
**Risk:** “Helpful” copilot or experiment framework accidentally gating a safety message.

**NFR_8. Privacy & occupancy ethics:** Occupancy and popularity prefer counts and opted-in trails over biometric identification. Inferred demographics are analytical, not CRM. Guest location beyond coarse zone counts requires consent. Camera use (e.g. piranha) is purpose-limited and retention-bounded. Honour applicable visitor-privacy law (treat GDPR-class rights as the default bar unless a later ADR sets a jurisdiction).
**Risk:** Cameras sold as “popularity” that become unlawful surveillance. Cohort AI leaking into marketing profiles.

**NFR_9. Data retention (initial bar):** Financial/ticket records: retain per tax rules (plan for 7 years). Occupancy/telemetry: hot 30 days, warm 1 year, cold 3 years. Guest photos / camera clips: shortest of purpose or 90 days unless an incident hold. Keeper and veterinary records: retain for the life of the animal plus a defined legal period. Experiment assignments: retain long enough to evaluate return visits (90 days) plus audit.

## 4. Operational Excellence

**NFR_10. Expandability:** 3× visitor growth without a rewrite of ticketing, ingest, or intranet contracts. Feature flags / experiment snapshots roll out progressively. New MQTT device classes onboard without changing guest checkout.
**Risk:** Popularity pipeline hard-coded to today’s 40×55 layout.

**NFR_11. Observability:** Freshness of each edge cache visible to ops. MQTT gateway lag, drop, and replay metrics. P0 (gate down, animal-escape alarm path, payment outage) acknowledge ≤15 minutes. MTTD ≤5 minutes for ingest/gate failures.
**Risk:** Alert fatigue from flapping Wi-Fi; missing “data is stale” as a first-class signal.

**NFR_12. Cost:** Cloud cost visible by pipeline (ticketing, MQTT ingest, AI inference). Inference budget per check/alert tracked. Offline edge hardware is a capital cost; do not duplicate the same telemetry in an expensive hot store forever. Data tiering hot → warm → cold.
**Risk:** Vision models and unbounded GenAI copilots dominate opex before tickets do.

## 5. AI-Specific Requirements

**NFR_13. Validation & verification:** Every production AI capability has golden cases in `evals/`, a documented metric (e.g. forecast MAPE, health-alert recall, population census error, experiment CUPED/lift), shadow mode before it can act, and a fallback (rules, last week’s average, human). Judges’ bar: we can show the AI is working and detect when it starts misbehaving.
**Risk:** Non-deterministic GenAI with no eval; silent drift after a provider update.

**NFR_14. Uncertainty & portability:** Model and provider are behind an interface. Assume today’s best model/vendor may be worse, more expensive, or gone. Migration target: swap a provider for a given capability in ≤2 weeks without changing ticketing or MQTT contracts. Budget alerts on inference. Pin versions; do not auto-upgrade models on safety-adjacent paths.
**Risk:** Vendor shutdown, price shock, breaking API changes.

**NFR_15. Human-in-the-loop & characteristic fit:** AI additions use the same identity, event backbone, and offline story as the rest of the architecture - not a sidecar product. Confidence scores and evidence on alerts. Animal and ride decisions: human confirm. Pricing: human-approved bands; automation only inside them after shadow.
**Risk:** A separate “AI app” that does not work when Wi-Fi is patchy, violating the existing architectural characteristics.

## 6. Usability Requirements

**NFR_16. Guest surfaces:** Web + gate kiosk at minimum; mobile optional. Family-pass purchase understandable in one flow. Gate redeem is a QR/barcode (or equivalent) that works when the cloud is down. WCAG 2.1 AA on web/kiosk. Copy and prices must not depend on a live model call.
**Risk:** Beautiful online checkout that cannot be honoured at a disconnected gate.

**NFR_17. Staff intranet & keeper app:** Role-specific views (duty manager, ride ops, keepers, commercial, Countess). Keeper app usable outdoors, with offline note/feed entry and sync. Stale-data banners. Large targets for gloved use. Battery drain compatible with a full shift.
**Risk:** Intranet that is only a cloud dashboard; unusable in the animal houses where Wi-Fi dies.
