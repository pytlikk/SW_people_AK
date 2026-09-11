# Architecture decision records

Template: [ADR-000-template.md](ADR-000-template.md) — Kata Log shape (Five Nines, Pragmatic, BluzBrothers, CELUS Ceals).

A requirement is not a decision. Tech choice, MQTT bus, pricing algorithm, and AI placement wait for their own ADR.

| # | Title | Status | Date |
|---|---|---|---|
| [001](ADR-001-use-signed-static-QR-for-attraction-token-presentation.md) | Use signed static QR for attraction-token presentation | Proposed | 2026-09-11 |

## Plan (who writes which ADR next)

| Order | Topic | Owner | Needs | Not this ADR |
|---|---|---|---|---|
| 1 | Ticketing QR at rides + enclosures | pytlikk | PR #2 / ADR-001 | — |
| 2 | Popularity meter (checkpoint events → usable counts) | pytlikk | ADR-001 `validated` events | Not the pricing model |
| 3 | Dynamic pricing (AI) | mroj4n | Popularity + shop data | Not gate admission |
| 4 | Return incentive: AI Guide + engaging games in the app | pytlikk | Tokens + popularity | Not animal health (Crafterro) |

MQTT broker, kiosks, and gateway placement are still assumptions until someone writes those ADRs.
