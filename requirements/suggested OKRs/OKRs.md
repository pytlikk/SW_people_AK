# Suggested OKRs

> Note: Approximate current vs suggested OKRs for planning and for showing the business impact of the architecture. Several “current” values are **unmeasured** — that *is* the kata (no popularity signal, no loyalty loop). Prioritizing these OKRs should help phase the work.

| OKR objective | OKR key result | Current value | Planned value (within 3 years unless noted) |
|--|--|--|--|
| 1. Grow visitation without selling the carnivorous plants | 1.1 Average daily visitors | ~5,000 | ≥15,000 |
| | 1.2 Family-pass share of admission SKUs | Unmeasured | Tracked; mix set by experiment not guesswork |
| | 1.3 Gate redeem success when local gateway is up (cloud optional) | Unmeasured / assumed fragile | ≥99.5% p99 of attempted valid tickets |
| 2. Make the estate profitable | 2.1 Contribution margin per visitor (ticket + known ancillary) | Unmeasured | Trend up quarter-on-quarter; floors/ceilings stop ruinous prices |
| | 2.2 Yield impact of pricing/package experiments (holdout) | None | Positive lift on primary metric without safety/welfare regressions |
| 3. Earn returning visitors | 3.1 Share of visits from guests with a return within 90 days | Unmeasured | ≥30% of visits |
| | 3.2 Identified (membership) share of visitors | ~0 (anonymous tickets) | Enough to close the loop (suggested ≥40% of tickets) |
| | 3.3 Win-back opt-out and complaint rate | n/a | Opt-out easy; complaint rate monitored with a cap |
| 4. Know where to invest and staff | 4.1 Share of rides + displays with daily popularity/dwell | ~0 | ≥90% of 40 rides and 55 displays |
| | 4.2 Flow-forecast error (30–90 min occupancy) | n/a | Beats recency baseline; paused when MQTT gap exceeds threshold |
| | 4.3 Staffing suggestions accepted by duty manager | n/a | Tracked; target majority accepted at peak days after shadow |
| 5. Healthy animals, controlled cost | 5.1 Time-to-detect feeding/health anomaly (keeper-confirmed) | Manual / unknown | Materially faster than paper rounds; recall target in evals |
| | 5.2 Piranha population estimate vs human census | Unmeasured | Error band agreed with keepers; census remains ground truth |
| | 5.3 Sick-animal incident cost | High / unquantified | Trend down; AI never autonomous on welfare |
| 6. Keep historic rides usable and inspectable | 6.1 Inspection-due compliance | Unknown | 100% of rides have next-due and evidence on the intranet |
| | 6.2 Unplanned downtime on top-quartile popular rides | Unmeasured | Reduced vs baseline once instrumented; inspect-before-peak alerts in shadow then live |
| 7. Verifiable AI under uncertainty | 7.1 Production AI capabilities with golden cases + drift alarm | 0 | 100% of live AI capabilities (see FR#2 and FR#3) |
| | 7.2 Provider/model swap drill | Never | ≤2 weeks for a designated capability without changing gate contracts |
| 8. Operate despite patchy Wi-Fi | 8.1 MQTT gateway buffer survival (hours of islanding without dropping health/access events) | n/a | Multi-hour (exact SLO in ops ADR) |
| | 8.2 Ops views showing data age | n/a | 100% of live ops screens |
