# Container view - animal care

Key: [00-legend.md](00-legend.md). Scope and ownership: [animal-care-scope.md](../docs/animal-care-scope.md).

This view answers one question: **what still works when the estate loses connectivity?** Thick arrows are the paths that survive an outage. Everything else can wait.

```mermaid
flowchart LR
  keeper(["Keeper"])
  sensors["Enclosure<br/>MQTT devices"]

  subgraph FIELD["Field and estate edge - no network assumed"]
    direction TB
    app["Keeper Field App<br/>ADR-021 capture contract"]
    devlog[("Device-local<br/>append-only log")]
    tier0["Edge Safety Rules<br/>tier 0 thresholds"]
    gw["MQTT broker<br/>and gateways"]
  end

  subgraph CLOUD["Cloud"]
    direction TB
    obsin["Observation Ingest<br/>dedupe by idempotent id"]
    telin["Telemetry Ingest<br/>gap and freshness"]
    acs["Animal Care Service<br/>subjects, placements, alerts"]
    store[("Animal Care Store")]
    subgraph AI["AI capabilities"]
      direction TB
      det["Welfare Detector<br/>FR 2I"]
      pop["Population Estimator<br/>FR 2J"]
      narr["Narrative Service<br/>tier 3"]
    end
    harness["Eval Harness<br/>golden cases, coverage"]
  end

  intranet["Ops Intranet"]
  staff(["Keeper, vet,<br/>duty manager"])
  llm["LLM provider"]

  keeper ==> app
  app ==> devlog
  sensors ==> tier0
  tier0 ==> gw
  sensors -.-> gw
  devlog -.->|"idempotent batches"| obsin
  gw -.-> telin
  obsin --> acs
  telin --> acs
  acs --- store
  acs <-->|"features out, alerts back"| AI
  narr --> llm
  harness -.->|"verifies"| AI
  acs -->|"published contracts"| intranet
  intranet --> staff

  classDef owned fill:#dae8fc,stroke:#4a6fa5,stroke-width:2px,color:#000
  classDef other fill:#f5f5f5,stroke:#888888,stroke-width:2px,stroke-dasharray:6 4,color:#000
  classDef assumed fill:#fff2cc,stroke:#b8860b,stroke-width:2px,stroke-dasharray:2 3,color:#000
  classDef store fill:#e1d5e7,stroke:#9673a6,stroke-width:2px,color:#000
  classDef human fill:#d5e8d4,stroke:#82b366,stroke-width:2px,color:#000
  classDef vendor fill:#f8cecc,stroke:#b85450,stroke-width:2px,color:#000

  class tier0,obsin,telin,acs,det,pop,narr,harness owned
  class app,intranet other
  class gw,sensors assumed
  class devlog,store store
  class keeper,staff human
  class llm vendor
```

## How to read it

**The thick path is the architecture.** A keeper records an observation with no network, it lands durably on the device, and it syncs later. Separately, edge safety rules evaluate hard environment thresholds against sensor readings and raise an alert locally the moment a threshold is crossed. Neither path waits for the cloud. Detection and delivery are different concerns, and only delivery is allowed to be delayed by an outage.

**Everything in the cloud is asynchronous by design.** Nothing in animal care is a hot path in the sense ADR-002 uses for gate access - no guest is held at a barrier waiting for us.

**The vendor sits at the far edge of the picture, reachable only through the Narrative Service.** That placement is the whole of our exposure to a model provider changing price, behaviour, or disappearing. Delete the red box and welfare detection is unaffected; alerts simply lose their prose.

## What this view deliberately does not show

- **Staff screens.** The ops intranet is another workstream's container. We publish contracts to it and build no UI.
- **How the Keeper Field App is built.** The app is the intranet workstream's; ADR-021 only mandates the capture contract it must honour - local durability, idempotent ids, batch sync, deferred attachments.
- **Broker topology and gateway placement.** Drawn as one amber box because no workstream owns that decision. It is a seam, not a design.
- **Ticketing and popularity.** Enclosure checkpoint scans belong to ticketing ([ADR-002](../adrs/ADR-002-use-signed-static-QR-for-attraction-token-presentation.md)) and never touch animal care.
