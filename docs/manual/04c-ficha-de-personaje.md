---
title: "Ficha de Personaje (PJ) — Schema canónico"
order: 43
section: units
lang: es
---

# Ficha de Personaje (PJ) — Schema canónico

Anexo del manual que especifica la ficha del soldado individual.
Fuente de verdad estructural de una **Tropa** (PJ, unidad atómica
del juego; ver [04 — Unidades](04-unidades.md)). Define qué campos
tiene un soldado, su tipo, rango y de dónde sale cada valor.

No reabre los stats de combate: los 6 stats de
[ADR-013](../adr/013-stats-canonicos-y-catalogo-de-modificadores.md)
se adoptan tal cual y se suma un 7º stat (`VIG`, ver §2).

Convención de fuente (de dónde sale cada campo):

- `[CLASE]`   heredado de la plantilla de clase.
- `[ENGINE]`  asignado/derivado por el servidor en creación.
- `[AZAR]`    asignado aleatoriamente por el servidor (oculto al cliente).
- `[RUNTIME]` estado dinámico que cambia durante la partida.
- `[NARR]`    campo narrativo, sin efecto mecánico.

---

## 1. Identidad

Campos que individualizan al PJ. No afectan el cálculo de combate.

| Campo | Tipo | Fuente | Descripción |
|-------|------|--------|-------------|
| `id` | uint | ENGINE | Identificador único de la tropa |
| `nombre` | str | ENGINE | Nombre y apellido del soldado |
| `rank` | str | CLASE | Grado (faction-dep.: "Sargento"…) |
| `level` | L1–L5 | CLASE | Autoridad/radio/autonomía, NO poder |
| `faction` | enum | ENGINE | Bando; `null` si sin restricción |
| `compatible_squads` | list | CLASE | Tipos de escuadra que puede integrar |
| `hoja_de_vida` | str | NARR | Backstory canónico, 1–2 frases |

`hoja_de_vida` es el insumo que ancla la crónica de combate (evita que
el LLM invente backstory). Ver template de crónica.

---

## 2. Stats canónicos (7)

Escala 0–100, suma aritmética de modificadores con clamp `[1,100]`
(ADR-013 §1.2):

```
stat_efectivo(S) = clamp( base(S) + Σ mod_i(S) , 1, 100 )
```

| # | Stat | Sigla | Base L1 | Fuente | Origen |
|---|------|-------|---------|--------|--------|
| 1 | Puntería | `AIM` | 50 | CLASE | ADR-013 |
| 2 | Resistencia | `TGH` | 50 | CLASE | ADR-013 |
| 3 | Reacción | `RCT` | 50 | CLASE | ADR-013 |
| 4 | Temple | `NRV` | 50 | CLASE | ADR-013 |
| 5 | Movilidad | `MOV` | 50 | CLASE | ADR-013 |
| 6 | Valor | `VAL` | 50 | CLASE | ADR-013 / cap.09 |
| 7 | Vigor | `VIG` | 50 | CLASE | **nuevo (este ADR)** |

### 2.1 Vigor (`VIG`) — el 7º stat

Capacidad de aguante físico. **100 = fresco; bajo = exhausto.** La
*fatiga* es la condición de `VIG` bajo, igual que el *pánico* es la
condición de `VAL` bajo. Es un modificador pasivo global: cuando cae,
degrada todo el desempeño (paralelo exacto a `VAL`, ADR-013 §1.2).

- Vive en el mismo sistema suma+clamp `[1,100]` que el resto.
- Se degrada por **modificadores transitorios negativos** acumulados por
  esfuerzo (movimiento gastado, combates del turno, heridas cargadas) —
  análogos a los de `VAL`, reversibles.
- Se recupera en turnos de descanso/seguridad (sin combate ni movimiento).

Triggers exactos de degradación, magnitudes y curva de recuperación:
**Pendiente de definir** (supuesto de balance, sujeto a playtest). Este
schema solo fija que `VIG` existe, su dirección (más = mejor) y que
pertenece al sistema de suma+clamp. La formalización de su interacción
con las fórmulas de combate (ADR-011/012) queda para el ADR que lo cierre.

---

## 3. Modificadores (Perks / Traits / Phobias)

Relación 1-a-varios. Cada modificador expresa su efecto solo como suma
aditiva sobre uno o más stats del §2. Catálogo y magnitudes en ADR-013
§2. Asignados en creación por `[AZAR]` y **ocultos** hasta su primera
activación (cap. 04).

| Campo | Tipo | Fuente | Descripción |
|-------|------|--------|-------------|
| `tipo` | enum | AZAR | PERK \| TRAIT \| PHOBIA |
| `nombre` | str | AZAR | Entrada del catálogo (ADR-013 §2) |
| `efecto` | map | AZAR | `{stat: delta}` aditivo |
| `revelado` | bool | RUNTIME | `false` hasta primera activación |
| `disparador` | str | CLASE | Condición de revelado (ADR-013 §2.3) |

Mutación en partida (ganar/perder modificador): rara, determinista por
semilla, siempre notificada. Ver ADR-013 §3.

---

## 4. Heridas

Relación 1-a-varios. Modificadores **negativos permanentes** adquiridos
en combate (no se asignan en creación, no se recuperan). Definidas por
ADR-011 §7.

| Campo | Tipo | Fuente | Descripción |
|-------|------|--------|-------------|
| `sev` | float | RUNTIME | Severidad de la herida |
| `parte` | enum | RUNTIME | brazo→AIM, pierna→MOV/RCT, torso→TGH |
| `carga_W` | float | RUNTIME | Aporte a la carga total de heridas |

El soldado es KIA cuando `Σ carga_W ≥ W_kia` (`W_kia=60`, ADR-011 §7).

---

## 5. Estado dinámico (runtime)

Estado que cambia turno a turno. Todo `[RUNTIME]`.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `condicion` | enum | alive \| kia \| captured |
| `vig_actual` | 1–100 | Vigor efectivo (fatiga = valor bajo) |
| `val_actual` | 1–100 | Valor efectivo (pánico = valor bajo) |
| `combates_previos` | uint | Combates librados en la partida |
| `rol_turno` | str | toma_el_mando \| fuego_cobertura \| … |
| `mods_transitorios` | list | Penaltis reversibles sobre VAL/VIG |

`in_command` y los estados de escuadra (ACTIVE/RETREAT/ROUTED) se
computan a nivel Escuadra, no en la ficha del PJ (cap. 04, cap. 08).

---

## 6. Equipment

Relación 1-a-varios. Schema completo **Pendiente de definir**
(cap. 04 / ADR-021).

| Campo | Tipo | Fuente | Descripción |
|-------|------|--------|-------------|
| `item` | str | CLASE | Ítem portado (fusil, munición, …) |
| `efecto` | map | CLASE | Efecto en combate (pendiente) |

---

## 7. Campos computados

Derivados, no se almacenan:

- `stat_efectivo(S)` — §2, suma+clamp `[1,100]`.
- `has_radio` — `true` si `level >= 3` (excepción: Comunicaciones L1).
- `alive` — `condicion == alive`.

---

## Pendientes que este schema deja abiertos

- Triggers, magnitudes y recuperación de `VIG` (§2.1).
- Interacción de `VIG` con fórmulas de combate (ADR-011/012).
- Schema de Equipment (§6).
- Tabla fina de reparto de heridas por parte del cuerpo (ADR-011 §7).
