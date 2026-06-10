# ADR-024: Generación procedural de mapa MVP

## Estado

Aceptado — 2026-05-20

## Contexto

Hasta este ADR el manual declaraba mapas como "escenarios predefinidos"
(cap. [02 — El Campo de Batalla](../manual/02-campo-de-batalla.md)):
un diseñador hacía el mapa a mano. Esta decisión era provisional;
el proyecto siempre tuvo como objetivo la generación procedural para
garantizar variedad y replayability sin trabajo manual de escenario.

El modelo espacial está cerrado (ADR-023): grilla flat-top radio 20,
1.261 hexes jugables, coordenadas axiales `(q, r)`, distancia
hexagonal. El tipo de terreno sigue el catálogo de ADR-016.

Lo que faltaba formalizar:

- Cómo se genera un mapa válido desde una semilla.
- Qué invariantes geométricos garantiza el generador.
- Quiénes son los bandos, dónde están, y si es elegible.
- Qué es un HQ y cómo termina una partida si es destruido.
- Cómo se colocan las piezas en T0 y qué compone el MVP.
- Qué rol juega A* como contrato de juego (no solo heurística).
- Qué parámetros son tunables vs. qué está fijado.

Este ADR cierra todos esos puntos para el MVP de un solo escenario
procedimental de 2 asientos.

## Decisión

### 1. Invariantes geométricos del tablero

#### 1.1 Borde impasable

La grilla de 1.261 hexes jugables está definida por
`max(|x|, |y|, |z|) ≤ 20` (coordenadas cúbicas, ADR-023 §2).
"Borde impasable" es renderizado decorativo **más allá** de radio 20;
no son hexes del mapa — no tienen coordenadas axiales ni participan
en ningún cálculo de juego.

#### 1.2 Hex central siempre impasable

El hex `(0, 0)` es **siempre impasable**, independientemente del seed.
Es el núcleo del ombligo (§1.3) y garantiza que no haya hex pasable
exactamente en el centro del tablero.

#### 1.3 Ombligo

El **ombligo** es un blob hexagonal centrado en `(0, 0)`.

| Parámetro | Valor | Tunable |
|-----------|-------|---------|
| Tipo | aleatorio por seed: `LAGUNA` o `MONTAÑA` | no |
| Radio ρ | aleatorio por seed ∈ [ρ_min, ρ_max] | sí (ver §6) |

Todos los hexes con `hex_distance((q,r), (0,0)) ≤ ρ` quedan
marcados como impasables del tipo elegido.

En MVP ambos tipos (`LAGUNA`, `MONTAÑA`) son impasables sin distinción
funcional. La diferenciación futura (LOS, modificadores de combate,
propiedades de visión) requerirá un ADR aparte y no rompe este
invariante.

#### 1.4 Cintura probabilística

Para cada hex `(q, r)` **no** ocupado por el ombligo, la densidad
de obstáculos sigue una distribución gaussiana sobre la columna `q`:

```
P(impasable | q, r) = K · (1 − |q| / 20)²
```

- En bordes laterales (`|q| = 20`): P ≈ 0 %.
- En la columna central (`q = 0`): P = K.
- K es un parámetro tunable (ver §6). Default propuesto: `K = 0.3`.

K debe acotarse de modo que la probabilidad de que el generador
produzca un mapa sin camino HQ↔HQ sea inferior a ε ≈ 1e-6. La
cintura debe ser **visualmente rala**; K=0.3 es el punto de
partida para calibración.

#### 1.5 Aserción de conectividad

Tras generar el ombligo y la cintura, el servidor ejecuta A*
(§5) entre HQ Azul y HQ Rojo. Si no existe camino → `seed++` y
regenerar completo. Este caso es ultra-raro con K calibrado;
es una red de seguridad, no una etapa habitual.

### 2. Invariante de bandos (permanente del proyecto)

| Bando | Posición | HQ axial | Cube |
|-------|----------|----------|------|
| **Azul** | Izquierda (`q` mínimo) | `(-20, 10)` | `(-20, 10, 10)` |
| **Rojo** | Derecha (`q` máximo) | `(20, -10)` | `(20, -10, -10)` |

- Eje de simetría = columna `q = 0`.
- Espejo Azul↔Rojo en axial: `(q, r) → (-q, q + r)`.
  En cube: `(x, y, z) → (-x, -z, -y)`.
- El color y el lado son **posicionales, no elegibles**. El jugador
  no elige bando; el asiento determina el bando (§4).
- Esta convención es permanente: ningún diseño futuro puede
  invertirla sin un ADR explícito que la reemplace.

### 3. HQ como pieza-unidad

El HQ **no** es una casilla de terreno ni un marcador estático.
Es una **escuadra-unidad** que ocupa un hex y puede recibir órdenes
`MOVE`, `ATTACK` o `DEFEND` como cualquier escuadra (pool de MP y
stats se definen en la spec de unidad — fuera del scope de este ADR).

- El HQ puede moverse durante la partida.
- El HQ puede combatir.
- La **destrucción del HQ enemigo es condición de victoria
  automática e instantánea**.

Esta condición se añade al conjunto de condiciones de ADR-014 (no lo
reemplaza). ADR-014 sigue siendo la fuente de las demás condiciones
de fin de partida (puntos de victoria, límite de turnos, etc.).

### 4. Asientos MVP

La partida MVP tiene exactamente **2 asientos fijos**:

| Asiento | Bando | Lado |
|---------|-------|------|
| `seat0` | Azul | Izquierda |
| `seat1` | Rojo | Derecha |

Color y lado son posicionales. La composición de fuerzas también
es fija (no hay team-building en MVP); ver §5.3.

### 5. Lifecycle de partida MVP

#### 5.1 Generación de mapa

1. Server genera o recibe semilla `seed` (entero). La semilla se
   registra en la partida para reproducibilidad y replay.
2. Server construye el mapa:
   a. Marca `(0, 0)` como impasable.
   b. Sortea tipo y radio ρ del ombligo desde `seed`; marca todos
      los hexes con `hex_distance ≤ ρ` como impasables.
   c. Para cada hex restante, aplica `P = K · (1 − |q|/20)²`;
      decide independientemente con el RNG de `seed`.
   d. Ejecuta aserción A* HQ Azul `(-20, 10)` ↔ HQ Rojo `(20, -10)`.
      Si no hay camino → incrementa `seed` y vuelve al paso 2.
3. El mapa resultante es canónico para esa partida.

#### 5.2 Colocación de piezas

La colocación es aleatoria determinística (derivada de `seed`),
**independiente por bando** (no es espejo posicional).

**Composición fija MVP por bando:**

| Pieza | Cantidad | Tipo |
|-------|----------|------|
| HQ | 1 | Escuadra-unidad HQ |
| Infantería L3 | 3 | Escuadra Infantería nivel 3 |
| **Total** | **4** | — |

8 piezas en total sobre el tablero en T0.

**Bandas de despliegue:**

| Bando | Columnas `q` válidas | Ancho K_b |
|-------|----------------------|-----------|
| Azul | `[-20, -15]` | 6 (tunable) |
| Rojo | `[+15, +20]` | 6 (tunable) |

**Reglas de colocación por bando:**

1. Exactamente 1 pieza por hex en T0.
2. Saltar hexes impasables.
3. Separación mínima D=1 al HQ propio: los 6 vecinos inmediatos
   del HQ deben quedar libres al finalizar la colocación.
4. El stacking propio se habilita durante la partida (ADR-023 §5),
   no en T0.

#### 5.3 Inicio de turno

No hay fase de despliegue interactivo en MVP. En cuanto el mapa y
las piezas están colocados, **T1 arranca de inmediato**.

#### 5.4 Loop de turno

Briefing → Orders → Resolution (ADR-004), repetido hasta condición
de fin.

#### 5.5 Condiciones de fin

```
fin_partida = condiciones_ADR-014 ∪ {destrucción de HQ enemigo}
```

La primera condición que se satisfaga termina la partida.

### 6. Aleatoriedad permitida (toda determinística por seed)

| Elemento | Determinado por |
|----------|-----------------|
| Tipo de ombligo (`LAGUNA`/`MONTAÑA`) | `seed` |
| Radio ρ del ombligo | `seed` |
| Distribución de obstáculos de cintura | `seed` (P por hex) |
| Posiciones de piezas por banda | `seed` (independiente por bando) |
| Modificadores por soldado (Perks/Traits/Phobias — ADR-013) | `seed` |

La asimetría táctica Azul vs. Rojo emerge de: (a) cintura
asimétrica, (b) ombligo asimétrico, (c) distribuciones de piezas
independientes por bando, (d) modificadores de soldado.
No se exige simetría matemática entre bandos.

### 7. Contrato A* canónico

A* es la referencia de búsqueda de caminos en todos los contextos
(aserción de conectividad §5.1, pathfinding de órdenes MOVE,
predicción de UI en cliente).

**Especificación:**

| Elemento | Valor |
|----------|-------|
| Heurística | `hex_distance(nodo, goal)` (ADR-023 §4) |
| Orden de vecinos | Tabla canónica ADR-023 §3 |
| Costo de arco | 1 por hex pasable (MVP uniforme) |
| Desempate de cola | menor `f` → menor `hex_distance(nodo, goal)` → orden lex `(q, r)` |

**Autoridad:** el servidor corre A* con autoridad. El cliente puede
correr A* predictivo para la UI (highlight de alcance, preview de
ruta), pero el resultado del servidor prevalece (ADR-003).

**Nota forward-looking:** terreno difícil futuro (ej. bosque) sumará
peso al costo de arco pero no romperá conectividad. Los pesos
exactos se definirán en el ADR de terreno correspondiente; este
contrato no cambia.

### 8. Parámetros tunables (deferidos a calibración)

| Parámetro | Default propuesto | Descripción |
|-----------|-------------------|-------------|
| K | 0.3 | Densidad pico de cintura en `q=0` |
| ρ_min | 2 | Radio mínimo del ombligo |
| ρ_max | 5 | Radio máximo del ombligo |
| K_b | 6 | Ancho de banda de despliegue (columnas) |
| D | 1 | Separación mínima HQ ↔ piezas propias en T0 |

Todos los defaults son puntos de partida; requieren calibración
con partidas reales. K en particular debe validarse con análisis
de conectividad antes de release.

## Consecuencias

**Positivas:**

- El mapa es reproducible: dado un `seed`, el resultado es
  idéntico en cualquier máquina — base para replay y bug report.
- La generación es O(n) en el número de hexes: sin iteraciones
  costosas salvo el A* de aserción, que es el caso degenerado.
- La asimetría emergente (cintura, ombligo, colocación
  independiente) da variedad táctica sin trabajo manual de
  diseño de escenario.
- El HQ como pieza-unidad simplifica el modelo de datos: no hay
  entidad "base" separada — es una escuadra con stat especial.
- La condición de victoria por destrucción de HQ es simple,
  dramática y no requiere sistemas adicionales.
- A* con desempate determinístico produce rutas idénticas en
  servidor y cliente predictivo, reduciendo desfase visual.

**Negativas:**

- K = 0.3 es un supuesto no calibrado. Un K demasiado alto puede
  producir mapas con cuellos de botella que eliminan variedad
  táctica; demasiado bajo, mapas sin obstáculos centrales.
- La colocación independiente por bando puede producir
  configuraciones donde un bando queda muy concentrado o muy
  disperso — requiere playtest.
- El retry por `seed++` es correcto pero sin telemetría no se
  conoce la tasa real de rechazo. Si K escala mal, el bucle de
  retry puede volverse costoso.
- Sin fase de despliegue interactivo, los jugadores no tienen
  agencia inicial sobre la composición ni posición — decisión
  de MVP, no limitación del diseño a largo plazo.

## Alternativas descartadas

- **Mapas predefinidos (cap. 02 original):** requerían trabajo
  manual de balance por escenario; incompatible con el objetivo
  de variedad procedural.
- **Simetría matemática de cintura y colocación:** haría los
  mapas predecibles y eliminaría la asimetría táctica que es
  parte del diseño (D1 cerrado: la asimetría es feature, no bug).
- **Ombligo siempre en `(0,0)` pero desplazable por seed:**
  descartado — la columna central ya actúa como eje de simetría
  del invariante de bandos; mover el ombligo rompería la lectura
  geométrica del tablero.
- **HQ como marcador de terreno:** requeriría un tipo especial
  de hex, condiciones de victoria ligadas al terreno y no al
  combate, y no aprovecharía el sistema de escuadras ya definido.
- **Fase de despliegue interactivo en MVP:** añadiría una
  fase de protocolo (ADR-010), lógica de validación de zona y
  UI específica; diferido hasta que el loop básico esté estable.
- **A* no determinístico en desempate:** produciría rutas distintas
  entre cliente y servidor con el mismo mapa, generando artefactos
  visuales en el preview de movimiento.
