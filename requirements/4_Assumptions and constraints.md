# Assumptions and Constraints

## Assumptions

- **Greenfield**: There is no legacy estate software to migrate. Ticketing, intranet, telemetry, and AI are designed from scratch. Paper or spreadsheet practices may exist; they are not systems of record.
- **Assets in scope**: 40 historic amusement rides (recently passed safety inspection) and a previously private exotic/poisonous collection now opening to the public: 200+ animals, 55 displays/enclosures, mixed aquatic and land, including a jumping-piranha colony.
- **Visitation baseline**: ~5,000 visitors/day today; planning horizon is ≥15,000/day within three years.
- **MQTT hardware**: There is budget to install MQTT-capable devices throughout the park (occupancy, ride health, water quality, feeders, and optionally cameras at key exhibits). Device count and exact sensors are an ADR, not a constraint against using MQTT.
- **Cloud is allowed**: Training, A/B analysis, Countess reporting, and most AI jobs may run in the cloud, provided there is a reliable estate→cloud ingest path.
- **Ticketing channels**: Guests buy tickets (including family passes) via web and on-site kiosk at minimum. Identity is **optional** on the first visit; membership / email / QR identity is incentivized for return so the loyalty loop can close.
- **Entitlements ≠ SKUs**: A family pass is a product that grants a set of access rights (party size, validity window, possibly timed talks). Access control consumes entitlements, not raw SKU names.
- **Pricing is async**: Checkout and gates consume published price lists and experiment assignments. They do not call a pricing model inline.
- **Animal identity**: Enclosure- or colony-level tracking is sufficient except where individual IDs already exist; piranha are colony-level with periodic census. Individual tagging elsewhere is a later ADR.
- **Occupancy measurement**: Popularity can be built from gate counts, ride cycles, and MQTT zone counters first. Cameras are purpose-limited (e.g. piranha), not the default people-tracker. Exact sensing tech is an ADR.
- **Safety inspection is not “done”**: Passing inspection after removing asbestos, glass, and gnomes is a starting point. The system must keep producing evidence of inspection, incidents, and uptime.
- **Jurisdiction**: The kata does not name a country. Design as if visitor-privacy and animal-welfare duties are real and auditable (GDPR-class privacy as the default bar). Payment and tax details follow a later jurisdiction ADR.
- **Staffing model**: Human keepers, ride operators, and a duty manager remain in the loop. Software recommends; it does not replace veterinary or safety authority.

## Constraints

- **Patchy Wi-Fi**: Field devices and guest gates cannot assume continuous internet or even continuous estate WLAN. Architecture must be edge-first for access, prices, and keeper data entry.
- **Estate-to-cloud is mandatory if cloud is used**: Using cloud services implies designing gateways, buffers, retries, MQTT QoS, and dead-letter/replay - not “the device will just call the API”.
- **MQTT-capable hardware is the field bus we should assume**, not a proprietary always-on IoT suite with guaranteed bandwidth.
- **Growth bound**: Capacity, data volume, and ops UX must be credible at 15,000 visitors/day, 40 rides, and 55 displays.
- **Hazardous operations**: Poisonous/exotic animals and historic rides constrain experiments. We never A/B-test safety closures, welfare thresholds, evacuation copy, or accessibility of exits.
- **AI must match the existing characteristics**: Offline story, event backbone, and identity of AI features must match ticketing and telemetry - not a separate cloud-only AI product (kata judging criterion).
- **AI results must be verifiable**: Golden cases, metrics, drift detection, and human fallback are constraints on going live, not optional polish (kata judging criterion).
- **Budget reality**: There is MQTT hardware budget; there is not a stated unlimited cloud/AI budget. Inference and vision cost must stay visible and killable.
- **Core access is deterministic**: A paid, unexpired entitlement must admit the guest at the gate without a model vote.
