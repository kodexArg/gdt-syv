---
title: "Niebla de Guerra"
order: 10
section: fog
lang: es
---

# Niebla de Guerra

## Scope

Cada jugador ve solo lo que sus unidades pueden ver. La información del oponente no se oculta en el cliente — directamente no existe. El servidor envía a cada cliente únicamente su scope individual. No se puede hackear lo que no está.

## Visión

<!-- Pendiente: cómo se calcula la visión
     - Rango base por tipo de unidad?
     - Efecto de elevación (cuando se implemente)
     - Terreno que bloquea línea de visión
     - Visión combinada de todas las unidades -->

## Presentación visual

El terreno base (tipo, elevación) siempre es visible — el Comandante conoce el mapa. Los hexes fuera del alcance visual se muestran desaturados, sin información sobre unidades. Cuando una unidad avanza y gana línea de visión, los hexes recuperan color y muestran su contenido.

## Reconocimiento

Las escuadras de reconocimiento (2 efectivos: observador + francotirador) son los ojos del ejército. Su función principal es revelar scope, no combatir.
