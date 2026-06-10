# RPCs — Mapeo a la MultiplayerAPI de Godot

Este documento mapea los mensajes de [`messages.md`](messages.md) a RPCs
Godot concretos. Es el contrato de **forma de invocación**, no la
implementación (la implementación vive en `client/`, `server/`, `shared/`).

Restricciones aplicadas:

- **ADR-006**: nombres agnósticos al transporte. Ningún nombre menciona
  ENet/Steam. Válido sobre ambos sin renombrar.
- **ADR-003 / ADR-007**: la autoridad es siempre del servidor.
  `multiplayer.is_server()` bifurca; los RPCs C→S se declaran
  `@rpc("any_peer")` y el servidor **revalida el `caller`**; los S→C el
  servidor los invoca con autoridad.
- **ADR-005**: los RPCs S→C que dependen del jugador se invocan
  **dirigidos a un peer específico** (`rpc_id(peer_id, …)`), nunca como
  broadcast, porque el payload es distinto por scope.

## Convención de nombres (ADR-006, agnóstica)

| Verbo | Uso |
|---|---|
| `solicitar_*` | RPC C→S: el cliente pide algo (intención) |
| `notificar_*` | RPC S→C broadcast (payload idéntico para todos) |
| `entregar_*` | RPC S→C dirigido por peer (payload recortado por scope) |

`entregar_*` ⇒ siempre `rpc_id(peer, …)` ⇒ corresponde a los mensajes
`S→C*` de `messages.md`.

## Tabla de RPCs

| Mensaje (messages.md) | RPC | Decorador Godot | Invocación | Reliability |
|---|---|---|---|---|
| `hello` | `solicitar_handshake` | `@rpc("any_peer","reliable")` | client → server | reliable |
| `welcome` | `entregar_bienvenida` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |
| `handshake_rejected` | `entregar_rechazo_handshake` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |
| `lobby_state` | `entregar_estado_lobby` | `@rpc("authority","reliable")` | `rpc_id(peer)` (zona propia) | reliable |
| `select_faction` | `solicitar_seleccion_faccion` | `@rpc("any_peer","reliable")` | client → server | reliable |
| `submit_deployment` | `solicitar_despliegue` | `@rpc("any_peer","reliable")` | client → server | reliable |
| `deployment_ack` | `entregar_ack_despliegue` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |
| `match_start` | `notificar_inicio_partida` | `@rpc("authority","reliable")` | broadcast (datos públicos) | reliable |
| `briefing` | `entregar_briefing` | `@rpc("authority","reliable")` | `rpc_id(peer)` **por scope** | reliable |
| `briefing_ack` | `solicitar_ack_briefing` | `@rpc("any_peer","reliable")` | client → server | reliable |
| `submit_orders` | `solicitar_envio_ordenes` | `@rpc("any_peer","reliable")` | client → server | reliable |
| `orders_received` | `entregar_validacion_ordenes` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |
| `awaiting_opponent` | `entregar_espera_oponente` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |
| `resolution_begin` | `notificar_inicio_resolucion` | `@rpc("authority","reliable")` | broadcast (metadato) | reliable |
| `resolution_delta` | `entregar_delta_resolucion` | `@rpc("authority","reliable")` | `rpc_id(peer)` **por scope** | reliable (ordenado por `seq`) |
| `resolution_end` | `entregar_fin_resolucion` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |
| `heartbeat` | `notificar_latido` | `@rpc("any_peer","unreliable")` | bidireccional | unreliable |
| `resume_session` | `solicitar_reanudar_sesion` | `@rpc("any_peer","reliable")` | client → server | reliable |
| `session_closed` | `entregar_cierre_sesion` | `@rpc("authority","reliable")` | `rpc_id(peer)` | reliable |

Notas:

- Todos los RPCs de juego son `reliable` salvo `notificar_latido`: el modelo
  por turnos (ADR-004) no tolera pérdida de órdenes, briefings ni deltas.
- `entregar_delta_resolucion` se emite en orden de `seq` creciente. La
  reliability del transporte garantiza entrega; el cliente reordena por `seq`
  y **no asume contigüidad** (puede haber huecos por filtrado de scope —
  ver `messages.md` §5.2).
- Los RPCs `solicitar_*` se declaran `any_peer` porque el cliente los inicia,
  pero el servidor **valida `multiplayer.get_remote_sender_id()`** contra el
  `player_id` de la sesión antes de aceptar (ADR-003: el cliente no tiene
  autoridad; declarar el RPC no es autorizarlo).

## Máquina de estados de sesión (servidor)

```
            hello
   [NUEVO] ───────────► [HANDSHAKE]
                            │ welcome / handshake_rejected
                            ▼
                        [LOBBY] ◄── select_faction / submit_deployment
                            │ (ambos jugadores: deployment_ack.accepted)
                            │ match_start
                            ▼
        ┌──────────────► [BRIEFING]  (servidor calcula scope por jugador)
        │                   │ entregar_briefing (rpc_id por peer)
        │                   │ briefing_ack (ambos)
        │                   ▼
        │               [ORDERS]  (sin tráfico salvo cierre)
        │                   │ submit_orders → orders_received
        │                   │ (ambos aceptados) awaiting_opponent
        │                   ▼
        │              [RESOLUTION]
        │                   │ resolution_begin
        │                   │ resolution_delta* (rpc_id por peer, por seq)
        │                   │ resolution_end
        └───────────────────┘  (terminal=false ⇒ vuelve a BRIEFING, ADR-004)
                                (terminal=true  ⇒ [FIN], victory por jugador)

  Cualquier estado ── desconexión ──► [SUSPENDIDA]
  [SUSPENDIDA] ── resume_session ──► recomputa BRIEFING fresco por scope
```

Reglas de la máquina:

1. Un RPC recibido fuera de su fase ⇒ `session_closed` /
   `*_ack` con `ErrorCode.PHASE_VIOLATION`. El servidor no procesa intención
   fuera de fase (ADR-004 es un orden estricto).
2. La transición `RESOLUTION → BRIEFING` siempre revalida coherencia
   (ADR-004 §1): el estado final no se reenvía en `resolution_end`, se
   entrega como el siguiente `briefing` recortado por scope.
3. `resume_session` nunca re-emite deltas históricos: recomputa el scope
   actual y entrega un `briefing` fresco (evita fuga por reconstrucción —
   ADR-005).

## Frontera de fórmulas abiertas

Las fórmulas de combate, iniciativa, `strength` y `moral` (pendientes en
ADR-009) **no son parte de este contrato**. Los RPCs transportan el
*resultado* ya computado por el servidor (`COMBAT.outcome`,
`strength`/`moral` como enteros derivados). Cambiar una fórmula no cambia
ningún RPC ni payload de este contrato: el acoplamiento es con el *tipo* del
resultado, no con su matemática.
