# ADR-021: Agregacion de escuadra (strength/moral), reparto de heridas por zona y schema de Equipment

## Estado

Aceptado — 2026-05-18

## Contexto

[ADR-009](009-modelo-de-datos-fuerza-militar.md) deja cuatro huecos explicitos
en la entidad `Escuadra` y en la relacion Tropa–Equipment:

```
strength   = sum(tropa.level para tropa.alive)  # formula exacta pendiente
moral      = aggregate(tropa.moral)             # pendiente
equipment:  Equipment[]  # tabla Equipment pendiente de definir
```

A su vez:

- [ADR-011](011-resolucion-de-combate-por-soldado.md) §7 dejo **pendiente la
  tabla de reparto de la penalizacion de heridas por parte del cuerpo**
  (declaro el ejemplo "brazo→`AIM`, pierna→`MOV`/`RCT`, torso→`TGH`" pero no la
  tabla cerrada), aclarando que esto **no bloquea** porque el umbral usa la
  carga agregada `W` (`W_kia = 60`).
- ADR-011 §4 anticipa que la **tabla `Equipment` formal (ADR-009, pendiente)
  debera exponer al menos** `w_arma` y `r_eff`, y que la proteccion
  (casco/blindaje) entra como **modificador aditivo a `TGH_ef`** del defensor,
  no como factor aparte.
- [ADR-012](012-calculo-de-iniciativa-orden-de-resolucion.md) §5 usa `strength`
  de escuadra como **tercer criterio de desempate** de iniciativa (tras `A` y
  tipo de orden), por lo que `strength` debe ser un escalar total comparable y
  determinista.
- [ADR-013](013-stats-canonicos-y-catalogo-de-modificadores.md) §1 fija el
  conjunto canonico de seis stats (`AIM/TGH/RCT/NRV/MOV/VAL`), confirma que las
  Heridas reparten su penalizacion sobre esas siglas y nombra `VAL` (Valor) como
  el stat individual de moral, cuya **degradacion/recuperacion por triggers
  queda para el grupo de decisiones de Valor** (cap. 09 — futuro ADR-017), no
  para este ADR.

El capitulo [04 — Unidades](../manual/04-unidades.md) refuerza estos pendientes:
"Fuerza de la escuadra … ¿level directo o ponderado?", "`moral` = agregado de
moral individual <!-- pendiente -->", y "schema completo de Equipment — tipos,
efectos en combate, encimamiento de items — Pendiente". El capitulo
[07 — Combate](../manual/07-combate.md) describe el equipamiento como uno de los
cuatro factores que modifican la probabilidad (arma, accesorios, proteccion).

Este ADR cierra esos cuatro pendientes **sin reabrir** ADR-009/011/012/013 ni
el manual: adopta literalmente las siglas de ADR-013, la matematica de potencia
de ADR-011 (§2, §4: proteccion como aditivo a `TGH_ef`), el uso de `strength` en
desempates de ADR-012, y referencia a `VAL` (ADR-013/cap. 09) sin redefinir sus
triggers. Solo aporta la **estructura de agregacion** que faltaba.

Restricciones de coherencia respetadas:

- ADR-001 (glosario): Tropa, Escuadra, Grupo; relacion 1-a-varios; **L1–L5 no
  es poder de combate**.
- ADR-009: la `Escuadra` es la unidad leaf; `strength`/`moral` son campos
  *computed* derivados de tropas vivas; Tropa tiene `equipment: Equipment[]`.
- ADR-011: matematica de combate intacta; la proteccion sigue siendo un
  **modificador aditivo a `TGH_ef`** (ADR-011 §4), no un factor nuevo en
  `P_atk`/`P_def`.
- ADR-013: siglas canonicas y mecanica suma+clamp `[1,100]`; `VAL` es el stat
  de Valor y su dinamica de triggers **no se toca aqui**.

## Decision

### 1. Formula de `strength` de escuadra

`strength` es un escalar entero, **suma ponderada del nivel de cada tropa
viva**, no la suma cruda de `level`. La ponderacion materializa lo que cap. 04
ya afirma en prosa ("perder al lider L3 o a los L2 tiene mas impacto que perder
soldados L1") sin que ello convierta el `level` en poder de combate individual
(ADR-011/013): el `level` solo pesa en este **agregado estructural** de la
escuadra, jamas en `P_atk`/`P_def` de una tropa.

```
peso(level):  L1 → 1   L2 → 3   L3 → 5   L4 → 7   L5 → 9

strength(S) = Σ peso(tropa.level)   sobre tropa ∈ S con tropa.alive == true
```

- Escala lineal con paso 2 (1,3,5,7,9): un L3 vale 5 L1; perderlo recorta
  `strength` mas que perder cuatro L1 (4 vs 5). Refleja el impacto de mando del
  cap. 04 sin discontinuidades bruscas. **Supuesto de balance, sujeto a
  playtest** (los pesos son tuneables; la *estructura* "suma ponderada por
  nivel sobre vivos" es la decision estable).
- Determinista y monotono decreciente: cada baja solo puede bajar `strength`,
  nunca subirlo — requisito para su uso como **desempate de iniciativa**
  (ADR-012 §5.3, "mayor `strength`").
- No interviene `equipment`, stats ni heridas: `strength` mide **masa de mando
  viva**, no eficacia de combate (esa la resuelve ADR-011 por soldado). Esto
  mantiene ortogonales `strength` (estructura) y la potencia (combate),
  evitando doble conteo.
- Escuadra `ELIMINATED` o sin tropas vivas → `strength = 0`.

Ejemplo: escuadra con 1×L3 + 1×L2 + 8×L1 todas vivas →
`5 + 3 + 8·1 = 16`. Si cae el L3 → `3 + 8 = 11`. Si en vez del L3 caen 3 L1 →
`5 + 3 + 5 = 13`. Perder el lider duele mas, como exige cap. 04.

### 2. Agregacion estructural de `moral` de escuadra

`moral(S)` es la **agregacion estructural** de la moral individual de las
tropas vivas. La moral individual de una tropa es su stat **`VAL` efectivo**
(`VAL_ef`), tal como lo nombra y modela ADR-013 §1 (suma+clamp `[1,100]` sobre
la base de clase mas Perks/Traits/Phobias/Heridas y los modificadores
transitorios de Valor del cap. 09).

> **Alcance deliberadamente limitado.** Este ADR define **solo la formula de
> agregacion** (como se combinan los `VAL_ef` individuales en un escalar de
> escuadra). **No** define los *triggers* de Valor, su degradacion, su
> recuperacion, ni los umbrales que mueven el estado de escuadra
> `ACTIVE/RETREAT/ROUTED`: eso es competencia del cap. 09 y del grupo de
> decisiones de Valor (futuro ADR-017), y se referencia sin redefinirse. Aqui
> `moral(S)` es un **insumo** que ese mecanismo consumira; este ADR solo
> garantiza que el insumo exista y sea determinista.

```
moral(S) = clamp(
             round(  0.5 · VAL_ef(lider_operativo)
                    + 0.5 · media( VAL_ef(t) : t ∈ S, t.alive, t ≠ lider ) ),
             1, 100 )
```

- `lider_operativo`: el L3 en escuadra estandar, el L2 en escuadra especial
  (misma definicion de "lider operativo" que ADR-012 §2.1; si murio, pasa a la
  tropa viva de mayor `level`, empates por menor `id` — desempate determinista
  consistente con ADR-012 §5.4).
- **Ponderacion lider/tropa 0.5 / 0.5**: la moral colectiva depende tanto del
  temple del mando como de la masa de la tropa. Es una ponderacion mas suave
  que la de iniciativa (ADR-012 §2.1 usa `1 + 0.25·media`) **a proposito**: la
  iniciativa modela "capacidad de mando para hacer reaccionar"; la moral modela
  cohesion del conjunto, donde la tropa pesa tanto como el lider. **Supuesto de
  balance, sujeto a playtest**; la estructura estable es "combinacion convexa
  de VAL del lider y media VAL del resto, sobre vivos, con clamp".
- Si no hay tropas distintas del lider vivas, `moral(S) = VAL_ef(lider)`.
  Escuadra sin vivos → `moral(S) = 0` (no participa; coherente con
  `ELIMINATED`).
- Salida en escala **0–100** (misma que `VAL`), lista para que el mecanismo de
  Valor (cap. 09 / ADR-017) la mapee a transiciones de estado **sin renombrar
  nada** (igual contrato que ADR-013 dio a `VAL`).
- Determinista dado el estado del turno: apto para la verificacion de
  coherencia del Briefing (ADR-004), como `R`/mutacion en ADR-012/013.

### 3. Tabla de reparto de heridas por parte del cuerpo

Completa el pendiente de **ADR-011 §7** ("reparto de severidad por parte del
cuerpo … tabla pendiente") y de ADR-013 §2.4, **sin reabrir ADR-011**: ADR-011
ya fijo que (a) `sev` es la penalizacion total del impacto
(`sev = round(sev_base·(1+0.5·max(Δ,0)))`, `sev_base ~ U[8,22]`), (b) la herida
es un modificador negativo, y (c) el umbral usa la carga agregada
`W = Σ|penalizacion_i|` contra `W_kia = 60`. Lo unico que faltaba —y lo unico
que se fija aqui— es **sobre que stats canonicos (ADR-013 §1) se distribuye esa
`sev`**.

En cada impacto se tira la **zona** con esta distribucion (supuesto de balance,
sujeto a playtest; aproxima superficie corporal expuesta en combate de pie):

| Zona (tirada) | P(zona) | Reparto de `sev` sobre stats canonicos (ADR-013 §1) |
|---------------|---------|------------------------------------------------------|
| Brazo / mano | 30 % | `AIM −sev` (perdida de punteria/manejo del arma) |
| Pierna / pie | 25 % | `MOV −0.6·sev`, `RCT −0.4·sev` (movilidad y reaccion fisica) |
| Torso | 25 % | `TGH −0.7·sev`, `NRV −0.3·sev` (encaje fisico + conmocion) |
| Cabeza / casco | 12 % | `AIM −0.4·sev`, `NRV −0.4·sev`, `RCT −0.2·sev` |
| Heridas multiples / general | 8 % | `TGH −0.4·sev`, `AIM −0.3·sev`, `MOV −0.3·sev` |

Reglas:

- La suma de los repartos de cada fila es exactamente `sev`, de modo que la
  **carga `W` aportada por el impacto es siempre `sev`**, independientemente de
  la zona. Esto preserva intacta la aritmetica de umbral de ADR-011 §7
  (`W_kia = 60`): la zona decide *que capacidad* se degrada, **no** cuanto se
  acerca el soldado a morir. ADR-011 no se reabre.
- Cada componente se inserta como **un modificador negativo aditivo** sobre el
  stat indicado, redondeado a entero, exactamente con la mecanica suma+clamp
  `[1,100]` de ADR-011 §1 / ADR-013 §1.2 (`stat_ef = clamp(base+Σmod,1,100)`).
  El clamp no altera `W`: `W` se computa sobre el `|penalizacion_i|` nominal de
  la herida (ADR-011 §7), no sobre el stat ya recortado.
- `VAL` **no** se penaliza por heridas en esta tabla: la interaccion
  Heridas→Valor pertenece al cap. 09 / ADR-017 (Valor); aqui las heridas solo
  tocan capacidades fisicas/de combate (`AIM/TGH/RCT/NRV/MOV`), coherente con
  ADR-013 §2.4 (las Heridas reparten sobre las siglas canonicas, Valor es
  dominio del grupo de Valor).
- `NRV` aparece en torso/cabeza como **conmocion** (no como supresion: ADR-011
  reserva `NRV` para supresion futura y no lo lee en `p_hit`; aqui solo se
  degrada el stat por suma, sin introducir ninguna lectura nueva en la formula
  de ADR-011).
- La tirada de zona usa la **semilla del servidor del turno** (misma fuente que
  `u`/`sev_base` de ADR-011 §6–7 y `R` de ADR-012 §2.5): la Resolution sigue
  reproducible (ADR-004). Se tira despues de confirmar impacto y `sev`
  (ADR-011 §6 paso 5), antes de recomputar `W`.

Ejemplo: impacto con `sev = 14`, zona = Pierna/pie → modificadores
`MOV −8` (round 0.6·14=8.4→8) y `RCT −6` (round 0.4·14=5.6→6); aporte a
`W` = 8+6 = 14 = `sev` (consistente con ADR-011 §7).

### 4. Schema formal de Equipment y efectos en combate

Cierra el pendiente de ADR-009 (Tropa `equipment: Equipment[]`, "tabla
Equipment pendiente") y de ADR-011 §4 ("la tabla `Equipment` formal … debera
exponer al menos `w_arma` y `r_eff`"). La relacion **Tropa–Equipment es
1-a-varios** (ya en ADR-009: una tropa puede portar arma + proteccion + items;
cap. 04 §Equipment).

#### 4.1 Entidad `Equipment`

```
Equipment
  id:          String
  name:        String
  slot:        EquipSlot          # WEAPON | ARMOR | UTILITY
  w_arma:      float | null       # solo slot WEAPON; multiplicador de P_atk (ADR-011 §2/§4)
  r_eff:       float | null       # solo slot WEAPON; alcance efectivo en hex (ADR-011 §3)
  tgh_bonus:   int   | null       # solo slot ARMOR; modificador ADITIVO a TGH_ef del portador (ADR-011 §4)
  tags:        Array[String]      # p. ej. ["mg"], ["indirect"], ["medic"] — hooks de reglas existentes
```

- **`slot`** segmenta los tres tipos de efecto que cap. 07 enumera (arma,
  proteccion, accesorios). No se inventan nuevos factores en la matematica de
  ADR-011: cada slot mapea a un punto **ya existente** de sus formulas.
- Encimamiento (cap. 04, "encimamiento de items"): una Tropa puede portar
  varios `Equipment`. Reglas de combinacion en §4.3.

#### 4.2 `EquipSlot` y como entra cada uno en ADR-011

| Slot | Campo activo | Punto de entrada en ADR-011 (sin modificar ADR-011) |
|------|--------------|------------------------------------------------------|
| `WEAPON` | `w_arma`, `r_eff` | `w_arma` multiplica `P_atk` (ADR-011 §2: `P_atk = AIM_ef·w_arma·…`); `r_eff` parametriza `f_dist` (ADR-011 §3). **No** afecta `P_def`. |
| `ARMOR` | `tgh_bonus` | Modificador **aditivo a `TGH_ef`** del defensor (literal ADR-011 §4: "casco/blindaje … como modificador aditivo a `TGH_ef` … p. ej. `+10`"). Entra via el `Σ mod_i(TGH)` de la mecanica suma+clamp (ADR-013 §1.2). **No** introduce factor nuevo en `P_def`. |
| `UTILITY` | `tags` | Sin efecto directo en `P_atk`/`P_def`. Habilita reglas ya definidas en otros ADR/cap. via `tags` (p. ej. el rol Sanitario de ADR-011 §7-bis; municion: cap. 07 §111 declara que **no** hay rastreo de municion, asi que UTILITY nunca limita la capacidad de disparar). |

Tabla canonica de armas (slot `WEAPON`) — **copiada literal de ADR-011 §4, no
se altera ni un valor** (este ADR solo le da hogar de datos, no la redefine):

| `name` | `w_arma` | `r_eff` (hex) |
|--------|----------|---------------|
| Pistola | 0.60 | 1.0 |
| Fusil cerrojo | 1.00 | 3.0 |
| Fusil de asalto | 1.25 | 3.0 |
| MG pesada | 1.60 | 6.0 |
| Sin arma (desarmado) | 0.20 | 1.0 |

Proteccion (slot `ARMOR`) — magnitudes **supuesto de balance, sujeto a
playtest**, coherentes con el ejemplo `+5` (casco) y `+10` de ADR-011 §4/§8:

| `name` | `tgh_bonus` |
|--------|-------------|
| Casco | +5 |
| Blindaje ligero | +10 |
| Blindaje pesado | +18 |

#### 4.3 Reglas de combinacion (encimamiento)

- **Arma efectiva**: si una Tropa porta varios `WEAPON`, el combate usa **el de
  mayor `w_arma`** (su arma principal en ese intercambio). No se suman armas:
  un soldado dispara un arma por evento (coherente con ADR-011 §6, una tirada
  por soldado atacante). Sin ningun `WEAPON` → "Sin arma" (`w_arma=0.20`).
- **Proteccion efectiva**: los `tgh_bonus` de los `ARMOR` portados **se suman**
  (casco + blindaje ligero = `+15`) y entran como un unico aporte al
  `Σ mod_i(TGH)` de ADR-013 §1.2, antes del clamp `[1,100]` de ADR-011 §1. El
  clamp ya acota cualquier acumulacion excesiva; no se fija tope adicional.
- **UTILITY** no se combina aritmeticamente: cada `tag` activa (o no) su regla
  asociada de forma independiente.
- `equipment` no afecta `strength` (§1) ni `moral` (§2): es exclusivamente
  insumo de la potencia de combate de ADR-011. Ortogonalidad explicita para
  evitar doble conteo.

## Consecuencias

**Positivas:**

- Cierra los cuatro pendientes (ADR-009 `strength`/`moral`/`Equipment`,
  ADR-011 §7 tabla de zonas) con una sola decision coherente, sin tocar el
  manual ni reabrir ADR-009/011/012/013 (ADR-000 inmutabilidad).
- `strength` queda como escalar entero, monotono y determinista: directamente
  usable como desempate de ADR-012 §5.3 sin ajustar ADR-012.
- `moral(S)` provee el insumo que el cap. 09 / futuro ADR-017 necesitaba, en la
  escala de `VAL` (ADR-013) y **sin prejuzgar** triggers de Valor: separacion
  de responsabilidades limpia.
- El reparto de heridas por zona preserva exactamente la aritmetica de `W` y
  `W_kia=60` de ADR-011 §7 (cada fila suma `sev`): la zona da textura
  (que capacidad se pierde) sin alterar el umbral de muerte.
- El schema de Equipment no introduce ningun factor nuevo en `P_atk`/`P_def`:
  arma → `w_arma`/`r_eff` ya existentes (ADR-011 §2/§3), proteccion → aditivo a
  `TGH_ef` ya prescrito (ADR-011 §4), utilidad → hooks de reglas existentes.
  Reutiliza intacta la matematica aceptada.

**Negativas:**

- Nuevos numeros sin calibrar (pesos de `strength`, mezcla 0.5/0.5 de `moral`,
  P(zona) y fracciones de reparto, `tgh_bonus` de blindajes): ≈18 magnitudes
  "supuesto de balance, sujeto a playtest". La **estructura** (suma ponderada
  por nivel; combinacion convexa lider/tropa de `VAL_ef`; reparto que suma
  `sev`; slots mapeados a puntos existentes de ADR-011) es la decision estable;
  los numeros son tuneables sin reabrir este ADR.
- `moral(S)` depende de `VAL_ef`, cuya dinamica de triggers aun no esta cerrada
  (cap. 09 / ADR-017): hasta entonces `moral(S)` es computable pero su
  *variacion en partida* (y su mapeo a estados de escuadra) queda a ese grupo.
- La tirada de zona agrega una tirada mas por impacto (consume semilla del
  turno): impacto nulo en reproducibilidad (misma fuente determinista) pero un
  paso extra en el pipeline de Resolution.

## Alternativas descartadas

- **`strength` = suma cruda de `level` (ADR-009 literal)**: no expresa el
  enfasis del cap. 04 ("perder al L3 duele mas que perder L1"); la suma
  ponderada lo logra sin convertir `level` en poder de combate (lo que
  contradiria ADR-011/013).
- **`strength` ponderado por stats/equipo/heridas**: duplicaria lo que ADR-011
  ya resuelve por soldado y volveria a `strength` no monotono (un soldado
  herido bajaria `strength` ademas de bajar su potencia) — doble conteo y
  desempate inestable para ADR-012.
- **`moral(S)` = media simple de `VAL_ef`**: ignora el rol del lider en la
  cohesion (el cap. 08/09 hace de la cadena de mando un pilar); la combinacion
  convexa lider/tropa lo refleja sin sobreponderar al lider como en iniciativa.
- **Definir triggers/recuperacion de Valor aqui**: fuera de alcance; cap. 09 y
  el grupo de Valor (ADR-017) los poseen. Este ADR solo fija la **agregacion
  estructural**, igual que ADR-013 solo fijo el nombre `VAL`.
- **Reparto de heridas que cambia la carga `W` segun zona** (p. ej. torso
  pesa mas hacia `W_kia`): reabriria la aritmetica de umbral de ADR-011 §7. Se
  rechaza: la zona modela *que* se degrada, no *cuanto* se acerca a morir; cada
  fila suma `sev` para dejar ADR-011 intacto.
- **Proteccion como factor multiplicativo aparte en `P_def`**: ADR-011 §4 ya
  prescribe explicitamente aditivo a `TGH_ef`; un factor nuevo reabriria la
  formula de potencia. Aditivo reutiliza la mecanica suma+clamp de ADR-013.
- **Estados de herida discretos por zona (leve/grave)**: rechazado por cap. 07
  y ADR-011 (carga acumulada `W`, sin estados discretos); la zona solo elige
  sobre que stats cae el modificador continuo.
