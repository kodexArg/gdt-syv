# Esquema — `resolution_delta` (messages.md §5.2)

Eventos secuenciales recortados por scope (ADR-004 §3, ADR-005). Emisión
individualizada (`S→C*`): cada jugador recibe solo los eventos que su scope
le permite percibir.

```
ResolutionDelta {
  seq:   int            # orden estricto global; por jugador puede haber HUECOS
  event: Event          # uno de los tipos abajo
}
```

El cliente ordena por `seq` y **no asume contigüidad**: los huecos son
eventos filtrados por scope para ese jugador, no errores de transporte.

## Tipos de evento

```
MOVE          { unit_ref, from_hex: HexCoord, to_hex: HexCoord }
CONTACT       { opaque_enemy_ref: OpaqueEnemyRef, hex: HexCoord }
COMBAT        { attacker_ref, defender_ref, outcome: CombatOutcome }
STATE_CHANGE  { unit_ref, new_state: UnitState }
CASUALTY      { unit_ref, tropa_ids: [String] }     # tropa_ids solo si la unidad es propia
COMMAND_LOST  { unit_ref }                            # unidad propia salió de mando
OBJECTIVE     { hex: HexCoord, owner: PlayerId | NEUTRAL }
```

`unit_ref` es `OwnUnitRef` si la unidad es del destinatario, o
`OpaqueEnemyRef` si es enemiga visible. `CASUALTY.tropa_ids` solo se incluye
cuando `unit_ref` es propia; para bajas enemigas visibles se omite el
detalle de Tropas (ADR-005).

## CombatOutcome — resultado, no fórmula

```
CombatOutcome {
  result:        enum { ATTACKER_PREVAILS, DEFENDER_HOLDS, MUTUAL_LOSS, INDECISIVE }
  attacker_state: UnitState
  defender_state: UnitState
}
```

`CombatOutcome` transporta el **resultado ya computado por el servidor**.
La matemática de combate/iniciativa (pendiente en ADR-009) **no forma parte
de este contrato**: cambiarla no altera este esquema. El acoplamiento es con
el *tipo* del resultado, nunca con su cálculo.

## Reglas de scope por evento (ADR-005)

| Evento | Se emite a un jugador solo si… |
|---|---|
| `MOVE` | la unidad es propia, o enemiga y el origen/destino está en su visión |
| `CONTACT` | el jugador detecta al enemigo este turno (genera/actualiza `OpaqueEnemyRef`) |
| `COMBAT` | el jugador participa, o ambos beligerantes están en su visión |
| `STATE_CHANGE` | unidad propia, o enemiga visible |
| `CASUALTY` | unidad propia (con `tropa_ids`), o enemiga visible (sin detalle) |
| `COMMAND_LOST` | la unidad es propia (información de cadena de mando privada) |
| `OBJECTIVE` | el cambio de objetivo es observable por el jugador |

Combate enemigo-vs-enemigo no observado: **no genera delta** para ese
jugador (no se serializa). El estado final autoritativo llega como el
`briefing` del turno siguiente, no aquí (ADR-004 cierra el ciclo en
Briefing con revalidación de coherencia).
