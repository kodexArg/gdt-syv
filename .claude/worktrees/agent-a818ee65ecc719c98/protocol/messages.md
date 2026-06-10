# Catálogo de mensajes — SyV

Contrato de mensajes entre client y server. Cada mensaje declara
**Dirección**, **Fase**, **Payload** y **Nota de scope**. El mapeo a RPCs
Godot concretos está en [`rpc.md`](rpc.md); los esquemas de payload en
[`schema/`](schema/).

Leyenda de dirección: `C→S` client→server · `S→C` server→client (broadcast
idéntico) · `S→C*` server→client **individualizado por scope** (ADR-005).

Regla transversal (ADR-005): **ningún mensaje `S→C` / `S→C*` puede contener
información fuera del scope del jugador destinatario.** Donde un mensaje
podría exponer estado enemigo, el servidor lo recorta *antes* de serializar.
La "Nota de scope" de cada mensaje es la cláusula auditable de esa garantía.

---

## 1. Handshake (fase `Handshake`)

Establece la sesión lógica sobre el transporte ya conectado (ADR-006). El
transporte (ENet/Steam) ya resolvió la conexión física; aquí se negocia
versión de protocolo e identidad de jugador.

### 1.1 `hello` — C→S

| Campo | Contenido |
|---|---|
| Dirección | C→S |
| Fase | Handshake |
| Payload | `protocol_version: int`, `client_build: String`, `player_token: String` (credencial opaca; en prototipo puede ser un nick) |
| Nota de scope | N/A — el cliente solo declara identidad e intención de conectar. No transporta estado de juego. |

### 1.2 `welcome` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Handshake |
| Payload | `session_id: String`, `assigned_player_id: int`, `protocol_version: int`, `server_phase: PhaseEnum` (fase actual del ciclo, para reconexión) |
| Nota de scope | Solo identidad asignada al propio jugador. No incluye datos de otros jugadores ni del estado del juego. |

### 1.3 `handshake_rejected` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Handshake |
| Payload | `reason: ErrorCode` (ver §7), `detail: String` |
| Nota de scope | Sin estado de juego. Tras este mensaje el servidor cierra la sesión lógica. |

---

## 2. Lobby / Deploy (fase `Lobby`)

Selección de facción y despliegue inicial de la Sección (ADR-009) antes de
que arranque el ciclo de turno. El despliegue es *intención* del cliente; el
servidor valida composición y posiciones contra las reglas de `shared/`.

### 2.1 `lobby_state` — S→C*

| Campo | Contenido |
|---|---|
| Dirección | S→C* |
| Fase | Lobby |
| Payload | `players: [{player_id, display_name, faction_choice|null, ready: bool}]`, `map_id: String`, `deploy_zones: [HexCoord]` (zona de despliegue **del jugador destinatario únicamente**) |
| Nota de scope | `deploy_zones` contiene solo la zona propia. La lista `players` expone metadatos públicos de lobby (nombre, facción elegida, ready) — esto es información pública del lobby, no estado de juego; no incluye composición ni posiciones de fuerza. |

### 2.2 `select_faction` — C→S

| Campo | Contenido |
|---|---|
| Dirección | C→S |
| Fase | Lobby |
| Payload | `faction: FactionId` (`confederacion` \| `rojos`) |
| Nota de scope | Intención. El servidor valida disponibilidad y responde con `lobby_state`. |

### 2.3 `submit_deployment` — C→S

| Campo | Contenido |
|---|---|
| Dirección | C→S |
| Fase | Lobby |
| Payload | `section_layout`: estructura de despliegue — lista de `{escuadra_id, hex: HexCoord}` para cada Escuadra de la Sección propia (jerarquía Sección/Pelotón/Escuadra según ADR-009; las Tropas/Grupos viajan por referencia a una plantilla de escuadra, no como estado expandido) |
| Nota de scope | Intención sobre la fuerza **propia**. El servidor valida `compatible_squads`, zona de despliegue y composición contra `shared/`. El cliente nunca despliega ni referencia fuerza enemiga. |

### 2.4 `deployment_ack` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Lobby |
| Payload | `accepted: bool`, `violations: [{escuadra_id, code: ErrorCode, detail}]` (vacío si `accepted`) |
| Nota de scope | Solo referido al despliegue propio del jugador. Sin datos del oponente. |

### 2.5 `match_start` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Lobby → Briefing |
| Payload | `match_id: String`, `turn_number: 1`, `factions: {player_id: FactionId}` (qué facción juega cada jugador — información pública) |
| Nota de scope | Solo el mapeo público jugador↔facción. No incluye posiciones ni composición. Transiciona el ciclo a Briefing (ADR-004). |

---

## 3. Briefing (fase `Briefing`) — ADR-004 §1, ADR-005

El servidor calcula el estado y emite a **cada** jugador su scope individual.
Emisión individualizada (`S→C*`): no es un broadcast.

### 3.1 `briefing` — S→C*

| Campo | Contenido |
|---|---|
| Dirección | S→C* |
| Fase | Briefing |
| Payload | `turn_number: int`, `own_force`: estado completo de la Sección propia (Pelotones → Escuadras → Grupos → Tropas, con `hex`, `state`, `alive`, `in_command`, `strength`/`moral` computados por el servidor — ADR-009), `known_enemy`: `[{escuadra_id_opaco, hex, last_seen_turn, fidelity: Enum(FIRM|STALE|CONTACT)}]` solo para Escuadras enemigas **dentro del scope de visión propio**, `map_deltas`: cambios de terreno/objetivos visibles al jugador, `command_state`: cobertura de cadena de mando propia (BFS desde HQ, ADR-009) |
| Nota de scope | **Cláusula central de fog-of-war (ADR-005).** `own_force` íntegro. `known_enemy` contiene únicamente Escuadras enemigas que el scope propio detecta, con identificador **opaco** (no el `Escuadra.id` real del enemigo) y sin composición interna, fuerza real, moral ni órdenes. Lo no detectado **no se serializa**: no existe en el payload. El servidor calcula la visión; el cliente no filtra nada. |

### 3.2 `briefing_ack` — C→S

| Campo | Contenido |
|---|---|
| Dirección | C→S |
| Fase | Briefing |
| Payload | `turn_number: int` (acuse de recepción/aplicación del briefing) |
| Nota de scope | Sin estado. Permite al servidor saber que el cliente está listo para la fase Orders. |

---

## 4. Orders (fase `Orders`) — ADR-004 §2

Fase **enteramente local** al cliente. No hay tráfico de red durante la
planificación (undo/reset son locales). El único mensaje es el que la cierra.

### 4.1 `submit_orders` — C→S

| Campo | Contenido |
|---|---|
| Dirección | C→S |
| Fase | Orders |
| Payload | `turn_number: int`, `orders`: lista de órdenes sobre unidades **propias** — cada orden `{unit_ref: escuadra_id, type: OrderType, params}` donde `OrderType` ∈ {`MOVE`, `ATTACK`, `HOLD`, `REGROUP`, `GENERIC` (orden genérica a L5/HQ), …}; `params` referencia objetivos por `HexCoord` o por identificador opaco de contacto enemigo recibido en `briefing` |
| Nota de scope | Intención pura. El cliente solo ordena sobre su propia fuerza y solo puede referirse a enemigos mediante el identificador opaco que el servidor le entregó (no puede inventar conocimiento que no recibió). El servidor revalida cada orden contra el estado real completo y la cadena de mando (ADR-003, ADR-005). |

### 4.2 `orders_received` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Orders |
| Payload | `turn_number: int`, `accepted: bool`, `rejected_orders: [{unit_ref, code: ErrorCode, detail}]` (órdenes inválidas devueltas para corrección; el resto quedan encoladas) |
| Nota de scope | Solo validación de las órdenes propias. No revela si el oponente ya envió, ni su contenido. El servidor espera a ambos jugadores antes de Resolution (ADR-004). |

### 4.3 `awaiting_opponent` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Orders |
| Payload | `turn_number: int` (señal de que las propias órdenes están aceptadas y el servidor espera al otro jugador) |
| Nota de scope | Solo un flag de sincronización. No expone identidad de órdenes ni progreso del oponente más allá de "aún no confirmó". |

---

## 5. Resolution (fase `Resolution`) — ADR-004 §3, ADR-005

El servidor ejecuta la lógica y emite **eventos secuenciales** (deltas) ya
recortados por scope, para presentación progresiva en el cliente.

### 5.1 `resolution_begin` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Resolution |
| Payload | `turn_number: int`, `event_count: int` (cantidad de deltas que se emitirán, para barra de progreso) |
| Nota de scope | Sin estado de juego; solo metadato de la secuencia. |

### 5.2 `resolution_delta` — S→C*

| Campo | Contenido |
|---|---|
| Dirección | S→C* |
| Fase | Resolution |
| Payload | `seq: int` (orden estricto), `event`: uno de — `MOVE {unit_ref, from_hex, to_hex}`, `CONTACT {opaque_enemy_ref, hex}`, `COMBAT {attacker_ref, defender_ref, outcome}`, `STATE_CHANGE {unit_ref, new_state: UnitState}`, `CASUALTY {unit_ref, tropa_ids}`, `COMMAND_LOST {unit_ref}`, `OBJECTIVE {hex, owner}`. Las fórmulas de combate/iniciativa **no forman parte de este contrato**: el evento transporta el *resultado* (`outcome`) ya computado por el servidor, no la matemática. |
| Nota de scope | **Cada delta se emite por jugador y solo si ese jugador puede percibir el evento según su scope.** Movimientos enemigos fuera de visión no generan delta para ese jugador. Combate enemigo-vs-enemigo no observado no se emite. Unidades enemigas siempre se referencian con identificador opaco; nunca se revela composición interna ni fuerza real del enemigo. La secuencia `seq` puede tener huecos por jugador (eventos filtrados): el cliente ordena por `seq`, no asume contigüidad. |

### 5.3 `resolution_end` — S→C*

| Campo | Contenido |
|---|---|
| Dirección | S→C* |
| Fase | Resolution → Briefing |
| Payload | `turn_number: int`, `next_turn_number: int`, `terminal: bool`, `victory`: `null` \| `{result: Enum(WIN|LOSS|DRAW), reason}` (resultado **desde la perspectiva del jugador destinatario**) |
| Nota de scope | El estado final autoritativo **no se reenvía aquí**: se entrega como el `briefing` del turno siguiente (ADR-004 cierra el ciclo en Briefing, que revalida coherencia). `victory` se individualiza por jugador. Sin datos del oponente. |

---

## 6. Conexión / control (fase `Any`)

### 6.1 `heartbeat` — C→S / S→C

| Campo | Contenido |
|---|---|
| Dirección | C→S y S→C (independientes) |
| Fase | Any |
| Payload | `t: int` (timestamp monotónico del emisor) |
| Nota de scope | Sin estado de juego. Solo liveness; el contenido es transport-neutral (ADR-006). |

### 6.2 `resume_session` — C→S

| Campo | Contenido |
|---|---|
| Dirección | C→S |
| Fase | Any |
| Payload | `session_id: String`, `last_seen_turn: int`, `last_seen_seq: int` |
| Nota de scope | Solo identificadores de sincronización propios. El servidor responde con un `briefing` fresco del turno actual (recomputado por scope) en lugar de re-emitir deltas históricos, evitando fugas por reconstrucción. |

### 6.3 `session_closed` — S→C

| Campo | Contenido |
|---|---|
| Dirección | S→C |
| Fase | Any |
| Payload | `reason: ErrorCode`, `detail: String` |
| Nota de scope | Sin estado de juego. Termina la sesión lógica. |

---

## 7. Errores — `ErrorCode`

Catálogo cerrado. Todo mensaje de rechazo (`handshake_rejected`,
`deployment_ack.violations`, `orders_received.rejected_orders`,
`session_closed`) usa estos códigos. Los mensajes de error **nunca**
incluyen estado de juego ni del oponente (ADR-005).

| Código | Significado |
|---|---|
| `PROTOCOL_VERSION_MISMATCH` | `protocol_version` del cliente incompatible con el servidor |
| `AUTH_FAILED` | `player_token` inválido o rechazado |
| `SESSION_FULL` | La partida ya tiene los jugadores requeridos |
| `SESSION_NOT_FOUND` | `session_id` desconocido en `resume_session` |
| `PHASE_VIOLATION` | Mensaje recibido fuera de la fase válida (p.ej. `submit_orders` en Briefing) |
| `FACTION_UNAVAILABLE` | Facción ya elegida por el otro jugador |
| `DEPLOY_OUT_OF_ZONE` | Escuadra desplegada fuera de la zona propia |
| `DEPLOY_INVALID_COMPOSITION` | Composición viola reglas de `shared/` (`compatible_squads`, tamaños) |
| `ORDER_INVALID_UNIT` | `unit_ref` no pertenece a la fuerza del jugador o no existe |
| `ORDER_OUT_OF_COMMAND` | Unidad fuera de cadena de mando para esa orden (ADR-009) |
| `ORDER_ILLEGAL` | Orden mal formada o destino imposible según reglas |
| `ORDER_UNKNOWN_TARGET` | Referencia a un contacto enemigo no presente en el scope recibido |
| `TIMEOUT` | El jugador no confirmó dentro de la ventana (si una regla de juego la define) |
| `INTERNAL` | Error del servidor; sesión finalizada por seguridad |

---

## 8. Invariantes del contrato (checklist de auditoría ADR-005)

1. Ningún payload `S→C`/`S→C*` contiene `Escuadra.id`, composición, `strength`
   real, `moral`, ni órdenes de fuerza enemiga. Lo enemigo se representa con
   **identificador opaco** y solo si está dentro del scope.
2. La información fuera de scope **no se serializa** (no existe en el paquete);
   no se ofusca ni se encripta (ADR-005 alternativas descartadas).
3. El cliente solo emite intención (`select_faction`, `submit_deployment`,
   `submit_orders`, acuses). Nunca emite estado de juego.
4. El servidor revalida toda intención contra el estado real completo
   (ADR-003). El rechazo nunca filtra por qué a nivel de estado enemigo.
5. La reconexión recomputa scope fresco (`briefing`), nunca re-deriva desde
   deltas históricos.
6. Los nombres de mensaje/RPC son agnósticos al transporte (ADR-006).
