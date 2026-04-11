---
title: "Órdenes"
order: 6
section: orders
lang: es
---

# Órdenes

## El pool de órdenes

El Comandante tiene un número limitado de órdenes por turno, determinado por los oficiales vivos y conectados en el campo. **Cada oficial contribuye órdenes al pool según su rango**. Si un oficial cae o se desconecta de la cadena de mando, se pierden inmediatamente las órdenes que canalizaba. El pool es dinámico y puede cambiar durante la partida.

### Contribución por rango

Cada oficial "conectado" (dentro del alcance de la cadena de mando) aporta un número de órdenes que depende de su rango jerárquico:

- **L5** — aporta [Pendiente de definir] órdenes
- **L4** — aporta [Pendiente de definir] órdenes  
- **L3** — aporta [Pendiente de definir] órdenes

Un oficial desconectado no contribuye, aunque siga vivo en el tablero. Restaurar la conexión (reacercarse) restaura su aporte.

## Tipos de orden

### Órdenes específicas

Dirigidas a una unidad concreta. Consumen una orden del pool. **Una vez asignada, una orden no puede revocarse.** Durante la Resolution, la orden se transmite y ejecuta; no hay forma de cancelarla mid-turno. Para cambiar lo que hace una escuadra, el jugador debe esperar hasta la siguiente fase de Orders y asignar una nueva orden. Esto refuerza la filosofía de compromisos y consecuencias: las órdenes importan y no se pueden deshacer.

- **Atacar (ATTACK)** — la unidad avanza hacia un hex objetivo y ataca cualquier enemigo encontrado durante la Resolution. La intención es atacar, no solo repositionarse; la unidad continúa hacia su objetivo incluso si detecta enemigos en el camino.
- **Mover (MOVE)** — la unidad se mueve por un path de hasta 3 waypoints (ver [Sistema de movimiento](#sistema-de-movimiento)). Es una orden de repositionamiento; si detecta enemigos durante la travesía, **se detiene** sin avanzar más. No ataca a menos que el enemigo esté dentro de línea de vista en el hex donde se detiene.
- **Defender (DEFEND)** — la unidad mantiene posición con bonus defensivo. Si la misma escuadra recibe DEFEND en turnos consecutivos, el bonus defensivo se incrementa (fortification). Esto simula el atrincheramiento y mejora de posiciones a lo largo del tiempo. Una escuadra que ha estado defendiendo durante 3 turnos es mucho más difícil de desalojar que una que acaba de empezar a defender. **Scaling del bonus por turno de defensa consecutiva:** Pendiente de definir. Si la escuadra se mueve o recibe una orden diferente, el bonus de fortification se reinicia a cero. No consume del pool.

<!-- Pendiente: DEPLOY — solo en setup o también en juego? -->

### Sistema de movimiento

El movimiento está gobernado por un pool de **puntos de movimiento** que cada escuadra posee cada turno.

#### Puntos de movimiento

- **Pool base:** Cada tipo de unidad tiene un pool diferente de puntos de movimiento disponibles por turno
  - Infantería: menos puntos
  - Caballería: más puntos
  - Otros tipos: valores intermedios
  - **Valores específicos:** Pendiente de definir

#### Costo de terreno

Cada hex tiene un costo en puntos de movimiento para entrar, basado en su tipo de terreno:

- **Pasto/terreno base:** costo base (definir)
- **Agua:** imparable o costo muy alto (definir)
- **Cambios de elevación:** posible costo adicional (cuando exista elevación en el mapa; pendiente)

**Tabla de costos:** Pendiente de definir

#### Condiciones del movimiento

- Una escuadra **solo se mueve con una orden MOVE activa.** Sin orden de movimiento asignada en el turno, la escuadra no puede gastar puntos de movimiento ni cambiar de hex.
- El movimiento se gasta de forma progresiva: la escuadra elige un path de hasta 3 waypoints, calcula el costo total y valida que tenga suficientes puntos antes de ejecutar.
- **Puntos no gastados no se acumulan** al turno siguiente; se reinician en cada Briefing.

#### Movimiento en la línea de tiempo de Resolution

El movimiento no es instantáneo. Durante la Resolution, cada desplazamiento entre hexes genera **eventos con marca de tiempo** a lo largo de la línea de cuatro horas:

- "14:30 → Escuadra X sale de hex A"
- "15:10 → Escuadra X llega a hex B"

El tiempo transcurrido depende del tipo de terreno del hex de destino y la velocidad de la unidad. Durante el trayecto, la unidad se encuentra en un **estado intermedio** — no está completamente en el hex de origen ni en el de destino.

**Consecuencias:**
- Una unidad puede ser interceptada, detectada o atacada mientras está en tránsito
- El orden ATTACK vs. MOVE es táctico: ATTACK continúa persiguiendo; MOVE se detiene ante resistencia
- La secuencia de eventos en la línea de tiempo es determinante — una unidad con iniciativa alta puede ejecutar sus acciones (incluyendo movimiento) antes de que la enemiga complete el suyo

### Órdenes genéricas

Dirigidas a un oficial. El oficial interpreta la directiva y coordina sus unidades de forma autónoma. Libera órdenes del Comandante pero el oficial puede interpretar mal la directiva o tomar una mala decisión.

#### Interpretación por IA del oficial

Cuando un Comandante da una orden genérica a un oficial (ej: "Ataque a toda costa", "Mantenga posición"), el **oficial ejecuta un algoritmo de IA que interpreta la directiva**. Basándose en esa interpretación, el oficial **genera automáticamente sub-acciones específicas para sus escuadras subordinadas**.

Esas sub-acciones se insertan en la línea de tiempo de Resolution como cualquier otra acción — es decir, el efecto es como si el Comandante hubiera emitido múltiples órdenes específicas, pero con una acción de mando consumida.

**Ventaja táctica:** Una sola orden genérica reemplaza múltiples órdenes específicas, ahorrando recursos de mando.

**Riesgo:** La calidad de la interpretación depende del oficial. Un oficial con mejor entrenamiento (reflejado en estadísticas o rasgos) puede tomar decisiones más inteligentes. Un oficial inexperimentado puede interpretar mal la directiva o cometer errores tácticos.

**Patrones de comportamiento exactos por tipo de orden genérica:** Pendiente de definir

<!-- Pendiente: lista de órdenes genéricas disponibles
     ("Ataque a toda costa", "Mantenga posición", etc.) -->

## Comportamiento sin orden

Cuando una unidad no recibe orden en un turno, **no entra en "espera pasiva"**. En su lugar, ejecuta un comportamiento contextual que depende de su último estado:

- **Si estaba DEFENDIENDO** → continúa defendiendo (estado pasivo persiste)
- **Si estaba MOVIENDO** → se detiene en el hex actual (movimiento requiere orden activa)
- **Si estaba ATACANDO** → cesa el ataque (ataque requiere orden activa)

En otras palabras: los estados defensivos/pasivos persisten sin órdenes frescas, pero los estados activos/ofensivos se extinguen.

<!-- Pendiente: comportamiento exacto por estado de unidad
     (¿qué hace una unidad que no estaba en ninguno de estos tres?) -->

## Transmisión por radio

Cada orden genera un mensaje semántico que viaja por la cadena de mando: Comandante → Oficial (radio) → Unidades (radio). La transmisión puede fallar:

- El oficial muere antes de retransmitir
- La unidad pierde radio (disband)
- La unidad ignora la orden, se rebela, o actúa por cuenta propia (tirada de Valor)
- La orden llega con retraso de turnos anteriores y se ejecuta literalmente sobre un campo que ya cambió

## Cómo se asignan (interfaz)

El sistema Cycle-Tap heredado de antigravity:

- Tap sobre unidad → la selecciona
- Tap sobre la misma unidad → DEFEND
- Tap sobre hex adyacente → ATTACK
- Tap sobre el mismo adyacente de nuevo → inicia path de MOVE
- Taps sucesivos sobre hexes vecinos → extiende el path (máximo 3 waypoints)
- Tap en el origen → confirma MOVE

Las órdenes confirmadas aparecen como trazos tenues sobre el tablero. Los botones de la franja inferior acompañan el contexto (confirmar, cancelar).
