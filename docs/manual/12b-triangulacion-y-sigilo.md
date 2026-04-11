---
title: "Triangulación y Sigilo"
order: 12.5
section: communications
lang: es
---

# Triangulación y Sigilo

## La triangulación como regla absoluta

Toda señal de radio en el campo **puede ser triangulada**. Sin excepción.

El universo de SyV transcurre después de **El Fin de los Secretos**, evento que volvió
irreversible la transparencia de las comunicaciones. No existe tecnología que resuelva esto —
ninguna encriptación, ningún protocolo nuevo puede ocultar la fuente de una transmisión activa.

Consecuencia directa: transmitir con frecuencia tiene un costo estratégico real. La gestión de
la comunicación es tan importante como la gestión de las órdenes. Ver también:
[Comunicaciones](12-comunicaciones.md).

## El HQ siempre está revelado

El HQ del Comandante es una posición estática y conocida. El enemigo no necesita triangularlo:
ya sabe dónde está. No existe stealth para el HQ.

## Stealth

El **stealth** es un estado de una unidad. Una unidad en stealth no emite señal activa que
pueda ser detectada o triangulada.

Una unidad **mantiene** el stealth cuando:
- No ha transmitido por radio en el turno actual
- Recibe una orden del HQ sin confirmarla (si la orden no requiere confirmación)

Una unidad **pierde** el stealth cuando:
- Transmite una respuesta por radio (E-UHF o cualquier otro canal)
- Realiza una acción que genera señal activa

<!-- Pendiente: definir si el movimiento físico o el disparo también rompen stealth -->

Ver [Niebla de Guerra](10-niebla-de-guerra.md) para la relación entre stealth y scope enemigo.

## Comunicación HQ → Oficial de Escuadra

El HQ puede transmitir órdenes al Oficial de Escuadra (L3–L5) sin que el receptor emita señal
de respuesta, siempre que la orden no exija confirmación.

| Canal | Resultado |
|-------|-----------|
| **E-VHF** | Interceptada automáticamente. Global y pública — todos los enemigos en el tablero conocen el contenido de la transmisión. |
| **E-UHF** (sin confirmación requerida) | El receptor puede mantener stealth. No emite señal de respuesta. Puede o no ser interceptada según equipo enemigo disponible. |
| **E-UHF** (con confirmación requerida) | El receptor debe responder. Ver tabla siguiente. |

<!-- Pendiente: definir rango de E-VHF y si existe como canal diferenciado o es simplemente
     una radio sin cifrado -->

## Comunicación Oficial de Escuadra → HQ (respuesta E-UHF)

Cuando el Oficial responde al HQ, emite una señal E-UHF activa. Los efectos dependen del
equipo enemigo en proximidad:

| Condición enemiga | Efecto |
|-------------------|--------|
| Al menos **una unidad** con equipo de criptografía en rango | Esa unidad obtiene la dirección del emisor y una estimación de distancia por intensidad de señal. El Oficial **rompe stealth** ante esa unidad. |
| **Dos o más unidades** a 5 hexes o menos | El Oficial es **triangulado e ubicado físicamente** con precisión. |
| Cualquier unidad con equipo a 5 hexes | El contenido de la transmisión es **desencriptado**, independientemente de si la unidad puede ubicar físicamente al emisor. |

<!-- Pendiente: definir tirada de detección exacta para el caso de una sola unidad
     con criptografía — ¿es automático o con roll? -->
<!-- Pendiente: ¿la estimación de distancia es en hexes o en rangos cualitativos
     (cerca/lejos)? -->

## Implicaciones tácticas

**Pedir confirmación tiene un precio.** Cada vez que el HQ exige que el Oficial responda,
expone la posición de ese Oficial. Si el enemigo tiene cobertura de criptografía, la respuesta
revela dirección y distancia. Si tiene dos unidades en rango, lo ubica con precisión.

**No responder es una decisión táctica válida.** Un Oficial que recibe una orden sin confirmación
obligatoria puede elegir ejecutarla en silencio y mantenerse en stealth. El HQ no tiene
confirmación, pero la unidad está operativa.

**El HQ no puede ocultarse.** El costo de ser el origen de todas las comunicaciones es la
exposición permanente. La cadena de mando fluye desde un punto conocido.

**El canal importa.** Usar E-VHF es ceder información a todo el frente enemigo. Usar E-UHF
no garantiza seguridad — solo la hace condicional al equipo del adversario.

<!-- Pendiente: definir si existe algún mecanismo de silencio de radio ordenado por el HQ
     (blackout de comunicaciones) y sus consecuencias sobre la cadena de mando -->
