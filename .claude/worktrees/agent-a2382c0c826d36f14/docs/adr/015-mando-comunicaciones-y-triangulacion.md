# ADR-015: Mando, comunicaciones y triangulación — cierre del eje táctico

## Estado

Aceptado — 2026-05-18

## Contexto

El eje táctico de fog-of-war y aislamiento se apoya en tres capítulos del
manual: el 08 (Mando y Subordinación), el 12 (Comunicaciones) y el 12b
(Triangulación y Sigilo), con el alcance físico anclado en el 02b (Escala y
Distancias). Esos capítulos describen la cadena de mando por la regla de los
5 hexes encadenada, la radio como condición de pertenencia a esa cadena, y la
triangulación como regla absoluta del universo post–*El Fin de los Secretos*.

Pero ese eje quedó con bloqueantes mecánicos explícitos, marcados como
`Pendiente` en el manual (decisiones #31–#38 del backlog, "Grupo 7"):

1. Alcance exacto de la amplificación de la Tropa de Comunicaciones y su
   relación con la regla de 5 hexes (cap. 08, cap. 12).
2. Mecanismo de reestablecimiento de contacto por radio tras perder al L3
   (cap. 08).
3. Consecuencias de perder el HQ propio: fin de partida, sucesión, pool de
   órdenes, relays activos (cap. 08).
4. Rango y naturaleza de la **E-VHF** frente a la **E-UHF** (cap. 12, 12b).
5. Tirada de detección cuando hay **una sola** unidad enemiga con
   criptografía; si la estimación de distancia por triangulación es en hexes
   o cualitativa; si movimiento/disparo rompen stealth; si existe un blackout
   de radio ordenado por el HQ y sus consecuencias (cap. 12b).

Sin estas reglas el servidor no puede calcular `in_command` / `has_radio` de
forma estable, ni recortar el scope (ADR-005) de la información de
triangulación, ni alimentar el estado de mando `C` que ADR-012 ya consume
para la penalización de aislamiento.

Restricciones que la decisión respeta:

- Fidelidad al alcance físico del cap. 02b: 1 hex = 100 m; E-UHF táctica =
  500 m = 5 hexes; grilla de radio 20 (≈2 km de radio).
- **ADR-001** (nomenclatura): se usan los términos canónicos L1–L5, HQ,
  E-UHF, E-VHF, stealth, `in_command`, `has_radio` sin introducir sinónimos.
- **ADR-005** (seguridad por scope individual): toda información derivada de
  triangulación (dirección, distancia, contenido desencriptado, posición
  exacta) es estado del jugador que la obtiene y **se serializa solo en su
  scope**; nunca viaja al jugador triangulado. La triangulación no es un
  efecto de cliente: es estado calculado server-side y recortado por peer.
- **ADR-010** (contrato de protocolo de red): los códigos de respuesta E-UHF
  (`Recibido`, `Copiado`, `Negativo — imposible`, `En movimiento`) son los
  acuses C→S / mensajes S→C ya tipados; este ADR no añade mensajes nuevos al
  contrato, solo fija cuándo una transmisión es emitida y por tanto
  triangulable. El log de radio sigue siendo emisión individualizada por peer.
- **ADR-012** (cálculo de iniciativa): el estado de mando `C` ya está
  definido (in_command +15 / sin-mando-con-radio −5 / aislada −20, con
  excepción +5 para especiales L2). Este ADR **no toca** esa tabla; solo
  define las condiciones de borde (`has_radio`, recuperación de contacto,
  caída del HQ) que producen los valores que ADR-012 consume.

Este ADR es normativo sobre la **regla de juego**. La implementación Godot
(client/server/shared) y los esquemas viven donde ADR-008/010 los ubican.

## Decisión

### 1. Amplificación de la Tropa de Comunicaciones

La Tropa de Comunicaciones es L1 sin autoridad: no es origen de cadena, no
extiende mando por sí sola, no actúa como relay autónomo (cap. 08, cap. 12).
Su único efecto, cuando está asignada a una escuadra cuyo líder vivo es L3 o
superior:

- **Duplica el radio de relay de esa escuadra: de 5 a 10 hexes** (1.000 m),
  tanto para alcanzar al HQ de pelotón como para enlazar con otra escuadra ya
  en mando. La regla de los 5 hexes encadenada del cap. 08 se evalúa para esa
  escuadra con salto de **10 hexes** en lugar de 5; el flood-fill (BFS) desde
  el HQ es idéntico, solo cambia el peso del salto de ese nodo.
- El alcance amplificado **no es global**. Se descartó el "alcance ilimitado"
  para no neutralizar el pilar posicional del cap. 08 (mantener la cadena
  debe seguir siendo una decisión de terreno). El radio-operador / radio de
  largo alcance del HQ sí es ilimitado dentro de la grilla (cap. 12), pero la
  Tropa de Comunicaciones es un **multiplicador local ×2**, no una segunda
  estación de largo alcance.
- La amplificación exige líder L3+ **vivo** en la escuadra. Si el L3 muere,
  la escuadra pierde la radio (cap. 08) y la Tropa de Comunicaciones deja de
  amplificar nada: no rescata una escuadra ya aislada.
- Si la Tropa de Comunicaciones muere, la escuadra vuelve al radio estándar
  de 5 hexes en el mismo turno; si ese radio ya no alcanza HQ ni relay, queda
  fuera de mando (sin-mando-con-radio, `C = −5` en ADR-012; no aislada
  mientras el L3 viva).

Factor de amplificación ×2 / 10 hexes: valor de diseño, **sujeto a playtest**
(alternativa razonable: ×1,5 → 7–8 hexes si 10 aplana demasiado el terreno).

### 2. Reestablecimiento de contacto por radio tras perder al L3

Una escuadra cuyo L3 murió tiene `has_radio = false` y está **aislada**
(cap. 08; `C = −20` en ADR-012). El contacto **no** se recupera de forma
permanente ni automática. Mecanismo elegido (combinación proximidad + relevo
de mando):

- **Proximidad a una fuente de radio.** Al final de la Resolution, si la
  escuadra aislada está a **≤2 hexes (≤200 m)** de otra escuadra propia con
  `has_radio = true` y `in_command = true`, recupera contacto **solo para ese
  turno**: cuenta como en mando para recibir la orden del turno siguiente
  (la fuente vecina actúa como enlace de voz/runner, no como relay de radio
  propio). El estado no persiste: si al turno siguiente ya no hay vecino en
  ≤2 hexes, vuelve a aislada. Es contacto prestado, no radio recuperada.
- **Relevo de mando permanente.** El contacto **propio** solo se recupera de
  forma permanente si la escuadra obtiene de nuevo un líder L3 o superior:
  por refuerzo, por fusión con otra escuadra que aporte un L3+, o por
  promoción explícita de un L4/L5 presente a rol de líder de esa escuadra.
  Recuperado el L3+, `has_radio` vuelve a `true` desde el turno siguiente.
- **Oficial Criptógrafo no retransmite.** Se descarta usar al criptógrafo
  como repetidor amigo: en el universo de SyV el criptógrafo es un
  interceptor (cap. 12, 12b), no un nodo de la cadena propia. Mantenerlo
  exclusivamente ofensivo conserva la asimetría temática.

Umbral de proximidad de 2 hexes: valor de diseño, **sujeto a playtest**.

### 3. Pérdida del HQ propio

El HQ del Comandante es una casilla estática y siempre revelada (cap. 08,
cap. 12b). Si el enemigo la captura:

- **No es fin de partida inmediato.** La partida continúa; la victoria se
  resuelve por las condiciones del cap. 11, no por la sola caída del HQ.
- **No hay sucesión.** Ningún L5/L4 asume el rol de Comandante. El universo
  de SyV no premia la improvisación de mando: la cadena fluye desde un punto
  único y conocido (cap. 12b), y si ese punto cae, no se regenera.
- **El pool de órdenes se congela.** Desde el turno siguiente a la caída, el
  jugador **no puede emitir órdenes nuevas a ninguna unidad**. La fase Orders
  de ese jugador queda vacía por contrato (ADR-010: simplemente no envía
  `submit_orders` con órdenes nuevas; el servidor las rechaza server-side
  igual que cualquier intención inválida — ADR-005/010).
- **Los relays activos no se rcompensan, pero no se destruyen.** La topología
  de la regla de 5 hexes encadenada se sigue calculando (BFS), pero el origen
  del flood-fill (el HQ) desaparece: ninguna escuadra puede estar
  `in_command` porque no hay raíz. Todas pasan a operar con su última orden
  conocida: las que conservan L3 vivo quedan **sin-mando-con-radio**
  (`C = −5`), las que perdieron el L3 quedan **aisladas** (`C = −20`). Las
  escuadras especiales L2 conservan su autonomía deliberada (`C = +5`,
  excepción de ADR-012): son las únicas que siguen siendo tácticamente
  plenas tras la decapitación, lo cual es coherente con su rol de operación
  en profundidad (cap. 08).

Efecto neto: perder el HQ no termina la partida pero **decapita el mando** —
el jugador conserva unidades que ejecutan su última orden con reacción
degradada, salvo las especiales L2. Es un estado de derrota progresiva, no
una pantalla de derrota.

### 4. E-VHF frente a E-UHF

Quedan definidos como **dos canales diferenciados**, no como "radio con
cifrado" vs "radio sin cifrado" únicamente:

| Canal | Cifrado | Alcance | Dirección | Intercepción |
|-------|---------|---------|-----------|--------------|
| **E-UHF táctica** | Sí (alta rotación) | 500 m = 5 hexes | Omni | Condicional al equipo enemigo (cap. 12b) |
| **E-UHF largo alcance** (HQ / radio-operador / Tropa Comm) | Sí | Ilimitado en la grilla | Dirigida | Baja, condicional |
| **E-VHF** | Sí, pero **roto** | **Global (toda la grilla, radio 20 / ≈4 km extremo a extremo)** | Omni | **Automática y pública** |

- **E-VHF es global por naturaleza física**: la banda VHF de SyV se propaga
  sin atenuación útil dentro de la grilla; su alcance es **toda la grilla**,
  no un radio. No tiene sentido un "rango" en hexes para E-VHF: alcanza a
  todos.
- **E-VHF está comprometida por diseño del universo**: tras *El Fin de los
  Secretos* su cifrado es trivialmente roto. Toda transmisión E-VHF es
  **interceptada automáticamente y su contenido es público para todos los
  enemigos del tablero**, sin necesidad de criptógrafo ni triangulación
  (cap. 12b ya lo afirma; este ADR lo fija como regla cerrada). Es un canal
  de último recurso (p. ej. ordenar a una escuadra aislada sin L3 que aún
  porta un receptor VHF de respaldo) cuyo precio es ceder la orden completa
  al enemigo.
- E-VHF **no** restablece `has_radio` ni reintegra a la cadena de mando: es
  un canal de difusión de una sola vía, no un nodo de relay. Una orden por
  E-VHF llega a la unidad aislada pero no la vuelve `in_command`.

### 5. Triangulación, stealth y blackout

#### 5.1 Detección con una sola unidad enemiga con criptografía

El cap. 12b ya fija el caso de **dos o más** unidades con equipo a ≤5 hexes
(ubicación física precisa) y el de cualquier unidad con equipo a ≤5 hexes
(contenido desencriptado). Falta el caso de **una sola** unidad con equipo de
criptografía en rango cuando el Oficial emite respuesta E-UHF activa:

- **No es automático: hay tirada de detección.** Probabilidad base
  **60 %** de que la unidad enemiga obtenga *dirección* + *estimación de
  distancia*, modificada por la distancia al emisor dentro del rango de
  criptografía:
  - a 1–2 hexes: **+25 %** (señal fuerte) → 85 %
  - a 3–4 hexes: **+0 %** → 60 %
  - a 5 hexes (borde): **−20 %** → 40 %
- Tirada server-side, una por unidad enemiga con equipo (ADR-005: el
  resultado vive en el scope del jugador interceptor). Si la tirada tiene
  éxito, el Oficial **rompe stealth ante esa unidad** y el jugador enemigo ve
  dirección + distancia estimada en su Briefing. Si falla, el Oficial
  conserva stealth ante esa unidad ese turno (la transmisión ocurrió, pero la
  unidad no logró fijarla).
- Esto **no** altera el caso de ≥2 unidades a ≤5 hexes del cap. 12b: ahí la
  ubicación física precisa sigue siendo automática (la triangulación
  geométrica no necesita tirada). La tirada cubre **solo** el caso de fuente
  única.

Valores 60 % / ±25 / −20: de diseño, **sujetos a playtest**.

#### 5.2 Estimación de distancia: cualitativa, no en hexes

La estimación de distancia que obtiene una **única** unidad por intensidad de
señal es **cualitativa**, no un número de hexes:

| Banda | Hexes reales (oculto al jugador) | Lo que ve el jugador |
|-------|----------------------------------|----------------------|
| Cerca | 1–2 hexes | `Contacto cercano` |
| Media | 3–5 hexes | `Contacto a media distancia` |
| Lejos | 6+ hexes | `Contacto lejano` |

Solo la **triangulación con ≥2 unidades a ≤5 hexes** produce posición exacta
(hex preciso). Una sola unidad nunca entrega un hex: entrega dirección + banda
cualitativa. Esto preserva la asimetría informativa que ADR-005 protege:
saber "hay algo cerca al este" no es saber "está en el hex (4,−7)".

#### 5.3 Movimiento y disparo frente al stealth

Cierre del `Pendiente` del cap. 12b sobre si la acción física rompe stealth:

- **Mover no rompe stealth.** El stealth de SyV modela exclusivamente
  **silencio de radio**, no ocultación visual. Una unidad puede maniobrar y
  seguir en stealth mientras no transmita. (La visibilidad física se rige por
  el cap. 10 / niebla de guerra y ADR-005, que es un eje independiente.)
- **Disparar no rompe stealth de radio** por sí mismo, por la misma razón:
  stealth ≠ invisibilidad. Disparar puede revelar la unidad por
  línea de visión enemiga (cap. 10), pero **no la hace triangulable**: la
  triangulación es un fenómeno estrictamente de emisión de radio (cap. 12b,
  "regla absoluta": *toda señal de radio* puede triangularse — el disparo no
  es señal de radio).
- En consecuencia, lo único que rompe stealth (de radio) es **transmitir**:
  responder por E-UHF, usar E-VHF, o cualquier acuse que genere señal activa
  (ADR-010). Recibir una orden sin confirmar no rompe stealth (cap. 12b).

Esta separación —stealth de radio vs. detección visual— mantiene los dos ejes
de fog-of-war ortogonales y deja la niebla visual íntegramente bajo ADR-005 /
cap. 10.

#### 5.4 Blackout de radio ordenado por el HQ

Se define el mecanismo de silencio de radio (cierre del último `Pendiente`
del cap. 12b):

- El HQ puede declarar **blackout** sobre toda su fuerza al inicio de la fase
  Orders (es una orden global, no por unidad; viaja en el `submit_orders`
  del jugador — ADR-010, sin mensaje nuevo).
- Bajo blackout, durante la Resolution siguiente **ninguna unidad propia
  emite respuesta E-UHF**: todos los acuses se suprimen, ninguna unidad puede
  romper stealth por transmisión. El HQ tampoco emite por largo alcance.
- **Consecuencia sobre la cadena de mando:** las órdenes ya entregadas en la
  fase Orders **antes** de iniciar la Resolution siguen vigentes (el blackout
  silencia la *respuesta* y la transmisión durante la Resolution, no anula el
  briefing previo). Pero mientras el blackout está activo el HQ **no puede
  emitir órdenes nuevas a unidades fuera de su radio físico directo**: con la
  radio en silencio, la regla de 5 hexes encadenada se evalúa **sin relays de
  radio** — solo cuentan los enlaces por proximidad de la cláusula de
  recuperación de contacto (§2, ≤2 hexes). En la práctica, durante un
  blackout casi toda la fuerza opera con su última orden conocida: es un
  intercambio deliberado de **mando por invisibilidad**.
- El blackout dura **el turno en que se declara** y se levanta automáticamente
  al inicio de la siguiente fase Orders, salvo que el HQ lo renueve.
- El blackout **no** protege al HQ: el HQ siempre está revelado (cap. 12b),
  el blackout solo evita exponer a los Oficiales por respuesta.

Duración de un turno y umbral de 2 hexes para enlaces durante blackout:
valores de diseño, **sujetos a playtest**.

## Consecuencias

**Positivas:**

- Cierra los bloqueantes #31–#38: el servidor ya puede computar
  `has_radio`, `in_command` y el estado de mando `C` de ADR-012 de forma
  determinista, incluyendo los bordes (amplificación, pérdida de L3, caída de
  HQ, blackout).
- La triangulación queda completamente del lado del scope (ADR-005): toda
  información derivada (dirección, banda, posición exacta, contenido) es
  estado del interceptor, nunca viaja al triangulado. No requiere mensajes
  nuevos en ADR-010.
- Stealth de radio y niebla visual quedan ortogonales: dos ejes limpios de
  fog-of-war, cada uno con su capítulo (12b / 10) y sin acoplarse.
- Perder el HQ produce derrota progresiva (no pantalla de derrota), lo que
  preserva la tensión del cap. 11 y da valor narrativo a las escuadras
  especiales L2 como única fuerza que sobrevive a la decapitación.
- La amplificación local ×2 mantiene el pilar posicional del cap. 08 en lugar
  de neutralizarlo con alcance global.

**Negativas:**

- Introduce varios coeficientes sin calibrar (×2 amplificación, ≤2 hexes
  recuperación, 60 %±mods detección, duración de blackout): la sensación
  táctica real depende de playtest; todos marcados como tales.
- El blackout añade un modo de evaluación de la cadena de mando "sin relays
  de radio" que el servidor debe implementar como caso especial del BFS del
  cap. 08; complejidad extra en el cálculo de topología.
- La promoción de un L4/L5 a líder de escuadra (§2) toca el modelo de fuerza
  (ADR-009) en cómo se reasigna el rol de líder; este ADR fija la regla pero
  no el esquema de datos, que queda atado a la nomenclatura/estructura de
  ADR-009.
- La banda cualitativa de distancia (cerca/media/lejos) exige que el Briefing
  serialice un enum, no un escalar; trivial, pero es una forma más que el
  schema de scope (ADR-010) debe contemplar.

## Alternativas descartadas

- **Amplificación de alcance global para la Tropa de Comunicaciones:**
  neutraliza el pilar posicional del cap. 08 (mantener la cadena dejaría de
  ser una decisión de terreno). Multiplicador local ×2 conserva la tensión.
- **Recuperación de radio automática por mera supervivencia / paso del
  tiempo:** contradice el cap. 08 ("el valor de la radio está en el
  entrenamiento del líder, no en la electrónica"); sin L3+ no hay radio.
- **Sucesión de mando al caer el HQ (un L5 asume Comandante):** contradice el
  cap. 12b ("la cadena fluye desde un punto único y conocido"; "el HQ no
  puede ocultarse" implica que es irremplazable). Decapitación, no relevo.
- **Fin de partida inmediato al caer el HQ:** colapsa el cap. 11 (condiciones
  de victoria) en una sola casilla y elimina el valor de las especiales L2;
  preferimos derrota progresiva.
- **E-VHF como simple radio sin cifrado de rango corto:** desaprovecha la
  tensión narrativa de *El Fin de los Secretos*; como canal global público es
  una decisión de "último recurso con precio" mucho más expresiva.
- **Detección de fuente única automática (sin tirada):** elimina el matiz de
  la cobertura parcial de criptografía; la tirada con modificador por
  distancia da gradiente táctico sin contradecir el caso geométrico de ≥2
  unidades (que sí es automático).
- **Estimación de distancia en hexes exactos para fuente única:** rompe la
  asimetría informativa que ADR-005 protege; "algo cerca al este" no debe
  equivaler a una coordenada.
- **Movimiento/disparo rompen stealth:** acoplaría el eje de radio con el eje
  visual y haría redundante la niebla del cap. 10; mantenerlos ortogonales es
  más limpio y fiel a "stealth = silencio de radio".
- **Sin blackout (la radio nunca se puede silenciar):** elimina una decisión
  táctica central (mando vs. invisibilidad) que el propio cap. 12b sugiere
  como pendiente; incluirlo cierra el capítulo de forma coherente.
