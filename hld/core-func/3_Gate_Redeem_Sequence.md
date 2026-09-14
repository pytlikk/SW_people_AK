# Sequence - Buy, admit, reconcile

The estate's most critical flow. A family buys a pass at home, arrives in two groups, and gets in while the cloud link is down.

## Purchase - online by definition

```mermaid
sequenceDiagram
    actor G as Guest (web)
    participant T as Ticketing service
    participant S as Edge snapshot store
    participant P as Payment provider
    participant M as Entitlement minter

    G->>T: browse admission options
    T->>S: read published price list + experiment variant
    Note over T,S: snapshot, not a model call (ADR-0010)
    T-->>G: family pass 2+2, price, variant
    G->>T: purchase
    T->>P: tokenized payment
    P-->>T: authorized
    T->>M: issue entitlement (partyShape 2+2, validity, scope estate, variant)
    M->>M: sign claim with estate private key
    M-->>G: signed QR (also emailed, also reprintable at kiosk)
    T->>S: nothing - the gate needs no push
```

Two things to notice. The price the guest sees came from a snapshot, so checkout does not depend on any AI being alive. And the claim is signed **here**, where connectivity is guaranteed because a payment just succeeded - which is why the gate never needs to mint anything.

## Admission - offline, partial party

The estate link is down. This is the normal case, not the exception.

```mermaid
sequenceDiagram
    actor G as Guest (2 adults, 1 child)
    participant L as Gate lane device
    participant LED as Local party ledger
    participant GW as Zone gateway
    participant C as Cloud (unreachable)

    G->>L: show QR
    L->>L: verify signature (public key, local)
    L->>L: check validity window, scope
    L->>LED: how many of party 2+2 already admitted?
    LED-->>L: 0 of 4
    L->>LED: record 3 admitted
    L-->>G: admit 3
    Note over L,C: no call attempted - the path has no network dependency
    L->>GW: redeemed event (QoS 1, persisted)
    GW->>GW: buffer to disk
    GW--xC: bridge retries in background

    Note over G,LED: one hour later, fourth family member arrives
    actor G2 as Fourth guest
    G2->>L: show same QR
    L->>LED: how many admitted?
    LED-->>L: 3 of 4
    L->>LED: record 4 admitted
    L-->>G2: admit 1
    G2->>L: show same QR again
    L->>LED: how many admitted?
    LED-->>L: 4 of 4
    L-->>G2: refuse - party count exhausted
```

The party ledger is the whole point. A family pass that could only be scanned once would be wrong about how families arrive at a park.

## Reconciliation - after the link returns

```mermaid
sequenceDiagram
    participant GW as Zone gateway
    participant PS as Pub/Sub
    participant DF as Dataflow
    participant R as Reconciler
    participant T as Ticketing service
    participant BQ as BigQuery
    participant I as Ops intranet

    GW->>PS: drain buffered redeemed events (oldest first)
    PS->>DF: validate, dedupe on (device id, sequence)
    DF->>R: redemption events
    R->>T: mark entitlements consumed
    R->>R: detect cross-cluster over-admission
    R->>I: reconciliation exception if any
    DF->>BQ: redemption + occupancy facts
    Note over BQ: popularity now counts these scans (FR#2D)
    BQ->>I: updated heat map, with data age
```

Access events are drained oldest-first and are never shed - that classification is [ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md)'s guarantee. Popularity aggregates for the outage window are restated when the backfill lands, which is why "today's numbers" are provisional until buffers drain.

## Failure cases

| Case | Gate behaviour | Why |
|:--|:--|:--|
| Cloud down | Admit normally | No dependency in the path ([ADR-0002](../../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md)) |
| Zone gateway down | Admit normally; lane buffers locally | The ledger is on the lane, not the gateway |
| Lane device replaced mid-day | Party ledger restored from the cluster; if isolated, worst case is re-admitting one party | Bounded by party size |
| Tampered or forged QR | Refuse, distinct reason code | Signature verification is local and unforgeable |
| Expired or wrong-day pass | Refuse, distinct reason code | Validity window in the claim |
| Refunded pass, revocation not yet arrived | **Admits** | Accepted risk: revocation is eventually consistent, loss bounded at one admission |
| Two partitioned clusters, same pass | Both may admit up to the party count | Accepted and bounded; reconciler flags the pattern |
| Dead phone | Kiosk reprint, same artefact | Paper is a first-class presentation |
| Dirty lens or screen glare | Retry, then kiosk reprint | The realistic failure mode; an ops checklist, not a code path |

The refund row is the uncomfortable one, and it is stated rather than hidden. The alternative - checking revocation centrally at the gate - is the thing the whole record refuses.

## What this flow proves

- **Gate redeem never touches the cloud.** The NFR_2 commitment, drawn.
- **A price is read, never computed.** No model on the guest path (NFR_3).
- **Field writes are append-only; the cloud reconciles.** The Appendix A 2-6 conflict rule in action.
- **Popularity is a by-product of admission,** so the "what is popular?" question is answered by infrastructure the estate needed anyway.

Related: [ADR-0002](../../adrs/ADR-0002%20-%20Signed%20QR%20entitlement%20at%20the%20perimeter%20gate.md), [ADR-0003](../../adrs/ADR-0003%20-%20MQTT%20gateways%20as%20the%20estate-to-cloud%20path.md), [data structures](../data-structure/README.md).
