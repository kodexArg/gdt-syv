---
title: "El Campo de Batalla"
order: 2
section: battlefield
lang: es
---

# El Campo de Batalla

## La grilla hexagonal

El campo de batalla es una grilla de hexágonos con orientación flat-top y radio 20, lo que produce 1.261 casillas. Cada hexágono tiene seis vecinos y dos propiedades: un tipo de terreno y un nivel de elevación.

**Escala**: cada hexágono representa **100 metros de diámetro**. La grilla cubre aproximadamente 4 km de extremo a extremo. Ver [Escala y Distancias](02b-distancias.md) para la tabla completa de conversión.

## Terreno

Cada hex tiene un tipo de terreno que define si es transitable y cómo afecta a las unidades que lo ocupan. Actualmente existen dos tipos:

| Terreno | Transitable | Efecto |
|---------|-------------|--------|
| **Pasto** | Sí | Sin modificadores. Terreno estándar. |
| **Agua** | No | Intransitable. Bloquea movimiento. |

Cada tipo de terreno es una definición independiente con sus propias reglas de movimiento, visión y combate. Nuevos terrenos — bosque, urbano, rocoso, trinchera — se incorporan como definiciones adicionales sin modificar los existentes.

## Elevación

Cada hex tiene un valor de elevación expresado como float. La elevación puede ser positiva o negativa, y sus efectos en visión, combate y movimiento se definirán conforme el sistema de terreno se expanda. En esta etapa, todos los hexes tienen elevación `0.0`.

## Lo que ves en pantalla

El campo de batalla se presenta como un tablero visto desde arriba. Los hexágonos de pasto cubren la mayor parte de la superficie, con cursos de agua marcando los límites naturales del terreno.

Tu HQ aparece en tu borde del mapa: una posición fija desde la que operás. Tus unidades se ven como fichas desplegadas sobre el tablero, agrupadas cerca de sus oficiales. Las de la Confederación en verde militar, las Rojas en camuflado con marcas rojas.

Los hexágonos fuera del alcance visual de tus unidades se muestran desaturados. El terreno es visible — tu Comandante conoce el mapa — pero no hay información sobre lo que hay encima. Cuando una unidad avanza y gana línea de visión, los hexes recuperan color y muestran su contenido.

## Interfaz

No hay interfaz tradicional. La pantalla es el tablero, y el tablero es la interfaz. Se arrastra para desplazarse por el mapa. Las unidades son interactuables directamente sobre la grilla.

La pantalla se organiza en tres franjas:

**Barra superior** — Una franja baja con la información mínima que el jugador necesita: órdenes pendientes, fase actual, lo esencial. Nada más.

**El tablero** — Ocupa todo el espacio central. Es donde sucede el juego. Seleccionar una unidad ilumina los hexes donde puede actuar. Las órdenes confirmadas se dibujan como trazos tenues. En la Resolution, los resultados se muestran evento por evento: movimientos, choques, rebotes, órdenes tardías que se ejecutan sobre un campo que ya cambió.

**Franja inferior (~20%)** — Botones contextuales, centrados. Nunca más de cuatro opciones más un botón Cancelar que siempre está a la derecha, gris, accesible. Los botones cambian según la fase: "Siguiente turno", "Confirmar órdenes", o las opciones que correspondan al contexto. Cuando no hay decisión pendiente, la franja queda vacía.

**Log de radio** — Nace desde la barra superior, esquina izquierda. Contraído, muestra solo el último mensaje recibido; un click lo despliega sobre el tablero, línea a línea, y la barra superior pasa a mostrar "(contraer)". Desplegado flota sobre el mapa sin reemplazarlo. Es la interfaz real con la cadena de mando: "Inf-3: Avanzar a (12,4) — recibido", "Inf-7: Mantener posición — sin respuesta".
