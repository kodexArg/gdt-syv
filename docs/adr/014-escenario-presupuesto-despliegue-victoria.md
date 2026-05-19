# ADR-014: Escenario — presupuesto, despliegue, schema y condiciones de victoria

## Estado

Aceptado — 2026-05-18

## Contexto

El manual deja abiertos, de forma explícita y dispersa, los parámetros que
definen una partida completa de SyV:

- **Cap. 03b (Composición de Fuerzas)**: "Pendiente de definir: valores
  específicos por dificultad". Habla de partidas pequeñas (20–30 pts),
  medianas (50–70 pts) y grandes (100+ pts), pero los archivos
  `docs/premade-squads/*.yaml` ya tienen costes (`cost`) en un rango muy
  distinto (50–230 por escuadra). Hay **dos escalas numéricas
  incompatibles** sin conversión definida.
- **Cap. 02 (Campo de Batalla)**: el escenario define tablero, hexes
  objetivo, zonas de despliegue, límite de turnos y presupuesto, pero
  "Formato de escenario: Pendiente de definir (estructura, serialización,
  validación)".
- **Cap. 05 (El Turno)**: despliegue simultáneo y oculto, validado por el
  servidor; pendiente "tamaño y forma de zonas de despliegue, reglas de
  colocación (formaciones, distancia mínima, apilamiento), límite de tiempo
  de despliegue".
- **Cap. 11 (Victoria)**: dos condiciones confirmadas (captura de HQ
  enemigo; control de objetivos al límite de turnos) pero objetivos por
  escenario, valor de puntos por objetivo y "mecanismo de control exacto:
  pendiente de definir".

Sin estos valores cerrados no se puede serializar un escenario jugable ni
implementar la fase de despliegue (ADR-004), su validación (ADR-005,
ADR-010 `submit_deployment`) ni la condición de fin de partida. Este es el
último bloque transversal de parámetros pendientes (Grupo 8, decisiones
#39–#47, incluida la conversión de escala #40).

Este ADR fija valores normativos concretos. No define la implementación
Godot (vive en `shared/`/`server/`), ni las fórmulas de
combate/iniciativa/strength (ADR-009, ADR-011, ADR-012), ni el editor de
mapas (fuera de alcance, cap. 02).

## Decisión

### 1. Escala de puntos única y conversión (#40)

Existe **una sola escala de puntos** en SyV: la del campo `cost` de los
YAML premade (`docs/premade-squads/*.yaml`), rango actual observado 50–230
por escuadra. Esa escala es canónica y se denomina **puntos de fuerza
(PF)**.

El "20–70 / 100+" del cap. 03b corresponde a un borrador temprano (orden de
magnitud de *cantidad de escuadras*, no de coste real) y queda
**reinterpretado, no conservado**: el presupuesto de escenario se expresa
siempre en PF, la misma unidad que `cost`. No se introduce factor de
conversión ni segunda unidad ("puntos de escenario" deja de existir como
escala separada). Regla operativa de selección (cap. 03b §"Selección de
fuerzas"): la suma de `cost` de las escuadras elegidas por un jugador debe
ser **≤ presupuesto**; no se exige agotarlo.

### 2. Presupuesto por dificultad (#39)

El escenario declara `budget_pf` por facción. Valores normativos:

| Dificultad / tamaño | `budget_pf` por facción | Escuadras típicas | Referencia |
|---|---|---|---|
| **Pequeño** (escaramuza) | **300** | 2–4 | 1 infantería estándar (200) + 1 recon (50) + margen |
| **Medio** (estándar) | **600** | 4–6 | base esperada de playtest |
| **Grande** (operación) | **1000** | 7–10 | partidas largas |

El presupuesto **puede ser asimétrico** por facción (el escenario lo fija
explícitamente; cap. 02 — balance manual). Si el escenario no declara
`budget_pf`, el valor por defecto es **600** (Medio). Una escuadra de mando
de pelotón (`type: command`, `cost` ~110) es **obligatoria**: toda fuerza
desplegada debe incluir al menos un HQ de pelotón; el resto es libre dentro
del presupuesto. Los números de la tabla son **sujetos a playtest** (la
escala PF y la regla ≤ presupuesto no lo son).

### 3. Composición / equipamiento custom — alcance prototipo (#47)

Para el prototipo **solo se permiten escuadras premade** del catálogo de
la facción (`docs/premade-squads/{conf,rojos}-*.yaml`). **No** se permite:
modificar composiciones, agregar/cambiar equipamiento, crear escuadras
custom, ni bonificadores de experiencia/desgaste entre escenarios (cap. 03b
§Personalización queda cerrado como "no, en prototipo"). Repetir una misma
escuadra premade varias veces **sí** está permitido (cada instancia recibe
un `id` de instancia distinto; el `id` del YAML es el tipo). La
personalización es trabajo futuro post-prototipo y no condiciona este
schema (el schema ya soporta referenciar tipos por id, suficiente para
extenderlo luego).

### 4. Zonas de despliegue y reglas de colocación (#43–#45)

- **Forma**: cada zona de despliegue es un **conjunto explícito de
  HexCoord** listado en el escenario (no una fórmula geométrica). Esto
  permite formas arbitrarias y balance manual (cap. 02). Cada hex pertenece
  a lo sumo a una zona; las zonas de facciones distintas no se solapan.
- **Tamaño recomendado**: la zona debe contener al menos **1.5×** el número
  máximo plausible de escuadras de su dificultad (Pequeño ≥ 6 hex, Medio
  ≥ 10, Grande ≥ 15). Recomendación de diseño, validable; no es un error
  duro si el escenario justifica menos.
- **Apilamiento**: **una escuadra por hex** (coherente con ADR-009:
  Escuadra ocupa exactamente 1 hex). El servidor rechaza dos escuadras en
  el mismo hex.
- **Distancia mínima**: ninguna entre escuadras propias más allá del
  no-apilamiento (las escuadras de un pelotón se despliegan agrupadas por
  diseño, cap. 02; forzar separación contradice la doctrina de
  subordinación). Las zonas enemigas no se solapan, así que no hay
  proximidad entre facciones en despliegue.
- **Formaciones**: sin sistema de formaciones en prototipo. La colocación
  es hex-por-hex libre dentro de la zona propia.
- **HQ estratégico**: el hex del Comandante (`Seccion.hq_hex`, ADR-009) lo
  fija el escenario, **no** el jugador, y queda **fuera** de la zona de
  despliegue libre (es posición fija, cap. 02). Debe estar dentro o
  adyacente a la zona propia.
- **Colocación obligatoria**: toda escuadra seleccionada debe quedar
  colocada dentro de la zona propia; un despliegue con escuadras sin
  colocar o fuera de zona es inválido y el servidor lo rechaza (ADR-010
  `submit_deployment`, revalidación ADR-005).

### 5. Límite de tiempo de despliegue (#46)

El despliegue tiene un **timeout de 180 s** por jugador (declarable por
escenario vía `deployment_timeout_s`, default 180). Al expirar:

- Si el jugador no envió `submit_deployment`: se aplica un **despliegue por
  defecto** — las escuadras se autocolocan en los primeros hexes libres de
  la zona propia en orden de lectura del escenario; si no seleccionó
  fuerzas, se asigna la composición mínima (1 HQ de pelotón del catálogo).
- El servidor nunca queda bloqueado esperando a un jugador (coherente con
  WEGO simultáneo, cap. 05; el listen server no introduce excepción).

Valor sujeto a playtest; el principio "el turno no se bloquea" no lo es.

### 6. Objetivos y mecanismo de control → victoria (#41–#42)

Condiciones de victoria (cap. 11), sin cambios en su jerarquía:

1. **Captura de HQ enemigo**: victoria inmediata (sin cambios).
2. **Control de objetivos** al alcanzar `turn_limit`: gana quien controla
   más objetivos; este ADR cierra el mecanismo.

Definiciones normativas:

- El escenario declara una lista `objectives`, cada uno con `hex`
  (HexCoord) y `value` (entero ≥ 1, default 1).
- **Control de un objetivo** (resuelto al cierre de cada Resolution, ADR-004):
  - **Presencia sin oposición**: un objetivo está *controlado por la
    facción F* si, al cierre de la Resolution, hay al menos una escuadra de
    F en estado `ACTIVE` (ADR-009 `UnitState`) **sobre el hex objetivo o en
    un hex adyacente**, y **ninguna** escuadra enemiga `ACTIVE` sobre el hex
    o adyacente. Escuadras `ROUTED`/`RETREAT`/`ELIMINATED` no controlan ni
    disputan.
  - **Disputado**: si ambas facciones tienen escuadra `ACTIVE` en el radio
    (hex + adyacentes), el objetivo queda **disputado** y no cuenta para
    nadie ese turno. **No es captura por umbral de tropas**: la regla es
    presencia/oposición binaria (más simple y auditable; el umbral de
    tropas queda descartado, ver Alternativas).
  - **Persistencia**: el control persiste mientras se mantenga la condición;
    si la facción controladora pierde presencia y nadie más entra, el
    objetivo vuelve a **neutral** (no controlado por nadie).
- **Resultado de partida** al `turn_limit`: se suma `value` de los
  objetivos controlados por cada facción. Mayor suma gana. **Empate** →
  empate registrado (no hay desempate por bajas; cap. 11: el juego no
  premia aniquilación). La captura de HQ enemigo, si ocurre antes, prevalece
  e ignora el conteo de objetivos.
- `turn_limit` lo declara el escenario (entero ≥ 1, sin default — campo
  obligatorio; cap. 02/11). Valores de referencia sujetos a playtest:
  Pequeño ~8, Medio ~15, Grande ~25 turnos.

### 7. Formato, serialización y validación de escenario (#43)

- **Serialización**: **YAML**, coherente con `docs/premade-squads/*.yaml`
  y con `_template.yaml`. Un escenario es un único archivo
  `docs/scenarios/<id>.yaml`.
- **Esquema canónico** (campos `(!)` obligatorios):

```yaml
scenario:
  id: ""                       # (!) string único, kebab-case
  name: ""                     # (!) display
  schema_version: 1            # (!) entero; este ADR define la versión 1
  grid:                        # (!) coherente con cap. 02 / ADR-009
    radius: 20                 # (!) flat-top, radio 20 (1.261 hex)
  turn_limit: 0                # (!) entero >= 1
  deployment_timeout_s: 180    # (?) default 180
  factions:                    # (!) exactamente 2 entradas: confederacion, rojos
    - id: "confederacion"      # (!) ID de facción (ADR-001)
      budget_pf: 600           # (!) presupuesto en PF (ver §1–§2)
      hq_hex: [q, r]           # (!) posición fija del Comandante (ADR-009)
      deployment_zone:         # (!) lista explícita de hexes
        - [q, r]
    - id: "rojos"
      budget_pf: 600
      hq_hex: [q, r]
      deployment_zone:
        - [q, r]
  terrain:                     # (?) overrides; default todo "pasto", elev 0.0 (cap. 02)
    - hex: [q, r]
      type: "agua"             # pasto | agua (cap. 02, extensible)
  objectives:                  # (!) >= 0 entradas (lista vacía válida si solo HQ-win)
    - hex: [q, r]              # (!)
      value: 1                 # (?) entero >= 1, default 1
```

- **Validación** (servidor, en carga de escenario y en
  `submit_deployment`):
  1. `schema_version` reconocido (== 1) — error si no.
  2. Exactamente 2 facciones, ids ∈ {`confederacion`, `rojos`}, distintas.
  3. Todos los hexes (`hq_hex`, zonas, `terrain`, `objectives`) dentro de
     la grilla de radio 20 y referenciados con HexCoord válido.
  4. Hexes de objetivos y de despliegue **no** sobre `terrain: agua`
     (intransitable, cap. 02); si lo están → error.
  5. Zonas de despliegue de ambas facciones **disjuntas** entre sí.
  6. `hq_hex` de cada facción dentro o adyacente a su zona, y no en agua.
  7. `turn_limit >= 1`; `budget_pf >= 1`; cada `objectives[].value >= 1`.
  8. En `submit_deployment`: cada escuadra existe en el catálogo de su
     facción, Σ`cost` ≤ `budget_pf`, ≥ 1 escuadra `command`, todas las
     escuadras dentro de la zona propia, sin dos escuadras en el mismo hex.
  Un escenario o despliegue que falle cualquier regla es **rechazado**; el
  servidor no intenta auto-reparar (salvo el despliegue por defecto del
  §5 ante timeout, que es una ruta distinta del rechazo por inválido).

## Consecuencias

**Positivas:**

- Cierra el último bloque transversal de pendientes (cap. 02, 03b, 05, 11):
  un escenario es ahora serializable, validable y jugable de extremo a
  extremo.
- Una sola escala de puntos (PF) elimina la ambigüedad #40 sin tocar los
  ~16 YAML existentes ni el campo `cost` (ADR-009/template intactos).
- Mecanismo de control de objetivos binario (presencia/oposición):
  auditable, barato de computar al cierre de cada Resolution (ADR-004),
  sin acoplarse a `strength`/`moral` aún pendientes (ADR-009).
- Zonas como lista explícita de hexes habilitan balance manual (cap. 02)
  sin motor de geometría.
- El timeout con despliegue por defecto preserva el invariante WEGO de que
  el servidor nunca se bloquea (cap. 05, ADR-004).
- Schema versionado (`schema_version`) permite evolucionar (custom futuro)
  sin romper escenarios v1.

**Negativas:**

- Reinterpretar el "20–70 / 100+" del cap. 03b crea una discrepancia
  textual con ese capítulo hasta que el manual se actualice (fuera de
  alcance de este ADR por restricción de tarea); este ADR es la fuente
  normativa mientras tanto.
- Listas explícitas de hexes hacen los YAML de escenario verbosos; aceptable
  para un set curado de mapas (cap. 02: sin generación procedural).
- Los valores numéricos (presupuestos, turn_limits, timeout, tamaños de
  zona) son conjeturas iniciales sujetas a playtest; solo la escala PF, la
  regla Σ`cost` ≤ presupuesto, el no-apilamiento y la semántica de control
  son invariantes.
- Prohibir composición custom en prototipo limita la rejugabilidad temprana;
  decisión deliberada de alcance (#47) para acotar superficie.

## Alternativas descartadas

- **Mantener dos escalas con factor de conversión** (PF ↔ "puntos de
  escenario"): añade aritmética y una fuente de error sin beneficio; los
  YAML ya fijan una escala usable. Se prefiere reinterpretar el manual.
- **Reescalar los `cost` de los 16 YAML al rango 20–70 del manual**:
  tocaría datos ya consumidos por ADR-009/template y por otros bloques;
  más costoso y frágil que adoptar la escala existente.
- **Control de objetivo por umbral de tropas** (p. ej. "≥ N tropas vivas"):
  acopla la victoria a `strength`/conteo de tropas (fórmula pendiente,
  ADR-009) y es más difícil de auditar; la presencia-sin-oposición binaria
  es suficiente y estable.
- **Zona de despliegue como rectángulo/parámetro geométrico**: insuficiente
  para mapas asimétricos balanceados a mano (cap. 02); la lista explícita
  es más expresiva y trivial de validar.
- **Distancia mínima entre escuadras propias**: contradice la doctrina de
  subordinación (escuadras agrupadas por pelotón, cap. 02/ADR-009) y
  complica el despliegue sin aportar al prototipo.
- **Bloquear el turno hasta recibir ambos despliegues**: rompe el invariante
  WEGO (cap. 05); el despliegue por defecto ante timeout lo evita.
- **Permitir composición custom en prototipo**: amplía la superficie de
  balance y validación antes de tener datos de playtest; diferido.
- **Serializar escenarios en JSON**: rompe consistencia con
  `premade-squads/*.yaml` y `_template.yaml` sin ganancia.
