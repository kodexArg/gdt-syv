# ADR-016: Terreno, elevación y puntos de movimiento

## Estado

Aceptado — 2026-05-18

## Contexto

El terreno y el costo de moverse son transversales: alimentan la fase de
movimiento (cap. [06 — Órdenes](../manual/06-ordenes.md)), la visión (cap. 10) y
el combate (cap. 07). El capítulo [02 — El Campo de Batalla](../manual/02-campo-de-batalla.md)
define la **forma** del sistema —grilla flat-top radio 20 (1.261 hexes), cada
hex con un tipo de terreno y una elevación float, escala 100 m/hex
(cap. [02b](../manual/02b-distancias.md))— pero deja todos los valores como
"Pendiente de definir":

- Catálogo de terreno: solo Pasto/Agua, sin efectos cuantificados, declarado
  "abierto y extensible".
- Elevación: declarada con efecto en visión/combate pero con todos los hexes a
  `0.0` en prototipo; sin modelo de cómo `≠ 0.0` afecta.
- Pools de puntos de movimiento por tipo de escuadra (infantería < caballería <
  otros): sin valores.
- Tabla de costos de terreno para entrar a un hex: sin valores.
- Cap. 06 declara el escalado del bonus de fortificación de DEFEND por turno
  consecutivo como "Pendiente de definir".

Estos cinco huecos cierran el Grupo 11 (#56 terreno/elevación, #57 movimiento)
y la parte de movimiento/terreno del Grupo 6 (#22 pools, #23 costos, #24
fortificación). Sin valores concretos no se puede implementar la fase de
movimiento de la Resolution (ADR-004) ni alimentar los factores tácticos que
ADR-011 ya consume del terreno.

Restricciones de coherencia respetadas:

- Escala 100 m/hex y tabla de distancias (cap. 02b): un hex es 100 m, el costo
  de movimiento se interpreta como "esfuerzo de cruzar 100 m de ese terreno".
- Grilla flat-top radio 20, dos propiedades por hex: `terrain_type` (enum
  extensible) y `elevation` (float) — cap. 02; este ADR no cambia esa forma.
- [ADR-001](001-nomenclatura-y-glosario.md): terminología (Escuadra como unidad
  leaf que ocupa un hex; órdenes ATTACK/MOVE/DEFEND).
- [ADR-011](011-resolucion-de-combate-por-soldado.md): el combate ya consume
  `k_cobertura(D)` (abierto/ligera/pesada → ×1.0/×1.25/×1.55), elevación
  (atacante/defensor más alto → ×1.15) y "terreno difícil bajo el atacante"
  (×0.90). Este ADR define **de dónde salen** esos factores (el catálogo de
  terreno y el modelo de elevación), sin alterar los multiplicadores de
  ADR-011.
- [ADR-012](012-calculo-de-iniciativa-orden-de-resolucion.md): la iniciativa se
  ordena por escuadra y el bonus de fortificación de DEFEND es **ajeno** a la
  iniciativa (ADR-012 §2.2). Este ADR define ese bonus sin tocar `INIT`.

Este ADR **no modifica** el manual ni otros ADRs; los cap. 02 y 06 lo
referenciarán cuando se actualicen.

## Decisión

### 1. Catálogo de terreno extensible

El terreno se modela como un **catálogo de definiciones** indexado por un enum
`TerrainType`. Cada definición es un registro autónomo con cuatro campos de
efecto, de modo que añadir un tipo nuevo es agregar una fila sin tocar las
existentes (cap. 02: sistema "abierto y extensible"):

```
TerrainDef {
    id:            TerrainType        # enum estable
    passable:      bool               # false ⇒ ninguna escuadra puede entrar
    move_cost:     float              # puntos de movimiento para ENTRAR al hex (§4)
    cover_class:   {OPEN, LIGHT, HEAVY}  # mapea a k_cobertura de ADR-011 §3
    vision_block:  float ∈ [0,1]      # fracción de bloqueo de LOS aportada por el hex
}
```

Mapeo `cover_class → k_cobertura` (ADR-011 §3, no se redefine aquí, solo se
referencia): `OPEN → ×1.0`, `LIGHT → ×1.25`, `HEAVY → ×1.55`.

`vision_block` se reserva para el sistema de visión (cap. 10): `0.0` = no
estorba la línea de visión, `1.0` = la corta por completo en ese hex. El motor
de LOS de visión lo consumirá; aquí solo se fija el dato por tipo.

**Prototipo (dos tipos definidos):**

| `id` | `passable` | `move_cost` | `cover_class` | `vision_block` | Lectura |
|------|-----------|-------------|---------------|----------------|---------|
| `GRASS` (Pasto) | `true` | `1.0` | `OPEN` | `0.0` | Terreno base. Costo unitario; sin cobertura ni bloqueo de visión. Es el "costo base" del cap. 06. |
| `WATER` (Agua) | `false` | `∞` | `OPEN` | `0.0` | Intransitable. Bloquea movimiento (cap. 02). No otorga cobertura ni corta visión a quien ve a través/sobre él. `move_cost` formalmente `∞` (cualquier validador lo trata como "no se puede entrar"). |

**Estructura preparada para más (no se definen ahora; valores conjeturales,
sujetos a playtest cuando se incorporen):** tipos como bosque
(`LIGHT`/`HEAVY`, `move_cost` 2–3, `vision_block` 0.6–1.0), edificación
(`HEAVY`, `move_cost` 2, `vision_block` 1.0), barro/pantano (`OPEN`/`LIGHT`,
`move_cost` 3–4), camino (`OPEN`, `move_cost` 0.5). Estos **no** forman parte
de la decisión vigente; se listan solo para evidenciar que el esquema de cuatro
campos los absorbe sin cambios estructurales.

`GRASS` es el terreno por defecto de cualquier hex de escenario que no declare
otro tipo.

### 2. Modelo de elevación

Cada hex tiene `elevation: float` (metros sobre un cero arbitrario del
escenario). La elevación **no** afecta el movimiento en el prototipo y **no**
tiene costo propio mientras todos los hexes valgan `0.0`. Su efecto es
táctico (visión + combate) y se activa **solo cuando hay diferencia de
elevación entre dos hexes relevantes** (`Δe ≠ 0.0`):

- **Combate.** El factor ya existe en ADR-011 §3: si el hex del atacante tiene
  mayor `elevation` que el del defensor → `m_elev_atk = ×1.15`; si el defensor
  está más alto → `m_elev_def = ×1.15`. Este ADR fija **cuándo se considera
  "más alto"**: con un **umbral de significancia** `E_MIN = 5.0 m`.

  ```
  ventaja_alto(hex_X, hex_Y) = (elevation(X) - elevation(Y)) >= E_MIN
  ```

  Diferencias `|Δe| < E_MIN` se tratan como terreno igualado (sin bono de
  elevación a ninguno). Esto evita que ruido topográfico de pocos metros
  dispare el ×1.15. ADR-011 sigue siendo la única fuente del multiplicador;
  aquí solo se define el predicado de activación.

- **Visión.** El hex elevado "ve más allá": en el cálculo de línea de visión
  (cap. 10, motor pendiente), un observador en un hex de mayor `elevation`
  ignora el `vision_block` de hexes intermedios cuya `elevation` sea **menor
  que la del observador menos `E_MIN`**. La fórmula completa de LOS es
  responsabilidad del sistema de visión; este ADR aporta el dato (`elevation`
  por hex) y la regla de comparación (`E_MIN`), no el algoritmo de trazado.

- **Movimiento (futuro, conjetural).** Cap. 06 menciona "posible costo
  adicional por cambios de elevación". Se reserva, **inactivo en prototipo**
  (todos los hexes a `0.0` ⇒ `Δe = 0` siempre). Regla futura propuesta
  (sujeta a playtest, **no** vigente): subir un escalón añade
  `ceil(|Δe| / 10 m) · 0.5` puntos al `move_cost` del hex de destino; bajar no
  añade costo. Se documenta para que el esquema de costos (§4) ya contemple el
  término.

**Prototipo:** todos los hexes `elevation = 0.0` ⇒ ningún factor de elevación
se dispara; el combate de ADR-011 corre con `m_elev_* = 1.0` por defecto, como
ya asume su §8.

### 3. Pools de puntos de movimiento por tipo de escuadra

Cada escuadra tiene, por turno, un **pool de puntos de movimiento** `MP`
determinado por su tipo. El pool se reinicia en cada Briefing y los puntos no
gastados **no se acumulan** (cap. 06). Tipos y valores (supuesto de balance,
sujeto a playtest), ordenados infantería < otros < caballería como exige
cap. 06:

| Tipo de escuadra | `MP` por turno | Hexes de `GRASS` alcanzables | Racional |
|------------------|----------------|------------------------------|----------|
| Infantería | `4` | 4 (≈ 400 m / 4 h) | Pie a paso de combate; menos puntos. |
| Otros (apoyo, MG, mortero, especialistas pesados) | `5` | 5 (≈ 500 m) | Valor intermedio; lastrados por equipo pesado pero motorizables. |
| Caballería | `8` | 8 (≈ 800 m) | Movilidad montada; más puntos. |

Coherencia con la escala (cap. 02b): 4 h de turno, 100 m/hex. Infantería a
~400 m / 4 h ≈ 1,7 km/h efectivos en campaña con combate — deliberadamente
por debajo de la marcha teórica porque el turno incluye reconocimiento,
órdenes y fricción, no marcha pura. Caballería duplica el alcance de la
infantería, consistente con cap. 06 ("caballería: más puntos").

El "tipo de escuadra" se deriva de la composición/plantilla (cap. 04); el
mapeo exacto plantilla→tipo de movimiento es responsabilidad del modelo de
datos (ADR-009) y no se fija aquí más allá de las tres clases de pool.

### 4. Tabla de costos de terreno (entrar a un hex)

El costo de un movimiento es el `move_cost` del hex de **destino** (entrar
cuesta; salir es gratis), tomado del catálogo §1:

| Terreno destino | Costo de entrar | Nota |
|-----------------|-----------------|------|
| `GRASS` | `1.0` | Costo base (cap. 06). |
| `WATER` | `∞` (intransitable) | El path que incluya un hex `WATER` es inválido; el validador lo rechaza antes de ejecutar. |

Costo total de un path = suma de `move_cost` de cada hex **al que se entra**
(el hex de origen no se cuenta). Cuando exista elevación activa (§2, futuro),
se sumaría el término de elevación al `move_cost` del hex de destino; en
prototipo ese término es `0`.

**Validación del path (cap. 06: hasta 3 waypoints):**

```
costo_path = Σ  move_cost(hex_i)   para cada hex_i entrado a lo largo del path
path válido  ⟺  ningún hex_i es intransitable  ∧  costo_path <= MP(escuadra)
```

- La escuadra elige un path de **hasta 3 waypoints** (cap. 06), se calcula
  `costo_path` y se valida **antes** de ejecutar: si `costo_path > MP` o el
  path cruza un hex intransitable, el path se rechaza (no hay movimiento
  parcial fuera de lo que el pool permita; la escuadra puede acortar el path).
- El movimiento solo ocurre con una orden MOVE activa (cap. 06); ATTACK usa el
  mismo costo de terreno para su componente de avance.
- Ejemplo: infantería (`MP = 4`) con path de 4 hexes `GRASS` ⇒
  `costo_path = 4·1.0 = 4.0 ≤ 4` → válido, consume todo el pool. Un 5.º hex lo
  haría `5.0 > 4` → inválido. Caballería (`MP = 8`) recorrería 8 hexes
  `GRASS`. Cualquier path que toque `WATER` → inválido.

### 5. Escalado del bonus de fortificación por DEFEND consecutivo

Cap. 06: una escuadra que recibe DEFEND en turnos consecutivos acumula un
bonus defensivo (fortification) que se **reinicia a cero** si se mueve o
recibe otra orden. Este bonus es **multiplicativo sobre la potencia defensiva**
y se inserta como un factor adicional del lado del defensor en ADR-011, junto
a `m_DEFEND` (ADR-011 §3), del que es independiente: `m_DEFEND = ×1.30`
representa "estar en orden DEFEND este turno"; la fortificación representa
"llevar varios turnos atrincherándose en el mismo hex".

Sea `n` el número de **turnos consecutivos** que la escuadra lleva en DEFEND
**sin moverse** (el turno actual incluido; `n = 1` el primer turno de DEFEND):

```
m_fort(n) = 1 + FORT_STEP · (n - 1) ,  acotado a  m_fort <= FORT_MAX
```

Parámetros (supuesto de balance, sujeto a playtest):

| Param | Valor | Rol |
|-------|-------|-----|
| `FORT_STEP` | `0.15` | Incremento de fortificación por cada turno consecutivo extra de DEFEND. |
| `FORT_MAX` | `1.60` | Techo: atrincheramiento maduro alcanzado en 5 turnos. |

Tabla resultante:

| `n` (turnos DEFEND consecutivos) | `m_fort` | Lectura táctica |
|----------------------------------|----------|-----------------|
| 1 | 1.00 | Recién planta posición: sin bono extra (solo el `m_DEFEND` de ADR-011). |
| 2 | 1.15 | Una noche cavando. |
| 3 | 1.30 | Posición consolidada — coherente con cap. 06 ("3 turnos = mucho más difícil de desalojar"). |
| 4 | 1.45 | Fortín. |
| 5 | 1.60 | Atrincheramiento máximo (`FORT_MAX`). |
| ≥6 | 1.60 | Saturado: no crece más. |

**Interacción con ADR-011.** `m_fort` se aplica como factor multiplicativo
extra del lado del defensor, en la misma posición que `m_DEFEND`:

```
f_tactico_def = m_elev_def · m_DEFEND · m_fort        (k_cobertura aparte, ADR-011 §2)
```

ADR-011 no se modifica: `m_fort` es un factor que vale `1.0` siempre que la
escuadra no acumule DEFEND consecutivo (`n = 1` o no está en DEFEND), de modo
que el ejemplo numérico de ADR-011 §8 (que asume `m_DEFEND = 1.0`) sigue
siendo correcto sin cambios. Cuando el cap. 07/ADR-011 se actualicen,
incorporarán `m_fort` por referencia a este ADR.

**Reinicio.** Cualquier turno en que la escuadra reciba una orden distinta de
DEFEND, o se mueva de hex, o sea desplazada → `n` vuelve a `0` (próximo DEFEND
arranca en `n = 1`, `m_fort = 1.0`). Coherente con cap. 06 (el bonus se
reinicia a cero al moverse o cambiar de orden) y con ADR-012 §2.2 (la
fortificación es ajena a la iniciativa; no toca `INIT`).

### 6. Resumen de constantes nuevas (todas tuneables por playtest)

| Constante | Valor | §  | Efecto al subir |
|-----------|-------|----|-----------------|
| `move_cost(GRASS)` | 1.0 | 1,4 | Reduce alcance efectivo de todas las escuadras. |
| `MP` infantería / otros / caballería | 4 / 5 / 8 | 3 | Alcance por turno; mantener orden inf < otros < cab. |
| `E_MIN` (umbral de elevación significativa) | 5.0 m | 2 | Menos terreno cuenta como "alto"; suaviza ventaja de cota. |
| `FORT_STEP` | 0.15 | 5 | Atrincherarse rinde más rápido. |
| `FORT_MAX` | 1.60 | 5 | Techo de cuán inexpugnable llega a ser una posición. |

`WATER = intransitable` y `cover_class → k_cobertura` (×1.0/×1.25/×1.55) **no**
son tuneables aquí: el primero es regla del cap. 02, el segundo es propiedad de
ADR-011.

## Consecuencias

**Positivas:**

- Cierra Grupo 11 (#56, #57) y la parte movimiento/terreno del Grupo 6 (#22,
  #23, #24): el backend (ADR-003/004) tiene catálogo, pools, tabla de costos,
  modelo de elevación y escalado de fortificación concretos para validar paths
  y resolver la fase de movimiento de la Resolution.
- El catálogo de cuatro campos (`passable`, `move_cost`, `cover_class`,
  `vision_block`) es genuinamente abierto: nuevos terrenos son filas, no
  refactors — cumple el mandato de extensibilidad del cap. 02.
- No reabre ni contradice ADR-011/012: la elevación y la cobertura siguen
  produciendo exactamente los multiplicadores que ADR-011 ya define; este ADR
  solo define su **origen de datos** y el predicado de activación (`E_MIN`).
  `m_fort` se inserta como factor neutro (=1.0) en el caso por defecto, así que
  los ejemplos de ADR-011 no cambian.
- La fortificación es independiente de la iniciativa (ADR-012 §2.2 explícito):
  sin acoplamiento cruzado entre ADRs.
- Valores anclados a la escala 100 m/hex y 4 h/turno (cap. 02b): los pools se
  leen como distancias plausibles en metros.

**Negativas:**

- Solo dos terrenos definidos; el grueso del catálogo táctico (bosque, urbano,
  barro, camino) queda conjetural y sin cerrar — aceptable: el cap. 02 declara
  el prototipo de dos tipos y el esquema los absorbe sin cambios.
- El término de costo por elevación queda especificado pero inactivo; cuando
  haya mapas con relieve habrá que playtestearlo (no bloquea el prototipo
  plano).
- ≈7 constantes nuevas requieren playtest para calibrar (pools, `E_MIN`,
  `FORT_STEP`, `FORT_MAX`); sin telemetría el balance del movimiento y del
  atrincheramiento es conjetural.
- `m_fort` añade un factor más a la cadena defensiva de ADR-011; combinado con
  `m_DEFEND` y cobertura `HEAVY`, una escuadra muy atrincherada puede volverse
  difícil de desalojar (×1.30 · ×1.60 · ×1.55 ≈ ×3.2 en `P_def`) — es el
  efecto buscado por cap. 06, pero es el primer candidato a recalibrar si el
  juego defensivo domina.

## Alternativas descartadas

- **Costo de movimiento por par (salir+entrar) o por arista:** más fiel a
  wargames clásicos pero duplica el estado y complica la validación de paths de
  3 waypoints; el costo "al entrar al hex destino" es suficiente a escala
  100 m/hex y más simple de implementar.
- **Pools de movimiento en metros/velocidad continua:** choca con la grilla
  discreta y la regla de hasta 3 waypoints del cap. 06; los puntos enteros por
  hex son la abstracción natural del tablero.
- **Bonus de fortificación aditivo a un stat (estilo herida inversa):**
  rechazado por coherencia: ADR-011 ya modela el contexto táctico como
  **factores multiplicativos** del lado defensor; un término aditivo a `TGH`
  rompería la homogeneidad de `f_tactico_def`.
- **Fortificación con crecimiento sin techo o lineal ilimitado:** una posición
  infinitamente inexpugnable rompe el ritmo ofensivo; el techo `FORT_MAX` con
  saturación en 5 turnos preserva la lectura del cap. 06 sin volver el mapa
  estático.
- **Elevación sin umbral (`E_MIN = 0`):** cualquier diferencia mínima activaría
  el ×1.15 de ADR-011, haciendo el factor ruidoso y dependiente de la precisión
  del escenario; el umbral de 5 m da un comportamiento estable y legible.
- **Modelar elevación como otro tipo de terreno:** confunde dos ejes
  ortogonales (cap. 02 define terreno y elevación como **dos** propiedades
  separadas del hex); mantenerlos separados respeta esa forma y permite
  combinarlos libremente.
