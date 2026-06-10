# ADR-010: Contrato de protocolo de red client ↔ server

## Estado

Aceptado — 2026-05-18

## Contexto

ADR-008 define la carpeta `protocol/` como única fuente de verdad del
contrato de comunicación entre el cliente de presentación y el servidor
headless autoritativo. Hasta ahora esa carpeta estaba vacía: no existía una
especificación de qué mensajes se intercambian, en qué dirección, con qué
payload ni con qué garantía de scope.

Sin ese contrato formal, cada lado (client/server) puede asumir formas de
mensaje distintas, y — más grave — no hay una cláusula auditable que
garantice que la seguridad por scope (ADR-005) se respeta mensaje por
mensaje. El ciclo de turno (ADR-004) y el modelo de fuerza (ADR-009)
necesitan un protocolo concreto sobre el cual implementarse.

Este ADR no define la implementación Godot (eso vive en `client/`,
`server/`, `shared/`), ni las fórmulas de combate/iniciativa/strength/moral
(pendientes en ADR-009).

## Decisión

Se establece el contrato de protocolo de red en `protocol/`, compuesto por:

- `protocol/README.md` — principios rectores y convenciones de lectura.
- `protocol/messages.md` — catálogo de mensajes (handshake, lobby/deploy,
  briefing, submit-orders, resolution-delta, conexión/control, errores),
  cada uno con **dirección**, **fase**, **payload** y **nota de scope**.
- `protocol/rpc.md` — mapeo de cada mensaje a un RPC Godot con nombre
  agnóstico al transporte, decorador, modo de invocación y reliability, más
  la máquina de estados de sesión del servidor.
- `protocol/schema/` — esquemas de payload transport-neutral, con foco en
  el caso crítico de scope (`briefing`, `resolution_delta`).

Decisiones de diseño que fija este contrato:

1. **El cliente solo envía intención, nunca estado** (ADR-003). Los
   mensajes C→S son `select_faction`, `submit_deployment`, `submit_orders`
   y acuses. El servidor revalida toda intención contra el estado real
   completo; declarar un RPC no es autorizarlo.

2. **Todo mensaje S→C viaja ya recortado por scope** (ADR-005). Los
   mensajes dependientes del jugador (`briefing`, `resolution_delta`,
   `resolution_end`) son emisión **individualizada por peer** (`S→C*`,
   `rpc_id(peer,…)`), no broadcast. La información fuera de scope **no se
   serializa**: no existe en el paquete.

3. **La fuerza enemiga se referencia con un identificador opaco**
   (`OpaqueEnemyRef`), nunca con el `Escuadra.id` real ni con composición,
   strength real, moral u órdenes (ADR-005, ADR-009). El cliente solo puede
   referirse a enemigos mediante el token que el servidor le entregó.

4. **Los mensajes se estructuran por las fases del ciclo de turno**
   (ADR-004). La fase Orders no genera tráfico salvo el `submit_orders` que
   la cierra. El cierre del ciclo `Resolution → Briefing` revalida
   coherencia: el estado final se entrega como el siguiente Briefing
   recortado, no se reenvía en `resolution_end`.

5. **Nomenclatura agnóstica al transporte** (ADR-006): ningún nombre de
   mensaje, RPC o señal menciona ENet o Steam. El contrato es válido sobre
   ambos sin renombrar.

6. **Desacople de fórmulas abiertas**: los eventos transportan el
   *resultado* ya computado por el servidor (`CombatOutcome`, `strength`
   como entero derivado), no la matemática. Cambiar una fórmula de ADR-009
   no altera ningún mensaje ni payload de este contrato.

La reconexión (`resume_session`) recomputa un Briefing fresco por scope en
lugar de re-emitir deltas históricos, evitando fugas por reconstrucción.

## Consecuencias

**Positivas:**

- `protocol/` deja de estar vacía: existe un contrato único que client y
  server implementan sin divergir (ADR-008).
- ADR-005 se vuelve auditable: cada mensaje declara su nota de scope y
  `messages.md` §8 lista invariantes verificables.
- La separación intención/estado hace imposible por construcción que el
  cliente inyecte estado o referencie enemigos no detectados.
- Migrar ENet → Steam (ADR-006) no requiere tocar este contrato.
- Las fórmulas pendientes de ADR-009 pueden evolucionar sin romper el
  protocolo.

**Negativas:**

- El servidor debe serializar un payload distinto por jugador en Briefing y
  Resolution (costo de cómputo de scope, ya anticipado en ADR-005).
- Mantener el contrato sincronizado con cualquier nueva mecánica requiere
  disciplina: toda información nueva expuesta debe pasar por la nota de
  scope antes de añadirse a un mensaje.
- En listen server (ADR-007), el contrato se respeta a nivel de código pero
  la garantía de scope no es física (memoria compartida) — limitación ya
  documentada en ADR-007, no introducida aquí.

## Alternativas descartadas

- **Dejar el contrato implícito en el código de `client/` y `server/`**:
  contradice ADR-008 (contrato en un solo lugar) y hace imposible auditar
  ADR-005 sin leer la implementación completa de ambos lados.
- **Enviar estado completo y recortar en el cliente**: ya descartado en
  ADR-005. El contrato lo refuerza: el cliente no tiene mensajes capaces de
  recibir estado fuera de scope.
- **Acoplar el protocolo a las fórmulas de combate/iniciativa**: ataría el
  contrato a decisiones aún abiertas de ADR-009. Se transporta el resultado
  tipado, no la matemática.
- **RPCs con nombres por transporte (`enet_*`)**: viola ADR-006 y forzaría
  renombrados masivos al migrar a Steam.
