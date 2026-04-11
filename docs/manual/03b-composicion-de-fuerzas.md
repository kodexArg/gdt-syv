---
title: "Composición de Fuerzas"
order: 3.5
section: factions
lang: es
---

# Composición de Fuerzas

## Presupuesto de puntos

Al inicio de cada escenario, ambos jugadores reciben un presupuesto de puntos. Este presupuesto define la cantidad de tropas que pueden desplegar en el campo de batalla.

**Características del sistema:**

- **Presupuesto fijo:** El escenario especifica una cantidad de puntos disponibles (Pendiente de definir: valores específicos por dificultad).
- **Selección de escuadras:** Cada jugador elige escuadras del catálogo de su facción hasta agotar el presupuesto.
- **Coste por escuadra:** Cada escuadra predefinida tiene un coste en puntos, ya definido en los archivos YAML de cada facción (`docs/premade-squads/conf-*.yaml`, `docs/premade-squads/rojos-*.yaml`).
- **Presupuesto variable:** El presupuesto puede variar según el escenario, permitiendo partidas pequeñas (20–30 puntos), medianas (50–70 puntos) o grandes (100+ puntos).

## Selección de fuerzas

Antes de desplegar, el jugador:

1. Revisa el presupuesto asignado para la partida
2. Consulta el catálogo de escuadras de su facción
3. Selecciona escuadras hasta agotar o aproximarse al presupuesto
4. Coloca sus escuadras en la zona de despliegue simultáneamente con su oponente

**Ejemplo:** Un escenario con presupuesto de 60 puntos. La Confederación selecciona:
- Escuadra de Infantería A (15 puntos)
- Escuadra de Infantería B (15 puntos)
- Escuadra de Morteros (18 puntos)
- Escuadra de Reconocimiento (12 puntos)
- **Total: 60 puntos**

Los Rojos seleccionan su propia composición con el mismo presupuesto, sin verla.

## Catalogo de escuadras por facción

Cada facción mantiene su propio catálogo de tipos de escuadra disponibles. Los costes en puntos ya están definidos en:

- **La Confederación:** `docs/premade-squads/conf-*.yaml`
- **Los Rojos:** `docs/premade-squads/rojos-*.yaml`

Para detalles de composiciones concretas, ver [04b — Ejemplos de Escuadras](04b-ejemplos-de-escuadras.md).

## Personalización (Pendiente de definir)

Está pendiente definir si los jugadores pueden:
- Modificar composiciones predefinidas (reemplazar tropas, agregar equipamiento)
- Crear escuadras custom a partir de reglas base
- Aplicar bonificadores de experiencia o penalizadores de desgaste por escenarios anteriores
