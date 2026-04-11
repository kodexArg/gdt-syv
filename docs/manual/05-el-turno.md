---
title: "El Turno"
order: 5
section: turn
lang: es
---

# El Turno

## Turnos simultáneos

Ambos jugadores planifican al mismo tiempo en clientes separados, sin conocer las intenciones del oponente. Cada turno representa cuatro horas de operaciones.

## Briefing

El servidor envía a cada jugador su scope: el estado del campo según lo que sus unidades pueden ver. Posiciones, bajas, movimientos detectados. Lo que está fuera del alcance visual no aparece.

## Orders

Fase de planificación local. El jugador selecciona unidades y asigna órdenes (específicas o genéricas a oficiales). Puede cambiar de opinión las veces que quiera. Al confirmar, las órdenes se transmiten por radio a la cadena de mando. El oponente hace lo mismo en su pantalla, simultáneamente.

## Resolution

El servidor ejecuta las órdenes de ambos jugadores y muestra los resultados evento por evento. Las transmisiones que llegaron se ejecutan. Las que no llegaron, no. Las que llegaron tarde de turnos anteriores se ejecutan literalmente sobre un campo que ya cambió.

Después de la Resolution, vuelve el Briefing con el nuevo estado. El ciclo se repite hasta que se cumple una condición de victoria.

```
Briefing → Orders → Resolution → Briefing → ...
```
