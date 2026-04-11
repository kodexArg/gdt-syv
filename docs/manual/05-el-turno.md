---
title: "El Turno"
order: 5
section: turn
lang: es
---

# El Turno

## Despliegue inicial

Antes del primer turno, ambos jugadores despliegan sus fuerzas de forma simultánea y oculta. Cada jugador no ve dónde coloca unidades el oponente.

**Características del despliegue:**

- **Simultáneo:** Ambos jugadores despliegan al mismo tiempo, sin revelar sus posiciones mutuamente.
- **Oculto:** Coherente con la filosofía WEGO (simulación simultánea) y con la arquitectura fog-of-war del juego. No hay información compartida hasta que la primera Resolution ocurre.
- **Zona de despliegue:** Cada jugador tiene una zona de despliegue definida en el mapa — un área donde puede colocar sus unidades iniciales.
- **Validación:** El servidor recibe ambos despliegues, los valida contra las reglas del escenario, y confirma antes de que comience el primer Briefing.

**Pendiente de definir:**

- Tamaño y forma de las zonas de despliegue (por escenario)
- Reglas de colocación (formaciones, distancia mínima entre unidades, etc.)
- Límite de tiempo para completar el despliegue

## Turnos simultáneos

Ambos jugadores planifican al mismo tiempo en clientes separados, sin conocer las intenciones del oponente. Cada turno representa cuatro horas de operaciones.

## Briefing

El servidor envía a cada jugador su scope: el estado del campo según lo que sus unidades pueden ver. Posiciones, bajas, movimientos detectados. Lo que está fuera del alcance visual no aparece.

## Orders

Fase de planificación local. El jugador selecciona unidades y asigna órdenes (específicas o genéricas a oficiales). Puede cambiar de opinión las veces que quiera. Al confirmar, las órdenes se transmiten por radio a la cadena de mando. El oponente hace lo mismo en su pantalla, simultáneamente.

## Resolution

Cada turno representa **cuatro horas de operaciones**. La Resolution no es una resolución instantánea ni un resumen — es una **simulación de tiempo continuo** que produce una línea de tiempo de eventos discretos con marcas de tiempo precisas dentro de esa ventana de cuatro horas.

El servidor toma las órdenes de ambos jugadores y genera una cronología completa: cada movimiento, cada disparo, cada baja, en el orden exacto en que ocurren dentro del turno. Esta línea de tiempo es la Resolution. El servidor la transmite a los clientes como una secuencia de eventos que se presenta al jugador de forma progresiva.

**Ejemplo de lo que produce el servidor:**

```
14:23 → Artillería XX comienza el ataque sobre hex [4,7]
14:40 → Infantería YY comienza despliegue dirección NE
15:40 → YY tiene visual de enemigo ZZ dirección NW y comienza su ataque
16:05 → Soldado xx de ZZ abatido
16:45 → Soldado yy de ZZ abatido
17:45 → Soldado zz de YY abatido
```

Esta línea de tiempo **no es aleatoria en su orden**: la iniciativa de cada unidad determina cuándo dentro de la ventana de cuatro horas ocurre su acción. Las unidades con mayor iniciativa actúan antes; las más lentas, después. La fórmula exacta de iniciativa: **Pendiente de definir**.

Las transmisiones que llegaron se ejecutan. Las que no llegaron, no. Las que llegaron tarde de turnos anteriores se ejecutan literalmente sobre un campo que ya cambió.

Después de la Resolution, vuelve el Briefing con el nuevo estado. El ciclo se repite hasta que se cumple una condición de victoria.

```
Briefing → Orders → Resolution → Briefing → ...
```
