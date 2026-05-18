# ADR-011: Resolucion de combate estocastica por soldado

## Estado

Aceptado — 2026-05-18

## Contexto

El capitulo [07 — Combate](../manual/07-combate.md) define un modelo de combate
**estocastico, resuelto por soldado individual (L1)**: cada soldado realiza una
tirada independiente y la probabilidad de baja se modifica por cuatro factores
(stats + modificadores, equipamiento, estado de la escuadra, contexto tactico).
El capitulo describe la **forma** del modelo pero deja la **formula exacta**
marcada como "Pendiente de definir". Lo mismo ocurre con el umbral de eliminacion
por acumulacion de heridas (cap. 07 y cap. [04 — Unidades](../manual/04-unidades.md)).

Sin una formula concreta no es posible implementar la fase de Resolution
(ADR-004) ni cerrar el bloqueante de combate. Este ADR fija la matematica de
resolucion de **un intercambio ya ordenado en la linea de tiempo** (la iniciativa
y el orden temporal los resuelve otro mecanismo; aqui se asume "dado un
intercambio que toca resolverse, asi se calcula su efecto").

Restricciones de coherencia respetadas:

- Escala 100 m/hex y tabla de distancias del cap. [02b](../manual/02b-distancias.md).
- Modelo de datos de fuerza de [ADR-009](009-modelo-de-datos-fuerza-militar.md)
  (Tropa con `level`, `equipment`, modificadores; Escuadra con `state`).
- L1–L5 **no** determina poder de combate (cap. 04): el `level` no entra en la
  potencia del soldado, solo en `strength` agregado de la escuadra.
- Heridas como modificadores negativos acumulables (cap. 07), sin estados
  discretos.

Este ADR **no modifica** el capitulo del manual ni otros ADRs; el cap. 07 lo
referenciara cuando se actualice.

## Decision

### 1. Stats canonicos del soldado

Se fijan **cuatro stats base** por soldado, en escala 0–100, heredados de la
plantilla de clase (cap. 04). Valores base de referencia para la clase Soldado
de Infanteria L1 (supuesto de balance, sujeto a playtest):

| Stat | Sigla | Que representa | Base infanteria |
|------|-------|----------------|-----------------|
| Punteria | `AIM` | Probabilidad de conectar fuego ofensivo | 50 |
| Resistencia | `TGH` | Capacidad de absorber daño sin caer | 50 |
| Reaccion | `RCT` | Rapidez de respuesta (alimenta iniciativa, fuera de scope aqui; aqui solo aporta un pequeño bono defensivo) | 50 |
| Temple | `NRV` | Estabilidad bajo fuego (resiste supresion/panico) | 50 |

Los modificadores (Perks +, Traits +/-, Phobias -, Heridas -) se aplican como
**suma aritmetica sobre el stat afectado**, con clamp final a `[1, 100]`:

```
stat_efectivo(S) = clamp( base(S) + Σ mod_i(S) , 1, 100 )
```

Ejemplo: base `AIM`=50, Perk "Tirador" `+15`, Herida en brazo `-20`
→ `AIM_ef = clamp(50 + 15 - 20, 1, 100) = 45`.

El `level` (L1–L5) **no aparece** en ninguna formula de potencia individual,
coherente con cap. 04.

### 2. Potencia ofensiva y defensiva por soldado

Cada soldado entra a un intercambio como **atacante** (A, el que dispara en este
evento) contra un **defensor** (D, el blanco designado). Se computan dos
escalares:

```
P_atk = AIM_ef(A) · w_arma(A) · f_estado(A) · f_tactico_atk(A,D)
P_def = TGH_ef(D) · k_cobertura(D) · f_estado(D) · f_tactico_def(D) + 0.15 · RCT_ef(D)
```

donde:

- `w_arma`: multiplicador de equipamiento (tabla §4). Pistola 0.6, fusil 1.0,
  fusil de asalto 1.25, MG 1.6, mortero/artilleria via regla de fuego indirecto
  (no cubierto aqui).
- `f_estado`: estado de la escuadra del soldado — `ACTIVE` = 1.0,
  `RETREAT` = 0.75, `ROUTED` = 0.0 (no dispara; si es defensor, `f_estado(D)`
  aplica como 0.6: una escuadra desbandada es mas vulnerable, no menos).
- `f_tactico_atk` / `f_tactico_def`: producto de modificadores tacticos (§3).
- `k_cobertura(D)`: bono multiplicativo de terreno/cobertura del hex del defensor
  (§3), tipicamente `1.0`–`1.6`.

### 3. Factor de distancia y contexto tactico

**Distancia (decaimiento).** El alcance se mide en hexes (100 m/hex, cap. 02b).
Se usa una caida exponencial suave con un alcance efectivo `r_eff` dependiente
del arma:

```
f_dist(d) = exp( -((d - 1) / r_eff)^2 )      para d >= 1 hex
```

`r_eff` por arma (supuesto de balance):

| Arma | `r_eff` (hex) | Lectura |
|------|---------------|---------|
| Pistola | 1.0 | letal solo en hex adyacente |
| Fusil | 3.0 | efectivo 2–3 hex (coherente con cap. 02b) |
| Fusil de asalto | 3.0 | idem, mayor volumen de fuego (via `w_arma`) |
| MG pesada | 6.0 | efectivo hasta ~6 hex |

Tabla resultante `f_dist` para fusil (`r_eff=3`):

| d (hex) | metros | f_dist |
|---------|--------|--------|
| 1 | 100 | 1.000 |
| 2 | 200 | 0.895 |
| 3 | 300 | 0.641 |
| 4 | 400 | 0.368 |
| 5 | 500 | 0.169 |
| 6 | 600 | 0.061 |
| 8 | 800 | 0.0038 |

`f_dist` es un componente de `f_tactico_atk`.

**Contexto tactico.** Cada modificador tactico es un factor multiplicativo
aplicado al lado correspondiente (supuesto de balance):

| Factor | Lado | Multiplicador |
|--------|------|---------------|
| `f_dist(d)` (distancia) | atacante | 0.0–1.0 (tabla) |
| Elevacion: atacante mas alto que defensor | atacante | ×1.15 |
| Elevacion: defensor mas alto | defensor | ×1.15 |
| Flanqueo: atacante sin enfrentar el frente del defensor | atacante | ×1.25 |
| Cobertura del hex (`k_cobertura`): abierto / ligera / pesada | defensor | ×1.0 / ×1.25 / ×1.55 |
| Orden DEFEND activa en el defensor | defensor | ×1.30 |
| Terreno dificil bajo el atacante (sin apoyo estable) | atacante | ×0.90 |

```
f_tactico_atk = f_dist(d) · m_elev_atk · m_flanqueo
f_tactico_def = m_elev_def · m_DEFEND        (k_cobertura va aparte en P_def)
```

### 4. Tabla de equipamiento (`w_arma`)

Supuesto de balance, sujeto a playtest. La tabla `Equipment` formal (ADR-009,
pendiente) debera exponer al menos estos campos:

| Arma | `w_arma` | `r_eff` (hex) |
|------|----------|---------------|
| Pistola | 0.60 | 1.0 |
| Fusil cerrojo | 1.00 | 3.0 |
| Fusil de asalto | 1.25 | 3.0 |
| MG pesada | 1.60 | 6.0 |
| Sin arma (desarmado) | 0.20 | 1.0 |

Proteccion (casco/blindaje ligero) entra como modificador aditivo a `TGH_ef`
del defensor (p. ej. `+10`), no como factor aparte, para reutilizar el sistema
de modificadores.

### 5. Mapeo a probabilidad del intercambio

Se calcula la **ventaja relativa** del atacante:

```
Δ = (P_atk - P_def) / (P_atk + P_def)        ∈ (-1, 1)
```

`Δ` se mapea a la probabilidad de impacto efectivo mediante una logistica
centrada en una probabilidad base `p0` con pendiente `s`:

```
p_hit = p0 + (p_max - p0) · ( 1 / (1 + exp(-s · Δ)) - 0.5 ) · 2        si Δ >= 0
p_hit = p0 · ( 1 / (1 + exp(-s · Δ)) ) / 0.5                             si Δ < 0
```

Equivalente operativo y mas simple de implementar (se adopta esta forma
canonica):

```
p_hit = clamp( p0 + g · Δ , p_min , p_max )
```

Parametros (supuesto de balance):

| Param | Valor | Rol |
|-------|-------|-----|
| `p0` | 0.18 | probabilidad base de un intercambio con fuerzas igualadas (Δ=0) |
| `g`  | 0.55 | ganancia: cuanto pesa la ventaja relativa |
| `p_min` | 0.02 | el lado mas debil siempre conserva chance real |
| `p_max` | 0.85 | el lado mas fuerte nunca tiene victoria garantizada |

El valor bajo de `p0` materializa el principio del cap. 07 ("la mayoria de los
enfrentamientos no producen resultado" — desgaste/erosion lenta).

### 6. Resolucion por soldado (procedimiento)

Para cada soldado atacante con blanco asignado, en el orden ya determinado por
la linea de tiempo:

1. Si `f_estado(A) == 0` (escuadra ROUTED) o `A.alive == false` o el blanco ya
   no esta vivo → el disparo no se resuelve. Continuar.
2. Calcular `P_atk`, `P_def`, `Δ`, `p_hit` (§2–§5).
3. Tirar `u ~ Uniforme[0,1)`.
4. Si `u >= p_hit` → **bounce**: sin efecto. Emitir evento `MISS`.
5. Si `u < p_hit` → **impacto**. Calcular severidad de la herida (§7),
   aplicarla como modificador negativo al defensor, reevaluar umbral de
   eliminacion (§7). Emitir evento `HIT` (y `KIA` si fue eliminado).
6. Las heridas/eliminaciones aplicadas son visibles para disparos **posteriores**
   en la misma linea de tiempo (un defensor ya KIA deja de ser blanco valido).

Cada soldado tira **independientemente**. No hay agregacion a nivel de escuadra:
la escuadra solo provee `f_estado` y la posicion (hex/cobertura/elevacion).

### 7. Heridas, severidad y umbral de eliminacion

En un impacto se determina la **severidad** con una tirada secundaria modulada
por la magnitud de la ventaja del atacante:

```
sev_base ~ Uniforme[8, 22]                    # puntos de penalizacion
sev = round( sev_base · (1 + 0.5 · max(Δ, 0)) )   # ventaja fuerte → heridas peores
```

La herida se inserta como modificador negativo. Su penalizacion se reparte
sobre stats segun una tabla de "parte del cuerpo" (supuesto, p. ej. brazo→`AIM`,
pierna→`MOV`/`RCT`, torso→`TGH`). Para el calculo de umbral lo que cuenta es la
**carga de heridas acumulada**:

```
W = Σ |penalizacion_i|   sobre todas las heridas activas del soldado
```

**Umbral de eliminacion** `W_kia = 60` (supuesto de balance). Reglas:

- `W >= W_kia` → el soldado es eliminado (`alive = false`), salvo §7-bis.
- Un solo impacto con `sev` alto puede cruzar el umbral (muerte directa
  plausible); varios impactos leves tambien (erosion).
- Las heridas son permanentes; `W` solo crece (cap. 07).

**§7-bis — Sanitario.** Si un Sanitario vivo esta en la misma escuadra y el
soldado quedaria eliminado por umbral (no por impacto masivo `sev >= 45`), el
soldado queda **estabilizado**: sobrevive con `W` congelada en el valor actual;
no recibe penalizaciones adicionales por "complicaciones" en turnos futuros,
pero conserva todas sus heridas y stats degradados. El Sanitario no remueve
heridas ni baja `W` (cap. 07: preservador, no sanador). Un impacto con
`sev >= 45` es letal aunque haya Sanitario (herida critica inmediata).

### 8. Ejemplo numerico trabajado

Escenario: soldado A (fusil de asalto, escuadra ACTIVE) dispara a soldado D a
**3 hexes** (300 m), terreno con **cobertura ligera**, A en **elevacion
superior**, D **sin orden DEFEND**, D en escuadra ACTIVE.

Stats efectivos:
- A: `AIM_ef = 60` (base 50 + Perk Tirador +10).
- D: `TGH_ef = 55` (base 50 + casco +5), `RCT_ef = 50`.

Factores:
- `w_arma(A)` = 1.25 (fusil de asalto), `r_eff` = 3.
- `f_dist(3)` con `r_eff=3` = `exp(-((3-1)/3)^2)` = `exp(-0.444)` = **0.641**.
- `m_elev_atk` = 1.15; `m_flanqueo` = 1.0; `f_estado(A)` = 1.0.
- `f_tactico_atk` = 0.641 · 1.15 · 1.0 = **0.737**.
- `k_cobertura(D)` = 1.25 (ligera); `m_elev_def` = 1.0; `m_DEFEND` = 1.0;
  `f_estado(D)` = 1.0; `f_tactico_def` = 1.0.

Potencias:
- `P_atk` = 60 · 1.25 · 1.0 · 0.737 = **55.3**.
- `P_def` = 55 · 1.25 · 1.0 · 1.0 + 0.15 · 50 = 68.75 + 7.5 = **76.25**.

Ventaja y probabilidad:
- `Δ` = (55.3 − 76.25) / (55.3 + 76.25) = −20.95 / 131.55 = **−0.159**.
- `p_hit` = clamp(0.18 + 0.55 · (−0.159), 0.02, 0.85)
  = clamp(0.18 − 0.0875, …) = **0.092**.

Resultado: ~9% de baja efectiva en este intercambio. Coherente con el desgaste:
disparar a un defensor cubierto a 300 m raramente surte efecto, pero la
posibilidad existe. Si hubiera impacto: `sev_base`∈[8,22], `Δ<0` ⇒ `sev = sev_base`
(p. ej. 14). Una sola herida no cruza `W_kia=60`; haria falta acumulacion.

Contraejemplo (asalto a 1 hex, atacante flanqueando, defensor ROUTED):
`f_dist(1)=1.0`, `m_flanqueo=1.25`, `f_estado(D)=0.6`. `P_atk` sube y `P_def`
baja → `Δ` ≈ +0.45 → `p_hit` = clamp(0.18 + 0.55·0.45) ≈ **0.43**, tope util muy
por debajo de 1.0: ni el mejor caso garantiza la baja.

## Parametros tuneables (balance)

Todos marcados como **supuesto de balance, sujeto a playtest**:

| Param | Valor inicial | Efecto al subir |
|-------|---------------|-----------------|
| `p0` | 0.18 | mas letalidad global / partidas mas rapidas |
| `g` | 0.55 | la ventaja relativa decide mas (menos azar) |
| `p_min` / `p_max` | 0.02 / 0.85 | piso/techo de azar; nunca 0 ni 1 |
| Stats base de clase | 50 c/u | calibracion por clase/faccion |
| `w_arma` por arma | tabla §4 | poder relativo del equipamiento |
| `r_eff` por arma | tabla §3 | alcance efectivo / forma del decaimiento |
| Multiplicadores tacticos | tabla §3 | peso de elevacion/flanqueo/cobertura/DEFEND |
| `f_estado` (RETREAT/ROUTED) | 0.75 / 0.0(atk) 0.6(def) | severidad de la penalizacion por estado |
| `sev_base` rango | [8, 22] | granularidad de heridas |
| Escalado `sev` por Δ | 0.5 | cuanto agravan las heridas las ventajas fuertes |
| `W_kia` | 60 | resiliencia del soldado (cuantas heridas aguanta) |
| Umbral letal Sanitario | `sev >= 45` | que tan a menudo el medico puede salvar |

## Consecuencias

**Positivas:**

- Formula explicita, deterministica salvo dos tiradas uniformes (`u` de impacto,
  `sev_base`): facil de implementar, testear y reproducir en el backend
  autoritativo (ADR-003/004).
- Respeta cap. 04: `level` no entra en potencia individual; los stats y
  modificadores si.
- Reutiliza el sistema abierto de modificadores para Perks/Traits/Phobias,
  Heridas y proteccion (suma aditiva sobre stats) — sin estados discretos.
- `p0` bajo + `p_max=0.85` codifican el desgaste/erosion del cap. 07: la mayoria
  de intercambios "rebotan".
- Distancia anclada a la escala 100 m/hex y a la tabla del cap. 02b.
- Resolucion estrictamente por soldado e independiente: compatible con la
  designacion de objetivos por soldado (cap. 07) y con la linea de tiempo.

**Negativas:**

- Muchos parametros (≈15) requieren playtest para calibrar; sin telemetria el
  balance es conjetural.
- La forma lineal `p0 + g·Δ` es una simplificacion de la logistica; en los
  extremos de `Δ` puede saturar antes — aceptable por el clamp `[p_min,p_max]`.
- El reparto de severidad por "parte del cuerpo" se deja como tabla pendiente
  (no bloquea: el umbral usa la carga agregada `W`).
- No modela supresion/panico como salida del combate (alimenta Valor/estado en
  otros capitulos); aqui Temple `NRV` solo se reserva para esa interaccion
  futura y no entra todavia en `p_hit`.

## Alternativas descartadas

- **Razon de fuerzas `P_atk / P_def` directa a probabilidad**: sin acotacion
  natural; explota con `P_def→0` y es poco intuitiva de tunear. La forma
  normalizada `Δ ∈ (-1,1)` es estable y simetrica.
- **Tabla de resultados tipo wargame (CRT 2d6)**: granularidad fija y dificil de
  parametrizar por arma/distancia continua; choca con el modelo continuo
  estocastico del cap. 07.
- **Estados de herida discretos (leve/grave/critico)**: explicitamente rechazado
  por el cap. 07; se usa carga acumulada `W` contra umbral.
- **Incluir `level` como multiplicador de potencia**: contradice cap. 04 (el
  nivel es autoridad/radio, no poder de combate).
- **Decaimiento lineal de distancia**: penaliza poco el fuego largo y mucho el
  corto; el gaussiano `exp(-((d-1)/r_eff)^2)` da meseta cercana y caida marcada
  en el limite del alcance, coherente con la tabla del cap. 02b.
```
