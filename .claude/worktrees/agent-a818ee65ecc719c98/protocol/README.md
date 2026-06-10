# protocol/ — Contrato de comunicación client ↔ server

Este directorio es la **única fuente de verdad** del contrato de red entre el
cliente de presentación y el servidor headless autoritativo de SyV
(ADR-003, ADR-008). No contiene implementación Godot: define qué mensajes
existen, en qué dirección viajan, qué payload llevan y qué garantía de scope
aplica a cada uno.

La decisión que motiva y enmarca este contrato está registrada en
[ADR-010](../docs/adr/010-contrato-de-protocolo-de-red.md).

## Principios rectores

1. **Autoridad del servidor (ADR-003).** El servidor es la única autoridad
   sobre el estado del juego. El cliente nunca envía estado: solo envía
   *intención* (órdenes y acciones de lobby/UI). Todo mensaje client→server
   es una solicitud que el servidor valida contra el estado real completo.

2. **Scope server-side (ADR-005).** Todo mensaje server→client viaja **ya
   recortado por niebla de guerra**. La información fuera del scope del
   jugador no se oculta: no se serializa ni se transmite. Cada mensaje de
   este contrato declara explícitamente su **Nota de scope**.

3. **Ciclo de turno (ADR-004).** Los mensajes se agrupan por fase
   `Briefing → Orders → Resolution`. La fase `Orders` es enteramente local
   al cliente: no hay tráfico de red durante ella salvo el envío final de
   órdenes que la cierra.

4. **Transporte agnóstico (ADR-006).** Los nombres de RPCs, señales y
   mensajes no mencionan el transporte (nada de `enet_*` ni `steam_*`).
   Este contrato es válido tanto sobre ENet (prototipo) como sobre Steam
   Networking Sockets (producción) sin renombrar nada.

5. **Glosario canónico (ADR-001).** Términos como Briefing, Orders,
   Resolution, Scope, Sección, Pelotón, Escuadra, Grupo, Tropa, L1–L5 se
   usan con el significado fijado en ADR-001 / ADR-001-01.

## Archivos

| Archivo | Contenido |
|---------|-----------|
| [`messages.md`](messages.md) | Catálogo de mensajes: dirección, payload, fase, nota de scope, errores |
| [`rpc.md`](rpc.md) | Mapeo de mensajes a RPCs Godot: nombres agnósticos, autoridad, reliability, máquina de estados |
| [`schema/`](schema/) | Esquemas de payload (formato legible, transport-neutral) |

## Convenciones de lectura

- **Dirección**: `C→S` (client a server) o `S→C` (server a client).
  `S→C*` indica emisión **individualizada** (un payload distinto por jugador,
  ver ADR-005); nunca es un broadcast del mismo paquete.
- **Fase**: `Handshake`, `Lobby`, `Briefing`, `Orders`, `Resolution`, o
  `Any` (fuera del ciclo de turno: control de conexión / error).
- **Nota de scope**: garantía explícita sobre qué información puede o no
  puede contener el payload. Es la cláusula que hace auditable ADR-005.
- Los nombres de campos siguen `snake_case`; los IDs de entidad son los
  definidos en ADR-009 (`Tropa.id`, `Escuadra.id`, etc.).
