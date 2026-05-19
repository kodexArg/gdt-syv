# ADR-009: Modelo de datos — Fuerza militar

## Estado

Aceptado — 2026-04-11

## Contexto

La arquitectura de SyV separa logica compartida en `shared/` (ADR-008). Para implementar
el ciclo de turno (ADR-004) se necesita un modelo de datos canonico que represente la fuerza
militar de cada jugador: sus tropas, su organizacion interna y su cadena de mando.

Este ADR define las entidades de fuerza militar que viven en `shared/`. Excluye identidad
de jugador, autenticacion, persistencia, UI y protocolo de red.

## Decision

La fuerza militar se representa con cinco entidades organizadas jerarquicamente:

```
Seccion
  └── Peloton
        ├── (HQ de Peloton: Teniente L5 + Sargento Primero L4 + tropas)
        └── Escuadra[]
              ├── Grupo[] (0..N agrupaciones internas)
              │     └── Tropa[]
              └── Tropa[] (tropas directamente en la escuadra, sin grupo)
```

### Tropa

La unidad atomica. Representa un combatiente individual.

```
id:                 String
rank:               String          # grado (faction-dependiente: "Sargento", "Sargento de Segunda")
level:              int             # 1..5 — determina autoridad, radio y autonomia
faction:            Faction | null  # null = sin restriccion de faccion
squad_ref:          Escuadra        # escuadra a la que pertenece
compatible_squads:  Array[String]   # tipos de escuadra que puede integrar
                                    # ejemplo: jinete → ["infernales"]
                                    # ejemplo: soldado → ["infantry", "mortars", "recon", ...]
equipment:          Equipment[]     # relacion 1-a-varios; tabla Equipment pendiente de definir
has_radio:          bool            # calculado: true si level >= 3
                                    # excepcion: Tropa de Comunicaciones L1 tiene radio global
alive:              bool
```

**Niveles y sus propiedades:**

| Nivel | Autoridad | Radio | Autonomia |
|-------|-----------|-------|-----------|
| L5 | Mando de Peloton | Si (5 hex) | Recibe ordenes genericas del HQ |
| L4 | Subjefe de Peloton | Si (5 hex) | Alta |
| L3 | Jefe de Escuadra | Si (5 hex) | Media |
| L2 | Sub-lider de Grupo o lider de escuadra especial | No | Limitada |
| L1 | Tropa basica | No | Ninguna — se desbanda sin mando |

**Excepcion radio global**: La Tropa de Comunicaciones es L1 pero porta una radio de
alcance ilimitado. Por si sola no extiende la cadena de mando. Cuando esta asignada a una
escuadra con un oficial L3+, amplifica el radio de ese oficial (alcance exacto pendiente).

### Grupo

Agrupacion organizacional interna de una Escuadra. No ocupa hex propio; es parte de su Escuadra.

```
id:      String
name:    String  # nombre segun tipo de escuadra: "Grupo de Fuego", "Grupo Scout",
                 # "Grupo de Ingenieria", "Los Artilleros", etc.
leader:  Tropa   # el L2 que lidera este grupo
members: Array[Tropa]  # 2..5 tropas (sin incluir al lider)
```

Los Grupos permiten definir mecanicas especificas para sub-conjuntos dentro de una escuadra.
Ejemplos: un Grupo MG ejecuta una MG como sistema (apuntador + asistentes). Un Grupo Scout
(sniper + oteador) puede tener reglas de vision especiales.

### Escuadra

La unidad leaf de combate. Ocupa exactamente 1 hex. Computa el combate.

```
id:            String
name:          String
type:          SquadType    # INFANTRY, CAVALRY, RECON, MORTARS, ENGINEERS, COMMAND, ...
faction:       Faction
squad_level:   int          # nivel L del lider (L3 = estandar, L2 = escuadra especial)
special_rules: bool         # true si squad_level == L2 (Infernales, commandos, zapadores)
hex:           HexCoord     # posicion en el tablero
state:         UnitState    # ACTIVE, ROUTED, RETREAT, ELIMINATED
groups:        Array[Grupo] # 0..N grupos internos
members:       Array[Tropa] # tropas fuera de grupo (lider L3, tropas directas)
```

Computed:
```
strength   = sum(tropa.level para tropa.alive)  # formula exacta pendiente
moral      = aggregate(tropa.moral)             # pendiente
has_radio  = any(tropa.has_radio and tropa.alive)
in_command = (ver ADR-008 BFS desde HQ, distancia de hex <= 5 por hop)
```

**Escuadra estandar (L3)**: liderada por un Sargento (L3) con radio. Tiene grupos internos
opcionales. Minimo 4 miembros; maximo ~15.

**Escuadra especial (L2)**: liderada por un L2 (Cabo Primero, Jefe de Partida, etc.). Sin radio.
Opera con ordenes genericas. Puede activar reglas especiales de su tipo. Ejemplos: Los Infernales
(caballeria), commandos, zapadores/ingenieros, recon avanzado.

### Peloton

Agrupacion de Escuadras bajo un mando de peloton (L5 Teniente + L4 Sargento Primero).
El Peloton incluye su propio elemento de mando como una Escuadra especial de tipo COMMAND.

```
id:      String
name:    String
hq:      Escuadra    # escuadra de mando (Teniente L5 + Sargento Primero L4 + tropas)
squads:  Array[Escuadra]
```

### Seccion

La fuerza completa de un jugador. Contiene Pelotones.

```
faction:      Faction
hq_hex:       HexCoord     # posicion fija del Comandante (HQ estrategico)
platoons:     Array[Peloton]
order_pool:   int          # calculado: sum de contribuciones L3/L4/L5 vivos
```

## Consecuencias

**Positivas:**
- Jerarquia clara y derivable: Seccion > Peloton > Escuadra > Grupo > Tropa
- Escuadra como unidad leaf permite computar combate sin ambiguedad
- `compatible_squads` en Tropa permite restricciones de composicion (jinete solo en caballeria)
- L2 como nivel de excepcion habilita escuadras especiales sin reglas ad-hoc
- Grupo como entidad organizacional permite definir mecanicas de sub-conjunto (MG, mortero, scout)

**Negativas:**
- `compatible_squads` requiere mantenimiento al agregar nuevos tipos de escuadra
- `strength` y `moral` como valores computados necesitan formula definitiva (pendiente)
- La relacion Tropa-Equipment queda sin schema hasta que se desarrolle ese sistema

## Alternativas descartadas

- **Tropa directamente en Escuadra sin Grupo**: pierde la capacidad de definir mecanicas
  para sub-conjuntos (MG como sistema, mortero como pieza, scouts como duo).
- **Escuadra especial como tipo separado**: duplica la jerarquia. El campo `squad_level`
  y `special_rules` son suficientes para distinguir comportamientos.
- **Pelotón como entidad plana (sin HQ propio)**: el mando de peloton tiene posicion
  en el tablero y participa en la cadena de mando, por lo que necesita representacion
  como Escuadra de tipo COMMAND.
