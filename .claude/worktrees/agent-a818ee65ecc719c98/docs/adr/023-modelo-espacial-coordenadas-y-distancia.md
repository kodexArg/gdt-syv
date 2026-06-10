# ADR-023: Modelo espacial — coordenadas y distancia

## Estado

Aceptado — 2026-05-19

## Contexto

El tablero hexagonal está definido en el cap.
[02 — El Campo de Batalla](../manual/02-campo-de-batalla.md): grilla
flat-top, radio 20, 1.261 hexes, escala 100 m/hex. Lo que no estaba
fijado hasta ahora es el **sistema de coordenadas canónico** que
identifica cada hex, la métrica de distancia entre ellos y la
granularidad espacial de las unidades sobre ese tablero.

Esa omisión genera ambigüedad en:

- El protocolo de red ([ADR-010](010-contrato-de-protocolo-de-red.md)):
  ¿qué par de enteros identifica un hex en los mensajes?
- El modelo de datos ([ADR-009](009-modelo-de-datos-fuerza-militar.md)):
  ¿qué almacena `position` de una escuadra?
- Las reglas de alcance ([ADR-011](011-resolucion-de-combate-por-soldado.md),
  ADR-016) y mando ([ADR-015](015-mando-comunicaciones-y-triangulacion.md)):
  ¿cuántos "pasos" separan dos hexes?
- El sistema de visión (cap. 10, pendiente): necesita un sistema de
  coordenadas estable para el trazado de línea de visión.

Restricciones de coherencia relevantes:

- La grilla es flat-top, radio 20 — cap. 02. Este ADR **no** cambia
  la forma del tablero, solo la nombra con precisión.
- Una escuadra ocupa exactamente 1 hex; stacking propio ilimitado —
  cap. [02b](../manual/02b-distancias.md) §Ocupación de hexes. Este
  ADR fija la granularidad, no la cambia.
- "Distancia Manhattan" aparecía en menciones informales internas como
  sinónimo de "pasos entre hexes". La terminología es incorrecta en una
  grilla hexagonal y este ADR la reemplaza.

## Decisión

### 1. Sistema de coordenadas canónico: axial (q, r)

El identificador canónico de un hex es el par de enteros `(q, r)` en
el sistema **axial** (también llamado "trapezoidal").

El origen `(0, 0)` es el hex central del tablero. Los ejes crecen en
las direcciones axiales estándar de una grilla flat-top:

```
  q crece hacia el este (derecha)
  r crece hacia el sureste
```

**Rango válido en prototipo:** todo par `(q, r)` tal que su
representación cúbica satisfaga `max(|x|,|y|,|z|) ≤ 20`
(radio 20, ver §2).

Las coordenadas axiales se usan en todos los contextos:
almacenamiento, protocolo de red, mensajes de log, API interna.

### 2. Coordenadas cúbicas — derivadas, no almacenadas

Para cálculos geométricos (distancia, vecinos, rotaciones) se
usa la representación **cúbica** `(x, y, z)`, derivada on-the-fly
de axial:

```
x = q
z = r
y = -x - z          (invariante: x + y + z = 0)
```

Las coordenadas cúbicas **no se almacenan ni se transmiten**; se
calculan cuando se necesitan y se descartan. La representación
canónica persistida es siempre axial.

### 3. Vecinos en grilla flat-top

Los 6 hexes adyacentes a `(q, r)` en axial, en sentido horario
desde el este:

```
Dirección       Δq   Δr
────────────────────────
Este            +1    0
Noreste         +1   -1
Noroeste         0   -1
Oeste           -1    0
Suroeste        -1   +1
Sureste          0   +1
```

Verificación: cada par `(Δq, Δr)` tiene distancia hexagonal = 1
(§4). La suma de los seis deltas es `(0, 0)` — simetría correcta.

### 4. Métrica canónica: distancia hexagonal

La distancia entre dos hexes `A` y `B` es el número mínimo de
pasos entre hexes adyacentes necesarios para ir de `A` a `B`.

En coordenadas cúbicas:

```
hex_distance(A, B) =
    (|x_A - x_B| + |y_A - y_B| + |z_A - z_B|) / 2
```

Equivalente en axial (sin conversión explícita):

```
hex_distance(A, B) =
    max(|Δq|, |Δr|, |Δq + Δr|)
```

Ambas formas son equivalentes. La forma cúbica es más legible en
las fórmulas; la forma axial evita crear variables intermedias.

**Terminología:** el término "distancia Manhattan" queda
**descartado** para esta métrica. La distancia Manhattan es
`|Δx| + |Δy|` en cuadrícula ortogonal; en hexagonal no es
equivalente. El término correcto es **distancia hexagonal** o
**distancia en hexes**.

Ejemplos:

| A | B | dist |
|---|---|------|
| (0,0) | (3,0) | 3 |
| (0,0) | (2,-3) | 3 |
| (0,0) | (-1,2) | 2 |
| (5,5) | (5,5) | 0 |

### 5. Granularidad espacial de unidad (prototipo)

Una **escuadra** ocupa exactamente **1 hex**, identificado por un
par axial `(q, r)`. Es la unidad mínima de presencia en el mapa.

El **soldado** no tiene sub-posición dentro del hex. No existe
ningún campo `(sx, sy)` ni coordenada intra-hex en el modelo de
datos. El soldado existe solo como entrada lógica de la escuadra
(composición, stats, estado de heridas); su posición espacial es
la del hex de su escuadra.

Stacking:

- **Propias:** ilimitado — múltiples escuadras del mismo bando
  pueden compartir hex (cap. 02b §Ocupación de hexes).
- **Enemigas:** permitido; activa reglas de combate en el mismo
  hex (cap. 02b §Ocupación de hexes).

**Diferido:** footprint físico del soldado (formación intra-hex,
dispersión táctica) y sub-posición de escuadra. Se pospone hasta
que el sistema de animación / resolución intra-hex lo requiera.

## Consecuencias

**Positivas:**

- Un único sistema de coordenadas en todo el stack: almacenamiento
  (ADR-009), protocolo (ADR-010), reglas (ADR-011, ADR-015,
  ADR-016, futuro cap. 10) y log hablan el mismo idioma.
- El par axial `(q, r)` es mínimo (2 enteros) y suficiente; la
  forma cúbica se deriva sin almacenamiento extra.
- La fórmula de distancia hexagonal es O(1), sin búsqueda de
  caminos ni estado extra.
- La terminología correcta ("distancia hexagonal") evita bugs de
  implementación que surgen de confundir Manhattan con hex-distance.
- La ausencia de sub-posición de soldado mantiene el modelo de
  datos plano y el protocolo simple — coherente con el alcance del
  prototipo.

**Negativas:**

- El par `(q, r)` no es inmediatamente intuitivo para diseñadores
  de escenarios que vienen de cuadrículas ortogonales; se necesita
  documentación de referencia (la tabla de vecinos de §3 y los
  ejemplos de §4 son ese material).
- Sin sub-posición intra-hex, no se puede modelar formación ni
  dispersión táctica de soldados — aceptable en prototipo, pero
  limita el nivel de detalle visual de la resolución. Cuando se
  implemente, requerirá un ADR nuevo que extienda el modelo de
  datos de ADR-009.
- La elección de origen en el centro `(0,0)` implica que los hexes
  del borde tienen coordenadas negativas en algunos ejes — normal
  en axial, pero hay que asegurarse de que el serializador de
  escenarios admita enteros negativos.

## Alternativas descartadas

- **Offset coordinates (par/impar):** ampliamente usadas en
  motores de juego por su apariencia "natural" (el hex de la
  esquina superior izquierda es (0,0)). Descartadas porque la
  fórmula de distancia y el cálculo de vecinos requieren casos
  especiales según la paridad de la fila/columna, lo que introduce
  bugs sutiles. Axial es algebraicamente limpio.

- **Coordenadas cúbicas como canónicas:** triple de enteros en
  lugar de par, con el invariante `x+y+z=0` que hace redundante
  un campo. Descartado: mayor costo de serialización y más campos
  que sincronizar; axial es la proyección mínima de cúbico.

- **Distancia Euclidiana en píxeles/metros:** útil para
  renderizado, irrelevante para las reglas (el juego es discreto
  por hex). Las reglas operan sobre pasos, no sobre distancias
  continuas.

- **Sub-posición de soldado en prototipo:** requeriría extender
  ADR-009 (campo `sub_pos` por soldado), el protocolo ADR-010
  (transmitir posiciones intra-hex) y el motor de resolución.
  El valor táctico en prototipo es mínimo frente al costo de
  implementación; diferido explícitamente.
