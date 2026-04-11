---
title: "Unidades"
order: 4
section: units
lang: es
---

# Unidades

En Subordinación y Valor, la fuerza militar de un jugador se organiza en cinco entidades jerárquicas: Sección, Pelotón, Escuadra, Grupo y Tropa. Las tres entidades que el jugador gestiona directamente durante una partida son la Tropa (individuo), el Grupo (agrupación interna) y la Escuadra (unidad de combate).

## Las tres entidades operativas

### Tropa

La unidad atómica. Representa un combatiente individual dentro de una escuadra.

| Atributo | Descripción |
|----------|-------------|
| `rank` | Grado militar (faction-dependiente: "Sargento", "Basto", "Infernal") |
| `level` | Nivel jerárquico L1–L5 — determina autoridad, radio y autonomía |
| `faction` | Facción de origen; `null` si no tiene restricción |
| `compatible_squads` | Tipos de escuadra que esta tropa puede integrar |
| `equipment` | Equipamiento individual (relación 1-a-varios; ver más abajo) |
| `has_radio` | Calculado: `true` si `level >= 3`. Excepción: Tropa de Comunicaciones L1 |
| `alive` | Si la tropa sigue en combate |

#### Compatible_squads

No toda tropa puede integrar cualquier escuadra. El atributo `compatible_squads` define la lista de tipos que esa tropa puede ocupar. Un jinete solo puede integrar escuadras de caballería (`["infernales"]`); un soldado de infantería puede integrar casi cualquier tipo (`["infantry", "mortars", "recon", "engineers", ...]`). Esto impide, por ejemplo, que un jinete sea asignado a una escuadra de morteros o que un artillero sea enviado a reconocimiento.

Al armar la sección antes de la partida, estas restricciones se validan automáticamente.

#### Equipment

Cada tropa puede cargar uno o varios ítems de equipamiento. La relación es 1-a-varios: una misma tropa puede portar fusil, munición extra y un botiquín. La tabla de equipamiento no está completamente definida aún.

<!-- Pendiente: schema completo de Equipment — tipos, efectos en combate, encimamiento de ítems -->

### Grupo

Agrupación organizacional interna de una Escuadra. No ocupa hex propio; forma parte de su escuadra y siempre se mueve con ella.

| Atributo | Descripción |
|----------|-------------|
| `name` | Nombre funcional ("Grupo de Fuego", "Grupo Scout", "Los Artilleros") |
| `leader` | La tropa L2 que lidera este grupo |
| `members` | 2 a 5 tropas bajo ese L2 (sin contar al líder) |

Los Grupos permiten definir mecánicas específicas para sub-conjuntos de la escuadra. Una ametralladora pesada puede requerir un sistema de dos tropas (apuntador + asistente): eso es un Grupo. Un dúo de sniper y oteador tiene reglas de visión especiales: también es un Grupo. Si el líder L2 del grupo cae, el grupo pierde su organización interna aunque los miembros sigan vivos.

Una escuadra puede tener entre cero y varios grupos. Las tropas que no están en ningún grupo dependen directamente del líder de la escuadra.

### Escuadra

La unidad leaf de combate. Ocupa exactamente **un hex** en el tablero. Es la unidad que computa el combate.

| Atributo | Descripción |
|----------|-------------|
| `name` | Nombre de la escuadra |
| `type` | Tipo: `INFANTRY`, `CAVALRY`, `RECON`, `MORTARS`, `ENGINEERS`, `COMMAND`, ... |
| `faction` | Facción |
| `squad_level` | Nivel L del líder: L3 (estándar) o L2 (especial) |
| `special_rules` | `true` cuando `squad_level == L2` — habilita reglas propias del tipo |
| `hex` | Posición en el tablero |
| `state` | Estado operativo (ver más abajo) |
| `groups` | Grupos internos (0 o más) |
| `members` | Tropas directamente en la escuadra, fuera de grupo (incluye al líder) |

Atributos computados:

| Calculado | Fórmula |
|-----------|---------|
| `strength` | Suma del nivel de cada tropa viva <!-- fórmula exacta pendiente --> |
| `moral` | Agregado de moral individual <!-- pendiente --> |
| `has_radio` | `true` si al menos una tropa viva porta radio |
| `in_command` | BFS desde HQ, distancia Manhattan ≤ 5 por hop (ver [Mando y Subordinación](08-mando-y-subordinacion.md)) |

#### Escuadra estándar (L3)

Liderada por un Sargento (L3), que porta radio y actúa como jefe operativo. Puede incluir grupos internos con sub-líderes L2. El mínimo es 4 miembros; el máximo es aproximadamente 15. Esta es la escuadra base de infantería y la más común en ambas facciones.

#### Escuadra especial (L2)

Liderada por un L2 (Cabo Primero, Jefe de Partida, según facción). **No porta radio.** Opera con órdenes genéricas asignadas por su oficial de pelotón; no recibe órdenes directas del HQ turno a turno. Tiene `special_rules = true`, lo que habilita mecánicas propias según su tipo.

Ejemplos de escuadras especiales: Los Infernales (caballería), commandos, zapadores, recon avanzado. Ver ejemplos detallados en [04b — Ejemplos de Escuadras](04b-ejemplos-de-escuadras.md).

## Niveles L1–L5

El sistema de niveles define la autoridad, el radio y la autonomía de cada tropa. No es un "tipo" fijo sino una propiedad que determina qué puede hacer y qué le pasa cuando queda sin mando.

| Nivel | Descripción | Radio | Autonomía |
|-------|-------------|-------|-----------|
| **L5** | Teniente — mando de pelotón | Sí (5 hex) | Alta — recibe órdenes genéricas del HQ |
| **L4** | Sargento Primero — subjefe de pelotón | Sí (5 hex) | Alta |
| **L3** | Sargento — jefe de escuadra estándar | Sí (5 hex) | Media — lidera la escuadra operativamente |
| **L2** | Cabo / sub-líder de grupo o líder de escuadra especial | No | Limitada (ver nota) |
| **L1** | Soldado — tropa básica | No | Ninguna — se desbanda sin mando superior |

**Nota sobre L2:** El nivel L2 tiene dos usos distintos. Dentro de una escuadra estándar, el L2 es el sub-líder de un Grupo: organiza 2–5 tropas internamente pero no porta radio y depende del L3. Como líder de una escuadra especial, el L2 manda a toda la escuadra: opera sin radio, recibe órdenes genéricas y puede activar reglas especiales de su tipo.

**Tropa de Comunicaciones (L1 especial):** Porta una radio de alcance global, pero esta radio solo amplifica el rango de su oficial L3/L4/L5 asignado — no extiende la cadena de mando por sí sola, y no actúa con autonomía.

<!-- Pendiente: alcance exacto de la amplificación de radio de la Tropa de Comunicaciones -->

## Fuerza de la escuadra

La fuerza de una escuadra se calcula sobre sus tropas vivas. Cuantas más bajas recibe una escuadra, menos fuerza tiene en el siguiente combate. Una escuadra que arranca con 14 miembros y pierde 4 sigue operando, pero con capacidad reducida.

```
strength = sum(tropa.level para tropa.alive en la escuadra)
```

<!-- Pendiente: fórmula exacta de strength — ¿level directo o ponderado? -->

Esto también significa que perder al líder L3 o a los L2 tiene más impacto que perder soldados L1: no solo por la cadena de mando, sino porque los niveles altos contribuyen más a la fuerza total.

## Estados de la escuadra

| Estado | Descripción |
|--------|-------------|
| **ACTIVE** | Operativa. Ejecuta órdenes normalmente. |
| **ROUTED** | <!-- Pendiente: qué causa este estado, qué puede hacer la escuadra --> |
| **RETREAT** | <!-- Pendiente: qué causa este estado, qué puede hacer la escuadra --> |
| **ELIMINATED** | Removida del juego. |

Transición general: `ACTIVE → ROUTED → RETREAT → ELIMINATED`

<!-- Pendiente: qué causa cada transición, si son reversibles, relación con tiradas de Valor -->

## Despliegue

<!-- Pendiente: cómo se colocan las escuadras al inicio —
     zona de despliegue, restricciones, simultaneidad -->
