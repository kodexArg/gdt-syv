---
title: "Visión General — Cómo funcionaría Subordinación y Valor"
order: 0
section: overview
lang: es
date: 2026-05-18
status: síntesis narrativa basada en specs (docs/adr + docs/manual + docs/premade-squads)
---

# Subordinación y Valor — Visión General

> Documento de síntesis. No define nada nuevo: relata, en una sola narrativa
> coherente, cómo el juego funcionaría **según lo ya documentado**. Todo lo que el
> plan deja sin resolver se marca explícitamente como **decisión abierta**, sin
> inventar valores. Fecha de redacción: **2026-05-18**.

---

## 1. La historia / el mundo

Año 2178. La Argentina ya no es un país: es una herida partida en dos sobre la
misma línea donde el siglo XIX la había cortado por primera vez. La acción
transcurre en la **Franja de Alsina**, esa vieja zanja que alguna vez separó la
tierra de los pueblos originarios y que ahora, por una ironía del destino, vuelve
a ser frontera —esta vez entre dos ejércitos de la misma sangre.

Desde el norte baja **La Confederación Argentina**: una teocracia militar que
gobierna desde Ciudad Dársena y marcha al sur para recuperar Bahía Blanca. Es un
ejército regular, de estructura clásica y disciplina formal, con la particularidad
de que la Iglesia no acompaña desde afuera sino que está integrada en la cadena de
mando: hay un Monseñor en el comando y Capellanes Médicos en la tropa. Visten
cascos verdes y, en pantalla, su Comandante aparece siempre a la derecha.

Desde el sur resisten **Los Rojos**: un movimiento de raíz comunista-patriótica,
bien pertrechado y decidido a no rendirse. Su estructura es deliberadamente más
horizontal y sus rangos más informales —la escuadra roja estándar es un Sargento,
un Basto y seis soldados sin especialización, plana a propósito—. Llevan cascos
camuflados con vetas rojas, un aspecto más decontracturado, y guardan un cuerpo
especial único: **Los Infernales**, caballería ligera montada en motos eléctricas
todoterreno, con nomenclatura gaucha (Jefe de Partida, Baqueano, Jinete). En
pantalla, su líder aparece siempre a la izquierda.

Lo notable es que el conflicto **no se decide por superioridad mecánica**. Ambas
facciones juegan con exactamente las mismas reglas, las mismas fórmulas y las
mismas mecánicas de combate: no hay bonificadores por bandera, no hay daño
distinto, no hay rasgos exclusivos por afiliación. La única asimetría es
**compositiva** —qué tipos y cuántas escuadras puede desplegar cada bando—. El
balance se logra por variedad de fuerzas y por ajuste manual de cada escenario, no
por excepciones de reglamento. El sistema jerárquico es agnóstico: cualquier
facción futura solo necesita mapear sus propios rangos sobre los mismos niveles
L1–L5.

El tono del juego está en su filosofía, y conviene decirla con sus propias
palabras: las unidades se desgastan, las órdenes se pierden y *el plan perfecto no
sobrevive al contacto con el enemigo*. No se gana con un golpe decisivo sino
acumulando ventajas posicionales, **como olas erosionando un acantilado**. Lo que
hace único a SyV no es mover soldados: es lo que ocurre cuando la cadena de mando
falla, la moral quiebra y el campo deja de parecerse a lo que planeaste. El
jugador no es un titiritero omnisciente. Es un Comandante encerrado en un puesto
de mando, hablando por radio a oficiales que pueden no escuchar, no entender, o ya
estar muertos cuando su orden llega. Sobre ese cuarto cerrado se construye todo.

Otro pilar del mundo es **El Fin de los Secretos**: un evento histórico que volvió
irreversible la transparencia de las comunicaciones. Toda señal de radio puede ser
escuchada y, eventualmente, descifrada. No hay tecnología que lo resuelva. De ahí
nace una de las reglas más características del juego: la tropa básica (L1 y L2) no
porta radio personal —una señal abierta sería interceptada de inmediato— y solo
los oficiales L3 y superiores operan la radio táctica encriptada **E-UHF**.
Comunicar tiene un costo estratégico permanente; gestionar la radio es tan
importante como gestionar las órdenes.

El universo extendido —historia, personajes, geografía completa— vive fuera de
este repositorio, en `kodexArg/syv`. Para el juego basta saber esto: dos ejércitos
hermanos, una zanja con memoria, y una guerra que se gana erosionando, no
aplastando.

---

## 2. Anatomía de un turno

SyV es un juego de **turnos simultáneos** (WEGO): ambos jugadores planifican al
mismo tiempo, en clientes separados, sin conocer las intenciones del otro. La
**autoridad es siempre del servidor**: la verdad del juego nunca vive en el
cliente. Antes de que empiece el primer turno hay un **despliegue inicial**
simultáneo y oculto —cada jugador coloca sus fuerzas en su zona de despliegue sin
ver al rival; el servidor recibe ambos despliegues, los valida contra el escenario
y recién entonces arranca el primer Briefing. (Decisión abierta: tamaño y forma de
las zonas de despliegue, reglas de colocación y límite de tiempo del despliegue.)

A partir de ahí, cada turno —que representa **cuatro horas de operaciones**— sigue
un ciclo estricto de tres fases: **Briefing → Orders → Resolution → Briefing → …**
hasta que se cumple una condición de victoria.

### Fase 1 — Briefing (servidor)

El servidor calcula el nuevo estado del juego y, en lugar de hacer un broadcast,
**serializa un estado distinto para cada jugador**: su *scope*. El scope es la
porción del estado que ese jugador tiene derecho a conocer; lo que está fuera de
su scope no se oculta en el cliente —directamente no llega a su máquina—. Esta es
la decisión arquitectónica central del juego: la niebla de guerra no es un filtro
visual, es una propiedad de la red. No se puede hackear lo que no está. El
servidor además valida coherencia: verifica que lo que emitió como eventos en la
Resolution anterior coincida con el estado actual.

El scope no es binario. El servidor calcula, para cada unidad enemiga vista desde
cada unidad amiga, un **nivel de revelación** continuo que depende de distancia,
tipo de observador, terreno intermedio, elevación relativa y (a futuro) clima.
Cuando varias unidades amigas observan al mismo enemigo, se usa la **mejor**
observación. El gradiente va desde "algo hay en esa dirección" hasta "composición,
estado y órdenes exactas", pasando por estimaciones de efectivos por rango (1–5,
6–10, 10+). El reconocimiento fino es un **privilegio del tipo de escuadra RECON**,
no una orden disponible para todos. (Decisión abierta: fórmula exacta y umbrales
de revelación; habilidades especiales concretas de las escuadras recon.)

Existe además una segunda fuente de inteligencia independiente de la línea de
visión: la **radiogoniometría**. El enemigo puede revelar parte de tu scope
triangulando tus transmisiones de radio, atravesando obstáculos. Usar radio seguido
es un riesgo estratégico real.

En pantalla, el jugador ve el tablero desde arriba —una grilla hexagonal
flat-top de radio 20 (1.261 hexes, ~4 km de lado, cada hex = 100 m de diámetro)—.
El terreno base siempre es visible (el Comandante conoce el mapa: pasto
transitable, agua intransitable; la elevación existe en el modelo pero en
prototipo todos los hexes están a `0.0`). Lo que está fuera de alcance visual
aparece desaturado, sin información de unidades. El jugador evalúa la situación:
posiciones propias, bajas, movimientos detectados.

### Fase 2 — Orders (cliente)

Fase **enteramente local**: el backend no participa en absoluto. El jugador
selecciona sus unidades y les asigna órdenes mediante el sistema *Cycle-Tap* (tap
selecciona; tap de nuevo en la unidad = DEFEND; tap en hex adyacente = ATTACK; tap
repetido inicia un path de MOVE de hasta 3 waypoints; tap en el origen confirma).
Puede usar undo y reset cuantas veces quiera, sin tocar al servidor. No hay límite
de tiempo impuesto por el sistema (puede añadirse como regla de juego). Al
confirmar, el cliente envía **un único mensaje** con todas las órdenes. El oponente
hace exactamente lo mismo, en paralelo, sin ver nada.

Las órdenes no son ilimitadas. El Comandante dispone de un **pool de órdenes**
dinámico: cada oficial L3/L4/L5 vivo y *conectado* a la cadena de mando aporta
órdenes al pool según su rango. Un oficial que cae —o que se desconecta— resta su
aporte de inmediato; reconectarlo (reacercarlo) lo restaura. (Decisión abierta:
cuántas órdenes aporta exactamente cada nivel L5/L4/L3.)

Hay tres órdenes específicas, y consumir el pool importa:

- **ATTACK** — la unidad avanza al hex objetivo y se compromete: si encuentra
  enemigos en el camino, los enfrenta y sigue hacia el objetivo. Es intención
  ofensiva.
- **MOVE** — repositionamiento por un path de hasta 3 waypoints, gobernado por un
  pool de puntos de movimiento por escuadra (más para caballería, menos para
  infantería; puntos no gastados no se acumulan y se reinician cada Briefing). Si
  detecta enemigos en tránsito, **se detiene**; solo dispara defensivamente si hay
  enemigo en línea de vista. Una escuadra **solo se mueve si tiene un MOVE
  activo**. (Decisión abierta: pools de movimiento por tipo y tabla de costos de
  terreno.)
- **DEFEND** — mantiene posición con bonus defensivo; **no consume pool**. Si se
  repite en turnos consecutivos, el bonus crece (fortificación / atrincheramiento);
  moverse o recibir otra orden lo reinicia a cero. (Decisión abierta: escala del
  bonus por turno consecutivo. Decisión abierta también: si DEPLOY existe solo en
  setup o también en juego.)

Además de las específicas, el Comandante puede dar **órdenes genéricas** a un
oficial ("Ataque a toda costa", "Mantenga posición"). El oficial corre un
algoritmo de IA que interpreta la directiva y **genera él mismo sub-acciones
específicas** para sus escuadras, que se insertan en la línea de tiempo como si
fueran órdenes propias —pero costando una sola acción de mando—. Es el corazón del
juego: una orden genérica ahorra recursos, pero el oficial puede interpretar mal o
decidir mal. La tensión está en cuánto control delegás. (Decisión abierta: lista
de órdenes genéricas disponibles y patrones de comportamiento exactos por tipo;
mejor entrenamiento del oficial = mejores decisiones, pero el mapeo concreto está
pendiente.)

Una orden, una vez asignada y confirmada, **no se revoca**: no hay cancelación a
mitad de turno. Para cambiar lo que hace una escuadra hay que esperar al próximo
Orders. Esto es deliberado: las órdenes son compromisos con consecuencias.

### Fase 3 — Resolution (servidor)

El servidor recibe las órdenes de ambos y **no produce un resultado instantáneo ni
un resumen**: simula tiempo continuo y genera una **línea de tiempo de eventos
discretos con marcas horarias** dentro de la ventana de cuatro horas. Cada
movimiento, cada disparo, cada baja, en el orden exacto en que ocurren:

```
14:23 → Artillería XX comienza el ataque sobre hex [4,7]
14:40 → Infantería YY comienza despliegue dirección NE
15:40 → YY tiene visual de enemigo ZZ dirección NW y comienza su ataque
16:05 → Soldado xx de ZZ abatido
```

Ese orden **no es aleatorio**: lo determina la **iniciativa** de cada unidad
—mayor iniciativa, acción más temprana; menor iniciativa, el campo ya cambió
cuando te toca actuar—. La iniciativa no es un desempate de último recurso: es el
mecanismo que ordena toda la línea de tiempo, y puede ser decisiva (una unidad
rápida abate a otra antes de que esa llegue a disparar este mismo turno).
**Decisión abierta: los factores que componen la iniciativa y su fórmula exacta no
están definidos.**

El movimiento no es instantáneo: cada salto entre hexes genera eventos con
timestamp ("14:30 sale de A", "15:10 llega a B"), y en tránsito la unidad puede ser
detectada, interceptada o atacada. Por eso ATTACK (persigue) y MOVE (se detiene
ante resistencia) son decisiones tácticas reales.

El **combate** se resuelve por **soldado individual (L1)** y es estocástico. Para
cada soldado se calcula una potencia de combate a partir de cuatro factores —stats
propios (plantilla de clase + perks/traits/phobias/heridas), equipamiento, estado
de la escuadra (ACTIVE / RETREAT / ROUTED), y contexto táctico (elevación,
terreno, flanqueo, bonus de DEFEND)—. La diferencia neta entre atacante y defensor
**modifica una probabilidad base** de tres resultados: muere el defensor, muere el
atacante, o ninguno (*bounce*). El lado más fuerte tiene más chance, nunca
certeza; el débil conserva una posibilidad real. El fuego se designa por soldado
(cada soldado puede apuntar a un enemigo distinto en hex distinto). No hay cuerpo
a cuerpo —todo es fuego, a cualquier distancia— ni logística —siempre se puede
disparar; la degradación viene solo de bajas y heridas—. **Decisión abierta: la
fórmula exacta que mapea los cuatro factores a la probabilidad final no está
definida; tampoco la designación de objetivos por soldado a nivel interfaz, ni la
distancia de detección y reacción, ni la secuencia interna de resolución
(movimiento → combate → post-procesado).**

Las heridas no tienen estados discretos: cada herida es un **modificador negativo
acumulable y permanente** (no se curan en toda la partida). El soldado es
eliminado cuando la suma de penalizadores cruza un umbral (**decisión abierta:
valor del umbral**). El **Sanitario** estabiliza —impide que empeore, mantiene
vivo a quien cruzaría el umbral— pero **nunca cura**: el daño persiste. Refuerza
el tema de erosión lenta. Morteros y artillería son **fuego indirecto puro**: no
necesitan línea de vista, disparan a coordenadas de hex, afectan **solo** ese hex
(sin splash a adyacentes). (Decisión abierta: si un observador mejora la precisión
del fuego indirecto.)

Sobre el resultado se aplica el **Valor**, que no es una tirada ni un recurso: es
un **modificador pasivo** de moral/disciplina por soldado, agregado a la escuadra
(promedio ponderado). Un Valor bajo degrada todo —combate, órdenes, resistencia—.
Se degrada por heridas, pérdida de compañeros, aislamiento (fuera del radio de
mando), presencia enemiga y traits/phobias; se recupera con tiempo seguro. El
Valor ponderado de la escuadra dispara transiciones de estado **reversibles**:
ACTIVE → RETREAT (no puede ATTACK, solo MOVE a retaguardia y DEFEND) → ROUTED
(ignora todas las órdenes, huye sola, no combate) → ELIMINATED. (Decisión
abierta: triggers y cantidades de degradación, fórmula de ponderación, umbrales
RETREAT/ROUTED, y velocidad/condiciones de recuperación.)

Tras el último eslabón del turno, el servidor recalcula la **cadena de mando** y
el **disband** (ver sección 3), transmite la cronología completa como eventos
secuenciales —el cliente los presenta de forma progresiva y dramática— y el ciclo
vuelve a Briefing.

---

## 3. Las decisiones del jugador

El jugador controla **un único personaje**: su Comandante, estático en el HQ. Todo
lo demás se decide a través de él. El árbol de decisiones de una partida se
recorre así:

### Antes de la partida — composición de fuerza

1. **Elegir facción** (Confederación o Rojos; se permite espejo Rojos vs Rojos o
   Conf vs Conf).
2. **Leer el presupuesto** de puntos que fija el escenario (partidas chicas
   ~20–30, medianas ~50–70, grandes 100+). *Decisión abierta: valores concretos
   por dificultad.*
3. **Seleccionar escuadras del catálogo de su facción** hasta agotar el
   presupuesto. Cada escuadra predefinida tiene un coste ya fijado en su YAML
   (`docs/premade-squads/conf-*.yaml`, `rojos-*.yaml`). Aquí el jugador define el
   carácter de su fuerza: infantería estándar L3 con grupos internos (Grupo de
   Fuego con MAG, Grupo Scout con sniper+observador), infantería pesada, morteros,
   artilleros, recon, zapadores, sanitarios, HQ de pelotón; o, del lado Rojo, la
   escuadra plana estándar y las opciones especiales. *Decisión abierta: si se
   permiten composiciones custom, equipamiento modificable, o bonos/penalizadores
   de experiencia entre escenarios.*

La estructura que compone es jerárquica y fija en su forma: **Sección** (su fuerza
completa, con el HQ en posición fija) → **Pelotones** (cada uno con su HQ propio:
Teniente L5 + Sargento Primero/de Primera L4, como una escuadra de tipo COMMAND) →
**Escuadras** (la unidad leaf que ocupa un hex y computa combate) → **Grupos**
(agrupaciones internas de 2–5 tropas bajo un L2, para mecánicas de subconjunto:
MG como sistema, dúo scout) → **Tropas** (L1, el átomo). Las **escuadras estándar**
las lidera un L3 con radio; las **escuadras especiales** (Infernales, commandos,
zapadores) las lidera un L2 sin radio y activan reglas propias. El campo
`compatible_squads` restringe qué tropa puede ir en qué escuadra (un Jinete solo
en Infernales).

### Al inicio — despliegue

4. **Colocar las escuadras** en su zona de despliegue, simultáneamente y a ciegas
   respecto del rival. La posición inicial es ya una decisión: define qué escuadras
   sostendrán el relay de mando hacia el frente. *Decisión abierta: reglas de
   colocación.*

### Cada turno — el ciclo de decisiones del Comandante

En cada **Orders**, el Comandante decide, dentro de su pool limitado:

- **Qué escuadras reciben orden específica** (ATTACK / MOVE / DEFEND) y cuáles no.
  No dar orden no es pasividad: una escuadra que defendía sigue defendiendo
  (estado pasivo persiste); una que se movía se detiene; una que atacaba cesa el
  ataque (estados activos se extinguen sin orden fresca). *Decisión abierta: qué
  hace una unidad que no estaba en ninguno de esos tres estados.*
- **Cuánto delegar**: gastar varias órdenes específicas microgestionando, o gastar
  **una** orden genérica sobre un oficial y dejar que su IA coordine al pelotón.
  Más delegación = más alcance con menos pool, pero menos control y riesgo de mala
  interpretación. Este es el dilema central, turno a turno.
- **Cómo sostener la cadena de mando.** Una escuadra está *en mando* si está a ≤5
  hexes (Manhattan) del HQ de Pelotón **o** de cualquier otra escuadra ya en mando
  —es un flood-fill hop a hop desde el HQ; cada escuadra en rango es repetidora
  involuntaria—. El jugador no asigna roles de relay: la topología se recalcula
  tras cada turno. Por eso una decisión posicional clave es no solo acercar tropas
  al HQ, sino mantener escuadras que sostengan el relay hacia el frente. Si el
  enemigo elimina un relay bien puesto, **aísla varias unidades de un solo golpe**.
- **Cómo gestionar la radio y el sigilo.** Solo L3/L4/L5 portan E-UHF (alcance 500
  m = 5 hexes, omnidireccional; de ahí que la regla de los 5 hexes sea literalmente
  el alcance físico de la radio). Si muere el **L3 líder**, la escuadra pierde la
  radio aunque queden L4/L5 —el valor está en el entrenamiento del líder, no en el
  aparato— y queda aislada: sigue combatiendo pero ya no recibe órdenes nuevas ni
  sirve de relay. La **Tropa de Comunicaciones** (L1 con radio global) no manda por
  sí sola: solo amplifica el alcance de un oficial L3+ al que esté asignada
  (*decisión abierta: cuánto amplifica*). Como toda señal puede triangularse y el
  HQ siempre está revelado, **pedir confirmación a un oficial expone su posición**:
  no responder y mantener *stealth* es una decisión táctica válida. Usar E-VHF
  entrega el contenido a todo el frente enemigo; E-UHF solo es seguro según el
  equipo de criptografía del rival. *Decisiones abiertas: rango y naturaleza
  exacta de E-VHF; tirada de detección para una sola unidad con criptografía; si
  movimiento o disparo rompen stealth; si existe un blackout de radio ordenado por
  el HQ; reestablecimiento de contacto por radio tras perder al L3.*
- **Cómo usar las escuadras especiales L2.** No entran en la cadena normal: reciben
  una **orden genérica** de su oficial de pelotón antes de operar ("Flanqueen por
  el este y corten la retirada") y la ejecutan con autonomía limitada, sin poder
  recibir nuevas instrucciones aunque el campo cambie. Son la herramienta de
  profundidad: flanqueo, incursión, sabotaje, donde mantener la línea de mando es
  imposible o indeseable. *Decisión abierta: en qué momento exacto se les asigna la
  orden genérica (¿fase Orders? ¿inicio de Briefing?).*

El jugador también convive con el **disband**: una tropa L1 que se queda sin ningún
superior L2+ vivo en su escuadra puede quebrarse —lo dispara una tirada de Valor;
si resiste, permanece; si falla, se quiebra—. Una escuadra con todos sus L1 en
disband cae a ROUTED o ELIMINATED. (Decisión abierta: condiciones exactas que
disparan la tirada de Valor para disband.)

### El cierre — cómo se gana

El jugador juega hacia **dos condiciones de victoria**:

- **Captura del HQ enemigo**: una unidad propia alcanza y captura/destruye el hex
  del HQ rival. Victoria inmediata, fin de partida.
- **Control de objetivos**: al agotarse el límite de turnos del escenario, gana
  quien controle más hexes-objetivo (presencia sin oposición activa).

No son condiciones de victoria la **eliminación total** (el juego es de desgaste,
no de aniquilación) ni existe **rendición**. *Decisiones abiertas: objetivos y
puntos por escenario, mecanismo exacto de control, y —notablemente— qué pasa al
perder el propio HQ (¿fin inmediato?, ¿asume el siguiente en la cadena?, ¿qué pasa
con el pool de órdenes y los relays activos?).*

---

## 4. Cierre — decisiones de diseño aún abiertas

El plan está completo en su **arquitectura y su filosofía**, pero deja
deliberadamente sin cerrar la **numerología y varias mecánicas de detalle**. Estas
son las decisiones abiertas declaradas por los specs, agrupadas:

**Combate e iniciativa**
- Fórmula exacta que mapea los cuatro factores de potencia → probabilidad final.
- Factores y fórmula de la **iniciativa**.
- Secuencia interna de resolución (movimiento → combate → post-procesado).
- Distancia de detección y mecanismos de reacción en movimiento.
- Designación de objetivos por soldado a nivel interfaz y lógica de distribución
  de fuego.
- Si un observador/spotter mejora la precisión del fuego indirecto.
- Umbral de eliminación por acumulación de heridas; nombres y efectos de heridas.

**Valor y estados**
- Triggers exactos y cantidades de degradación de Valor.
- Fórmula de ponderación del Valor de escuadra.
- Umbrales RETREAT y ROUTED; velocidad y condiciones de recuperación.
- Condiciones exactas que disparan la tirada de Valor para disband.
- Fórmula definitiva de `strength` y `moral` computados de la escuadra.

**Órdenes y movimiento**
- Cuántas órdenes aporta cada nivel L5/L4/L3 al pool.
- Pools de puntos de movimiento por tipo de unidad y tabla de costos de terreno.
- Escala del bonus de fortificación por turno consecutivo de DEFEND.
- Si DEPLOY existe solo en setup o también en juego.
- Lista de órdenes genéricas y patrones de comportamiento por tipo; mapeo de
  entrenamiento del oficial → calidad de interpretación.
- Comportamiento de una unidad sin orden que no estaba defendiendo/moviendo/atacando.
- Momento exacto en que se asignan órdenes genéricas a escuadras especiales L2.

**Mando y comunicaciones**
- Alcance exacto de la amplificación de la Tropa de Comunicaciones.
- Reestablecimiento de contacto por radio tras perder al L3.
- Consecuencias de perder el HQ propio (fin de partida, sucesión, pool, relays).
- Rango y naturaleza de E-VHF; tirada de detección para una sola unidad con
  criptografía; si la estimación de distancia es en hexes o cualitativa.
- Si movimiento o disparo rompen stealth; existencia de un blackout de radio.

**Escenarios y composición**
- Formato/serialización/validación de escenario.
- Valores de presupuesto por dificultad; objetivos y puntos de victoria por
  escenario; mecanismo de control de objetivos.
- Tamaño/forma de zonas de despliegue, reglas de colocación, límites de tiempo.
- Si se permiten composiciones custom, equipamiento modificable o progresión
  entre escenarios.
- Schema de la tabla Equipment (relación Tropa–Equipment).
- Mecánica especial pendiente del Cabo Rojo; catálogo extensible de terreno;
  sistema de elevación (todo `0.0` en prototipo).

Ninguno de estos huecos contradice la visión: son parámetros que el diseño
todavía no fijó. La narrativa del juego —Comandante encerrado, radio
interceptable, órdenes que se pierden, victoria por erosión— ya está completa y es
coherente de punta a punta.
