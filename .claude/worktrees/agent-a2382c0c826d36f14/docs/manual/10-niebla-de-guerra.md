---
title: "Niebla de Guerra"
order: 10
section: fog
lang: es
---

# Niebla de Guerra

## Scope

Cada jugador ve solo lo que sus unidades pueden ver. La información del oponente no se oculta en el cliente — directamente no existe. El servidor envía a cada cliente únicamente su scope individual. No se puede hackear lo que no está.

## Inteligencia sobre enemigos: fórmula continua

La cantidad de información que un jugador recibe sobre unidades enemigas **no depende de capas discretas**, sino de una **fórmula continua** que evalúa múltiples factores.

### Cálculo de nivel de revelación

El servidor calcula para cada unidad enemiga un **"nivel de revelación"** basado en:

- **Distancia** del enemigo a unidades amigas
- **Tipo de observador** (infantería regular vs. escuadra especializada)
- **Terreno entre observador y objetivo** (obstáculos, vegetación)
- **Elevación relativa** (visibilidad vertical)
- **Condiciones climáticas** (futuro; ej. niebla, lluvia)

### Gradiente de información

El nivel de revelación forma un **gradiente continuo**:

- **Mínimo**: "algo está en esa dirección/área" — presencia confirmada sin detalles
- **Intermedio**: tipo de unidad, aproximadamente cuántos efectivos (rango: 1–5, 6–10, 10+)
- **Máximo**: composición exacta, estado, equipo, órdenes en curso

**Fórmula exacta y umbrales de revelación: Pendiente de definir**

El servidor decide qué información transmitir al cliente basándose en el nivel calculado. No existe mapeado fijo de "revelación X = información Y" — es un continuo.

### Observación múltiple

Cuando **múltiples unidades amigas** observan el mismo enemigo, el servidor usa la **mejor observación** (mayor nivel de revelación). El jugador ve el enemigo a través de los ojos de su mejor observador en ese momento.

## Reconocimiento: privilegio exclusivo de escuadras de tipo RECON

El reconocimiento mejorado **es una capacidad de tipo de unidad**, no una orden general.

- Solo las escuadras de tipo **"recon"** (definidas en el YAML de escuadras premontadas) tienen capacidad de observación mejorada
- Las escuadras de tipo "recon" tienen **valores de observación más altos** que la infantería regular, permitiéndoles revelar información a mayor distancia y con mayor precisión
- Infantería regular, apoyo, y otros tipos ven únicamente lo que la fórmula base de visión les permite
- Las escuadras recon pueden tener **habilidades especiales** (mayor rango, mejor detalle a distancia): Pendiente de definir
- **No existe una orden "RECONOCIMIENTO"** para todas las unidades — el reconocimiento es una propiedad del tipo de escuadra

## Inteligencia por radiogoniometría

Existe una **segunda fuente de inteligencia enemiga completamente independiente**: la triangulación de señales de radio.

El enemigo puede obtener información sobre posiciones amigas (y así revelar scope del jugador) interceptando y triangulando transmisiones de radio. Esto es gobernado por reglas separadas:

**Ver [Comunicaciones](12-comunicaciones.md) y [Triangulación y Sigilo](12b-triangulacion-y-sigilo.md)** para cómo las transmisiones E-UHF y E-VHF exponen posiciones y son interceptadas.

La radiogoniometría **no es afectada por línea de visión** — una transmisión puede ser triangulada a través de obstáculos. Es un riesgo estratégico de usar radio frecuentemente.

## Presentación visual

El terreno base (tipo, elevación) siempre es visible — el Comandante conoce el mapa. Los hexes fuera del alcance visual se muestran desaturados, sin información sobre unidades. Cuando una unidad avanza y gana visión sobre un enemigo, el hex donde está ese enemigo recupera información visual basada en el nivel de revelación calculado.

## Cálculo de scope (arquitectura servidor)

El servidor calcula el scope per jugador evaluando la fórmula de revelación desde la perspectiva de **cada unidad amiga**, para **cada unidad enemiga**. La información más detallada es transmitida al cliente. 

Este cálculo es **independiente de la presentación visual** — internamente el servidor conoce todo, pero envía al cliente solo lo permitido por la fórmula.
