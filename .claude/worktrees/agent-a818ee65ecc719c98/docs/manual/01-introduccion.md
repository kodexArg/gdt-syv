---
title: "Introducción"
order: 1
section: intro
lang: es
---

# Introducción

## Qué es Subordinación y Valor

Subordinación y Valor es un juego de estrategia por turnos simultáneos en el que dos jugadores comandan fuerzas de infantería sobre un campo de batalla hexagonal. Cada jugador controla a un único personaje — su Comandante — desde un puesto de mando estático, el HQ, y transmite órdenes por radio a sus oficiales, quienes las retransmiten a las unidades en el campo.

El resto de la cadena de mando está automatizado. Los oficiales y la tropa ejecutan las órdenes recibidas — si es que las reciben. Las transmisiones pueden fallar, los oficiales pueden caer antes de retransmitir, y las unidades pueden ignorar una orden, rebelarse, o actuar por cuenta propia. Cada orden genera un mensaje semántico que viaja por la cadena de mando; si llega tarde, se ejecuta literalmente sobre un campo de batalla que ya cambió.

El juego es multijugador por diseño: dos clientes separados, simultáneos, asíncronos. Cada jugador ve solo lo que sus unidades pueden ver. Lo que está fuera de su alcance no existe en su pantalla.

La partida transcurre en la Franja de Alsina, año 2178. La Confederación Argentina marcha al sur para recuperar Bahía Blanca y encuentra una resistencia inesperada: las fuerzas Rojas, bien pertrechadas y comprometidas. Por una ironía del destino, la guerra se libra exactamente sobre la vieja zanja de Alsina — aquella que en el siglo XIX dividía la tierra a los pueblos originarios, ahora trinchera entre dos ejércitos.

La Confederación viste cascos verdes e incluye miniaturas eclesiásticas. Los Rojos llevan cascos camuflados con vetas rojas y un aspecto más decontracturado. En pantalla, el líder Rojo aparece siempre a la izquierda y el Comandante verde a la derecha. El jugador elige su facción — e incluso puede enfrentar Rojos contra Rojos.

## El mundo

Año 2178. La Confederación Argentina — una teocracia militar que gobierna desde Ciudad Dársena — marcha al sur para recuperar Bahía Blanca. Del otro lado de la Franja de Alsina los esperan las fuerzas Rojas: una resistencia bien pertrechada que no piensa rendirse.

La Franja de Alsina es la vieja zanja que en el siglo XIX separaba el territorio de los pueblos originarios. Ahora es trinchera entre dos ejércitos en una guerra intestina, donde la Confederación despliega infantería de cascos verdes y miniaturas eclesiásticas, y los Rojos responden con tropas de cascos camuflados y vetas rojas.

El universo completo de Subordinación y Valor — su historia, facciones, personajes y geografía — está documentado en [kodexArg/syv](https://github.com/kodexArg/syv).

## La filosofía del juego

Las unidades se desgastan, las órdenes se pierden, y el plan perfecto no sobrevive al contacto con el enemigo. La victoria se construye acumulando ventajas posicionales — no con un golpe decisivo sino con presión sostenida, como olas erosionando un acantilado.

Tu habilidad no está en mover soldados. Está en adaptarte a lo que realmente pasa cuando la cadena de mando falla, la moral quiebra, y el campo ya no se parece a lo que planeaste.

## Tu primera partida

Al conectarte a una partida, elegís tu facción y configurás la composición de tu ejército. El campo de batalla aparece desde arriba: una grilla de hexágonos con terreno y elevación variable. Tu HQ está en tu lado del mapa — es tu puesto de mando, la casilla que el enemigo quiere conquistar.

Cada turno representa cuatro horas de operaciones y tiene tres fases:

1. **Briefing** — El servidor te muestra el estado del campo según lo que tus unidades pueden ver. Evaluás la situación: posiciones, bajas, movimientos detectados.

2. **Orders** — Seleccionás unidades y les asignás órdenes: mover, atacar, defender. Cada orden se traduce en un mensaje que tu Comandante transmite por radio a la cadena de mando. Podés cambiar de opinión las veces que quieras. Cuando estás listo, confirmás. Tu oponente hace lo mismo en su propia pantalla, al mismo tiempo, sin saber qué ordenaste.

   También podés dar órdenes genéricas a un oficial — "Ataque a toda costa", "Mantenga posición" — y el oficial actuará según su criterio, coordinando sus infanterías de forma autónoma. Esto libera órdenes del Comandante, pero te arriesgás: el oficial puede interpretar mal la directiva, o tomar una decisión desastrosa. Este es, en esencia, un juego de órdenes — y la tensión está en cuánto control delegás.

3. **Resolution** — El servidor ejecuta las órdenes de ambos jugadores y muestra los resultados, evento por evento. Las transmisiones que llegaron se ejecutan. Las que no llegaron, no. Las que llegaron tarde de turnos anteriores... también se ejecutan, literalmente, sobre un campo que ya cambió.

Después de la Resolution, vuelve el Briefing con el nuevo estado. El ciclo se repite hasta que se cumple una condición de victoria.
