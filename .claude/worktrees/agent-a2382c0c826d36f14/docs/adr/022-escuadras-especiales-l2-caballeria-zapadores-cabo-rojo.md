# ADR-022: Escuadras especiales L2 — caballería, zapadores, Cabo Rojo y reasignación de la orden genérica

## Estado

Aceptado — 2026-05-18

## Contexto

El eje de escuadras especiales L2 quedó cerrado en su *estructura* (ADR-009:
`squad_level == L2` ⇒ `special_rules = true`, sin radio, no relay) y en su
*estado de mando* (ADR-015 §3: las especiales L2 conservan `C = +5` incluso
tras la caída del HQ, excepción de ADR-012). Pero el contenido táctico de cada
tipo de escuadra especial sigue declarado `Pendiente` en el manual, con cuatro
huecos explícitos que forman el Grupo 12 (#58–#61):

1. **#58 — Reglas de CAVALRY.** `docs/manual/04b-ejemplos-de-escuadras.md`
   §3 (Los Infernales) deja un `<!-- Pendiente: reglas especiales de
   caballería — ¿movimiento extra? ¿restricciones de terreno? ¿bonificación
   de carga? -->`. ADR-016 §3 ya fija el pool de movimiento de caballería
   (`MP = 8`) pero no la carga ni las restricciones de terreno propias.
2. **#59 — Reglas de ENGINEERS.** El mismo documento §4 (Zapadores) deja
   `<!-- Pendiente: reglas especiales de ENGINEERS — ¿qué estructuras pueden
   afectar? ¿cómo interactúa con el sistema de terreno y hexes? -->`. ADR-016
   §1 modela el terreno como catálogo extensible pero no define quién lo
   modifica en partida.
3. **#60 — Mecánica del "Cabo Rojo".** `docs/manual/03-facciones.md` describe
   al **Cabo** de Los Rojos (L2, no confundir con el Basto/Cabo Primero) como
   "especialista o tercero al mando … tiene una mecánica especial pendiente
   de definir".
4. **#61 — Reasignación de la orden genérica L2.** `04b` §4 deja
   `<!-- Pendiente: mecánica para reasignar órdenes genéricas a escuadras L2
   durante la partida -->`; el cap. 08 (línea 72) pregunta *cuándo* se asigna
   esa orden (¿Briefing? ¿Orders?).

Sin estas reglas el backend (ADR-003/004) no puede resolver la fase de
Resolution de una escuadra CAVALRY o ENGINEERS (no sabe qué efecto producen),
ni instanciar la mecánica del Cabo Rojo, ni decidir si la fase Orders puede
emitir una nueva directiva a una L2 ya desplegada.

Restricciones de coherencia que esta decisión respeta (no reabre ninguno):

- **ADR-001** (nomenclatura): se usan los términos canónicos
  L1–L5, Escuadra, Tropa, órdenes ATTACK/MOVE/DEFEND, orden genérica,
  `special_rules`, `compatible_squads`, sin introducir sinónimos.
- **ADR-009** (modelo de datos): `squad_level == L2` ⇒ `special_rules = true`,
  sin radio, no relay; `compatible_squads` restringe composición
  (`["infernales"]` para CAVALRY de Los Rojos). El tipo de escuadra
  (CAVALRY/ENGINEERS/…) se deriva de la plantilla (cap. 04); este ADR no
  cambia el schema, solo define el comportamiento que `special_rules`
  habilita.
- **ADR-011** (combate por soldado): los efectos de combate se expresan
  **exclusivamente** como factores que ADR-011 ya consume —`f_tactico_atk`
  (§3: `f_dist · m_elev_atk · m_flanqueo`) y `f_tactico_def`— o como
  modificadores aditivos de stat vía ADR-013. No se añade ningún término
  nuevo a `P_atk`/`P_def`; la carga de caballería se modela reutilizando
  `m_flanqueo` y el factor de orden.
- **ADR-013** (stats y catálogo): cualquier bono individual es un modificador
  aditivo sobre un stat canónico (`AIM/TGH/RCT/NRV/MOV/VAL`) con suma+clamp
  `[1,100]`; no se crean stats nuevos. El `level` L1–L5 no entra en ninguna
  fórmula de potencia (ADR-013 §1.2).
- **ADR-015** (mando): las especiales L2 no tienen radio, no son relay,
  conservan `C = +5` tras la decapitación del HQ (ADR-015 §3). Este ADR no
  toca esa autonomía; la usa como premisa.
- **ADR-016** (terreno y movimiento): el pool de movimiento de caballería es
  `MP = 8` (ADR-016 §3, ya aceptado, **no se redefine**); `move_cost` por
  terreno y la validación de path (§4) son la única fuente de costo de
  movimiento. CAVALRY y ENGINEERS consumen ese sistema; este ADR solo añade
  un **predicado de restricción de terreno** para CAVALRY y una **acción de
  edición de `TerrainDef` en partida** para ENGINEERS, sin alterar la tabla
  de pools ni los `move_cost` del prototipo.

Este ADR es normativo sobre la **regla de juego**. No modifica el manual ni
otros ADRs; los cap. 03 y 04b lo referenciarán cuando se actualicen. Valores
concretos donde el balance lo permite; lo genuinamente abierto se marca
**sujeto a playtest**.

## Decisión

### 1. Reglas especiales de CAVALRY (Los Infernales)

CAVALRY es una escuadra especial L2 (`squad_level = L2`, `special_rules =
true`, sin radio — ADR-009/015) cuya identidad mecánica son tres reglas:
movilidad, carga y restricción de terreno.

#### 1.1 Movilidad

El pool de movimiento de caballería es `MP = 8` **tal como lo fija ADR-016
§3** (no se redefine aquí). La validación de path (hasta 3 waypoints, suma de
`move_cost` de cada hex entrado, rechazo si `costo_path > MP` o si toca un hex
intransitable) es exactamente la de **ADR-016 §4**. CAVALRY no recibe un
término de movimiento extra fuera de su pool: su ventaja ya está cuantificada
en el pool 8 vs 4 (infantería) de ADR-016.

#### 1.2 Carga (charge)

Cuando una escuadra CAVALRY ejecuta una orden **ATTACK** y su path de avance
hasta el hex enemigo recorre **`d_charge >= 3` hexes** consumidos del pool en
ese mismo turno (carrera de aproximación montada), la escuadra obtiene un
**bono de carga** que se modela **reutilizando `m_flanqueo` de ADR-011 §3**
(el multiplicador táctico ofensivo que ADR-011 ya tiene en
`f_tactico_atk = f_dist · m_elev_atk · m_flanqueo`):

```
m_flanqueo(CAVALRY en carga) = M_CHARGE          si d_charge >= 3
m_flanqueo(CAVALRY sin carga) = valor normal de ADR-011 §3 (1.0 / 1.25)
```

| Constante | Valor | Rol | Tuneable |
|-----------|-------|-----|----------|
| `D_CHARGE_MIN` | `3` hexes | Distancia de aproximación mínima para que cuente como carga (≈300 m de carrera, escala ADR-016/cap. 02b). | sujeto a playtest |
| `M_CHARGE` | `1.40` | Valor de `m_flanqueo` durante la carga (ofensivo). Por encima del flanqueo normal `1.25` de ADR-011 §3 pero acotado para no romper `Δ`. | sujeto a playtest |

Reglas de la carga:

- Es **ofensiva y de un turno**: solo aplica en el turno en que se ejecuta el
  ATTACK con avance `>= D_CHARGE_MIN`. No persiste; el turno siguiente,
  sin nueva carrera, `m_flanqueo` vuelve a su valor normal.
- **No es aditiva con el flanqueo posicional**: si la carga aplica, `M_CHARGE`
  **sustituye** (no multiplica) al `m_flanqueo` que ADR-011 §3 daría por
  flanqueo (`1.25`). Se toma `m_flanqueo = max(M_CHARGE, flanqueo_posicional)`
  — nunca el producto — para no apilar dos multiplicadores ofensivos y
  mantener acotado `Δ = (P_atk − P_def)/(P_atk + P_def)` de ADR-011 §4.
- **Solo afecta al lado atacante** (`f_tactico_atk`). No modifica `P_def` ni
  `k_cobertura`; ADR-011 no se reabre — `M_CHARGE` es simplemente el valor que
  toma una variable que ADR-011 §3 ya define.
- El ejemplo numérico de ADR-011 §8 (que asume `m_flanqueo = 1.0`) **sigue
  siendo válido sin cambios**: una escuadra que no es CAVALRY o no cargó
  nunca ve `M_CHARGE`.

#### 1.3 Restricción de terreno

La movilidad montada se paga con **fragilidad fuera de terreno corrible**.
CAVALRY no puede **entrar** a un hex cuyo `TerrainDef.cover_class == HEAVY`
(ADR-016 §1), porque la motocicleta todoterreno no opera en edificación densa
ni espesura cerrada:

```
path válido para CAVALRY  ⟺  (path válido por ADR-016 §4)
                            ∧  ningún hex_i del path tiene cover_class == HEAVY
```

- En el **prototipo de ADR-016** solo existen `GRASS` (OPEN) y `WATER`
  (intransitable para todos): la restricción CAVALRY **no recorta nada**
  todavía (no hay terreno `HEAVY` definido). Es una regla que se activa sola
  cuando el catálogo de terreno incorpore tipos `HEAVY` (bosque cerrado,
  edificación — los conjeturales de ADR-016 §1), sin tocar ADR-016.
- El validador de path (ADR-016 §4, mismo punto de control) añade este
  predicado para escuadras de tipo CAVALRY; el rechazo es server-side
  (ADR-005/010) igual que cualquier path inválido.
- CAVALRY **sí** puede entrar a terreno `OPEN` y `LIGHT`. Solo `HEAVY` está
  vedado.

### 2. Reglas especiales de ENGINEERS (Zapadores)

ENGINEERS es una escuadra especial L2 (ADR-009/015, sin radio, orden
genérica). Su valor no es la potencia de combate sino **editar el tablero**:
su acción modifica el `TerrainDef` de hexes objetivo dentro del marco del
catálogo de ADR-016 §1, **sin introducir un eje nuevo** — solo cambia el
`TerrainType` asignado a un hex, lo cual ya es un dato mutable del escenario.

#### 2.1 Acción de ingeniería

Una escuadra ENGINEERS, mientras tenga vivos **>= 2 operadores de carga** (el
Grupo de Ingeniería L1 de `04b` §4) y su Cabo/Cabo Primero L2 vivo, puede
ejecutar **una** acción de ingeniería por turno sobre **un hex objetivo
adyacente** (a 1 hex del hex que ocupa la escuadra), declarada por su orden
genérica (§4). Dos efectos canónicos:

| Acción | Efecto sobre el hex objetivo | Restricción |
|--------|------------------------------|-------------|
| **Brecha (breach)** | Reduce el `cover_class` del `TerrainDef` del hex un escalón: `HEAVY → LIGHT`, `LIGHT → OPEN`. Reduce su `move_cost` a `move_cost(GRASS) = 1.0` (ADR-016 §1/§4). No cambia `passable` ni `vision_block` salvo que el escalón lo implique. | El hex debe ser `passable` y de `cover_class != OPEN`. |
| **Demolición (destroy)** | Marca el hex como **intransitable**: `passable = false`, `move_cost = ∞` (ADR-016 §1, mismo tratamiento que `WATER`: cualquier validador de path lo rechaza). Modela cráter / puente volado / obstáculo. | Solo sobre hex `passable`; irreversible en la partida. |

- El efecto **se materializa en la fase Resolution** del turno en que la
  acción se resuelve, igual que el fuego indirecto del mortero (`04b` §2):
  el cambio de `TerrainDef` queda activo desde el turno siguiente para
  validación de paths (ADR-016 §4), cobertura `k_cobertura` (ADR-011 §3 vía
  ADR-016 §1) y visión (`vision_block`, cap. 10).
- El cambio es **persistente y autoritativo** (estado de escenario server-side,
  ADR-003/005): una vez aplicado, todo el sistema lo ve. No hay reversión por
  parte del enemigo (no hay "reconstruir" en el prototipo; conjetural, sujeto
  a playtest si se añade contraingeniería).
- ENGINEERS **no** crea cobertura nueva ni eleva terreno: solo opera **dentro**
  del catálogo de ADR-016 (degradar cobertura, bloquear paso). Elevación
  (eje ortogonal, ADR-016 §2) **no** es afectada por zapadores.

| Constante | Valor | Rol | Tuneable |
|-----------|-------|-----|----------|
| `ENG_MIN_OPERATORS` | `2` | Operadores de carga L1 vivos mínimos para ejecutar la acción (coherente con `04b` §4 "mueren los operadores de carga ⇒ pierde la función"). | sujeto a playtest |
| `ENG_RANGE` | `1` hex | La acción es sobre hex adyacente (los zapadores deben llegar al objetivo). | sujeto a playtest |
| `ENG_ACTIONS_PER_TURN` | `1` | Una acción de ingeniería por turno por escuadra. | sujeto a playtest |

#### 2.2 Combate de ENGINEERS

ENGINEERS combate con las reglas normales de ADR-011 (es una escuadra débil:
5 tropas, fuerza baja — `04b` §4). No tiene bono de carga ni multiplicador
especial: su factor táctico es el estándar de ADR-011 §3. La escolta de
combate (2× L1) existe solo para autodefensa.

### 3. Mecánica especial del "Cabo Rojo"

El **Cabo** de Los Rojos (`docs/manual/03-facciones.md`, L2, *distinto* del
**Basto**/Cabo Primero que es el segundo al mando con radio) es un
**especialista o tercero al mando** sin función de radio. Su mecánica especial
se define así, coherente con ADR-013 (modificadores aditivos) y ADR-009
(el `level` no entra en fórmulas de potencia):

**El Cabo Rojo es un sub-líder de Grupo que confiere un modificador de
cohesión a su grupo interno mientras esté vivo.**

- Si la escuadra roja tiene un Grupo interno liderado por el Cabo (ADR-009:
  `Group.leader` = el L2 Cabo), **mientras el Cabo está vivo** cada Tropa L1
  **de ese grupo** recibe un modificador aditivo transitorio
  **`NRV +N_CABO`** (Temple, stat canónico de ADR-013 §1), aplicado con la
  misma mecánica suma+clamp `[1,100]` de ADR-013 §1.2 — análogo a un Trait
  reversible, ligado a la presencia del líder, **no** a su nivel (respeta
  ADR-013 §1.2: el `level` no aparece en fórmulas).

| Constante | Valor | Rol | Tuneable |
|-----------|-------|-----|----------|
| `N_CABO` | `+8` | Bono `NRV` (Temple) a las L1 del grupo del Cabo mientras el Cabo vive. Magnitud entre un Trait (`+6`) y un Perk (`+15`) del catálogo de ADR-013 §2. | sujeto a playtest |

- **Es de grupo, no de escuadra**: solo afecta a las L1 del Grupo cuyo
  `leader` es el Cabo. La escuadra plana estándar de Los Rojos (un Sargento,
  un Basto, seis soldados, **sin** grupo del Cabo — `03-facciones.md`) **no**
  recibe el bono: el Cabo solo aparece en composiciones que lo incluyen como
  líder de grupo (p. ej. la escuadra de Zapadores de Los Rojos, `04b` §4,
  donde el Cabo lidera el Grupo de Ingeniería).
- **Se pierde al morir el Cabo**: el modificador `NRV +N_CABO` es transitorio;
  si el Cabo (L2) cae, se retira de las L1 de su grupo en el mismo turno
  (coherente con `04b`: "muere el Cabo L2 ⇒ el grupo pierde su organización
  interna"). No hay sucesión del rol.
- **Apila por suma con el resto**, como cualquier modificador de ADR-013
  (Perks/Traits/Heridas) — no es un multiplicador; no toca ADR-011 (entra vía
  `NRV_ef`, reservado por ADR-011 §1 para supresión futura; hoy no altera
  ningún ejemplo numérico de ADR-011 §8 porque `NRV` aún no pesa en
  `P_atk`/`P_def`). Es deliberadamente un bono de **moral/estabilidad**
  acorde al rol de "tercero al mando que sostiene al grupo", no de pegada.

Lectura temática: el Cabo Rojo no manda la escuadra (eso es del Sargento) ni
porta radio (eso es del Basto); su valor es **mantener firme a su grupo bajo
fuego**. Coherente con la estructura plana e informal de Los Rojos
(`03-facciones.md`): la cohesión es por presencia, no por jerarquía.

### 4. Asignación y reasignación de la orden genérica a escuadras L2

Cap. 08 (línea 72) y `04b` §4 dejan abierto **cuándo** se asigna la orden
genérica de una escuadra especial L2 y si puede **reasignarse** durante la
partida. Se decide, coherente con el ciclo Briefing → Orders → Resolution
(ADR-004) y con la autonomía sin radio de ADR-009/015:

#### 4.1 Cuándo se asigna

- La orden genérica de una escuadra especial L2 se asigna **en la fase Orders**
  (ADR-004), igual que cualquier otra orden, **pero** la escuadra L2 solo es
  un destino de orden válido **si su oficial de pelotón (L5) que la comanda
  está `in_command`** en ese turno (regla de 5 hexes encadenada, ADR-015 §3):
  la directiva sale del pelotón, no del HQ directamente (cap. 08 línea 64–66).
- Una vez emitida, la escuadra **ejecuta esa directiva con autonomía limitada**
  durante los turnos siguientes **sin** recibir órdenes nuevas, exactamente
  como describe el manual (cap. 08 líneas 66–70; `04b` §3/§4): la L2 no tiene
  radio, no está en la cadena, opera "en profundidad".

#### 4.2 Si se permite reasignar — **sí, con condición de contacto**

Se **permite reasignar** la orden genérica de una escuadra especial L2 durante
la partida, **pero solo en un turno en que la escuadra esté dentro del radio
de mando de un L3+ aliado o del relay encadenado** (regla de 5 hexes,
ADR-015), es decir, cuando **`in_command == true`** para esa escuadra ese
turno. Justificación: sin radio, la única forma de hacerle llegar una nueva
directiva es por **contacto/cercanía física** con la cadena de mando.

```
reasignar_orden_generica(L2)  permitido en el turno T  ⟺
        squad.special_rules == true
     ∧  in_command(squad, T) == true        (regla 5 hexes, ADR-015)
```

- **Caso normal (en profundidad, fuera de mando):** la escuadra está aislada
  por diseño (flanqueo, incursión); `in_command == false` ⇒ **no** se puede
  reasignar; ejecuta la última directiva (cap. 08 línea 68–70). Esto preserva
  la tensión del manual: mandarla en profundidad significa perder el control
  fino sobre ella.
- **Caso recontacto:** si la operación trae a la L2 de vuelta a rango de la
  cadena (`in_command == true`), el oficial L5 **puede** emitirle una nueva
  orden genérica en la fase Orders de ese turno, como a cualquier unidad en
  mando.
- **Sin HQ (ADR-015 §3):** si el HQ cayó, el pool de órdenes está congelado
  (ADR-015 §3: el jugador no puede emitir órdenes nuevas a ninguna unidad) ⇒
  **no** se puede reasignar a nadie, incluidas las L2. Las especiales L2
  conservan su `C = +5` y siguen ejecutando su última directiva: son las
  únicas tácticamente plenas tras la decapitación (ADR-015 §3), pero su orden
  queda **fija** desde la caída del HQ. No hay contradicción con ADR-015: el
  `+5` es estado de mando para iniciativa (ADR-012), no un canal para recibir
  órdenes nuevas.
- **No ejecutable (`04b` §3/§4):** si la directiva ya no puede ejecutarse
  (objetivo destruido por otro medio, terreno cambiado por ENGINEERS §2), la
  escuadra queda en limbo defensivo por defecto hasta un turno en que sea
  reasignable (`in_command == true` y HQ vivo). Mientras no lo sea, el riesgo
  de disband de las L2 es el ya definido en cap. 08 (línea 83) y `04b`: "solo
  si la orden genérica ya no puede ejecutarse".

#### 4.3 Resumen de la regla de reasignación

| Situación de la L2 ese turno | ¿Reasignable? | Qué hace |
|------------------------------|---------------|----------|
| `in_command == true`, HQ vivo | **Sí** (fase Orders, vía L5) | Recibe nueva orden genérica |
| `in_command == false`, HQ vivo | No | Ejecuta última directiva (autonomía, `C = +5`) |
| HQ caído (ADR-015 §3) | No (pool congelado) | Ejecuta última directiva, orden fija, `C = +5` |
| Directiva no ejecutable | No hasta recontacto | Defensa por defecto; riesgo disband (cap. 08 / `04b`) |

### 5. Resumen de constantes nuevas (todas tuneables por playtest)

| Constante | Valor | § | Efecto al subir |
|-----------|-------|---|-----------------|
| `D_CHARGE_MIN` | 3 hexes | 1.2 | Carga más difícil de activar (exige más carrera). |
| `M_CHARGE` | 1.40 | 1.2 | Caballería en carga más letal; primer candidato a recortar si la carga domina. |
| `ENG_MIN_OPERATORS` | 2 | 2.1 | Zapadores más frágiles a perder su función. |
| `ENG_RANGE` | 1 hex | 2.1 | Deben acercarse más al objetivo. |
| `ENG_ACTIONS_PER_TURN` | 1 | 2.1 | Ritmo de edición del tablero. |
| `N_CABO` | +8 (`NRV`) | 3 | Grupo del Cabo Rojo más estable bajo fuego. |

No tuneables aquí (propiedad de otro ADR): `MP` caballería = 8 (ADR-016 §3),
`m_flanqueo`/`f_tactico_atk` como mecanismo (ADR-011 §3), suma+clamp de
modificadores y stats canónicos (ADR-013 §1), regla de 5 hexes e `in_command`
(ADR-015), `WATER`/`∞` y catálogo de terreno (ADR-016 §1).

## Consecuencias

**Positivas:**

- Cierra el Grupo 12 (#58 CAVALRY, #59 ENGINEERS, #60 Cabo Rojo, #61
  reasignación L2): el backend tiene reglas concretas para resolver la
  Resolution de escuadras especiales y para validar emisión/reasignación de
  órdenes genéricas.
- **No reabre ningún ADR.** La carga es un *valor* de `m_flanqueo` que
  ADR-011 §3 ya define; ENGINEERS edita `TerrainDef` dentro del catálogo de
  ADR-016 §1; el Cabo Rojo es un modificador aditivo de ADR-013 §2; la
  reasignación se ancla en `in_command` de ADR-015 y el ciclo de ADR-004.
  Los ejemplos numéricos de ADR-011 §8 siguen exactos (los nuevos términos
  valen su neutro en el caso por defecto).
- **Asimetría por composición, no por regla** (`03-facciones.md`): CAVALRY y
  el Cabo Rojo solo existen en Los Rojos por *qué escuadras tienen*, pero las
  reglas (carga, modificador `NRV`) son ruleset único — coherente con la
  decisión deliberada del cap. 03 de simetría mecánica.
- La restricción de terreno de CAVALRY y la edición de ENGINEERS están
  preparadas para el catálogo de terreno extensible de ADR-016 sin
  modificarlo: se activan solas cuando aparezcan terrenos `HEAVY`.
- La regla de reasignación preserva la tensión narrativa del cap. 08:
  mandar una L2 en profundidad es renunciar al control fino, recuperable solo
  por recontacto físico.

**Negativas:**

- 6 constantes nuevas requieren playtest; `M_CHARGE` apilado con elevación
  ofensiva (`m_elev_atk = 1.15`, ADR-011 §3) puede hacer una carga en cota
  muy letal (`f_tactico_atk` ≈ `f_dist · 1.15 · 1.40`) — buscado, pero
  primer candidato a recalibrar si la caballería domina el juego ofensivo.
- La edición de terreno por ENGINEERS es **irreversible** en el prototipo
  (no hay contraingeniería): un mapa muy trabajado por zapadores puede
  volverse estático; aceptable para el prototipo, conjetural si se añade
  reconstrucción.
- El bono del Cabo Rojo recae en `NRV`, que ADR-011 §1 aún tiene "reservado"
  (supresión futura): hasta que el sistema de supresión exista, el efecto del
  Cabo Rojo es **latente** (no altera combate hoy). Es coherente y sin
  contradicción, pero su impacto real depende de un sistema futuro.
- La reasignación atada a `in_command` añade una consulta más al BFS de
  mando (ADR-015) en la validación de la fase Orders para destinos L2; coste
  de implementación menor pero real.

## Alternativas descartadas

- **Carga como término aditivo nuevo en `P_atk` (ADR-011):** rechazado por
  coherencia — ADR-011 modela el contexto ofensivo como factores
  multiplicativos dentro de `f_tactico_atk`; reutilizar `m_flanqueo` mantiene
  la homogeneidad y **no** reabre ADR-011. Un término aditivo nuevo lo
  habría modificado.
- **Carga multiplicando además del flanqueo posicional:** apilar `M_CHARGE ·
  1.25` infla `Δ` sin techo natural (ADR-011 §4) y vuelve la caballería
  irresistible; se toma `max(M_CHARGE, flanqueo)` para acotar.
- **ENGINEERS creando cobertura/elevación nueva:** confunde los dos ejes
  ortogonales que ADR-016 §2 mantiene separados (terreno vs elevación) y
  obligaría a inventar tipos fuera del catálogo; degradar/bloquear **dentro**
  del catálogo respeta ADR-016 sin tocarlo.
- **Cabo Rojo como bono de combate ofensivo (`AIM`):** rompe el rol del cap.
  03 ("tercero al mando", no especialista de pegada) y solaparía con Perks
  ofensivos de ADR-013 §2; el bono de `NRV`/cohesión es fiel al rol de
  segundo/tercero que sostiene la unidad y a la estructura plana de Los Rojos.
- **Cabo Rojo con efecto de escuadra completa:** sobrevaloraría a una L2 y
  rompería el balance plano de la escuadra roja estándar
  (`03-facciones.md`); limitarlo al **grupo que lidera** lo mantiene
  excepcional y compositivo.
- **Reasignar la orden genérica L2 libremente cada turno:** anula la tensión
  del cap. 08 (la L2 sin radio en profundidad es deliberadamente
  incontrolable) y contradice ADR-009/015 (sin radio ⇒ fuera de la cadena);
  atarla a `in_command` por contacto físico es la única vía coherente.
- **Asignar la orden genérica solo en Briefing:** ADR-004 ubica la emisión de
  intenciones en Orders; meter un canal de asignación distinto en Briefing
  duplicaría el flujo de órdenes sin necesidad. Orders con condición
  `in_command` reutiliza el ciclo existente.
