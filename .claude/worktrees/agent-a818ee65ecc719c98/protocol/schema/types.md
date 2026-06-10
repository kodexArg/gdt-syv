# Tipos comunes

Transport-neutral (ADR-006). Vocabulario alineado a ADR-001 / ADR-009.

## Primitivos

```
HexCoord      { q: int, r: int }          # coordenada axial de la grilla hexagonal
PlayerId      int                          # asignado en `welcome`
FactionId     enum { confederacion, rojos }   # IDs canónicos, ADR-001
SessionId     String                       # opaco, generado por el servidor
```

## Enums de ciclo y estado

```
PhaseEnum     enum { HANDSHAKE, LOBBY, BRIEFING, ORDERS, RESOLUTION, FIN, SUSPENDIDA }
UnitState     enum { ACTIVE, ROUTED, RETREAT, ELIMINATED }   # = ADR-009 Escuadra.state
OrderType     enum { MOVE, ATTACK, HOLD, REGROUP, GENERIC }   # ampliable; GENERIC = orden a L5/HQ
Fidelity      enum { FIRM, STALE, CONTACT }                   # calidad de un contacto enemigo
VictoryResult enum { WIN, LOSS, DRAW }                        # siempre perspectiva del destinatario
```

## Referencias de unidad

```
OwnUnitRef    String          # = Escuadra.id real de la fuerza PROPIA (ADR-009)
OpaqueEnemyRef String          # token opaco y estable-por-partida que el servidor
                               # asigna a una Escuadra enemiga detectada.
                               # NO es el Escuadra.id real del enemigo (ADR-005).
                               # El cliente solo puede referir enemigos con este token.
```

Regla de scope (ADR-005): cualquier campo que pueda apuntar a fuerza enemiga
usa `OpaqueEnemyRef`. El `Escuadra.id` real enemigo, su composición
(Grupos/Tropas), `strength` real, `moral` y órdenes **nunca** cruzan a un
cliente que no sea su dueño.

## Error

```
ErrorCode     enum  # catálogo cerrado en messages.md §7
Violation     { ref: String, code: ErrorCode, detail: String }
```
