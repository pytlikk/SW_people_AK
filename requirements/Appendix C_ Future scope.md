# Extra Ideas 

## Deferred AI / product ideas

- Full computer-vision people tracking across all 55 exhibits (privacy and cost currently argue against it; piranha is the exception).
- Individual animal tagging and veterinary imaging models for every species.
- Fully automatic ride return-to-service (AI will not clear safety holds in v1).
- In-app gamification of the estate (badges, scavenger hunts) beyond simple itineraries.
- Voice copilot for keepers in gloves-on, noisy houses.
- Carbon / conservation storytelling as a guest-facing product.
- Dynamic F&B menu and staffing (only if POS data is later integrated).
- Geo-economic “where to add a new ride” simulation beyond the ranked investment list.
- Multi-language guest app as a default (kiosk/web can start with one language + simple pictograms).

## Future enhancements for core functionality

- Zone-level access control (venomous house as a timed entitlement), once gates and Wi-Fi islands are proven at the perimeter.
- Membership-first identity (today: optional after first visit).
- Ancillary spend attached to the same party id (F&B, photos, tours) for true yield-per-visitor.
- Partner / group sales and school bookings as first-class SKUs.
- Public real-time wait times on the web (only when freshness SLOs are met).
- Deeper offline keeper protocols (multi-day islanding of an animal house).
- Accessibility routing as a first-class itinerary constraint (v1 only forbids unsafe shortcuts).

## Proposed deep-dives vs mention-only

To avoid a shopping list , the architecture narrative should deep-dive:

1. **Async dynamic pricing + A/B + AI cohort analysis** (growth and profit).
2. **Estate popularity / flow from MQTT + staffing** (the “where to invest & deploy” challenge).
3. **Animal health + piranha population** (welfare and cost).

Ticketing, intranet, and maintenance are the **platform** those three sit on — specified in [Appendix A](Appendix%20A_%20Core%20functionality.md), not deferred.

## Implementation sketch 

| Phase | Emphasis |
|:--|:--|
| 1 | Ticketing + offline gates, MQTT ingest, intranet heat map, animal/feed logs, experiment snapshots with manual prices |
| 2 | Shadow AI: flow forecast, health anomalies, piranha estimate; A/B on family-pass shapes |
| 3 | Cohort analysis, win-back, staffing recommendations, predictive maintenance on instrumented rides |
| Later | Items in this appendix |

**Traceability:** related to FR#1–FR#3 and NFR_13–NFR_15. ADRs and `evals/` golden cases belong with the three deep-dives first.
