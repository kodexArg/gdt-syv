# ADR-013: Stats canonicos del soldado y catalogo de modificadores

## Estado

Aceptado — 2026-05-18

## Contexto

El capitulo [04 — Unidades](../manual/04-unidades.md) define que todo soldado
hereda una **plantilla de clase** de stats base y que sobre ella se aplican
**modificadores abiertos**: Perks (bonificadores), Traits (innatos), Phobias
(penalizadores psicologicos) y Heridas (penalizadores permanentes de combate).
El mismo capitulo deja explicitamente pendiente tres cosas:

- *"Nombres de stats especificos: Pendiente de definir."*
- *"Catalogo de Perks, Traits, Phobias: Pendiente de definir."*
- *"Eventos disparadores especificos y probabilidades [de mutacion]: Pendiente
  de definir."*

El capitulo [09 — Valor](../manual/09-valor.md) define el **Valor** como un stat
individual por soldado (plantilla de clase + perks/traits/phobias + heridas),
modificador pasivo que degrada todo desempeño, pero tampoco lo nombra
canonicamente ni fija sus triggers.

Mientras tanto, dos ADR aceptados ya consumen estos stats sin esperar la
nomenclatura:

- [ADR-011](011-resolucion-de-combate-por-soldado.md) fija **cuatro stats base
  provisionales** con siglas `AIM` (Punteria), `TGH` (Resistencia), `RCT`
  (Reaccion), `NRV` (Temple), en escala 0–100, modificadores aditivos con clamp
  `[1,100]`, y referencia ademas `MOV` en su tabla de reparto de heridas por
  parte del cuerpo (§7: "pierna→`MOV`/`RCT`").
- [ADR-012](012-calculo-de-iniciativa-orden-de-resolucion.md) usa un stat de
  **Reaccion** (`reaccion`) por soldado, normalizado 0–100, alimentando el
  factor `A` de la iniciativa, y declara explicitamente que queda *"atado a la
  nomenclatura de stats pendiente. Si esa nomenclatura nombra el stat de otra
  forma, este ADR sigue valido (mapea por rol, no por nombre)."*

Es el bloqueante de mas alto apalancamiento del Grupo 1: sin nombres canonicos
de stats y sin catalogos de modificadores no se puede instanciar una Tropa
jugable, ni calibrar ADR-011/012, ni cablear Valor (cap. 09) ni la asignacion
aleatoria/descubrimiento/mutacion (cap. 04). Este ADR los cierra.

Restricciones de coherencia respetadas:

- ADR-001 (glosario): Tropa, Escuadra, modificador 1-a-varios, L1–L5 **no** es
  poder de combate.
- cap. 04: plantilla de clase + Perks/Traits/Phobias/Heridas como suma de
  modificadores; asignacion aleatoria oculta; mutacion de baja frecuencia.
- cap. 09: Valor individual, pasivo, degrada todo; agregado a escuadra mueve
  estados ACTIVE/RETREAT/ROUTED.
- **No contradecir** ADR-011 ni ADR-012: se adoptan sus siglas tal cual y se
  mapea por rol el stat "Reaccion" de ADR-012.

Este ADR **no modifica** el manual ni otros ADR; cap. 04 y cap. 09 lo
referenciaran cuando se actualicen. No reabre 010/011/012.

## Decision

### 1. Stats canonicos del soldado

Se fija un **conjunto canonico de seis stats base** por soldado, escala 0–100,
heredados de la plantilla de clase (cap. 04). Es un **superconjunto** que
contiene exactamente las siglas ya usadas por ADR-011 (`AIM/TGH/RCT/NRV`),
incorpora `MOV` (ya referenciado en ADR-011 §7) y nombra el Valor del cap. 09
como `VAL`. No se renombra ninguna sigla preexistente: ADR-011 y ADR-012 quedan
validos sin tocarlos.

| # | Stat (canonico) | Sigla | Que representa | Base infanteria L1 | Consumido por |
|---|-----------------|-------|----------------|--------------------|---------------|
| 1 | Punteria | `AIM` | Probabilidad de conectar fuego ofensivo | 50 | ADR-011 `P_atk` |
| 2 | Resistencia | `TGH` | Capacidad de absorber daño sin caer | 50 | ADR-011 `P_def`, umbral `W_kia` |
| 3 | Reaccion | `RCT` | Rapidez de respuesta; bono defensivo + insumo de iniciativa | 50 | ADR-011 `P_def` (0.15·RCT); ADR-012 factor `A` |
| 4 | Temple | `NRV` | Estabilidad bajo fuego (resiste supresion/panico) | 50 | ADR-011 (reservado); cap. 07 supresion futura |
| 5 | Movilidad | `MOV` | Velocidad de desplazamiento y reaccion fisica | 50 | ADR-011 §7 (reparto de heridas pierna); MOVE/avance |
| 6 | Valor | `VAL` | Moral/disciplina; modificador pasivo global (cap. 09) | 50 | cap. 09 (degrada todo; agrega a estado de escuadra) |

Base **50** para todos en infanteria L1, coherente con ADR-011 ("Stats base de
clase 50 c/u", supuesto de balance, sujeto a playtest). Otras clases ajustan su
plantilla sobre esta referencia (p. ej. Recon +RCT/+MOV, Sanitario sin sesgo
ofensivo); las plantillas por clase se calibran con playtest y no se fijan aqui
salvo la de infanteria L1.

#### 1.1 Mapeo con los ADR previos (autoridad de nombres)

| Concepto en ADR previo | Sigla canonica de este ADR | Nota |
|------------------------|----------------------------|------|
| ADR-011 §1 `AIM` (Punteria) | `AIM` | Identico, sin cambio |
| ADR-011 §1 `TGH` (Resistencia) | `TGH` | Identico |
| ADR-011 §1 `RCT` (Reaccion) | `RCT` | Identico |
| ADR-011 §1 `NRV` (Temple) | `NRV` | Identico |
| ADR-011 §7 `MOV` (reparto heridas pierna) | `MOV` | Se formaliza como stat #5 |
| ADR-012 §2.1 `reaccion` / "Reaccion del lider" | `RCT` | **Mapeo por rol** (lo previsto por ADR-012: "mapea por rol, no por nombre"). El factor `A` de iniciativa usa `RCT_ef` |
| cap. 09 "stat de Valor individual" | `VAL` | Nombre canonico del Valor |

Con esto, ADR-012 deja de estar "atado a nomenclatura pendiente": su `reaccion`
es `RCT`. ADR-011 no requiere ningun ajuste: sus cuatro siglas son un
subconjunto exacto del canonico.

#### 1.2 Aplicacion de modificadores (reafirma ADR-011 §1, no lo cambia)

Todos los modificadores (Perks +, Traits +/-, Phobias -, Heridas -) se aplican
como **suma aritmetica** sobre el/los stat(s) afectado(s), con clamp final a
`[1, 100]`, exactamente como ya define ADR-011 §1:

```
stat_efectivo(S) = clamp( base(S) + Σ mod_i(S) , 1, 100 )
```

`VAL` usa la misma mecanica de suma+clamp. Su degradacion por triggers (cap. 09)
se modela como modificadores negativos transitorios sobre `VAL` (analogos a
Heridas pero reversibles); la formalizacion de triggers/recuperacion de Valor
**no** es alcance de este ADR (queda para el Grupo de Valor) — aqui solo se fija
el **nombre** `VAL` y que vive en el mismo sistema de suma+clamp.

El `level` (L1–L5) **no** aparece en ninguna formula de potencia individual
(cap. 04, reafirmado por ADR-011/012).

### 2. Catalogo inicial de modificadores (set jugable para prototipo)

Cada entrada expresa su efecto **exclusivamente** como modificador aditivo
sobre uno o mas stats canonicos del §1. Las magnitudes son **supuesto de
balance, sujeto a playtest**. Cada clase de tropa expone un *pool* del que la
asignacion aleatoria del servidor extrae (cap. 04: oculto al jugador hasta su
primera activacion). El pool de infanteria L1 es el catalogo completo de abajo;
otras clases restringen/extienden su pool en calibracion posterior.

Reglas de asignacion inicial (prototipo, sujeto a playtest):

- Cada Tropa recibe **0–2 Traits**, **0–1 Perk**, **0–1 Phobia** en creacion.
- Probabilidad por slot: Perk 35 %, Phobia 25 %, cada Trait 50 % (hasta 2).
- Una Tropa puede quedar sin ningun modificador (es lo esperado para la
  mayoria; los modificadores son la excepcion que individualiza).

#### 2.1 Perks (bonificadores)

| Perk | Efecto (aditivo sobre stat canonico) | Lectura |
|------|--------------------------------------|---------|
| Tirador | `AIM +15` | Punteria sobresaliente (el ejemplo de ADR-011 §1/§8) |
| Curtido | `TGH +12` | Encaja castigo fisico |
| Reflejos | `RCT +12` | Reacciona antes (sube `A` de iniciativa, ADR-012) |
| Sangre fria | `NRV +15` | No se quiebra bajo fuego |
| Veloz | `MOV +12` | Desplazamiento agil |
| Aguante moral | `VAL +12` | Moral mas firme que la media |
| Veterano | `AIM +6, NRV +6, VAL +6` | Competencia transversal moderada |
| Cazador | `AIM +8, RCT +6` | Bueno abriendo fuego primero y certero |

#### 2.2 Traits (innatos, +/-)

| Trait | Efecto | Lectura |
|-------|--------|---------|
| Pulso firme | `AIM +8` | Innato fino, menor que el Perk Tirador |
| Corpulento | `TGH +10, MOV -5` | Aguanta mas, se mueve menos |
| Nervioso | `RCT +6, NRV -8` | Rapido pero inestable |
| Estoico | `NRV +8, VAL +6` | Temperamento firme |
| Liviano | `MOV +8, TGH -5` | Agil pero fragil |
| Lider nato | `VAL +6, RCT +5` | Arrastra a la unidad (relevante si es lider, ADR-012 `A`) |
| Pesimista | `VAL -6` | Moral basal baja |
| Temerario | `AIM +6, NRV +6, TGH -8` | Eficaz pero se expone |
| Lento | `RCT -8, MOV -6` | Reaccion y pies pesados |
| Pulso tembloroso | `AIM -8` | Mal tirador de base |

#### 2.3 Phobias (penalizadores psicologicos)

| Phobia | Efecto | Disparador de revelado (cap. 04) | Lectura |
|--------|--------|----------------------------------|---------|
| Pavor a la artilleria | `NRV -18, RCT -8` | Proyectil indirecto cae a ≤2 hex | El ejemplo de cap. 04 §descubrimiento |
| Claustrofobia de combate | `NRV -12, AIM -6` | Combate a 1 hex (cuerpo a cuerpo / asalto) | Se quiebra en el asalto cercano |
| Hemofobia | `VAL -12, NRV -8` | Un companero de su escuadra es KIA en el turno | Ver sangre lo desarma |
| Aracnofobia de trinchera | `NRV -10, MOV -6` | Escuadra en cobertura pesada/terreno cerrado | Penaliza fortificarse |
| Panico al aislamiento | `VAL -15, RCT -6` | Escuadra fuera de mando (`in_command=false`, cap. 08) | Refuerza el pilar de subordinacion |
| Miedo al flanqueo | `NRV -12, AIM -6` | El soldado es atacado con modificador de flanqueo (ADR-011 §3) | Se desordena cuando lo rodean |
| Fobia al fuego de MG | `NRV -14` | Recibe fuego de un arma con `w_arma ≥ 1.6` (MG, ADR-011 §4) | Lo paraliza el volumen de fuego |

#### 2.4 Heridas (referencia, no se cataloga aqui)

Las Heridas son modificadores negativos definidos por ADR-011 §7
(`sev`, reparto por parte del cuerpo, carga `W`, `W_kia=60`). Este ADR solo
confirma que su reparto por parte del cuerpo usa las siglas canonicas del §1
(brazo→`AIM`, pierna→`MOV`/`RCT`, torso→`TGH`; tabla fina sigue pendiente en
ADR-011, no se cierra aqui).

### 3. Mutacion de modificadores en partida

cap. 04 establece que un soldado puede ganar/perder un modificador por un
**evento extraordinario**, con **frecuencia muy baja**, y que no es progresion
por uso. Se fija el modelo concreto (supuesto de balance, sujeto a playtest):

#### 3.1 Eventos disparadores y probabilidad

La mutacion se evalua **una sola vez por soldado al final de un turno**, y solo
si el soldado vivio ≥1 evento disparador en ese turno. La probabilidad es la
del evento de mayor peso disparado (no se acumulan); como mucho **una** mutacion
por soldado por turno.

| Evento disparador (en el turno) | Mutacion candidata | P(mutacion) |
|---------------------------------|--------------------|-------------|
| Sobrevivio un turno donde recibio ≥1 HIT y su escuadra no quedo ELIMINATED | Gana Trait `Curtido` o `Estoico` (aleatorio entre los que no tenga) | 4 % |
| Presencio el disparador de una Phobia que **no** posee, dos turnos distintos | Adquiere esa Phobia | 6 % |
| Supero el disparador de una Phobia que **si** posee sin degradar de estado de escuadra, 3 turnos acumulados | Pierde esa Phobia | 8 % |
| Acumulo ≥3 eventos `MISS` propios como atacante con `p_hit ≥ 0.4` (fracaso repetido pese a ventaja) y 0 `HIT` propios en el turno | Pierde un Perk ofensivo (`Tirador`/`Cazador`/`Pulso firme`) si lo tiene | 3 % |
| Su escuadra gano un combate decisivo (≥2 KIA enemigos, 0 bajas propias) estando `in_command` | Gana Perk `Veterano` (si no tiene ningun Perk) | 2 % |

Notas:

- Solo se considera un disparador "valido" si el soldado estuvo `alive` todo el
  turno. Un soldado KIA no muta.
- "Dos/tres turnos" no requieren ser consecutivos: se cuenta un contador
  persistente por (soldado, evento) que **no** se resetea salvo que ocurra la
  mutacion (entonces se limpia el contador asociado).
- Si la tirada de mutacion procede, se aplica al final de la fase Resolution y
  se emite el mismo evento de revelado/notificacion que cap. 04 define para la
  primera activacion (la mutacion **siempre** es visible para el dueño: ganar o
  perder un modificador se notifica).
- Las Heridas **no** mutan por este sistema (son permanentes, ADR-011 §7).
- `VAL` no se "muta" por aqui: su variacion es la degradacion/recuperacion
  pasiva del cap. 09 (alcance de otro grupo de decisiones).

#### 3.2 Determinismo

La tirada de mutacion usa la **semilla del servidor del turno** (misma fuente
que `R` en ADR-012 §2.5 y las tiradas de ADR-011): la Resolution sigue siendo
reproducible dado estado + ordenes + semilla (compatible con la verificacion de
coherencia del Briefing, ADR-004). Orden de aplicacion dentro del turno:
combate (ADR-011) → estados de escuadra → **evaluacion de mutacion** (ultimo
paso, sobre el estado ya resuelto del turno).

### 4. Parametros tuneables (balance)

Todos **supuesto de balance, sujeto a playtest**:

| Parametro | Valor inicial | Efecto al subir |
|-----------|---------------|-----------------|
| Base de stats infanteria L1 | 50 c/u | Calibracion por clase/faccion |
| Slots de asignacion (Trait/Perk/Phobia) | 0–2 / 0–1 / 0–1 | Densidad de individualizacion |
| P(slot) Perk / Phobia / Trait | 35 % / 25 % / 50 % | Cuanto se aparta cada soldado de su clase |
| Magnitudes de Perks/Traits/Phobias | tablas §2 | Peso de los modificadores frente a los stats base |
| P(mutacion) por evento | tabla §3.1 (2–8 %) | Cuanto evolucionan las tropas en partida |
| Turnos-umbral de eventos repetidos | 2 / 3 / 3 | Cuan "extraordinario" debe ser para mutar |
| Tope de mutaciones por soldado/turno | 1 | Estabilidad del roster |

## Consecuencias

**Positivas:**

- Cierra las tres pendientes de cap. 04 ("nombres de stats", "catalogo de
  Perks/Traits/Phobias", "eventos disparadores y probabilidades de mutacion") y
  nombra el Valor (`VAL`) del cap. 09, sin tocar el manual ni reabrir ADR.
- **No contradice** ADR-011 ni ADR-012: adopta sus siglas literalmente y mapea
  por rol el `reaccion` de ADR-012 a `RCT` — exactamente lo que ADR-012 previo.
- Todo efecto de modificador esta expresado como suma aditiva sobre stats
  nombrados, reutilizando intacto el sistema de modificadores de ADR-011 §1.
- El set de catalogos es chico y jugable: instancia Tropas para prototipo ya.
- La mutacion es rara, determinista por semilla y siempre notificada: coherente
  con cap. 04 (eventos extraordinarios, no progresion por uso) y con la
  verificacion de coherencia del Briefing (ADR-004).

**Negativas:**

- ≈20 magnitudes + 5 probabilidades de mutacion sin calibrar: el balance real
  exige telemetria de playtest. La **estructura** (siglas canonicas,
  suma+clamp, pools por clase, una mutacion/turno por semilla) es la decision
  estable; los numeros son ajustables sin reabrir este ADR.
- Introduce dos stats que ADR-011 no usaba directamente (`MOV` ya estaba
  referenciado; `VAL` es nuevo como nombre): no rompen nada porque la formula
  de combate de ADR-011 no los lee — son insumo de movimiento/Valor, dominios
  de otros grupos.
- La interaccion `VAL` ↔ formulas de ADR-011/012 (Valor como modificador global
  del cap. 09) queda nombrada pero no formalizada aqui: depende del grupo de
  decisiones de Valor; este ADR solo garantiza que `VAL` vive en el mismo
  sistema de suma+clamp para que ese grupo no tenga que renombrar nada.

## Alternativas descartadas

- **Renombrar las siglas de ADR-011 a un set "mas limpio"**: contradiria dos
  ADR aceptados y obligaria a reabrir 011/012. El costo de un superconjunto que
  conserva `AIM/TGH/RCT/NRV` es nulo y preserva la inmutabilidad de ADR (ADR-000).
- **Stats como multiplicadores en vez de aditivos**: ADR-011 ya fijo
  suma+clamp; cambiarlo romperia su matematica de potencia. Aditivo ademas hace
  trivial el "encimamiento" de modificadores 1-a-varios (cap. 04).
- **Catalogo grande y exhaustivo**: sobredimensionado para prototipo; dificulta
  calibrar. Un set chico cubre todos los disparadores de revelado que cap. 04
  ejemplifica (tirador, fobia a artilleria, resistencia excepcional).
- **Mutacion como progresion por uso (estilo RPG/XP)**: explicitamente
  rechazado por cap. 04 ("No es un sistema de progresion por uso"). Se modela
  como evento extraordinario raro.
- **Mutacion con azar independiente por evento**: romperia la reproducibilidad
  del Briefing (ADR-004). Se ata a la semilla del turno, como ADR-011/012.
- **Definir aqui triggers y recuperacion de Valor**: fuera de alcance; cap. 09
  los deja a un grupo de decisiones propio. Este ADR solo fija el nombre `VAL`
  y su pertenencia al sistema de modificadores para no prejuzgar ese trabajo.
```
