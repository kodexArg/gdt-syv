# ADR-020: Fórmula de revelación, umbrales de inteligencia y reconocimiento

## Estado

Aceptado — 2026-05-18

## Contexto

El capítulo [10 — Niebla de Guerra](../manual/10-niebla-de-guerra.md) define el
eje de fog-of-war **visual** (distinto del eje de radio/triangulación, que
cierra [ADR-015](015-mando-comunicaciones-y-triangulacion.md)). El capítulo es
explícito en su forma —revelación **continua, sin capas discretas**, evaluada
server-side por unidad amiga contra cada unidad enemiga, con la **mejor
observación** ganando— pero deja cuatro huecos marcados `Pendiente`
(decisiones #52, #53, #54 del "Grupo 10", más #55):

- **#52** Fórmula exacta de revelación continua: cómo combina distancia, tipo
  de observador, terreno (`vision_block`), elevación y clima.
- **#53** Umbrales de inteligencia: a qué nivel calculado se transmite
  presencia / tipo / detalle.
- **#54** Habilidades concretas de las escuadras `recon` (mayor rango y
  detalle): cap. 10 solo dice "valores de observación más altos: Pendiente".
- **#55** Efecto concreto del **Grupo Scout** interno en visión (cap.
  [04b](../manual/04b-ejemplos-de-escuadras.md) tiene un `<!-- Pendiente -->`
  explícito).

Sin estos valores el servidor no puede calcular el scope visual de ADR-005 ni
poblar el `EnemyContact` recortado del contrato de protocolo (ADR-010): el
campo `fidelity` (FIRM/STALE/CONTACT) y la decisión de qué campos omitir
dependen directamente de un nivel de revelación que hoy no está definido.

Restricciones de coherencia respetadas:

- **Fidelidad al cap. 10:** revelación **continua**, sin capas discretas; el
  servidor evalúa la fórmula desde **cada unidad amiga** contra **cada unidad
  enemiga** y se queda con la **mejor observación**; el reconocimiento es una
  **propiedad de tipo de escuadra** (`recon`), no una orden; la
  radiogoniometría es un eje **separado** (ADR-015 / cap. 12b) y no se toca
  aquí; el terreno base y la elevación del mapa siempre son visibles
  (el Comandante conoce el mapa).
- **[ADR-001](001-nomenclatura-y-glosario.md):** términos canónicos (Escuadra,
  hex, `recon`, Grupo, L1–L5, scope); sin sinónimos nuevos.
- **[ADR-005](005-seguridad-por-scope-individual.md):** el nivel de revelación
  y todo dato derivado es estado del jugador observador, calculado
  server-side, serializado **solo** en su scope; lo no revelado **no existe**
  en el payload (no se oculta: no se serializa).
- **[ADR-010](010-contrato-de-protocolo-de-red.md):** este ADR **no añade
  mensajes ni campos** al contrato. Mapea su salida sobre el `EnemyContact` ya
  tipado (`ref` opaco, `hex`, `last_seen_turn`, `fidelity ∈
  {FIRM, STALE, CONTACT}`) y sobre la regla ya escrita de campos prohibidos
  (`type` exacto, composición, `strength`, `moral`, órdenes, niveles).
- **[ADR-015](015-mando-comunicaciones-y-triangulacion.md):** la niebla visual
  y el stealth de radio son **ortogonales** (ADR-015 §5.3 explícito: mover y
  disparar no rompen stealth de radio; disparar **sí** puede revelar por línea
  de visión bajo *este* ADR). Este ADR es el dueño del eje visual; no toca
  triangulación.
- **[ADR-016](016-terreno-elevacion-y-puntos-de-movimiento.md):** consume
  `vision_block ∈ [0,1]` por tipo de terreno y la regla de elevación con
  umbral `E_MIN = 5.0 m` **tal como ADR-016 §1–§2 los define**. ADR-016 fija
  el dato y el predicado de comparación de cota; este ADR aporta el
  **algoritmo de trazado de LOS y la fórmula de revelación** que ADR-016 §2
  declaró explícitamente "responsabilidad del sistema de visión".
- **[ADR-013](013-stats-canonicos-y-catalogo-de-modificadores.md):** los 6
  stats canónicos **no incluyen** un stat de percepción y este ADR **no
  introduce uno** (ADR-013 cierra la lista). La capacidad de observación es
  una **propiedad del tipo de escuadra** (cap. 10 explícito), expresada como
  parámetros de revelación por tipo, no como un 7.º stat. Coherente con la
  nota de ADR-013 §1 ("Recon +RCT/+MOV": el sesgo de la clase recon vive en
  sus stats de combate; su superioridad de observación vive aquí).

Este ADR es normativo sobre la **regla de juego**. La implementación Godot y
los esquemas viven donde ADR-008/010 los ubican. **No modifica** el manual ni
otros ADRs; el cap. 10 y el cap. 04b lo referenciarán cuando se actualicen.

## Decisión

### 1. Fórmula continua de revelación (#52)

Para cada par (unidad amiga `O` observadora, unidad enemiga `T` objetivo) el
servidor calcula un escalar continuo **`R ∈ [0, 1]`** ("nivel de revelación").
No hay capas: `R` es un número real; los umbrales del §2 solo deciden **qué se
serializa**, no segmentan el cálculo.

```
R(O, T) = R_base(d) · V_los(O, T) · M_obs(O) · M_clima
```

donde `d` es la distancia en hexes (cap. 02b, eje hexagonal) entre el hex de
`O` y el de `T`.

**1.1 Atenuación por distancia `R_base(d)` — continua.**
Decaimiento lineal por tramos anclado a la tabla de distancias del cap. 02b
(combate directo 1 hex; fusilero efectivo 2–3; límite táctico 4–5;
visibilidad máxima en abierto hasta ~10):

```
R_base(d) = clamp( 1 - (d - 1) / D_VIS , 0 , 1 )
```

con **`D_VIS = 10`** hexes (1 km) = alcance de observación base de una escuadra
no-recon en terreno totalmente despejado. `R_base(1) = 1.0` (hex adyacente,
visión total); decae linealmente hasta `R_base(11) = 0.0` (más allá de 1 km la
infantería regular no aporta inteligencia visual por sí sola). Es **continuo**
en `d` salvo por la discretización natural de la grilla.

**1.2 Factor de línea de visión `V_los(O, T) ∈ [0,1]`.**
Es el complemento del bloqueo acumulado de terreno entre `O` y `T`, usando
`vision_block` de ADR-016 §1 y la regla de elevación de ADR-016 §2:

- Se traza la cadena de hexes de la línea recta `O→T` (excluidos los hexes
  extremos). Sea `H` el conjunto de hexes intermedios.
- **Regla de elevación (ADR-016 §2, no se redefine):** un hex intermedio
  `h ∈ H` **no cuenta** su `vision_block` si el observador domina la cota,
  es decir si `elevation(O) - elevation(h) >= E_MIN` (`E_MIN = 5.0 m`,
  ADR-016). Recíprocamente, si `elevation(T) - elevation(O) >= E_MIN` el
  objetivo está sobre una cresta y se aplica una bonificación: el bloqueo
  efectivo se reduce a la mitad (un objetivo elevado se recorta contra el
  cielo). Estas dos reglas usan exactamente el predicado de cota de
  ADR-016 §2; este ADR solo decide **cómo entra** en el trazado.
- Bloqueo acumulado y factor LOS:

  ```
  B = clamp( Σ_{h ∈ H, h no anulado por cota}  vision_block(h) , 0 , 1 )
  B_ef = B · (0.5 si elevation(T) - elevation(O) >= E_MIN  else 1.0)
  V_los = 1 - B_ef
  ```

  Un solo hex con `vision_block = 1.0` no anulado por cota corta la LOS por
  completo (`V_los = 0` ⇒ `R = 0`: el objetivo no se ve en absoluto a través
  de ese hex). Varios hexes ligeros (`vision_block` 0.3–0.6) suman y degradan
  la revelación de forma continua sin necesariamente anularla. En el
  prototipo de ADR-016 (`GRASS`/`WATER`, ambos `vision_block = 0.0`)
  `V_los = 1` siempre: el terreno no estorba hasta que se incorporen tipos
  con `vision_block > 0` (bosque/urbano, conjeturales en ADR-016 §1).

**1.3 Multiplicador de observador `M_obs(O)`.**
Propiedad del **tipo de escuadra** (no un stat, §contexto / ADR-013):

| Tipo de escuadra observadora | `M_obs` |
|------------------------------|---------|
| `recon` | **1.8** |
| Cualquier otro tipo (infantería, apoyo, MG, mortero, etc.) | **1.0** |

`M_obs` multiplica el alcance y la calidad efectivos: con `M_obs = 1.8` una
escuadra `recon` cruza el primer umbral del §2 a una distancia ~1,8× mayor que
la infantería regular y entrega más detalle a igual distancia (desarrollo en
§3). Una escuadra con un **Grupo Scout** interno activo recibe un `M_obs`
intermedio definido en §4.

**1.4 Factor climático `M_clima ∈ (0,1]`.**
Reservado y **neutro en el prototipo** (`M_clima = 1.0` siempre, no hay clima
implementado), igual que ADR-016 reservó el término de elevación de
movimiento. Cuando se incorpore clima, será un multiplicador global por turno
sobre **todos** los pares (niebla densa / lluvia degradan toda la visión por
igual). Valores conjeturales, **no vigentes**, **sujetos a playtest**: niebla
densa `0.4`, lluvia `0.7`, noche `0.5`, despejado `1.0`. Se documentan solo
para evidenciar que la fórmula los absorbe sin cambio estructural.

**1.5 Mejor observación (cap. 10, no se redefine).**
La revelación que el jugador obtiene de `T` es
`R*(T) = max_{O ∈ unidades amigas vivas} R(O, T)`.
El servidor evalúa el máximo y serializa según §2. Esto implementa
literalmente "el jugador ve el enemigo a través de los ojos de su mejor
observador" (cap. 10).

### 2. Umbrales de inteligencia (#53)

`R*` es continuo; estos umbrales **solo deciden qué campos se serializan** en
el `EnemyContact` de ADR-010 (no segmentan el cálculo — cap. 10: "no existe
mapeado fijo… es un continuo", pero el contrato de red necesita una
discretización **de salida**). Se mapean sobre el enum `fidelity` ya existente
en ADR-010 (`CONTACT` / `FIRM`) más la regla de campos prohibidos:

| `R*` | Nivel narrativo (cap. 10) | `fidelity` (ADR-010) | Qué se serializa en `EnemyContact` |
|------|---------------------------|----------------------|-------------------------------------|
| `R* < R_PRES = 0.15` | nada | — (no se emite) | **El contacto no existe** en el payload (ADR-005). |
| `R_PRES ≤ R* < R_TIPO = 0.40` | presencia | `CONTACT` | `ref` opaco + `hex` (área) + `last_seen_turn`. **Sin** tipo, sin fuerza. "Algo está ahí." |
| `R_TIPO ≤ R* < R_DET = 0.75` | tipo + magnitud aproximada | `CONTACT` | Lo anterior **+** tipo de unidad **+** banda de efectivos cualitativa: `1–5` / `6–10` / `10+` (cap. 10). Sigue prohibido: composición, `strength` exacto, `moral`, órdenes, niveles (ADR-010). |
| `R* ≥ R_DET = 0.75` | detalle | `FIRM` | Tipo exacto + banda de efectivos + estado general (intacta / mermada / crítica, derivado del cap. 09 sin exponer `moral` ni stats). Sigue prohibido el `Escuadra.id` real (siempre `ref` opaco) y las **órdenes en curso** (nunca se serializan: la "orden" del nivel máximo del cap. 10 se interpreta como *postura observable* — moviéndose / estática / disparando—, no como la orden interna del enemigo, que ADR-010 prohíbe). |

- **Continuidad preservada:** el cálculo de `R*` es un real; los cortes son una
  **proyección de salida** impuesta por que `EnemyContact.fidelity` es un
  enum de 3 valores (ADR-010, no modificable aquí). No hay "capas" en la
  evaluación: mover al observador un hex cambia `R*` de forma continua y puede
  cruzar un corte sin discontinuidad de modelo.
- **`STALE` (memoria).** ADR-010 define `STALE` = recuerdo. Cuando `R*` cae por
  debajo de `R_PRES` pero el jugador **vio** a `T` en un turno reciente, el
  servidor conserva el último `EnemyContact` con `fidelity = STALE` y su
  `hex`/`last_seen_turn` congelados, durante **`MEM_TURNS = 2`** turnos; luego
  el contacto desaparece del payload. El recuerdo nunca se actualiza con
  información que el jugador ya no observa (ADR-005: no se serializa estado
  ajeno actual).
- **Disparar y la niebla visual.** Coherente con ADR-015 §5.3: disparar no
  rompe stealth de radio, pero **sí** es un evento físico observable. Una
  unidad enemiga que dispara recibe, ese turno, un bono aditivo
  `+R_FIRE = 0.20` a `R*` **solo** para los observadores con `V_los > 0`
  hacia ella (un fogonazo no atraviesa un bloqueo total de LOS). Esto
  mantiene los dos ejes ortogonales: el disparo afecta la **niebla visual**
  (este ADR), nunca la triangulación (ADR-015).

### 3. Habilidades concretas de las escuadras `recon` (#54)

El cap. 10 reserva el reconocimiento mejorado al **tipo** `recon` (presente en
`conf-recon.yaml` / `conf-recon-01.yaml`). Sus capacidades concretas, además
del `M_obs = 1.8` del §1.3:

1. **Mayor alcance.** Por el `M_obs = 1.8`, una `recon` en terreno despejado
   mantiene `R_base · M_obs ≥ R_PRES` hasta ≈**18 hexes (1,8 km)** y entrega
   *tipo* (`R_TIPO`) a distancias donde la infantería solo da presencia. A
   igual distancia siempre revela **un nivel más** que una escuadra regular.
2. **Penetración de cobertura ligera.** Una `recon` ignora el primer `0.30` de
   bloqueo `vision_block` acumulado en la LOS (oteador entrenado): en §1.2
   usa `B_recon = clamp(B - 0.30, 0, 1)` antes de calcular `V_los`. **No**
   penetra un bloqueo total (un hex `vision_block = 1.0` sigue cortando: no
   ve a través de un muro, sí a través de matorral ligero).
3. **Detalle a distancia.** El umbral de detalle de una `recon` se relaja:
   alcanza `FIRM` con `R* ≥ R_DET_RECON = 0.60` (vs. `0.75` general). Ve
   *qué* es y *en qué estado está* desde más lejos — su función táctica.
4. **Sin orden de reconocimiento.** Coherente con cap. 10: estas capacidades
   son pasivas y permanentes por ser propiedad del tipo; no existe una orden
   "RECONOCIMIENTO" y ninguna otra escuadra puede adquirirlas por orden.
5. **Degradación por bajas.** Si la `recon` pierde a su líder L2 (las
   premontadas son `squad_level: 2`), pierde su organización (cap. 04 / cap.
   10) y revierte a `M_obs = 1.0` y umbrales generales hasta recomponer
   liderazgo (coherente con la regla de Grupo del cap. 04: sin L2 vivo, sin
   capacidad especial).

### 4. Efecto del Grupo Scout interno en visión (#55)

El cap. 04b describe escuadras de infantería con un **Grupo Scout** interno
(un L2 + L1 con sniper y oteador) y deja su efecto en niebla como `Pendiente`.
Decisión: una escuadra **no-`recon`** que contiene un **Grupo Scout
organizado y vivo** recibe una fracción de la capacidad recon, sin igualarla
(la `recon` dedicada debe seguir siendo superior):

- **`M_obs = 1.35`** para la escuadra mientras su Grupo Scout esté
  **organizado** (líder L2 del grupo vivo — regla de Grupo del cap. 04: "si el
  líder L2 del grupo cae, el grupo pierde su organización"). Es un valor
  intermedio entre `1.0` (regular) y `1.8` (`recon`).
- **Penetración de cobertura ligera reducida:** ignora el primer `0.15` de
  `vision_block` acumulado (mitad del beneficio recon del §3.2).
- **No** obtiene el umbral de detalle relajado del §3.3 (eso queda exclusivo
  de la `recon` dedicada): el Grupo Scout extiende **alcance/penetración**, no
  **detalle a distancia**. Refuerza la jerarquía: `recon` > infantería con
  Grupo Scout > infantería sin él.
- **Pérdida del Grupo Scout.** Si muere el L2 del Grupo Scout o se desorganiza
  (cap. 04), la escuadra revierte a `M_obs = 1.0` desde ese turno; el resto de
  la escuadra no se ve afectado en lo demás (los miembros sobrevivientes
  pasan a tropas sueltas, cap. 04).
- Una escuadra `recon` **no** acumula este bono con el del §3 (no es aditivo):
  `recon` ya es el tope; el Grupo Scout es el mecanismo para que la infantería
  *regular* compre algo de visión sin ser `recon`.

`M_obs` resultante por observador (resumen): `recon` = 1.8 ·
(1.0 si L2 vivo, si no 1.0 base) ; infantería con Grupo Scout organizado =
1.35 ; cualquier otro = 1.0.

### 5. Resumen de constantes nuevas (todas tuneables por playtest)

| Constante | Valor | §  | Efecto al subir |
|-----------|-------|----|-----------------|
| `D_VIS` (alcance base de visión) | 10 hexes (1 km) | 1.1 | Más visión para todos; aplana la ventaja de la `recon`. |
| `M_obs` `recon` | 1.8 | 1.3 / 3 | Cuánto supera la `recon` a la infantería en alcance/detalle. |
| `M_obs` Grupo Scout | 1.35 | 4 | Cuánta visión compra la infantería regular con Scout interno. |
| `R_PRES` | 0.15 | 2 | Umbral de "presencia"; bajarlo revela contactos más lejanos. |
| `R_TIPO` | 0.40 | 2 | Umbral de "tipo + magnitud". |
| `R_DET` (general) / `R_DET_RECON` | 0.75 / 0.60 | 2 / 3.3 | Umbral de "detalle"; la `recon` lo alcanza antes. |
| `MEM_TURNS` (vida del recuerdo `STALE`) | 2 turnos | 2 | Cuánto persiste un contacto perdido en el Briefing. |
| `R_FIRE` (bono por disparo observable) | +0.20 | 2 | Cuánto delata disparar ante observadores con LOS. |
| Penetración cobertura `recon` / Scout | 0.30 / 0.15 | 3.2 / 4 | Cuánto matorral ligero ignora cada uno. |
| `M_clima` (prototipo) | 1.0 (neutro) | 1.4 | Reservado; clima futuro lo activa globalmente. |

`vision_block`, `E_MIN = 5.0 m` y el predicado de cota **no** son tuneables
aquí: son propiedad de ADR-016. El enum `fidelity` y la lista de campos
prohibidos **no** son tuneables aquí: son propiedad de ADR-010.

## Consecuencias

**Positivas:**

- Cierra el Grupo 10 (#52 fórmula, #53 umbrales, #54 `recon`) y #55 (Grupo
  Scout): el servidor ya tiene un escalar `R` determinista y la proyección
  exacta sobre el `EnemyContact` de ADR-010 para recortar el scope de ADR-005
  sin añadir mensajes ni campos al contrato.
- La fórmula es **genuinamente continua** (cap. 10): el cálculo nunca usa
  capas; la única discretización es la de salida, impuesta por el enum
  `fidelity` que ADR-010 ya fijó y este ADR no puede ni debe cambiar.
- No reabre ADR-016: consume `vision_block` y la regla `E_MIN` exactamente
  como ADR-016 §1–§2 las dejó, completando el "algoritmo de trazado" que
  ADR-016 §2 declaró ajeno a sí mismo. En el prototipo `GRASS`/`WATER`
  (`vision_block = 0`) el término LOS es neutro, igual que el término de
  elevación de movimiento de ADR-016.
- No introduce un 7.º stat: respeta el cierre de ADR-013; la superioridad de
  observación vive como propiedad de tipo (cap. 10 explícito).
- Mantiene ortogonales los dos ejes de fog-of-war: el disparo afecta solo la
  niebla visual (este ADR), nunca la triangulación (ADR-015 §5.3), y el
  stealth de radio sigue siendo exclusivo de ADR-015.
- Jerarquía de visión legible y temática: `recon` > infantería con Grupo
  Scout > infantería sin él, anclada en las reglas de Grupo del cap. 04.

**Negativas:**

- ≈10 constantes nuevas requieren playtest (alcances, umbrales, multiplicadores,
  vida del recuerdo): sin telemetría la sensación de "cuánto veo" es
  conjetural; todas marcadas como tuneables.
- El trazado de LOS hex a hex con regla de cota añade un cálculo geométrico
  por par (O, T) en el peor caso O(unidades_amigas × unidades_enemigas ×
  longitud_LOS); en el prototipo plano (`vision_block = 0`) es trivial, pero
  con terreno real el cómputo de scope crece — costo aceptado del modelo
  server-side de ADR-005.
- La proyección continuo→enum de 3 valores pierde resolución en el payload
  (dos `R*` distintos pueden mapear al mismo `EnemyContact`): es una limitación
  heredada de ADR-010, no de este modelo; el cálculo interno conserva el
  continuo para "mejor observación" y futuros refinamientos.
- El bono `R_FIRE` por disparo introduce un acoplamiento controlado entre la
  fase de combate y el cálculo de scope del turno; está acotado a observadores
  con `V_los > 0` para no volverse omnisciente, pero es un caso especial más
  en el pipeline de Briefing.

## Alternativas descartadas

- **Capas discretas de revelación (niebla en 3 anillos fijos):** contradice el
  cap. 10 ("no depende de capas discretas… es un continuo"). El escalar `R`
  con proyección de salida preserva el continuo y aun así alimenta el enum de
  ADR-010.
- **Stat de percepción (`PER`) por soldado:** reabriría ADR-013 (que cierra la
  lista de 6 stats) y contradiría el cap. 10 ("el reconocimiento es una
  capacidad de **tipo de unidad**"). Modelarlo como `M_obs` por tipo es fiel
  y no toca ADR-013.
- **Decaimiento de distancia exponencial / gaussiano:** más "realista" pero
  menos legible y más difícil de anclar a la tabla de distancias del cap. 02b;
  el decaimiento lineal por `D_VIS` se lee directamente en hexes/metros.
- **Añadir campos nuevos al `EnemyContact` (p. ej. un `reveal_level` float):**
  violaría la restricción de no tocar ADR-010 y expondría más resolución de la
  que el contrato concede; se proyecta sobre `fidelity` + reglas de campos ya
  existentes.
- **Recuperación/recuerdo infinito de contactos vistos:** rompe ADR-005 (el
  cliente terminaría con un mapa de inteligencia que no decae); `MEM_TURNS`
  finito mantiene la niebla viva.
- **Disparar rompe también stealth de radio (un solo eje de detección):**
  acoplaría visual y radio y haría redundante ADR-015 §5.3; se mantienen
  ortogonales — disparar solo añade `R_FIRE` al eje visual.
- **Grupo Scout que iguala a la `recon` dedicada:** anularía el valor de
  comprar una escuadra `recon`; el bono intermedio (`M_obs = 1.35`, sin
  detalle a distancia) preserva la jerarquía y el sentido económico del
  cap. 14 (presupuesto/coste).
- **Clima activo desde el prototipo:** no hay sistema de clima; activarlo sería
  especular sin base. Se reserva neutro (`M_clima = 1.0`), igual criterio que
  ADR-016 con la elevación de movimiento.
