# ADR-017: Degradación, recuperación y umbrales de Valor

## Estado

Aceptado — 2026-05-18

## Contexto

El Valor es el eje psicológico de Subordinación y Valor. El capítulo
[09 — Valor](../manual/09-valor.md) lo define como un **modificador pasivo**
individual por soldado que degrada todo desempeño, y establece que su agregado
por escuadra gobierna las transiciones de estado ACTIVE → RETREAT → ROUTED. El
mismo capítulo deja explícitamente pendientes cinco cosas:

- *"Triggers exactos y cantidades de degradación: Pendiente de definir."*
- *"Fórmula de ponderación [del agregado de escuadra]: Pendiente de definir."*
- *"Umbral RETREAT / Umbral ROUTED: Por definir."*
- *"Mecánica de recuperación: Pendiente de definir."*
- *"Velocidad de recuperación y condiciones exactas: Pendiente de definir."*

El capítulo [08 — Mando y Subordinación](../manual/08-mando-y-subordinacion.md)
deja a su vez pendiente *"condiciones exactas que disparan la tirada de Valor
para disband"* de una Tropa L1 que pierde todo mando superior.

Tres ADR aceptados ya tocan el borde de este sistema sin cerrarlo:

- [ADR-013](013-stats-canonicos-y-catalogo-de-modificadores.md) fija el nombre
  canónico `VAL` (stat #6, base infantería L1 = 50, escala 0–100), declara que
  `VAL` vive en el mismo sistema de **suma aditiva con clamp `[1,100]`** que el
  resto de stats (§1.2), modela su degradación por triggers como
  *"modificadores negativos transitorios sobre `VAL` (análogos a Heridas pero
  reversibles)"* y delega explícitamente *"la formalización de
  triggers/recuperación de Valor … para el Grupo de Valor"*.
- [ADR-012](012-calculo-de-iniciativa-orden-de-resolucion.md) ya consume el
  estado moral de la escuadra (ACTIVE/RETREAT/ROUTED) como factor `M` de la
  iniciativa (+10 / −10 / −25). Este ADR define **cuándo** una escuadra entra
  en cada estado; no toca `M`, que sigue siendo de ADR-012.
- [ADR-011](011-resolucion-de-combate-por-soldado.md) define HIT/KIA por
  soldado, el sistema de Heridas (`sev`, carga `W`, `W_kia = 60`) y la semilla
  determinista del turno. Este ADR reutiliza esos eventos como triggers de
  Valor; no modifica la matemática de combate.

Es el cierre del Grupo 5 (decisiones #16–#20): sin triggers, agregación,
umbrales, recuperación y condición de disband no se puede simular el pilar
psicológico ni cablear el factor `M` de ADR-012 con un estado real.

Restricciones de coherencia respetadas:

- ADR-001 (glosario): Tropa, Escuadra, L1–L5 **no** es poder de combate;
  el disband de L1 es la consecuencia de quedar sin mando.
- cap. 09: Valor pasivo, individual, agregado **ponderado** a escuadra; estados
  progresivos y **reversibles**; recuperación por tiempo seguro.
- cap. 08: el disband de una Tropa L1 sin superior vivo lo dispara una **tirada
  de Valor**; éxito = resiste, fallo = se quiebra.
- cap. 04: estados de escuadra distintos del sistema de Heridas individual;
  las Heridas son permanentes, el Valor es reversible.
- ADR-013: `VAL` ∈ `[1,100]`, base 50, suma+clamp; la degradación es un
  modificador negativo **reversible** sobre `VAL`. No se renombra nada.
- ADR-012: no se toca el factor `M`; este ADR sólo decide la transición de
  estado que `M` lee.

Este ADR **no modifica** el manual ni otros ADR; cap. 09 y cap. 08 lo
referenciarán cuando se actualicen. No reabre 010–016.

## Decisión

### 1. Modelo del Valor efectivo individual

Cada Tropa viva tiene un Valor efectivo `VAL_ef` calculado con la mecánica
canónica de ADR-013 §1.2 (suma aditiva + clamp), separando los modificadores
**permanentes** (plantilla de clase + Perks/Traits/Phobias + Heridas) de un
**componente moral transitorio** `Δmoral` (negativo o cero) que acumula la
degradación de este ADR y se reduce con la recuperación (§4):

```
VAL_ef(t) = clamp( base(VAL) + Σ mod_perm(VAL) + Δmoral(t) , 1, 100 )
```

- `Σ mod_perm(VAL)`: Perks/Traits/Phobias/Heridas que afectan `VAL`
  (catálogo ADR-013 §2: p. ej. `Aguante moral +12`, `Pesimista −6`,
  `Hemofobia −12`, `Pánico al aislamiento −15`). Estos son **permanentes** y
  no son alcance de este ADR salvo como insumo del valor base psicológico.
- `Δmoral(t)`: acumulador transitorio, **`≤ 0`**, inicia en `0`, se hace más
  negativo con los triggers de §2 y vuelve hacia `0` con la recuperación de §4.
  Es exactamente el *"modificador negativo transitorio reversible"* que
  ADR-013 §1.2 previó. No baja de `−99` (el clamp final lo acota igual).

El clamp `[1,100]` es el de ADR-013: ningún soldado llega a `VAL_ef = 0`.

> Nota de coherencia: la degradación por trauma se modela sobre `Δmoral`, no
> sobre el modificador permanente; las Phobias/Traits sí mueven el **piso**
> psicológico permanente (vía `Σ mod_perm`). Una Tropa con `Pesimista` o
> `Pánico al aislamiento` entra a cada turno con menos colchón de Valor: el
> catálogo de ADR-013 y este ADR componen aditivamente, sin reglas ad-hoc.

### 2. Triggers de degradación de `Δmoral` (individual)

La degradación se evalúa **al final de la fase Resolution** (ADR-004), después
del combate de ADR-011 y antes de la evaluación de mutación de ADR-013 §3.2.
El orden canónico del turno queda:

```
combate (ADR-011) → degradación/recuperación de Valor (este ADR §2–§4)
  → re-evaluación de estado de escuadra (§5) → disband L1 (§6)
  → evaluación de mutación (ADR-013 §3)
```

Para cada Tropa viva se suma a `Δmoral` la contribución (negativa) de cada
trigger ocurrido en ese turno. Los efectos **se acumulan** dentro del turno
(a diferencia de la mutación de ADR-013, que toma sólo el de mayor peso).

| # | Trigger (en el turno, sobre la Tropa viva) | Δ a `Δmoral` | Fuente |
|---|--------------------------------------------|--------------|--------|
| T1 | La Tropa recibió ≥1 HIT propio (resultó herida) | **−6** por turno con HIT (no por cada HIT) | cap. 09 "heridas sufridas"; ADR-011 HIT |
| T2 | La Tropa cruzó un umbral de Herida grave (carga `W` acumulada ≥ 30, mitad de `W_kia`=60) por primera vez | **−8** una sola vez (latch) | cap. 09; ADR-011 §7 `W`, `W_kia=60` |
| T3 | Un compañero de su escuadra resultó KIA este turno | **−4** por compañero KIA, máx **−16**/turno | cap. 09 "pérdida de compañeros"; ADR-011 KIA |
| T4 | El líder operativo de la escuadra (L3 estándar / L2 especial) resultó KIA este turno | **−10** adicional (a todas las tropas vivas de la escuadra) | cap. 08 "romper un eslabón"; cap. 09 |
| T5 | La escuadra terminó el turno fuera de mando (`in_command=false`, BFS 5 hex, cap. 08) | **−5** por turno aislada | cap. 09 "pérdida de contacto"; cap. 08 |
| T6 | La escuadra terminó el turno sin radio por L3 muerto (`has_radio=false` por pérdida del líder, cap. 08) | **−5** adicional a T5 (aislamiento severo) | cap. 08; cap. 09 |
| T7 | La Tropa fue atacada con modificador de flanqueo o quedó rodeada (≥2 enemigos adyacentes atacándola), ADR-011 §3 | **−4** por turno | cap. 09 "presencia enemiga / flanqueado / rodeado" |
| T8 | La Tropa recibió fuego de un arma de alto volumen (`w_arma ≥ 1.6`, MG, ADR-011 §4) sin ser KIA | **−3** por turno | cap. 09 "bajo fuego" |

Notas:

- **Excepción de diseño escuadra especial L2** (Los Infernales, commandos):
  T5 y T6 **no aplican** — operan sin radio por diseño y con autonomía
  entrenada, coherente con la excepción equivalente de ADR-012 §2.3
  (`C = +5` en vez de aislada). Sí sufren T1–T4, T7, T8.
- Los modificadores permanentes de ADR-013 **amplifican el dolor sin reglas
  nuevas**: la Phobia `Hemofobia` (`VAL −12`) ya hunde el piso de un soldado
  que ve morir a un compañero; `Pánico al aislamiento` (`VAL −15`) lo hace
  con T5. Este ADR no duplica ese efecto: `Σ mod_perm` y los triggers son
  términos aditivos independientes (ADR-013 §1.2).
- Magnitudes T1–T8: **supuesto de balance, sujeto a playtest**. La estructura
  (acumulación aditiva sobre `Δmoral`, evaluación post-combate, excepción L2)
  es la decisión estable.

### 3. Agregación de Valor de escuadra — promedio **ponderado por nivel**

El Valor de escuadra `VAL_sq` es un **promedio ponderado por el `level`
(L1–L5) de cada tropa viva**, no un promedio simple:

```
VAL_sq = Σ ( level(t) · VAL_ef(t) )  /  Σ level(t)      para t en tropas vivas
```

Racional (resuelve la pregunta abierta de cap. 09 "puede que el líder cuente
más, o promedio simple"):

- cap. 04/ADR-009 ya establecen que **perder niveles altos pesa más** que
  perder L1 (mismo principio que `strength = Σ level`). La cohesión moral de
  una escuadra depende desproporcionadamente de que el líder L3 y los L2
  mantengan el temple: si el Sargento se quiebra, la escuadra se quiebra. Un
  promedio simple ignoraría esto y haría a la escuadra inmune al colapso de
  su mando mientras la masa de L1 aguante.
- Reutiliza la **misma noción de peso por nivel** que ADR-009 §`strength`,
  sin introducir un esquema de ponderación nuevo y ad-hoc — coherencia con
  ADR-001 (L1–L5 modela autoridad/cohesión, no poder de combate: aquí pesa
  por cohesión, no por daño).
- El líder no recibe un peso arbitrario "×N"; recibe su peso natural `level`
  (un L3 pesa 3× un L1, un L2 pesa 2×). Determinista y sin parámetro extra.

`VAL_sq` se recalcula al final de cada turno (orden de §2) sobre las tropas
**vivas**. Si todas las tropas están muertas, la escuadra es `ELIMINATED`
(ADR-009, cap. 04) y `VAL_sq` no se evalúa.

> Coherencia ADR-009: el modelo de datos lista `moral = aggregate(tropa.moral)`
> como *"pendiente"*. Este ADR fija ese agregado como el promedio ponderado de
> arriba, sin tocar el archivo de ADR-009 (cap. 04 / ADR-009 lo referenciarán
> al actualizarse).

### 4. Recuperación de `Δmoral` (individual)

`Δmoral` regresa hacia `0` (recuperación de moral) sólo en turnos en que la
Tropa cumple **todas** las condiciones de "turno seguro":

- No recibió ningún HIT este turno (no se activó T1/T2).
- Ningún compañero de su escuadra resultó KIA este turno (no T3/T4).
- La escuadra no recibió fuego enemigo este turno (no T7/T8).

Si se cumple turno seguro, se aplica:

```
Δmoral ← min( 0 , Δmoral + REC )
```

con tasa de recuperación `REC` dependiente del contexto (el mejor caso
aplicable, no acumulativo):

| Condición del turno seguro | `REC` (puntos hacia 0 / turno) |
|----------------------------|--------------------------------|
| Turno seguro **y** `in_command = true` (recibe contención de mando) | **+8** |
| Turno seguro pero fuera de mando (`in_command = false`) | **+4** |
| Escuadra especial L2 en turno seguro (autonomía entrenada, sin penalizar aislamiento) | **+6** |

Racional: la recuperación con mando es el doble que aislada — refuerza el
pilar de subordinación (la presencia de la cadena de mando estabiliza la
moral), coherente con cap. 09 *"posiblemente dentro del radio de mando
(mejora pero tal vez no obligatorio)"*: aquí **mejora, no es obligatorio**.
La recuperación es **más lenta que la peor degradación de un turno** (un mal
turno puede costar ≫ 8): recuperarse toma varios turnos seguros, como exige
cap. 09 (*"el Valor aumenta lentamente durante estos turnos seguros"*).
`REC` es **supuesto de balance, sujeto a playtest**.

Una Herida nunca se "cura" (cap. 04, ADR-011 §7 permanente): si la pérdida de
`VAL` venía de `Σ mod_perm` (Herida/Phobia), esa parte **no** se recupera;
sólo `Δmoral` (el trauma transitorio) vuelve hacia `0`. Esto es exactamente
la separación que ADR-013 §1.2 pidió.

### 5. Umbrales de estado de escuadra y transición

El estado se deriva de `VAL_sq` (§3) al final de cada turno, con
**histéresis** para evitar parpadeo de estado entre turnos:

| Umbral | Valor `VAL_sq` |
|--------|----------------|
| Entra a **RETREAT** (desde ACTIVE) | `VAL_sq` cae a **≤ 40** |
| Entra a **ROUTED** (desde RETREAT) | `VAL_sq` cae a **≤ 20** |
| Recupera a **RETREAT** (desde ROUTED) | `VAL_sq` sube a **≥ 30** |
| Recupera a **ACTIVE** (desde RETREAT) | `VAL_sq` sube a **≥ 50** |

- Banda de histéresis de **10 puntos** en cada transición (entra ROUTED a 20,
  sale a 30; entra RETREAT a 40, sale a 50): una escuadra que oscila alrededor
  del umbral no alterna de estado cada turno. Coherente con cap. 09
  ("transiciones progresivas y reversibles") y cap. 04 (tabla de estados:
  ACTIVE = Valor > umbral RETREAT; RETREAT = ≤ RETREAT y > ROUTED;
  ROUTED = ≤ ROUTED).
- La transición es **de a un escalón por turno** (ACTIVE↔RETREAT↔ROUTED): no
  se salta de ACTIVE directo a ROUTED en un solo turno aunque `VAL_sq` caiga
  por debajo de 20 — el turno siguiente, si sigue ≤20, pasa de RETREAT a
  ROUTED. Esto preserva la progresividad que exige cap. 09 ("de forma
  progresiva") y da una ventana de un turno para reaccionar.
- `ELIMINATED` no usa umbral: es todas las tropas muertas (cap. 04, ADR-009),
  tiene prioridad sobre cualquier estado moral.
- El estado resultante alimenta el factor `M` de ADR-012 §2.4
  (ACTIVE +10 / RETREAT −10 / ROUTED −25) sin que este ADR lo modifique.
- Umbrales 40/20 y bandas de histéresis: **supuesto de balance, sujeto a
  playtest**. La estructura (dos umbrales con histéresis, un escalón por
  turno, derivado de `VAL_sq` ponderado) es la decisión estable.

Efectos de estado (reafirma cap. 04 / cap. 09, no los cambia): ACTIVE = todas
las órdenes; RETREAT = sólo MOVE a retaguardia + DEFEND, ATTACK prohibido;
ROUTED = ignora órdenes, repliegue automático, no combate.

### 6. Condición de la tirada de Valor para disband (cap. 08)

cap. 08 establece que una Tropa **L1** sin **ningún** superior L2+ vivo en su
escuadra **puede** desbandar, y que lo dispara una **tirada de Valor**
(éxito = resiste, fallo = se quiebra). Se fija la condición y la tirada:

**Cuándo se dispara** (evaluado en el orden de §2, paso "disband L1", después
de recalcular estado de escuadra):

Una Tropa `t` tira disband este turno si y sólo si **todas**:

1. `t.level == 1` (sólo L1; la Tropa de Comunicaciones L1 también aplica).
2. **No** queda viva en su escuadra ninguna tropa de `level ≥ 2` (sin L3 ni
   ningún L2 de grupo: la escuadra perdió toda la cadena interna). Coherente
   con cap. 08 ("sin ningún superior L2 o superior vivo en su escuadra").
3. La condición 2 se cumplió **al cierre de este turno** (la tirada se hace
   **post-Resolution**, una vez por turno por cada L1 afectado, mientras la
   condición persista — no es de una sola vez: cada turno que siga sin mando
   interno se vuelve a tirar, modelando el desgaste de cap. 08).

Escuadra especial L2: su líder es L2; mientras ese L2 viva, sus L1 **no**
tiran disband (hay superior vivo). Si el L2 líder muere, sus L1 quedan sin
superior y aplican 1–3. Esto es coherente con la tabla de cap. 08 ("L2
especial: riesgo de disband sólo si la orden genérica ya no puede
ejecutarse") leído como: el riesgo de la escuadra especial se materializa
cuando pierde a su L2 (entonces es el caso L1 estándar).

**La tirada** (determinista por la semilla del turno, ADR-011/012/013):

```
roll = uniform(0, 100)        # semilla del servidor del turno
resiste  si  roll < VAL_ef(t)         # éxito: la Tropa permanece
se quiebra si roll ≥ VAL_ef(t)        # fallo: la Tropa desbanda
```

- Probabilidad de resistir = `VAL_ef(t) / 100`. Un soldado con Valor alto
  casi siempre aguanta; uno ya degradado por trauma/Phobia (`VAL_ef` bajo)
  se quiebra con alta probabilidad. El Valor **no se gasta** en la tirada
  (cap. 09: "no es una tirada de salvación ni un recurso que se agota") —
  modula la probabilidad, no se consume; `Δmoral` no cambia por tirar.
- Una Tropa que se quiebra deja de funcionar como unidad coordinada (cap. 08):
  se marca desbandada y no contribuye a `strength` ni a `VAL_sq` los turnos
  siguientes (cuenta como baja de cohesión, no como KIA).
- **Cuando todas las tropas L1 de la escuadra están desbandadas**, la escuadra
  pasa a `ROUTED` o `ELIMINATED` según severidad (cap. 08): `ELIMINATED` si
  además no queda ninguna tropa viva operativa; `ROUTED` si quedan tropas
  vivas pero todas desbandadas (pérdida total de cohesión, no de vidas).
- El desband de un L1 cuenta como "pérdida de compañero" para los efectos de
  T3 sobre el resto **sólo si** la tropa además muere; un desband sin muerte
  no dispara T3 (es abandono de cohesión, no una baja).

### 7. Parámetros tuneables (balance)

Todos **supuesto de balance, sujeto a playtest**; la **estructura** es estable.

| Parámetro | Valor inicial | Efecto al subir |
|-----------|---------------|-----------------|
| Magnitudes triggers T1–T8 | −6/−8/−4(máx−16)/−10/−5/−5/−4/−3 | Cuán rápido colapsa la moral bajo presión |
| Ponderación de agregado de escuadra | por `level` (Σ level·VAL / Σ level) | Cuánto pesa el mando en la cohesión |
| `REC` (en mando / aislado / especial L2) | +8 / +4 / +6 | Cuán rápido se recupera una escuadra en seguridad |
| Umbral RETREAT / ROUTED | 40 / 20 | Cuán frágil es la escuadra ante la presión moral |
| Histéresis de salida (RETREAT / ROUTED) | 50 / 30 | Resistencia al parpadeo de estado |
| Escalones por turno | 1 (no se saltan estados) | Progresividad de la degradación |
| Tirada de disband | `uniform(0,100) < VAL_ef` | Probabilidad de que un L1 sin mando resista |

## Consecuencias

**Positivas:**

- Cierra el Grupo 5 (#16–#20): triggers+magnitudes, agregación, umbrales,
  recuperación y condición de disband, sin tocar el manual ni reabrir ADR.
- **No contradice** ADR-011/012/013: reutiliza la suma+clamp y el `VAL`
  canónico de ADR-013 §1.2 (degradación como `Δmoral` reversible — exactamente
  lo que ADR-013 previó), consume eventos HIT/KIA/`W` de ADR-011 sin cambiar
  su matemática, y alimenta el factor `M` de ADR-012 §2.4 sin modificarlo.
- El agregado ponderado por `level` reutiliza el mismo principio que
  `strength` de ADR-009: perder al líder hunde la cohesión, refuerza el pilar
  de subordinación (ADR-001) sin un esquema de pesos nuevo.
- La recuperación con mando al doble que aislada y el disband modulado por
  `VAL_ef` hacen del aislamiento un costo psicológico medible, no sólo de
  órdenes: coherente con la tesis del juego (cap. 08/09).
- Histéresis + un escalón/turno: estado de escuadra estable y progresivo,
  reproducible dado estado+órdenes+semilla (compatible con la verificación de
  coherencia del Briefing, ADR-004).

**Negativas:**

- ~18 magnitudes + 4 umbrales sin calibrar: el balance real exige telemetría
  de playtest. La **estructura** (Δmoral reversible separado de mod_perm,
  agregado ponderado por level, dos umbrales con histéresis, disband como
  tirada `uniform < VAL_ef`) es la decisión estable; los números se ajustan
  sin reabrir este ADR.
- Introduce un acumulador de estado por soldado (`Δmoral`) que persiste entre
  turnos y un latch por (soldado, T2). Es estado adicional en `shared/`
  (ADR-008/009), aceptable: ya existe estado persistente por soldado (Heridas,
  contadores de mutación de ADR-013 §3.1).
- La separación `Δmoral` (recuperable) vs `Σ mod_perm` (permanente) obliga al
  modelo de datos a distinguir ambos componentes de `VAL`; ADR-013 §1.2 ya
  anticipó esta distinción, por lo que no rompe nada.

## Alternativas descartadas

- **Promedio simple del Valor de escuadra**: ignora que perder al líder L3
  debe hundir la cohesión más que perder un L1. Contradice el principio de
  ADR-009 (`strength` pondera por level) y el pilar de subordinación. El
  ponderado por `level` no añade parámetros nuevos.
- **Ponderar al líder con un factor arbitrario "×N"**: introduce un parámetro
  ad-hoc sin anclaje. El `level` (L3=3×, L2=2×, L1=1×) ya es la escala de
  cohesión natural del juego.
- **Valor como recurso que se gasta en la tirada de disband**: explícitamente
  rechazado por cap. 09 ("no es una tirada de salvación ni un recurso que se
  agota"). Se modela como probabilidad `VAL_ef/100` sin consumir el stat.
- **Transición directa ACTIVE→ROUTED en un turno**: contradice cap. 09 ("de
  forma progresiva"). Se fija un escalón por turno con histéresis.
- **Sin histéresis (un único umbral por transición)**: produce parpadeo de
  estado turno a turno cuando `VAL_sq` oscila cerca del umbral, lo que rompe
  la lectura táctica y el factor `M` de ADR-012 oscilaría con él.
- **Degradar el modificador permanente de `VAL` en vez de un `Δmoral`
  aparte**: haría irreversible el trauma (las Heridas no se curan, ADR-011 §7)
  y contradiría cap. 09 ("reversible … recuperación de moral"). ADR-013 §1.2
  ya pidió modelar la degradación como modificador **reversible** separado.
- **Disparar disband cada turno fuera de mando aunque haya un L2 vivo**:
  contradice cap. 08, que exige la pérdida de **todo** superior L2+ vivo en
  la escuadra. El aislamiento por radio degrada `VAL` (T5/T6) pero no fuerza
  la tirada de disband mientras quede mando interno.
