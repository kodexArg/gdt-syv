# Esquema — `submit_orders` (messages.md §4.1) y validación

Intención pura del cliente (ADR-003). Cierra la fase Orders (ADR-004 §2),
que por lo demás es enteramente local.

```
SubmitOrders {
  turn_number: int
  orders:      [Order]
}

Order {
  unit_ref: OwnUnitRef        # SIEMPRE una Escuadra propia; nunca enemiga
  type:     OrderType         # MOVE | ATTACK | HOLD | REGROUP | GENERIC
  params:   OrderParams
}

OrderParams {            # campos según `type`; los no usados se omiten
  path?:        [HexCoord]        # MOVE / REGROUP
  target_hex?:  HexCoord          # ATTACK contra posición
  target_ref?:  OpaqueEnemyRef    # ATTACK contra contacto recibido en briefing
  posture?:     enum { HOLD_FIRE, RETURN_FIRE, FREE }   # HOLD
}
```

## Reglas de validación (server-side, ADR-003 / ADR-005)

El servidor revalida cada `Order` contra el **estado real completo**. El
cliente no tiene autoridad: declarar la orden no la autoriza.

1. `unit_ref` debe pertenecer a la Sección del jugador → si no:
   `ORDER_INVALID_UNIT`.
2. `target_ref` debe ser un `OpaqueEnemyRef` **presente en el `briefing`
   del turno** entregado a ese jugador → si no: `ORDER_UNKNOWN_TARGET`.
   (El cliente no puede inventar conocimiento que no recibió — ADR-005.)
3. La unidad debe estar en cadena de mando para órdenes no triviales
   (ADR-009, BFS desde HQ) → si no: `ORDER_OUT_OF_COMMAND`.
4. `path`/`target_hex` deben ser alcanzables/legales según reglas de
   `shared/` → si no: `ORDER_ILLEGAL`.
5. Fase distinta de Orders → `PHASE_VIOLATION` (no se procesa nada).

Las órdenes inválidas se devuelven en `orders_received.rejected_orders`
para corrección; las válidas quedan encoladas. El rechazo nunca explica
nada del estado enemigo: solo el código y el `unit_ref` propio.

## Frontera con fórmulas abiertas

El servidor decide *si* una orden es válida (pertenencia, mando,
alcanzabilidad). El *resultado* de ejecutarla (combate/iniciativa) se
resuelve en Resolution con fórmulas pendientes de ADR-009 — fuera de este
contrato. `submit_orders` no transporta ni asume ninguna fórmula.
