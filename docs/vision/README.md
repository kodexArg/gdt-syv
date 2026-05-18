# Vision — comprension holistica del proyecto

> Material de referencia y onboarding. Sintetiza el plan documentado (ADRs +
> manual + premade-squads) en piezas explicativas. **No es spec normativa**:
> ante cualquier discrepancia mandan los ADRs (`../adr/`) y el manual (`../manual/`).
> Fecha de generacion: 2026-05-18.

## Documentos

| Doc | Que responde |
|-----|--------------|
| [VISION-GENERAL.md](VISION-GENERAL.md) | La historia/mundo, anatomia de un turno (Briefing→Orders→Resolution) y el arbol de decisiones del jugador. |
| [ESTRUCTURA-EJERCITO.md](ESTRUCTURA-EJERCITO.md) | Que es una unidad, jerarquia Tropa→Grupo→Escuadra→Peloton→Seccion, armado de fuerza y composicion de la fuerza de combate (diagramas). |

## Bloqueantes de implementacion cerrados en esta iteracion

Esta exploracion tambien cerro los tres bloqueantes que impedian empezar a codear.
Las decisiones son normativas y viven en `../adr/`:

| ADR | Bloqueante resuelto |
|-----|---------------------|
| [010 — Contrato de protocolo de red](../adr/010-contrato-de-protocolo-de-red.md) | `protocol/` ya no esta vacio: mensajes, RPCs y schemas con scope server-side. |
| [011 — Resolucion de combate por soldado](../adr/011-resolucion-de-combate-por-soldado.md) | Formula estocastica de combate (cap. 07 ya no "Pendiente"). |
| [012 — Calculo de iniciativa](../adr/012-calculo-de-iniciativa-orden-de-resolucion.md) | Orden de resolucion de la fase Resolution. |

El contrato de red detallado vive en [`../../protocol/`](../../protocol/).
