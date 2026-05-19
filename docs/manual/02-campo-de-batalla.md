---
title: "El Campo de Batalla"
order: 2
section: battlefield
lang: es
---

# El Campo de Batalla

## Mapas: Escenarios predefinidos

Cada partida utiliza un **escenario previamente diseñado**. No hay generación procedural de mapas ni editor de mapas (por ahora).

El escenario define:
- Diseño del tablero: distribución de terreno, agua, elevación
- Hexes objetivos y sus puntos de victoria
- Zonas de despliegue por facción
- Límite de turnos para la partida
- Presupuesto de puntos de despliegue por facción

**Implicación táctica:** Toda la balance del juego es manual. Cada mapa se ajusta a mano para garantizar que ambas facciones tengan oportunidades simétricas de victoria según sus fuerzas.

**Formato de escenario:** Pendiente de definir (estructura, serialización, validación)

## La grilla hexagonal

El campo de batalla es una grilla de hexágonos con orientación flat-top y radio 20, lo que produce 1.261 casillas. Cada hexágono tiene seis vecinos y dos propiedades: un tipo de terreno y un nivel de elevación.

**Escala**: cada hexágono representa **100 metros de diámetro**. La grilla cubre aproximadamente 4 km de extremo a extremo. Ver [Escala y Distancias](02b-distancias.md) para la tabla completa de conversión.

## Sistema de coordenadas

Cada hex se identifica con un par de enteros `(q, r)` en el sistema
**axial** (flat-top). El origen `(0, 0)` es el hex central. La
distancia entre dos hexes es la **distancia hexagonal** (pasos mínimos
entre adyacentes), no la distancia Manhattan.

La especificación completa — vecinos, fórmula de distancia,
coordenadas cúbicas derivadas y granularidad de unidad — está en
[ADR-023](../adr/023-modelo-espacial-coordenadas-y-distancia.md).

## Terreno

Cada hex tiene un tipo de terreno que define si es transitable y cómo afecta a las unidades que lo ocupan. Actualmente existen dos tipos:

| Terreno | Transitable | Efecto |
|---------|-------------|--------|
| **Pasto** | Sí | Sin modificadores. Terreno estándar. |
| **Agua** | No | Intransitable. Bloquea movimiento. |

El sistema de terreno está diseñado como **abierto y extensible**: cada tipo de terreno define independientemente sus reglas de movimiento, visión y combate. Nuevos tipos de terreno se incorporan como definiciones adicionales sin modificar los existentes. En fases futuras, el catálogo podrá expandirse, pero los tipos y efectos específicos se definirán en su momento.

## Elevación

Cada hex tiene un valor de elevación expresado como float. La elevación afecta tanto a la **visión** como al **combate**: el terreno elevado ve más allá (menos bloqueos de línea de visión) y proporciona ventaja táctica en el combate. Sin embargo, en la fase de prototipo, **todos los hexes tienen elevación `0.0`** — este es un sistema futuro que se implementará conforme el juego evolucione.

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
