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

Dirigidas a una unidad concreta. Consumen una orden del pool.

- **Atacar (ATTACK)** — la unidad ataca un hex adyacente
- **Mover (MOVE)** — la unidad se mueve por un path de hasta 3 waypoints
- **Defender (DEFEND)** — la unidad mantiene posición con bonus defensivo (+1 fuerza). No consume del pool.
- **Cancelar (CANCEL)** — revoca una orden previamente asignada, devuelve la orden al pool

<!-- Pendiente: DEPLOY — solo en setup o también en juego? -->

### Órdenes genéricas

Dirigidas a un oficial. El oficial interpreta la directiva y coordina sus unidades de forma autónoma. Libera órdenes del Comandante pero el oficial puede interpretar mal la directiva o tomar una mala decisión.

<!-- Pendiente: lista de órdenes genéricas disponibles
     ("Ataque a toda costa", "Mantenga posición", etc.)
     y cómo el sistema las resuelve -->

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
- Tap sobre la misma una tercera vez → CANCEL
- Tap sobre hex adyacente → ATTACK
- Tap sobre el mismo adyacente de nuevo → inicia path de MOVE
- Taps sucesivos sobre hexes vecinos → extiende el path (máximo 3 waypoints)
- Tap en el origen → confirma MOVE

Las órdenes confirmadas aparecen como trazos tenues sobre el tablero. Los botones de la franja inferior acompañan el contexto (confirmar, cancelar).
