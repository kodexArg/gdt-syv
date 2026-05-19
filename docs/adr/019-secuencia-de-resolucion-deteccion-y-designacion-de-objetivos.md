# ADR-019: Secuencia de resolución, detección en movimiento y designación de objetivos

## Estado

Aceptado — 2026-05-18

## Contexto

El capítulo [07 — Combate](../manual/07-combate.md) define el modelo de combate
estocástico por soldado y su inserción en una línea de tiempo, pero deja tres
huecos marcados como "Pendiente de definir" que componen el Grupo 3 restante:

- **#10 — Secuencia interna de resolución de un evento.** El cap. 07 (sección
  "Secuencia de resolución") enumera, en comentario, qué debe resolverse
  —movimiento, combate, post-procesado— sin fijar el orden ni cómo encaja con
  el orden de iniciativa de [ADR-012](012-calculo-de-iniciativa-orden-de-resolucion.md).
- **#11 — Detección y reacción durante el movimiento.** El cap. 07 (sección
  "Encuentros durante el movimiento") fija la **política** —MOVE se detiene al
  detectar enemigo, ATTACK continúa— pero deja "Distancia de detección y
  mecanismos de reacción: Pendiente de definir".
- **#12 — Designación de objetivos por soldado.** El cap. 07 (sección
  "Designación de objetivos por soldado") establece que el fuego se designa por
  soldado (L1) y que un ATTACK puede repartir blancos, pero deja "la mecánica
  exacta de la interfaz de objetivos y la lógica de distribución automática:
  Pendiente de definir".

Sin estos tres cierres el servidor autoritativo (ADR-003/004) no puede ejecutar
la fase de Resolution: tiene el **orden** de las escuadras (ADR-012) y la
**matemática** de un intercambio (ADR-011), pero no la **secuencia** que
encadena movimiento→encuentro→combate→post dentro de cada acción, ni la regla
que interrumpe un MOVE, ni cómo se asignan los blancos por soldado antes de
invocar ADR-011.

Restricciones de coherencia respetadas (no se reabren):

- [ADR-001](001-nomenclatura-y-glosario.md): terminología — Escuadra como
  unidad leaf que ocupa un hex; órdenes ATTACK/MOVE/DEFEND; soldado = L1.
- [ADR-011](011-resolucion-de-combate-por-soldado.md): la matemática de **un
  intercambio ya ordenado** (P_atk/P_def, Δ, p_hit con clamp `[0.02, 0.85]`,
  severidad `sev`, carga de heridas `W`, umbral `W_kia = 60`, §7-bis Sanitario).
  Este ADR **no** recalcula nada de eso: solo decide **a quién** dispara cada
  soldado y **cuándo** se invoca el procedimiento §6 de ADR-011.
- [ADR-012](012-calculo-de-iniciativa-orden-de-resolucion.md): la línea de
  tiempo se ordena **por escuadra** vía `INIT`; `t(S)` es el instante de
  **inicio** de la acción de la escuadra; un defensor ya KIA por una acción
  previa deja de ser blanco válido. Este ADR define qué ocurre **a partir** de
  `t(S)`, sin tocar `INIT` ni la interpolación a la ventana de 240 min.
- [ADR-016](016-terreno-elevacion-y-puntos-de-movimiento.md): `MP` por tipo de
  escuadra (inf 4 / otros 5 / cab 8), `move_cost` por hex de destino, paths de
  hasta 3 waypoints validados **antes** de ejecutar, `cover_class`/`vision_block`
  por terreno, `E_MIN = 5.0 m`, `m_fort`. Este ADR consume esos datos; no
  redefine pools, costos ni el predicado de elevación.
- Escala 100 m/hex y tabla de distancias del cap.
  [02b](../manual/02b-distancias.md): fusilería efectiva 2–3 hex, MG/mortero
  6–10 hex.

Este ADR **no modifica** el manual ni otros ADRs; el cap. 07 lo referenciará
cuando se actualice.

## Decisión

### 1. La acción de una escuadra como secuencia de cuatro etapas

A partir de su instante de inicio `t(S)` (ADR-012 §4), cada escuadra ejecuta su
orden como una **secuencia interna de cuatro etapas en orden fijo**. La línea de
tiempo global sigue ordenada por escuadra (ADR-012): este ADR define qué pasa
**dentro** del bloque de tiempo de una escuadra una vez que le toca actuar.

```
Para la escuadra S, en su turno de la línea de tiempo (ADR-012):
  Etapa 1 — MOVIMIENTO
  Etapa 2 — ENCUENTRO (detección + decisión de interrupción)
  Etapa 3 — COMBATE (designación de objetivos + ADR-011 §6 por soldado)
  Etapa 4 — POST-PROCESADO (estado, Valor, regla 5 hexes, disband)
```

**Por qué este orden y no "todo el movimiento global primero".** ADR-012 ya
serializó las escuadras por iniciativa: no hay una "fase de movimiento global
simultánea". Cada escuadra mueve **dentro de su propio bloque**, en el orden de
la línea de tiempo. Esto preserva la consecuencia táctica central del cap. 07 y
de ADR-012 §6: una escuadra de alta iniciativa **mueve y dispara antes** de que
una escuadra lenta empiece a moverse, y puede abatir a quien aún no se ha
desplazado. Resolver todo el movimiento de todas las escuadras antes que todo el
combate destruiría esa propiedad (la iniciativa dejaría de decidir quién dispara
primero) y contradiría ADR-012 §6 y el cap. 07 ("una unidad con alta iniciativa
puede abatir a un enemigo antes de que ese enemigo llegue a disparar").

**Encadenamiento con ADR-012.** El instante `t(S)` marca el inicio de la
Etapa 1. Las cuatro etapas consumen tiempo simulado a partir de `t(S)`; los
eventos emitidos (`MOVE`, `CONTACT`, `HIT`/`MISS`/`KIA`, cambios de estado)
llevan su timestamp dentro de la ventana de 240 min. Una escuadra que arranca
más tarde (menor `INIT`) encuentra el campo ya modificado por las etapas 1–4 de
las escuadras que arrancaron antes — exactamente la lectura de ADR-012 §6.

#### Etapa 1 — Movimiento

- Solo se ejecuta si la orden es MOVE o ATTACK (DEFEND no mueve; cap. 06).
- El path (hasta 3 waypoints) ya fue validado en el Briefing contra `MP` y
  `move_cost` (ADR-016 §4). El path es válido en su totalidad **al planificarse**;
  esta etapa lo **recorre hex a hex**, en orden, gastando del pool `MP` el
  `move_cost` de cada hex al que se entra (ADR-016 §4).
- El recorrido es **incremental y observable**: tras entrar a cada hex se
  evalúa la Etapa 2 (Encuentro) **antes** de consumir el siguiente hex. Esto
  hace posible que un MOVE se interrumpa a mitad de path.
- Si la orden es DEFEND, la Etapa 1 es nula (la escuadra permanece en su hex) y
  se pasa directo a la Etapa 2 (puede detectar y reaccionar a enemigos que
  entren en su alcance).

#### Etapa 2 — Encuentro (detección y decisión de interrupción)

Tras cada hex entrado en la Etapa 1 (o, para DEFEND, una sola vez en el hex
propio), la escuadra evalúa detección (§2) y decide si interrumpe su movimiento
según la **política del cap. 07** (no se reabre):

- **MOVE:** si detecta al menos un enemigo, **se detiene en el hex actual**
  (cesa el recorrido del path; no consume más `MP`). Pasa a Etapa 3 con postura
  defensiva (§3.4).
- **ATTACK:** **continúa el path** aunque detecte enemigos; los encuentros se
  acumulan como blancos potenciales para la Etapa 3, pero el avance no se
  interrumpe hasta agotar el path o alcanzar el hex objetivo.
- **DEFEND:** no hay path; si detecta enemigos pasa a Etapa 3 (responde al
  fuego desde su posición fortificada).

Si no hay detección, la escuadra que aún tiene path pendiente vuelve a Etapa 1
(siguiente hex). Si el path se agotó sin detección, pasa a Etapa 4.

#### Etapa 3 — Combate

Si en el hex final (o de detención) hay enemigos detectados y en alcance, la
escuadra resuelve fuego: se designan blancos por soldado (§3) y se invoca el
**procedimiento §6 de ADR-011 por cada soldado atacante** (no se reabre la
matemática). El orden de evaluación de los soldados dentro de la escuadra es por
`squad_id`/`troop_id` ascendente (determinista, reproducible para la
verificación de coherencia del Briefing, ADR-004); como cada soldado tira
independientemente (ADR-011 §6) ese orden solo afecta qué blancos siguen vivos
para los disparos posteriores **del mismo evento**, coherente con ADR-011 §6
paso 6 y ADR-012 (un blanco ya KIA deja de ser válido).

#### Etapa 4 — Post-procesado

Una vez resueltos todos los disparos de la escuadra en este evento, se aplican,
en este orden:

1. **Bajas y heridas:** ya escritas por ADR-011 §6/§7 durante la Etapa 3
   (`alive`, carga `W`, modificadores de herida). Aquí solo se consolida el
   estado resultante de la escuadra.
2. **Estado de escuadra:** recálculo de ACTIVE/RETREAT/ROUTED y de
   `strength` agregado (cap. 04/09). El nuevo estado afecta a escuadras que
   actúen **más tarde** en la línea de tiempo (vía `f_estado` de ADR-011 y `M`
   de ADR-012 ya fue fijado al ordenar; el estado intra-turno se propaga al
   combate, no re-ordena la línea de tiempo — ADR-012 no se reabre).
3. **Valor:** ajustes de moral/Valor derivados del resultado (cap. 09),
   evaluados con el estado consolidado.
4. **Regla de 5 hexes y aislamiento:** reevaluación de `in_command` por el BFS
   de 5 hexes desde el HQ y de `has_radio` (cap. 08/12) con las posiciones ya
   actualizadas por la Etapa 1. El resultado condiciona **turnos futuros** (su
   `C` de ADR-012 se recalculará en el próximo ordenamiento), no re-ordena el
   turno actual.
5. **Disband:** si la escuadra cae por debajo del umbral de disband (cap. 04),
   se marca ELIMINATED y deja de ser blanco/actor para eventos posteriores de
   esta misma línea de tiempo.

El post-procesado de una escuadra es **local a su bloque**: no se difiere a un
"barrido global" al final del turno. Esto mantiene la propiedad de ADR-012 §6
(las escuadras lentas ven el campo ya alterado).

### 2. Detección y reacción durante el movimiento

#### 2.1 Radio de detección

La detección se mide en hexes (100 m/hex, cap. 02b). Una escuadra `S` detecta a
una escuadra enemiga `E` si se cumplen, en este orden, las dos condiciones:

```
distancia_hex(S, E) <= R_det(S)        # dentro del radio de detección
∧  hay_linea_de_vista(S, E)            # LOS no cortada (cap. 10 / ADR-016 vision_block)
```

`R_det(S)` es el **radio de detección efectivo** de la escuadra, derivado del
mayor alcance efectivo `r_eff` de las armas de sus tropas vivas (tabla ADR-011
§3) más un margen de **vigilancia visual** que no depende del arma:

```
R_det(S) = max( R_VIS , max_{t ∈ tropas_vivas(S)} r_eff(arma(t)) )
```

| Símbolo | Valor inicial | Rol |
|---------|---------------|-----|
| `R_VIS` | `5` hex (≈ 500 m) | Vigilancia visual a ojo desnudo en terreno abierto, independiente del arma. Coherente con cap. 02b ("visibilidad… en terreno abierto") y con el límite de 4–5 hex de la cadena táctica. |
| `r_eff` por arma | tabla ADR-011 §3 | Fusil 3, fusil de asalto 3, MG 6, pistola 1. Una escuadra con MG "ve para disparar" hasta 6 hex. |

Lectura: una escuadra de fusileros detecta a ~5 hex (domina `R_VIS`); una con
MG pesada detecta a ~6 hex (domina `r_eff` de la MG). Nunca por debajo de
`R_VIS` mientras tenga al menos una tropa viva con visión (no hay "ceguera por
arma corta").

**LOS y terreno (no se reabre).** `hay_linea_de_vista` es responsabilidad del
sistema de visión (cap. 10, motor pendiente); este ADR consume su resultado
booleano. El dato de terreno que lo alimenta es `vision_block` por hex y la
regla de elevación `E_MIN` de ADR-016 §1–§2; aquí solo se referencia. Mientras
el prototipo tenga todos los hexes a `elevation = 0.0` y solo `GRASS`/`WATER`
(`vision_block = 0.0`), `hay_linea_de_vista` es verdadera dentro de `R_det`
(sin obstáculos), y la detección queda gobernada por el radio.

#### 2.2 Reacción: cómo se interrumpe un MOVE

La detección se evalúa en la Etapa 2, **tras entrar a cada hex** del path
(§1, Etapa 1). El mecanismo de reacción es discreto por hex (no continuo):

- La escuadra entra al hex `i` (gasta `move_cost(i)` del pool, ADR-016 §4).
- Se evalúa detección desde el hex `i` contra todas las escuadras enemigas.
- **MOVE + detección:** el recorrido del path **se detiene en el hex `i`**. La
  escuadra no entra al hex `i+1`; el `MP` restante se descarta (no se acumula,
  ADR-016 §3). Se emite evento `CONTACT` con timestamp y se pasa a Etapa 3 en
  postura defensiva (§3.4). Esto materializa la "cautela" del cap. 07: MOVE
  evita la emboscada accidental deteniéndose al primer contacto.
- **ATTACK + detección:** se emite `CONTACT` pero el path **no** se interrumpe;
  el enemigo detectado se registra como candidato a blanco (§3) y la escuadra
  sigue avanzando hasta agotar el path o llegar al hex objetivo, donde se
  resuelve la Etapa 3. Esto materializa el "compromiso" del cap. 07.
- Si no hay detección tras el hex `i`, el recorrido continúa al hex `i+1`
  (Etapa 1) hasta agotar el path.

**Resolución de quién detecta primero entre dos escuadras que se acercan.** Si
dos escuadras enemigas tienen `MOVE`/`ATTACK` y sus paths las acercan, la que
**actúa antes en la línea de tiempo** (mayor `INIT`, ADR-012) ejecuta su
Etapa 1–2 primero y, por tanto, detecta y reacciona primero. No se introduce
una tirada de detección extra: el orden ya lo fija ADR-012 (no se reabre). Esta
es la palanca por la que la iniciativa "gana la emboscada".

#### 2.3 Reacción sin orden / aislada

Una escuadra aislada o sin orden nueva (cap. 08/12; ADR-012 §2.3) ejecuta su
**última orden conocida** con el mismo mecanismo de detección, pero su `INIT`
ya viene penalizado por ADR-012 (`C` negativo), de modo que tiende a detectar y
reaccionar **más tarde** que las escuadras coordinadas. No se añade aquí ninguna
penalización extra de detección: el costo del aislamiento ya está expresado en
la línea de tiempo (ADR-012, no se reabre). `R_det` no se degrada por
aislamiento — una escuadra aislada ve igual de lejos, solo reacciona más tarde.

### 3. Designación de objetivos por soldado

#### 3.1 Principio

El fuego se designa **por soldado (L1)** (cap. 07; ADR-011 resuelve cada
intercambio atacante↔defensor individualmente). Una orden ATTACK de escuadra
puede dirigirse a uno o varios hexes objetivo; la asignación final
soldado→blanco la produce una **regla de auto-distribución de fuego**
determinista, con posibilidad de **override manual** por el jugador.

#### 3.2 Conjunto de blancos válidos

Para un soldado atacante `a` de la escuadra `S`, el conjunto de blancos válidos
`B(a)` son las **tropas enemigas vivas** que cumplen:

```
b ∈ B(a)  ⟺  b.alive
            ∧ distancia_hex(hex(a), hex(b)) <= r_eff(arma(a))   # ADR-011 §3
            ∧ hay_linea_de_vista(S, hex(b))                       # cap. 10 (infantería)
            ∧ hex(b) ∈ hexes_objetivo_de_la_orden(S)              # ver §3.3
```

- El alcance es el `r_eff` del arma del **soldado** (ADR-011 §3), no el
  `R_det` de la escuadra: detectar (ver) y poder batir con eficacia son cosas
  distintas — un fusilero detecta a 5 hex pero su `f_dist` a 5 hex es ≈0.17
  (ADR-011 §3); se le permite disparar pero ADR-011 ya castiga el tiro largo.
  No se redefine `f_dist`.
- **Fuego indirecto (mortero/artillería):** excepción del cap. 07 — **no**
  requiere LOS y apunta a **coordenadas de hex**, no a tropa visible. Para esas
  armas `B(a)` se sustituye por el hex objetivo designado en la orden; la
  resolución de daño al hex sigue la regla de fuego indirecto del cap. 07 (no
  cubierta por ADR-011 §2, que es fuego directo). Este ADR solo fija que el
  blanco indirecto es un **hex**, no una tropa.

#### 3.3 Hexes objetivo de la orden

- **ATTACK** especifica uno o más **hexes objetivo** (cap. 07: "puede
  especificar múltiples hexes objetivo"). `hexes_objetivo_de_la_orden(S)` es
  ese conjunto.
- **MOVE interrumpido** y **DEFEND**: no hay hexes objetivo explícitos;
  `hexes_objetivo_de_la_orden(S)` = todos los hexes con enemigos detectados en
  alcance (fuego defensivo reactivo, postura del §3.4).

#### 3.4 Regla de auto-distribución de fuego (por defecto, determinista)

Cuando el jugador no asigna blancos manualmente, el servidor distribuye el fuego
de los soldados de `S` con esta regla, **determinista y reproducible** (ADR-004):

1. Ordenar los soldados atacantes vivos de `S` por `troop_id` ascendente
   (orden estable; coincide con el de la Etapa 3).
2. Para cada soldado `a` en ese orden, calcular `B(a)` (§3.2). Si `B(a) = ∅`,
   `a` no dispara este evento (sin blanco válido).
3. Elegir el blanco `b* ∈ B(a)` que **maximiza la prioridad**:

   ```
   prioridad(b) =  ( ¬cubierta_ya_letalmente(b) ,   # 1) repartir, no sobrematar
                     -distancia_hex(hex(a), hex(b)) , # 2) preferir el más cercano (mejor f_dist, ADR-011 §3)
                     -b.troop_id )                    # 3) desempate determinista
   ```

   comparación lexicográfica (primero el criterio 1, luego 2, luego 3).
   `cubierta_ya_letalmente(b)` es verdadera si a `b` ya se le asignaron
   suficientes atacantes en este evento como para que su probabilidad acumulada
   de baja supere `P_SAT = 0.80` (estimada con el `p_hit` de ADR-011 §5 de los
   atacantes ya asignados a `b`; no se recalcula la fórmula, solo se acumula
   `1 − Π(1 − p_hit_i)`). Esto **reparte** el fuego: una vez que un blanco está
   "saturado", los siguientes soldados prefieren otro blanco, evitando
   malgastar 10 disparos en un solo enemigo cuando hay varios.
4. Asignar `a → b*`, acumular el `p_hit` estimado de `a` sobre `b*` para el
   cálculo de saturación de los soldados siguientes.
5. Tras asignar todos los soldados, se ejecuta la Etapa 3 invocando ADR-011 §6
   por cada par `(a, b*)`. La saturación es solo una **heurística de
   asignación**; la baja real la decide la tirada de ADR-011 §6 (un blanco
   "saturado" puede sobrevivir; uno con un solo tirador puede caer).

| Param | Valor inicial | Rol |
|-------|---------------|-----|
| `P_SAT` | `0.80` | Umbral de probabilidad acumulada de baja sobre un blanco a partir del cual se considera "cubierto" y el reparto pasa al siguiente. Subir ⇒ más concentración de fuego; bajar ⇒ más dispersión. |

Todos sujetos a playtest.

#### 3.5 Override manual (interfaz)

El jugador, al emitir la orden ATTACK en el Briefing, puede **asignar blancos
explícitamente** a nivel soldado o a nivel hex:

- **Por hex (modo normal):** el jugador marca uno o más hexes objetivo; la
  regla §3.4 distribuye los soldados entre los blancos válidos de esos hexes.
  Es el modo esperado para la mayoría de las órdenes (carga cognitiva baja).
- **Por soldado (modo fino):** el jugador asigna `soldado → hex/tropa`
  explícitamente para soldados concretos; los soldados sin asignación manual
  caen en la regla §3.4 sobre los hexes objetivo restantes. Una asignación
  manual a un blanco fuera de `B(a)` (sin LOS, fuera de `r_eff`, o blanco
  muerto al resolverse) se trata como "sin blanco válido" en la Etapa 3 (el
  soldado no dispara ese evento); no se busca un blanco alternativo
  automáticamente cuando hubo override manual — el jugador eligió.
- La interfaz **no** permite asignar a un hex sin enemigos para fuego directo
  (infantería): se rechaza en el Briefing (validación de coherencia, ADR-004).
  El fuego indirecto **sí** admite hex sin enemigo visible (fuego ciego a
  coordenadas, cap. 07).

La designación es **parte de la orden** (se fija en el Briefing); la resolución
ocurre en la Etapa 3 con el estado del campo en ese instante de la línea de
tiempo (un blanco designado puede haber muerto antes por una escuadra de mayor
iniciativa — ADR-012 §6 / ADR-011 §6 paso 6; entonces ese soldado no dispara).

### 4. Resumen de constantes nuevas (todas tuneables por playtest)

| Constante | Valor | § | Efecto al subir |
|-----------|-------|---|-----------------|
| `R_VIS` (radio de vigilancia visual) | `5` hex | 2.1 | Las escuadras detectan antes; menos emboscadas, combate más temprano. |
| `P_SAT` (saturación de blanco) | `0.80` | 3.4 | Más concentración de fuego sobre un mismo enemigo; bajar reparte más. |

`R_det` deriva de `R_VIS` y de los `r_eff` de ADR-011 §3 (no tuneable aquí: es
propiedad de ADR-011). El orden de las cuatro etapas (§1) y el orden por
escuadra de la línea de tiempo (ADR-012) son la **estructura estable**; solo
estas dos constantes son calibrables sin reabrir este ADR.

## Consecuencias

**Positivas:**

- Cierra el Grupo 3 restante (#10, #11, #12): el backend (ADR-003/004) tiene la
  secuencia exacta (movimiento→encuentro→combate→post por escuadra), la regla
  de detección/interrupción de MOVE y la lógica de distribución de fuego,
  encajadas con el orden de iniciativa de ADR-012 sin re-ordenarlo.
- No reabre ADR-011/012/016: la matemática de intercambio, el cálculo de `INIT`
  y los datos de terreno/pools se **consumen**, no se redefinen. La saturación
  §3.4 es una heurística de asignación que **alimenta** el `p_hit` de ADR-011,
  no lo altera.
- Preserva la tesis del juego (subordinación + iniciativa): mover dentro del
  bloque de cada escuadra mantiene que la alta iniciativa golpea primero
  (ADR-012 §6); el aislamiento ya penaliza la reacción vía la línea de tiempo,
  sin reglas extra.
- Determinista dado estado + órdenes + semilla (ADR-004): los desempates
  (`troop_id`, `squad_id` ascendente) y la regla §3.4 son reproducibles para la
  verificación de coherencia del Briefing.
- La auto-distribución con saturación evita el "sobrematar" sin obligar al
  jugador a microgestionar 10 soldados por escuadra; el override fino queda
  disponible para quien lo quiera.

**Negativas:**

- La detección discreta "tras cada hex" puede dejar pasar un contacto que
  ocurriría a media transición entre hexes; aceptable a escala 100 m/hex (un
  hex es la unidad mínima del tablero, cap. 02b) y mucho más simple que un
  trazado continuo.
- `R_VIS = 5` y `P_SAT = 0.80` son hipótesis de diseño; sin telemetría el
  equilibrio entre "combate temprano" y "emboscada" y entre concentración y
  dispersión de fuego es conjetural.
- La estimación de saturación recomputa un `p_hit` aproximado por blanco para
  decidir la asignación: pequeño costo de cómputo extra antes de la resolución
  real; se acota recalculando solo sobre blancos candidatos y reutilizando los
  factores de ADR-011 ya disponibles.
- El post-procesado local por escuadra (no global) implica que el estado de
  escuadra puede cambiar varias veces dentro de un turno a medida que distintas
  escuadras la baten; es el comportamiento buscado (presión sostenida del
  cap. 07) pero exige que la UI de replay muestre estados con timestamp, no un
  estado final único.

## Alternativas descartadas

- **Resolver todo el movimiento global y luego todo el combate (dos fases
  separadas):** contradice ADR-012 §6 y el cap. 07 ("alta iniciativa abate
  antes de que el enemigo dispare"): si todos mueven antes de que nadie dispare,
  la iniciativa deja de decidir quién pega primero. Rechazado por reabrir de
  facto la decisión de ADR-012.
- **Detección continua a lo largo de la transición entre hexes (sub-hex):**
  más fiel pero introduce geometría continua que choca con la grilla discreta
  y la regla de hasta 3 waypoints de ADR-016/cap. 06; el chequeo por hex
  entrado es suficiente a 100 m/hex.
- **Tirada de detección probabilística (sigilo/sorpresa con azar propio):**
  añadiría una segunda fuente de azar sobre la que ADR-012 ya inyectó (`R` de
  iniciativa) y volvería no reproducible la verificación de coherencia; el
  orden de detección lo decide ya la iniciativa (quien actúa antes ve antes).
- **`R_det` único fijo para todas las escuadras:** ignora que una escuadra con
  MG "ve para batir" más lejos que una de fusileros; derivarlo de `r_eff`
  (ADR-011) con un piso visual `R_VIS` reaprovecha datos existentes sin nueva
  tabla.
- **Auto-distribución "todos al blanco más cercano":** colapsa los 10 disparos
  sobre un solo enemigo y desperdicia la potencia de la escuadra; la regla de
  saturación reparte como lo haría un oficial de escuadra (cap. 07: "distribución
  asignada por el oficial de escuadra").
- **Sin auto-distribución (asignación manual obligatoria por soldado):**
  microgestión inviable con escuadras de ~10 soldados y muchas escuadras por
  turno; el cap. 07 explícitamente contempla "una lógica de distribución
  automática". El override fino cubre al jugador que quiera control total.
- **Re-ordenar la línea de tiempo tras cada baja (recálculo de `INIT`
  intra-turno):** rompería ADR-012 (la línea de tiempo se fija una vez por
  turno con semilla); el estado intra-turno se propaga al **combate**
  (`f_estado`, blancos vivos) pero no re-secuencia el turno.
