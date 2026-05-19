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

#### Atributos y estadísticas (Stats)

**Plantilla de clase:** Todos los soldados de la misma clase comienzan con estadísticas base idénticas heredadas de una plantilla de clase. Por ejemplo, todos los Soldados de Infantería L1 arrancan con los mismos valores base de Ataque, Defensa, Movimiento, etc.

**Sistema de modificadores:** Sobre esa base, cada soldado puede llevar modificadores abiertos que alteran sus stats: Perks (bonificadores), Traits (características innatas), Phobias (penalizadores), y **Heridas** (penalizadores permanentes adquiridos en combate). Cada soldado tiene una relación 1-a-varios con estos modificadores. De este modo, los soldados arrancan iguales por clase pero pueden divergir según los modificadores asignados.

Las **Heridas** son un tipo especial de modificador: no se asignan al inicio, sino que se adquieren durante el combate como resultado del daño. Cada herida es un penalizador negativo que persiste para toda la partida. Un soldado es eliminado cuando la acumulación total de penalizadores de heridas cruza un umbral: **Pendiente de definir**. Para más detalle sobre el sistema de heridas, ver [Combate — Heridas y degradación del soldado](07-combate.md#heridas-y-degradación-del-soldado).

Nombres de stats específicos: **Pendiente de definir**.
Catálogo de Perks, Traits, Phobias: **Pendiente de definir**.

#### Asignación aleatoria e invisibilidad de modificadores

Este es uno de los elementos de diseño más distintivos de SyV: **el jugador no sabe qué Perks, Traits ni Phobias tienen sus soldados al comenzar la partida.**

Al crear una escuadra durante la composición de fuerzas, cada tropa recibe modificadores asignados aleatoriamente desde el pool posible de su plantilla de clase. La asignación ocurre en el servidor en el momento de la creación y nunca se comunica directamente al cliente. El jugador ve los nombres y niveles de sus soldados, su equipamiento, y sus stats base de clase — pero los modificadores permanecen ocultos.

**Mecánica de descubrimiento:** Los modificadores se revelan durante la partida cuando se vuelven evidentes por el comportamiento del soldado.

- Un soldado con un Perk de puntería excepcional solo se revela cuando conecta un disparo improbable.
- Un soldado con una Fobia a la artillería solo se revela cuando cae un proyectil cerca y reacciona de forma adversa.
- Un soldado con un Trait de resistencia excepcional se revela cuando sobrevive un impacto que hubiera eliminado a otro.

Cuando un modificador se activa por primera vez, el cliente recibe una notificación del evento y el modificador queda visible en la ficha del soldado desde ese momento. Antes de su primera activación, no aparece.

Esta mecánica refuerza la filosofía de niebla de guerra del juego: **el jugador no conoce completamente ni siquiera sus propias fuerzas.** Comandar bien significa aprender sobre tus tropas comandándolas, no leyendo sus fichas antes del primer turno.

<!-- Pendiente: qué campos exactos se revelan al cliente en el evento de activación; formato del mensaje de descubrimiento -->

#### Mutación de modificadores durante la partida

Los modificadores no son completamente estáticos. Un soldado puede ganar o perder un Perk, Trait o Phobia como consecuencia de un evento significativo durante el combate.

- Un soldado que sobrevive un combate especialmente brutal podría adquirir un nuevo Trait.
- Un soldado que presenció algo traumático podría desarrollar una Phobia que no tenía.
- Un soldado que supera repetidamente una situación que antes lo paralizaba podría perder esa Phobia.
- Un soldado que acumula demasiados fracasos podría perder la confianza reflejada en un Perk previo.

**Esta mutación ocurre con frecuencia muy baja.** La mayoría de los soldados conservan sus modificadores iniciales durante toda la partida. No es un sistema de progresión por uso — es un reflejo de eventos extraordinarios que transforman a un individuo.

Eventos disparadores específicos y probabilidades: **Pendiente de definir**.

#### Progresión entre partidas

*Futuro — no prioridad en prototipo.*

En modo campaña, los soldados sobrevivientes podrían acumular experiencia entre partidas: ganar nuevos Perks, perder Phobias superadas, o modificar sus stats base según su historial de combate. La arquitectura de modificadores está diseñada para soportarlo — agregar un modificador de campaña es una operación del mismo tipo que asignar uno al inicio de partida.

Esta funcionalidad no se implementa en el prototipo. El sistema de modificadores debe diseñarse con este uso futuro en mente, pero no debe construirse ni exponerse en la interfaz hasta que el modo campaña sea una prioridad.

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
| `in_command` | BFS desde HQ, distancia hex ≤ 5 por hop (ver [Mando y Subordinación](08-mando-y-subordinacion.md)) |

#### Escuadra estándar (L3)

Liderada por un Sargento (L3), que porta radio y actúa como jefe operativo. Puede incluir grupos internos con sub-líderes L2. El mínimo es 4 miembros; el máximo es aproximadamente 15. Esta es la escuadra base de infantería y la más común en ambas facciones.

#### Escuadra especial (L2)

Liderada por un L2 (Cabo Primero, Jefe de Partida, según facción). **No porta radio.** Opera con órdenes genéricas asignadas por su oficial de pelotón; no recibe órdenes directas del HQ turno a turno. Tiene `special_rules = true`, lo que habilita mecánicas propias según su tipo.

Ejemplos de escuadras especiales: Los Infernales (caballería), commandos, zapadores, recon avanzado. Ver ejemplos detallados en [04b — Ejemplos de Escuadras](04b-ejemplos-de-escuadras.md).

## Niveles L1–L5

El sistema de niveles define la **autoridad, el radio y la autonomía** de cada tropa. **No determina la potencia de combate.** Un L1 es tan capaz de infligir daño como cualquier otro soldado; la diferencia es que no puede tomar decisiones autónomas en ausencia de mando superior.

| Nivel | Descripción | Radio | Autonomía | Poder de combate |
|-------|-------------|-------|-----------|------------------|
| **L5** | Teniente — mando de pelotón | Sí (5 hex) | Alta — recibe órdenes genéricas del HQ | Depende de sus stats, equipo y modificadores |
| **L4** | Sargento Primero — subjefe de pelotón | Sí (5 hex) | Alta | Depende de sus stats, equipo y modificadores |
| **L3** | Sargento — jefe de escuadra estándar | Sí (5 hex) | Media — lidera la escuadra operativamente | Depende de sus stats, equipo y modificadores |
| **L2** | Cabo / sub-líder de grupo o líder de escuadra especial | No | Limitada (ver nota) | Depende de sus stats, equipo y modificadores |
| **L1** | Soldado — tropa básica | No | Ninguna — se desbanda sin mando superior | Depende de sus stats, equipo y modificadores |

**Nota importante:** El nivel L1–L5 define autoridad y radio, no capacidad de combate. Un L1 puede tener excelentes stats y modificadores, mientras que un L5 podría tener stats bajos o penalizadores.

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

Los estados están **impulsados por el Valor ponderado de la escuadra** (promedio de Valor individual de todos los miembros). Ver [Valor — Transiciones de Estado](09-valor.md#transiciones-de-estado-por-valor-de-escuadra) para el modelo completo.

| Estado | Descripción | Órdenes permitidas | Causa |
|--------|-------------|-------------------|-------|
| **ACTIVE** | Operativa, rendimiento normal. | MOVE, DEFEND, ATTACK | Valor > umbral RETREAT |
| **RETREAT** | Retirarse controladamente. Sub-rendimiento. | MOVE (hacia retaguardia), DEFEND | Valor ≤ umbral RETREAT, pero > umbral ROUTED |
| **ROUTED** | Pérdida de cohesión, movimiento automático hacia retaguardia. Incontrolable. | Ninguna (ignora órdenes). Movimiento automático. | Valor ≤ umbral ROUTED |
| **ELIMINATED** | Removida del juego. | N/A | Todos los miembros muertos. |

### Transición y reversibilidad

- **Degradación:** ACTIVE → RETREAT → ROUTED (cuando Valor cae)
- **Recuperación:** ROUTED → RETREAT → ACTIVE (cuando Valor sube durante períodos de seguridad)
- Los estados son **reversibles**: una escuadra ROUTED puede recuperarse gradualmente si pasa tiempo sin amenaza inmediata
- Ver [Valor — Recuperación de ROUTED](09-valor.md#progresión-y-recuperación) para detalles de mecánica

**Nota:** Los estados de escuadra (ACTIVE, ROUTED, RETREAT) son distintos del sistema de heridas individual. Una escuadra puede estar en estado ACTIVE pero tener soldados individuales con heridas acumuladas que degradan su capacidad de combate gradualmente. El daño individual es persistente — las heridas del soldado no se limpian ni se recuperan durante la partida. Un Sanitario puede estabilizar a un soldado herido para evitar que empeore, pero el daño ya infligido permanece.

## Despliegue

<!-- Pendiente: cómo se colocan las escuadras al inicio —
     zona de despliegue, restricciones, simultaneidad -->
