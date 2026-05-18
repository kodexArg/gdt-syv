# schema/ — Esquemas de payload

Esquemas legibles y **transport-neutral** (ADR-006) de los payloads
referenciados en [`../messages.md`](../messages.md). No son código Godot:
describen la forma de los datos para que `client/`, `server/` y `shared/`
implementen una serialización consistente.

| Archivo | Cubre |
|---|---|
| [`types.md`](types.md) | Tipos primitivos y enums comunes (HexCoord, enums de fase/estado/error, FactionId) |
| [`briefing.md`](briefing.md) | Payload de `briefing` (§3.1) — el caso crítico de scope (ADR-005) |
| [`orders.md`](orders.md) | Payload de `submit_orders` (§4.1) y validación |
| [`resolution.md`](resolution.md) | Eventos de `resolution_delta` (§5.2) |

Notación: pseudo-esquema con `campo: Tipo` · `?` opcional/nullable ·
`[]` lista · `|` unión. Los IDs de entidad son los de ADR-009.
