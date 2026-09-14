# Animal care - data contract

> Note: This is the field-level contract behind [animal-care-scope.md](animal-care-scope.md) section 3. It expands the single `Animal` row in [Appendix A](../requirements/Appendix%20A_%20Core%20functionality.md) section 4 into something an implementer and the ops intranet workstream can both build against.
>
> Granularity is fixed by [ADR-020](../adrs/ADR-020-use-care-subject-as-unit-of-record.md). Capture guarantees are fixed by [ADR-021](../adrs/ADR-021-keep-keeper-observations-off-the-telemetry-path.md). Detection and population semantics are fixed by [ADR-022](../adrs/ADR-022-detect-welfare-anomalies-against-per-subject-baselines.md) and [ADR-023](../adrs/ADR-023-anchor-piranha-population-on-human-census.md). **This document decides no architecture.** Where an ADR left something open, it is marked TBD rather than invented.

Storage design, wire format, and API shape are deliberately absent. This is *what* is recorded and *what it means*, not how it is serialised.

## 1. Why the fields matter

The detector's ceiling is set by the schema, not by the model. You cannot detect "eating less well" if nobody records leftover weight, and you cannot measure recall if the system never records the events it failed to raise. Three fields below exist purely because an ADR would otherwise be unimplementable, and they are the ones most likely to be dropped as noise by someone who has not read the reasoning:

| Field | On | Without it |
|---|---|---|
| `prompted_by_alert_id` | `observation` | Recall has no denominator. The detector can only measure its own opinion of itself (ADR-022). |
| `investigated` | `alert_decision` | A keeper too busy to look teaches the model that busy days are healthy (ADR-022 label bias). |
| `feed_protocol` | `feed_record` | Per-capita intake is unobservable, and the entire between-census population signal collapses (ADR-023). |

## 2. Conventions

| Convention | Rule |
|---|---|
| Identifiers | Opaque strings, unique across the estate. No meaning is encoded in them. |
| Time | UTC, ISO-8601. |
| Field-captured time | Every field event carries **both** `observed_at` (device clock, may skew) and `received_at` (endpoint stamp), plus `device_seq`. Ordering is reliable **within** a device only; cross-device ordering is TBD (ADR-021). |
| Idempotency | Every field event carries a client-generated `event_id`. It is the deduplication key, and it is what makes at-least-once sync safe. |
| Append-only | Nothing is updated or deleted. A correction is a new event carrying `amends_event_id`. The original stays attributable. |
| Freshness | Every **published** state carries `as_of`, `data_age_seconds`, and a `freshness` enum. Stale is never rendered as normal. |
| Gaps | A missing reading is `unknown`, never zero (Appendix B). |

**Enums used throughout**

- `freshness`: `live` | `stale` | `unknown`
- `subject_type`: `individual` | `group` | `colony`
- `baseline_status`: `established` | `warming_up` | `stale`

## 3. Captured records

Everything in this section is append-only. Sections 3.1 to 3.6 arrive over the ADR-021 keeper path; section 3.7 arrives over MQTT telemetry.

### 3.1 `subject`

The unit of record (ADR-020).

| Field | Type | Req | Meaning |
|---|---|---|---|
| `subject_id` | id | yes | |
| `subject_type` | enum | yes | `individual`, `group`, or `colony`. Determines whether a population estimate applies. |
| `collection` | string | yes | Grouping for collection priors during warm-up (ADR-022). |
| `species` | string | no | An attribute, never a structure. There is no taxonomy engine. |
| `arrived_on` | date | no | Drives the warm-up window. |
| `lifecycle_state` | enum | yes | **TBD** - the vocabulary (birth, death, transfer, split, merge) is an open question in ADR-020. |

### 3.2 `placement`

Where a subject currently lives. Separate from the subject, because identity must survive a move (ADR-020).

| Field | Type | Req | Meaning |
|---|---|---|---|
| `placement_id` | id | yes | |
| `subject_id` | id | yes | |
| `enclosure_id` | id | yes | One of the 55 displays. |
| `placed_at` | timestamp | yes | |
| `ended_at` | timestamp | no | Null while current. |
| `reason` | enum | no | **TBD** - routine, quarantine, breeding, treatment. |

> **The join rule.** An observation is associated with environment readings through the placement covering its `observed_at`, not through the subject's *current* enclosure. This is the resolver drawn in orange in the [component view](../diagrams/c3-components-animal-care.md), and getting it wrong produces a wrong answer that looks right. If placement cannot be resolved for a timestamp, the observation carries a gap flag - it is never joined to the last known enclosure.

### 3.3 `observation`

A human record of what was seen. Unrecoverable if lost (ADR-021).

| Field | Type | Req | Meaning |
|---|---|---|---|
| `event_id` | id | yes | Client-generated. Deduplication key. |
| `subject_id` | id | yes | |
| `observation_type` | enum | yes | `health_note`, `behaviour`, `condition`, `welfare_event`. |
| `observed_at` | timestamp | yes | Device clock. |
| `device_seq` | integer | yes | Per-device monotonic sequence. |
| `device_id` | id | yes | |
| `received_at` | timestamp | yes | Set by the endpoint on sync, not by the device. |
| `recorded_by` | id | yes | Staff identity. |
| `text` | string | no | |
| `attachments` | list | no | Each with `attachment_id` and `status` of `pending` or `synced`. Deferred sync (ADR-021). |
| `prompted_by_alert_id` | id | no | **Null means the keeper opened this unprompted.** Unprompted confirmed welfare events are the recall denominator (ADR-022). |
| `amends_event_id` | id | no | Present on a correction. Never overwrites the original. |

### 3.4 `feed_record`

The data behind "how much / how well they are eating".

| Field | Type | Req | Meaning |
|---|---|---|---|
| `event_id` | id | yes | |
| `subject_id` | id | yes | |
| `observed_at`, `device_seq`, `device_id`, `received_at`, `recorded_by` | | yes | As section 3.3. |
| `offered_g` | number | yes | |
| `consumed_g` | number | yes | |
| `leftover_g` | number | yes | |
| `refused` | boolean | yes | A total refusal, distinct from partial consumption. |
| `aggression_noted` | boolean | no | |
| `feed_protocol` | enum | yes | `fixed_offer` or `to_appetite`. **Under `to_appetite`, per-capita intake is unobservable** and the colony change signal is void (ADR-023). |
| `amends_event_id` | id | no | |

### 3.5 `expected_state_declaration`

A human assertion that suppresses scoring, with a name and an expiry on it.

| Field | Type | Req | Meaning |
|---|---|---|---|
| `event_id` | id | yes | |
| `subject_id` | id | yes | |
| `state` | enum | yes | **TBD** - brumation, moulting, gravid, quarantine, under treatment. Vocabulary is an open question in ADR-022. |
| `declared_by` | id | yes | Must be an identified human. Never system-inferred. |
| `declared_at` | timestamp | yes | |
| `expected_until` | date | yes | Suppression does not outlive this without a fresh declaration. |
| `note` | string | no | |

### 3.6 `colony_event`

Facts about a colony that move the population estimate (ADR-023).

| Field | Type | Req | Meaning |
|---|---|---|---|
| `event_id` | id | yes | |
| `subject_id` | id | yes | Must be a `colony`. |
| `event_type` | enum | yes | `census`, `carcass_recovered`, `fry_observed`. |
| `observed_at`, `recorded_by` | | yes | As section 3.3. |
| `counted` | integer | when `census` | The anchor value. |
| `count` | integer | when `carcass_recovered` | Decrements deterministically by exactly this many. |
| `method` | string | no | **TBD** - census method is an open question in ADR-023. |

### 3.7 `environment_reading`

Machine telemetry. Attaches to an **enclosure**, never to a subject - a tank cannot be sick (ADR-020).

| Field | Type | Req | Meaning |
|---|---|---|---|
| `enclosure_id` | id | yes | |
| `parameter` | string | yes | Water temperature, dissolved oxygen, humidity, and so on. |
| `value` | number | yes | |
| `unit` | string | yes | |
| `observed_at` | timestamp | yes | |
| `device_id` | id | yes | |
| `gap_flag` | boolean | yes | True when a gap precedes this reading. Downstream must render unknown, not interpolate. |

## 4. Published contracts

What the ops intranet consumes. It builds the screens; we guarantee the meaning.

### 4.1 `subject_state`

| Field | Type | Meaning |
|---|---|---|
| `subject_id`, `subject_type`, `collection` | | |
| `current_enclosure_id` | id | Resolved from the placement ledger. |
| `as_of` | timestamp | |
| `data_age_seconds` | integer | |
| `freshness` | enum | |
| `baseline_status` | enum | `established`, `warming_up`, or `stale`. |
| `welfare_state` | enum | `ok`, `alerted`, or `unknown`. **Must be `unknown` when `baseline_status` is `stale`** - a subject nobody has observed for weeks is never reported as well. |
| `expected_state` | object | Null when none is declared. |

### 4.2 `welfare_alert`

| Field | Type | Meaning |
|---|---|---|
| `alert_id`, `subject_id` | id | |
| `signal` | string | What was detected. |
| `tier` | integer | 0, 1, or 2. Tier 3 never raises an alert. |
| `confidence` | object | Value plus the reason it is reduced, if it is. Warm-up subjects must say so. |
| `raised_at` | timestamp | |
| `evidence` | list | Field and value pairs. **Every entry must be a field that exists on the record** - no derived narrative claims. |
| `suggested_action` | object | `source` is always `playbook_lookup`, with `playbook_id` and `text`. **Never generated.** |
| `narrative` | string | Nullable prose from tier 3. |
| `narrative_source` | enum | `model` or `template`. Template is the vendor-outage fallback (ADR-022). |
| `rank` | integer | Position in the keeper's budgeted inbox. |
| `suppressed_by` | string | Null unless an expected state suppressed it. Suppression is always audited, never silent. |
| `state` | enum | `open`, `accepted`, `rejected`, `resolved`. |

### 4.3 `alert_decision`

Flows back **from** the intranet. This is the training label.

| Field | Type | Meaning |
|---|---|---|
| `alert_id` | id | |
| `decided_by` | id | |
| `decided_at` | timestamp | |
| `decision` | enum | `accept`, `reject`, `resolve`. |
| `investigated` | boolean | **Whether the keeper actually looked.** An uninvestigated reject is a weak label and is weighted down (ADR-022). |
| `note` | string | |

### 4.4 `population_estimate`

| Field | Type | Meaning |
|---|---|---|
| `subject_id` | id | A colony. |
| `as_of` | timestamp | |
| `status` | enum | `usable` or `unusable`. |
| `point` | integer | **Null when `status` is `unusable`.** No figure is published past the maximum anchor age. |
| `interval_low`, `interval_high` | integer | **Mandatory when usable.** An estimate without an interval is not a valid output. |
| `nominal_coverage` | number | What the interval claims, so coverage can be measured against census. |
| `anchor_census_at` | timestamp | |
| `anchor_count` | integer | |
| `days_since_anchor` | integer | Drives interval width. |
| `trend` | enum | `rising`, `falling`, `stable`, `unknown`. |
| `flags` | list | Breeding or loss flags, each requiring keeper confirmation. |

### 4.5 `containment_signal`

| Field | Type | Meaning |
|---|---|---|
| `signal_id`, `enclosure_id` | id | |
| `subject_id` | id | Nullable. |
| `detected_at` | timestamp | |
| `source` | string | |
| `severity` | enum | |

> **We emit this; the intranet runs the incident and evacuation flow.** That flow is deterministic and human-authored, and no model may raise, suppress, downrank, or delay this signal (NFR_7).

## 5. Invariants

Machine-checkable rules that hold across every contract above. These are the contract, as much as the field lists are.

1. A published `subject_state` with `baseline_status = stale` has `welfare_state = unknown`.
2. A `population_estimate` with `status = usable` has a non-null interval; with `status = unusable` it has a null `point`.
3. Every `evidence` entry on a `welfare_alert` names a field present on the underlying records.
4. `suggested_action.source` is always `playbook_lookup`.
5. A `narrative` never contains a diagnosis, dose, or treatment, and its absence never blocks the alert.
6. Replaying a sync batch produces no new records and no new labels.
7. An `amends_event_id` chain always resolves to an original that remains retrievable.
8. An observation whose placement cannot be resolved carries a gap flag and is not joined to any enclosure.
9. A `carcass_recovered` event decrements the point estimate by exactly `count` and leaves interval width unchanged.
10. Every published payload carries `as_of` and `data_age_seconds`.

## 6. Open questions

Carried from the ADRs. None are invented here.

| Question | Blocking | From |
|---|---|---|
| Subject lifecycle vocabulary | Section 3.1 | ADR-020 |
| Placement `reason` vocabulary | Section 3.2 | ADR-020 |
| Expected-state vocabulary | Section 3.5 | ADR-022 |
| Census method | Section 3.6 | ADR-023 |
| Cross-device event ordering under clock skew | Section 2 | ADR-021 |
| Attachment size cap and sync policy | Section 3.3 | ADR-021 |
| Warm-up window length per collection | Section 4.1 | ADR-022 |
| Alert budget per keeper per shift | Section 4.2 | ADR-022 |
| Maximum anchor age before `unusable` | Section 4.4 | ADR-023 |
| Retention period for keeper and veterinary records | All | NFR_9 leaves it undefined |
