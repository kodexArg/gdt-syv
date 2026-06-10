# Esquema — `briefing` (messages.md §3.1)

Caso crítico de scope (ADR-005). Emisión individualizada por jugador
(`S→C*`). El servidor calcula la visión; el cliente **no filtra nada**.

```
Briefing {
  turn_number:   int
  own_force:     OwnForce          # Sección propia, íntegra
  known_enemy:   [EnemyContact]    # SOLO lo detectado por scope propio
  map_deltas:    [MapDelta]        # cambios de mapa visibles al jugador
  command_state: CommandState      # cobertura de cadena de mando propia
}
```

## OwnForce — fuerza propia, completa (ADR-009)

```
OwnForce {
  faction:  FactionId
  hq_hex:   HexCoord
  platoons: [Platoon]
}
Platoon  { id, name, hq: Squad, squads: [Squad] }
Squad {
  id:          OwnUnitRef
  name, type, faction, squad_level, special_rules
  hex:         HexCoord
  state:       UnitState
  groups:      [Group]
  members:     [Trooper]
  strength:    int        # computado por el servidor (fórmula ADR-009 pendiente — no en contrato)
  moral:       int         # idem
  has_radio:   bool
  in_command:  bool        # BFS desde HQ, distancia <= 5 por hop (ADR-009)
}
Group   { id, name, leader: Trooper, members: [Trooper] }
Trooper { id, rank, level, faction?, alive, has_radio }
```

Sin recorte: el jugador tiene derecho a todo su propio estado.

## EnemyContact — enemigo detectado (recortado, ADR-005)

```
EnemyContact {
  ref:            OpaqueEnemyRef   # token opaco, NO el id real del enemigo
  hex:            HexCoord         # última posición conocida
  last_seen_turn: int
  fidelity:       Fidelity         # FIRM = visto este turno; STALE = recuerdo; CONTACT = detección parcial
}
```

**Prohibido en `EnemyContact`** (no se serializa): `Escuadra.id` real,
`type` exacto si el scope no lo revela, composición (`groups`/`members`),
`strength` real, `moral`, órdenes, niveles de Tropa. Lo no detectado **no
aparece en la lista** — no existe en el payload.

## MapDelta / CommandState

```
MapDelta {
  hex:    HexCoord
  kind:   enum { TERRAIN, OBJECTIVE }
  owner?: PlayerId | NEUTRAL        # solo si el jugador puede conocerlo
}
CommandState {
  covered_squads:   [OwnUnitRef]   # escuadras propias en cadena de mando
  isolated_squads:  [OwnUnitRef]   # propias fuera de mando (riesgo de desbande L1)
}
```

Invariante de auditoría: serializar `Briefing` para el jugador A nunca debe
poder producir un campo cuyo valor dependa del estado privado del jugador B,
salvo a través de `EnemyContact` ya recortado.
