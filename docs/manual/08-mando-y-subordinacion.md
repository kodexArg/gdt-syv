---
title: "Mando y Subordinación"
order: 8
section: command
lang: es
---

# Mando y Subordinación

## La cadena de mando

La fuerza militar de un jugador opera como una cadena: cada eslabón depende del anterior. El Comandante (HQ estratégico, estático) da órdenes genéricas a los Tenientes de Pelotón (L5). Ellos coordinan sus escuadras a través de sus Sargentos Primero (L4). Los Sargentos (L3) lideran las escuadras en el campo. Dentro de cada escuadra, los Cabos (L2) organizan grupos. La tropa básica (L1) ejecuta o se desbanda.

```
HQ (Comandante)
  └── Pelotón (Teniente L5 + Sargento Primero L4)
        └── Escuadra (Sargento L3)
              └── Grupo (Cabo L2)
                    └── Tropa (Soldado L1)
```

Romper un eslabón tiene consecuencias para todo lo que hay detrás.

## La regla de los 5 hexes encadenada

Una escuadra está **en mando** si se encuentra a 5 hexes o menos (distancia Manhattan) del HQ de Pelotón, **o** a 5 hexes o menos de cualquier otra escuadra que ya esté en mando.

Esto es un flood-fill desde el HQ: la red de mando se expande hop a hop. Cada escuadra dentro del radio actúa como repetidora para las que están detrás de ella.

**Ejemplo:** El HQ está en el centro. La Escuadra A está a 4 hexes del HQ — en mando. La Escuadra B está a 8 hexes del HQ, pero a 3 hexes de la Escuadra A — también en mando, a través del relay. La Escuadra C está a 7 hexes de la Escuadra A y a 11 del HQ — fuera de mando.

Esta mecánica fuerza decisiones posicionales reales: mantener la cadena no es solo acercar tropas al HQ, sino también mantener escuadras en posiciones que sostengan el relay hacia el frente.

## Relay y ruptura de cadena

Las escuadras intermedias son relay involuntarios. Si una escuadra que sostiene la conexión hacia el frente es eliminada, todas las escuadras que dependían de ese relay pueden quedar fuera de mando en el mismo turno.

El jugador no asigna explícitamente roles de relay: la topología se calcula automáticamente después de cada turno. Una escuadra puede ser relay para varias otras simultáneamente.

**Consecuencia táctica:** Una escuadra enemiga que elimina un relay bien posicionado puede aislar a varias unidades aliadas en una sola acción.

## Radio: quién la tiene y qué pasa si se pierde

Las tropas L3, L4 y L5 portan radio. Esto es lo que les permite mantenerse en la cadena de mando y actuar como relay.

Una escuadra tiene radio (`has_radio = true`) mientras al menos una de sus tropas portadoras esté viva. Si todas las tropas con radio en una escuadra mueren, la escuadra **pierde su radio**.

Una escuadra sin radio no puede actuar como relay. Además, si ya no cuenta con radio propia para comunicarse con el HQ, entra en situación de disband.

<!-- Pendiente: efecto mecánico exacto del disband — ¿la escuadra actúa sola? ¿se detiene? ¿intenta reagruparse? -->

## La Tropa de Comunicaciones

La Tropa de Comunicaciones es L1 con una radio de alcance global. Esto no la convierte en mando: no puede actuar de forma autónoma, no extiende la cadena de mando por sí misma, y no porta autoridad sobre nadie.

Lo que sí hace: cuando está asignada a una escuadra con un oficial L3 o superior, **amplifica el radio de ese oficial**. La escuadra alcanza más lejos dentro de la cadena de mando.

<!-- Pendiente: alcance exacto de la amplificación (¿radio doble? ¿alcance global del relay?) -->

Si la Tropa de Comunicaciones muere en una escuadra que dependía de esa amplificación, la escuadra vuelve al radio estándar de 5 hexes de su oficial. Si ese radio ya no alcanza al HQ ni a otro relay, la escuadra puede quedar fuera de mando.

## Escuadras especiales L2 y la autonomía sin radio

Las escuadras especiales (Los Infernales, commandos, zapadores) son lideradas por un L2 y operan **sin radio**. No participan en la cadena de mando normal: no reciben órdenes directas del HQ ni actúan como relay.

En cambio, reciben **órdenes genéricas** de su oficial de pelotón antes de entrar en operación: "Ataquen el flanco este", "Hostiguen y retiren", "Aseguren el cruce". Con esa directiva, la escuadra actúa con autonomía limitada según sus reglas especiales.

Esto las hace útiles para operaciones en profundidad — flanqueos, incursiones, sabotaje — donde mantener la línea de mando es imposible o indeseable. Pero también significa que no pueden recibir nuevas instrucciones durante esa operación. Si la situación cambia, la escuadra ejecuta la última orden genérica recibida.

**Ejemplo:** Los Infernales reciben "Flanqueen por el este y corten la retirada". Durante la resolución del turno, la posición del enemigo cambió y el flanco ya no es viable. Los Infernales ejecutan igualmente la orden, según su criterio dentro de esa directiva.

<!-- Pendiente: cuándo exactamente se asignan las órdenes genéricas a escuadras L2 — ¿en la fase Orders? ¿al inicio del Briefing? -->

## Disband

Una tropa L1 sin ningún superior L2 o superior vivo en su escuadra puede **desbandar**: abandonar su posición, actuar por cuenta propia, o simplemente dejar de funcionar como unidad coordinada.

La acción de desband no es automática: la dispara una tirada de Valor. Si la tirada tiene éxito, la tropa resiste y permanece. Si falla, se quiebra.

| Nivel | Qué pasa al desbandar |
|-------|-----------------------|
| **L1** | Puede desbandar si pierde todo mando superior. La tirada de Valor determina si resiste o se quiebra. |
| **L2 (especial)** | Opera con autonomía limitada incluso sin contacto con el pelotón. Riesgo de disband solo si la orden genérica ya no puede ejecutarse. |

Una escuadra con todas sus tropas L1 en disband pasa a estado `ROUTED` o `ELIMINATED` según la severidad.

<!-- Pendiente: condiciones exactas que disparan la tirada de Valor para disband — ¿post-resolución? ¿cada turno fuera de mando? -->

## Perder al Comandante

El HQ estratégico es una casilla en el tablero. Si el enemigo la captura, el Comandante cae.

<!-- Pendiente: consecuencias de perder el HQ —
     ¿es fin de partida inmediato? ¿el siguiente en la cadena asume?
     ¿qué pasa con el pool de órdenes? ¿qué pasa con los relays activos? -->
