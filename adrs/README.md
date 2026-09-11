# Architecture decision records

Template: [ADR-000-template.md](ADR-000-template.md) - Kata Log shape (Five Nines, Pragmatic, BluzBrothers, CELUS Ceals).

A requirement is not a decision. Tech choice, MQTT bus, pricing algorithm, and AI placement wait for their own ADR.

| # | Title | Status | Date |
|---|---|---|---|
| [001](ADR-001-use-home-bought-token-pool.md) | Use home-bought token pool, spend at live park prices | Proposed | 2026-09-11 |
| [002](ADR-002-use-signed-static-QR-for-attraction-token-presentation.md) | Use signed static QR for attraction-token presentation | Proposed | 2026-09-11 |

## Plan (which ADR next)

| Order | Topic | Needs | Not this ADR |
|---|---|---|---|
| 1 | Token pool: home-bought pool vs per-attraction tickets | ADR-001 (this log) | Prices, rates, refund policy |
| 2 | Ticketing QR at rides + enclosures | ADR-002; assumes ADR-001 | - |
| 3 | Popularity meter (checkpoint events → usable counts) | ADR-002 `validated` events | Not the pricing model |
| 4 | Dynamic pricing (AI) | Popularity + shop data | Not gate admission |
| 5 | Return incentive: AI Guide + engaging games in the app | Tokens + popularity | Not animal health |

MQTT broker, kiosks, and gateway placement are still assumptions until someone writes those ADRs.
