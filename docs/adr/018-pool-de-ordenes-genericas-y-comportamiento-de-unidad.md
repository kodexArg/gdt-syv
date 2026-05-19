# ADR-018: Pool de órdenes, órdenes genéricas e IA del oficial

## Estado

Aceptado — 2026-05-18

## Contexto

El capítulo [06 — Órdenes](../manual/06-ordenes.md) define el lazo de control
del jugador: el Comandante dispone de un **pool de órdenes** por turno que
alimentan los oficiales vivos y conectados, y puede gastar ese pool en órdenes
**específicas** (a una unidad) o **genéricas** (a un oficial, que las interpreta
con una IA y deriva sub-acciones a sus escuadras). El capítulo
[08 — Mando y Subordinación](../manual/08-mando-y-subordinacion.md) describe la
cadena L5→L4→L3→L2→L1 y el caso particular de las escuadras especiales L2, que
operan sin radio y reciben **órdenes genéricas** de su oficial de pelotón antes
de entrar en operación.

Cinco huecos del Grupo 6 quedaron como `Pendiente` en esos capítulos y bloquean
la implementación del lazo de mando:

1. **#21** — Cuántas órdenes aporta al pool cada nivel de oficial (L5/L4/L3);
   en el cap. 06 los tres están como `[Pendiente de definir]`.
2. **#26** — Lista concreta de órdenes genéricas para el prototipo y su
   semántica (cap. 06 solo cita ejemplos sueltos: "Ataque a toda costa",
   "Mantenga posición").
3. **#27** — Patrones de comportamiento exactos por tipo de orden genérica y
   por tipo/calidad de oficial que la interpreta (cap. 06: "Patrones de
   comportamiento exactos por tipo de orden genérica: Pendiente de definir").
4. **#29** — Comportamiento por defecto de una unidad **sin orden** que **no**
   estaba defendiendo, moviendo ni atacando (cap. 06 cierra los tres estados
   conocidos pero deja abierto el cuarto caso).
5. **#30** — Momento exacto en que se asignan las órdenes genéricas a las
   escuadras especiales L2: ¿en el Briefing o en la fase Orders? (cap. 08:
   "cuándo exactamente se asignan… ¿en la fase Orders? ¿al inicio del
   Briefing?").

Sin estas reglas el servidor no puede calcular el tamaño del pool por turno,
ni validar `submit_orders` (ADR-010), ni generar las sub-acciones del oficial
en la línea de tiempo de la Resolution, ni resolver la iniciativa de una
escuadra sin orden fresca.

Restricciones que la decisión respeta:

- **ADR-001** (nomenclatura): se usan los términos canónicos L1–L5, HQ,
  Comandante, Escuadra, ATTACK/MOVE/DEFEND, `in_command`, `has_radio` sin
  introducir sinónimos. Las genéricas son una capa sobre las específicas
  canónicas, no una cuarta orden de combate nueva.
- **ADR-004** (ciclo Briefing → Orders → Resolution): toda asignación de orden
  ocurre en la fase **Orders** (cliente, local); el oficial interpreta la
  genérica y deriva sub-acciones **durante la Resolution** (servidor); el
  tamaño del pool del turno se publica en el **Briefing**. Este ADR no añade
  fases ni mensajes nuevos al contrato (ADR-010): la genérica viaja dentro del
  `submit_orders` ya tipado, como una intención más.
- **ADR-012** (cálculo de iniciativa): las sub-acciones derivadas por el
  oficial son órdenes ATTACK/MOVE/DEFEND ordinarias para cada escuadra y usan
  el bono `O` ya tabulado (ATTACK +20 / MOVE +5 / DEFEND −10 / sin-orden −15).
  El comportamiento por defecto de una unidad sin orden (#29) **es** el caso
  `O = −15` de ADR-012 §2.2; este ADR define qué hace esa unidad, no su bono.
  Las especiales L2 conservan `C = +5` (excepción ADR-012 §2.3).
- **ADR-015** (mando/comunicaciones): un oficial solo aporta al pool si está
  **conectado** (`in_command`, BFS de 5 hexes encadenado del cap. 08, con la
  amplificación ×2 de la Tropa de Comunicaciones). Al caer el HQ el pool se
  **congela** (ADR-015 §3): este ADR define el tamaño del pool *mientras
  existe*, no reabre la regla de congelamiento. Las especiales L2 no toman del
  pool (su genérica se entrega fuera de la cadena de radio).
- **ADR-016** (movimiento/terreno): las sub-acciones MOVE derivadas por el
  oficial consumen puntos de movimiento y costos de terreno de ADR-016 como
  cualquier MOVE; este ADR no toca esa economía.

Este ADR es normativo sobre la **regla de juego**. La implementación Godot y
los esquemas viven donde ADR-008/010 los ubican. Todos los valores numéricos
son hipótesis de diseño iniciales **sujetas a playtest**; la estructura de las
reglas es la decisión estable.

## Decisión

### 1. Contribución al pool de órdenes por rango (#21)

Cada oficial **vivo y conectado** (`in_command = true` evaluado por la regla
de 5 hexes encadenada del cap. 08 / ADR-015) aporta un número fijo de órdenes
al pool del Comandante al inicio del turno:

| Nivel | Rol (ADR-001 / cap. 08) | Aporte al pool / turno |
|-------|-------------------------|------------------------|
| **L5** | Teniente de Pelotón | **3** |
| **L4** | Sargento Primero | **2** |
| **L3** | Sargento (líder de escuadra) | **1** |

- El pool del turno es `Σ aporte(o)` sobre todos los oficiales `o` con
  `in_command = true` **al momento del Briefing** (el Briefing publica el
  tamaño disponible; ADR-004 §1). Un oficial vivo pero **desconectado** (fuera
  del BFS, o aislado por L3 muerto — ADR-015) **no aporta** ese turno; al
  reconectarse vuelve a aportar (cap. 06: "el pool es dinámico").
- El gradiente 3/2/1 es deliberado: el mando fluye de arriba hacia abajo, por
  lo que perder un L5 cuesta más pool (y más cobertura, vía sus L4/L3
  subordinados) que perder un L3. Esto refuerza la tesis del juego —
  decapitar la jerarquía superior es estratégicamente más rentable que
  desgastar la base— y es coherente con ADR-015 §3 (al caer el HQ, raíz de
  todo, el pool se congela por completo).
- **Específicas vs. genéricas frente al pool** (cierre operativo del cap. 06):
  - Una **orden específica** (ATTACK / MOVE) consume **1** del pool.
  - **DEFEND no consume del pool** (regla explícita del cap. 06; se mantiene).
  - Una **orden genérica** a un oficial consume **1** del pool y libera al
    Comandante de emitir las específicas que el oficial generará: ese es el
    intercambio de "una acción de mando por múltiples sub-acciones" del
    cap. 06. El número de sub-acciones que el oficial deriva **no** se
    descuenta del pool (ya se pagó con la genérica).
  - El **blackout** (ADR-015 §5.4) y la **caída del HQ** (ADR-015 §3) operan
    *antes* que este conteo: si el pool está congelado, no hay órdenes que
    asignar aunque haya oficiales vivos.

Valores 3/2/1 y "DEFEND gratis": de diseño, **sujetos a playtest**
(alternativa razonable si 3/2/1 hace el pool demasiado holgado: 2/1/1, o
costo 1 también para DEFEND si el atrincheramiento gratuito resulta dominante).

### 2. Catálogo de órdenes genéricas del prototipo (#26)

Para el prototipo se definen **cuatro** órdenes genéricas. Cada una es una
*directiva de intención* dirigida a un oficial; el oficial la traduce a
sub-acciones ATTACK/MOVE/DEFEND para sus escuadras subordinadas (§3). El
parámetro de toda genérica es un **objetivo**: un hex o un eje (dirección)
sobre el tablero.

| Genérica | Símbolo | Semántica (intención que el oficial debe satisfacer) |
|----------|---------|------------------------------------------------------|
| **Asalto** | `ASSAULT(hex)` | Tomar el hex objetivo. El oficial empuja a todas sus escuadras hacia el objetivo y busca el contacto; acepta bajas para progresar. |
| **Asegurar** | `HOLD(hex)` | Mantener el hex/área objetivo. El oficial posiciona sus escuadras en y alrededor del objetivo en postura defensiva y no cede terreno. |
| **Avanzar** | `ADVANCE(hex)` | Reposicionar la fuerza hacia el objetivo **sin** buscar combate. El oficial mueve sus escuadras hacia el hex; ante resistencia se detiene, no fuerza el paso. |
| **Hostigar y replegar** | `HARASS(hex)` | Golpear el objetivo y retirarse. El oficial lanza un ataque limitado contra el área y repliega a sus escuadras hacia su origen; prioriza conservar la fuerza sobre tomar terreno. |

- Cuatro y no más para el prototipo: cubren el cuadrante ofensivo (`ASSAULT`),
  defensivo (`HOLD`), de maniobra (`ADVANCE`) y de incursión (`HARASS`) sin
  solapamientos. Quedan deliberadamente fuera del prototipo directivas más
  finas (escalonamiento, fijar-y-flanquear explícito, repliegue ordenado por
  fases): son refinamientos de §3, no nuevas primitivas.
- Toda genérica se asigna **en la fase Orders** (ADR-004 §2), igual que una
  específica, y viaja en el `submit_orders` (ADR-010). Una vez confirmada **no
  se revoca** (filosofía de compromisos del cap. 06): el oficial la
  interpretará en la Resolution siguiente con el campo tal como esté entonces,
  pudiendo errar (cap. 06: "la orden llega y se ejecuta literalmente sobre un
  campo que ya cambió").

### 3. Patrones de comportamiento por genérica y por oficial (#27)

#### 3.1 Mapeo base genérica → sub-acciones

El oficial, durante la Resolution, recorre sus escuadras subordinadas vivas y
en mando y emite para cada una una sub-acción según la genérica. Patrón base
(oficial de calidad **media**):

| Genérica | Sub-acción base por escuadra |
|----------|------------------------------|
| `ASSAULT(hex)` | `ATTACK` con destino = hex objetivo, para **todas** las escuadras. |
| `HOLD(hex)` | `DEFEND` para la escuadra más próxima al hex; las demás `DEFEND` en su posición actual si ya cubren el área, o `MOVE` un paso hacia el objetivo y luego `DEFEND`. |
| `ADVANCE(hex)` | `MOVE` con path hacia el hex (≤3 waypoints, ADR-016); ninguna `ATTACK`. |
| `HARASS(hex)` | La escuadra más apta (mayor `strength`, cap. 04) `ATTACK` hacia el hex; el resto `MOVE` de regreso hacia el origen del pelotón (repliegue). |

Las sub-acciones resultantes entran en la línea de tiempo de la Resolution
**como órdenes específicas ordinarias** y usan el bono `O` de ADR-012 §2.2
(ATTACK +20 / MOVE +5 / DEFEND −10) por escuadra, sin tratamiento especial.

#### 3.2 Modulación por calidad del oficial

La calidad de interpretación depende del oficial (cap. 06: "un oficial con
mejor entrenamiento puede tomar decisiones más inteligentes; uno inexperto
puede interpretar mal"). Se modela con un único escalar derivado de stats
canónicos de ADR-013 del oficial que interpreta:

```
q(oficial) = clamp01( ( VAL + RCT + (100 - NRV) ) / 300 )
```

donde `VAL` (Valor), `RCT` (Reacción) y `NRV` (Nervio, donde más alto = peor)
son stats canónicos de ADR-013, normalizados 0–100. `q ∈ [0,1]`: oficial
mediocre cerca de 0, oficial excelente cerca de 1.

A partir de `q` se aplica una **tirada de interpretación** server-side, una
por genérica, semilla del turno (determinista, coherente con la verificación
del Briefing de ADR-004, igual que `R` en ADR-012 §2.5):

- **Éxito** (prob. `0.4 + 0.5·q`): el oficial aplica el patrón base de §3.1
  de forma **competente** — escuadras heridas/ROUTED quedan en `DEFEND` en
  vez de ser empujadas a `ATTACK` suicida; el reparto respeta cobertura
  (ADR-016) y evita cruzar hexes `impassable`.
- **Fallo** (prob. `1 − (0.4 + 0.5·q)`): el oficial **malinterpreta**. Efecto
  determinista por genérica (no aleatorio adicional, para reproducibilidad):

  | Genérica | Malinterpretación al fallar |
  |----------|------------------------------|
  | `ASSAULT` | Empuja **todas** las escuadras al hex en línea recta ignorando cobertura y bajas (asalto frontal torpe). |
  | `HOLD` | Sobre-extiende: una escuadra avanza fuera del área a `ATTACK` y abandona la cobertura del objetivo. |
  | `ADVANCE` | Convierte el avance en `ATTACK` de la escuadra de punta (provoca el contacto que la directiva quería evitar). |
  | `HARASS` | Compromete de más: dos escuadras `ATTACK` y el repliegue se retrasa (la fuerza queda expuesta). |

- Un oficial **desconectado** (fuera de mando, ADR-015) no puede interpretar
  genéricas nuevas: sus escuadras caen al comportamiento sin orden de §4
  (`O = −15` en ADR-012). Las especiales L2 son un caso aparte (§5).

Coeficientes `0.4 + 0.5·q`, pesos `VAL/RCT/NRV` y los efectos de fallo: de
diseño, **sujetos a playtest** (alternativa: incluir `TGH` o un trait de
"mando" si ADR-013 lo expone; la estructura tirada-por-calidad es estable).

### 4. Comportamiento por defecto de una unidad sin orden (#29)

El cap. 06 cierra tres estados (DEFENDIENDO→persiste, MOVIENDO→se detiene,
ATACANDO→cesa). Falta el cuarto: una escuadra que **no estaba en ninguno de
esos tres estados** (recién desplegada, o cuyo único estado previo ya se
extinguió) y que **no recibe orden** este turno.

Regla de cierre: **una escuadra sin orden y sin estado activo/pasivo previo
adopta DEFEND pasivo en su hex actual** ("aguantar en posición"), con estas
precisiones:

- Es **DEFEND degradado**, no el DEFEND ordenado del cap. 06: **no acumula
  bonus de fortificación** (ADR-016 §, fortificación exige DEFEND *ordenado*
  en turnos consecutivos). Solo mantiene posición y reacciona si la alcanzan.
- Para la iniciativa (ADR-012) la escuadra usa `O = −15` (caso "sin orden /
  orden vieja"), **no** el `O = −10` de DEFEND ordenado: no hay impulso de
  mando reciente, así que actúa aún más tarde que una que defiende por orden.
- No gasta puntos de movimiento (ADR-016: el movimiento exige MOVE activa) ni
  inicia combate ofensivo (ATTACK exige orden, cap. 06).
- Generaliza el principio del cap. 06 ("los estados pasivos persisten, los
  activos se extinguen") al estado nulo: en ausencia de mando una unidad
  **se aferra al terreno**, nunca avanza ni ataca por iniciativa propia. Es
  coherente con la tesis de subordinación (sin orden no hay acción ofensiva)
  y con ADR-012 (la pasividad por falta de mando se paga en la línea de
  tiempo).
- Excepción: una escuadra **ROUTED** (cap. 04/09) ignora esta regla y se
  mueve descoordinada según ADR-012 §2.4 (`M = −25`); el DEFEND por defecto
  aplica solo a escuadras ACTIVE/RETREAT.

### 5. Momento de asignación de la genérica a escuadras especiales L2 (#30)

Las escuadras especiales L2 (Los Infernales, commandos, zapadores) operan sin
radio y fuera de la cadena (cap. 08; ADR-015 §3, ADR-012 §2.3, `C = +5`).
Reciben **una sola orden genérica** de su oficial de pelotón "antes de entrar
en operación" (cap. 08). Se fija el momento exacto:

- **La genérica a una escuadra especial L2 se asigna en la fase Orders**
  (ADR-004 §2), igual que cualquier otra orden, y viaja en el mismo
  `submit_orders` (ADR-010). **No** se asigna en el Briefing: el Briefing es
  fase de servidor de solo-lectura para el jugador (ADR-004 §1), no acepta
  intenciones.
- **No consume del pool de órdenes** (§1): la genérica de una especial L2 se
  entrega fuera de la cadena de radio (cara a cara con el oficial de pelotón
  antes de la incursión, cap. 08), no por la economía de mando del Comandante.
  Es una asignación de intención previa al despliegue, no una orden de mando
  del turno.
- **Persistencia:** una vez asignada, la genérica de la especial L2
  **persiste a través de los turnos** hasta que el jugador, en una fase Orders
  posterior **y solo si la escuadra está a ≤2 hexes de su oficial de pelotón
  o del HQ** (mismo umbral de contacto por proximidad de ADR-015 §2), emita
  una genérica de reemplazo. Mientras esté en operación profunda (sin esa
  proximidad) **no se le puede reasignar**: ejecuta la última genérica recibida
  turno tras turno con su autonomía entrenada (cap. 08; ADR-012 `C = +5`),
  incluso si el campo cambió (cap. 08, ejemplo de Los Infernales).
- **Primera asignación:** en el escenario, la genérica inicial de cada especial
  L2 se fija en la **primera fase Orders** (turno 1). No existe asignación en
  el despliegue/setup salvo que un ADR de escenario (ADR-014) lo establezca;
  este ADR no lo presupone.

Umbral de ≤2 hexes para reasignación: reutiliza deliberadamente el de
ADR-015 §2 por coherencia (mismo concepto de "contacto recuperado por
proximidad"); de diseño, **sujeto a playtest**.

## Consecuencias

**Positivas:**

- Cierra los bloqueantes #21, #26, #27, #29, #30: el servidor puede computar
  el tamaño del pool por turno desde el Briefing (ADR-004), validar
  `submit_orders` con genéricas (ADR-010), derivar sub-acciones deterministas
  en la Resolution y resolver la iniciativa de toda escuadra (incluida la sin
  orden) con ADR-012 sin huecos.
- El gradiente de pool 3/2/1 refuerza la tesis del juego de forma consistente
  con ADR-015 §3 (decapitar arriba cuesta más que desgastar abajo).
- Las sub-acciones del oficial reutilizan ATTACK/MOVE/DEFEND y el bono `O` de
  ADR-012 sin tipos nuevos: las genéricas son una capa de orquestación, no una
  cuarta primitiva de combate.
- El comportamiento sin orden (DEFEND degradado, `O = −15`) generaliza el
  principio del cap. 06 sin contradecir ADR-012, y mantiene "sin mando no hay
  ofensiva", núcleo temático del juego.
- Fijar la genérica L2 en Orders (no en Briefing) mantiene la simetría de
  ADR-004 (Briefing = servidor solo-lectura) y evita un canal de asignación
  excepcional; la persistencia + umbral ≤2 hexes reutiliza ADR-015 sin
  inventar mecánica nueva.

**Negativas:**

- Introduce coeficientes sin calibrar (3/2/1, `0.4 + 0.5·q`, pesos
  VAL/RCT/NRV, ≤2 hexes para reasignación L2): la sensación táctica real
  depende de playtest; todos marcados como tales.
- La tirada de interpretación del oficial añade una segunda fuente de azar
  determinista en la Resolution (además de `R` de ADR-012); el servidor debe
  derivarla de la misma semilla del turno para no romper la verificación de
  coherencia del Briefing (ADR-004).
- El "DEFEND degradado" sin fortificación es un cuarto estado de orden de
  facto que el servidor debe distinguir del DEFEND ordenado a efectos de
  fortificación (ADR-016) aunque comparta resolución de combate; matiz extra
  de estado por escuadra.
- `q(oficial)` depende de stats de ADR-013 (`VAL/RCT/NRV`); si esos stats se
  renombran, el mapeo es por rol y la fórmula sigue válida, pero acopla este
  ADR a la nomenclatura de stats.

## Alternativas descartadas

- **Pool plano (cada oficial aporta lo mismo, p. ej. 2):** no expresa que la
  jerarquía superior vale más; contradice el espíritu de ADR-015 §3 (la caída
  del HQ, raíz, congela todo). El gradiente 3/2/1 hace la decapitación
  estratégicamente legible.
- **Pool fijo por turno independiente de oficiales vivos:** rompe el cap. 06
  ("el pool es dinámico y depende de los oficiales vivos y conectados") y
  elimina el incentivo de proteger y conectar la cadena de mando.
- **Genéricas como nuevas primitivas de combate (no derivadas a
  ATTACK/MOVE/DEFEND):** duplicaría la resolución de combate y la tabla `O`
  de ADR-012; la capa de orquestación sobre las tres primitivas es más simple
  y fiel al cap. 06 ("el efecto es como si el Comandante hubiera emitido
  múltiples órdenes específicas").
- **Lista grande de genéricas (8+) en el prototipo:** sobredimensiona el
  espacio de interpretación de la IA antes de tener playtest; cuatro
  cuadrantes ortogonales bastan para validar el lazo y se extienden sin
  reabrir el ADR.
- **Interpretación del oficial puramente determinista (sin tirada de
  calidad):** desaprovecha el riesgo narrativo del cap. 06 ("el oficial puede
  interpretar mal"); la tirada modulada por stats de ADR-013 da gradiente sin
  romper el determinismo del Briefing (semilla del turno).
- **Unidad sin orden = espera pasiva inerte:** contradice el cap. 06
  explícitamente ("no entra en espera pasiva"). DEFEND degradado mantiene a
  la unidad reactiva pero sin premiarla con fortificación que no ordenó.
- **Unidad sin orden mantiene su última orden ofensiva:** contradice el
  cap. 06 ("los estados activos/ofensivos se extinguen"); sería además un
  agujero en la tesis de subordinación (acción ofensiva sin mando).
- **Asignar la genérica L2 en el Briefing:** rompe ADR-004 §1 (Briefing es
  servidor, solo-lectura para el jugador, no acepta intenciones); además
  obligaría a un canal de asignación excepcional. La fase Orders es el único
  punto donde el jugador emite intenciones (ADR-004 §2).
- **Permitir reasignar la genérica L2 en cualquier momento sin proximidad:**
  contradice el cap. 08 (las especiales "no pueden recibir nuevas
  instrucciones durante esa operación"); el umbral ≤2 hexes reutiliza la
  excepción de contacto por proximidad de ADR-015 §2 sin inventar regla nueva.
