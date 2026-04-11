---
title: "Facciones"
order: 3
section: factions
lang: es
---

# Facciones

## Abstracción jerárquica

Todas las facciones usan el mismo sistema de juego. La diferencia está en los nombres, la identidad y la estructura interna. Los niveles jerárquicos son genéricos (L1–L5) y cada facción los mapea a sus propios rangos.

## La Confederación

Ejército regular de la Confederación Argentina. Estructura militar clásica con presencia eclesiástica integrada en la cadena de mando. Cascos verdes.

| Nivel | Rango | Función |
|-------|-------|---------|
| HQ | Teniente Primero | Jefe de Sección (el jugador) |
| HQ | Monseñor | Guía espiritual del comando |
| HQ | Oficial de Comunicaciones | Gestión de radio y cadena de mando del HQ |
| L5 | Teniente | Jefe de pelotón |
| L4 | Sargento Primero | Subjefe de pelotón |
| L3 | Sargento | Jefe de escuadra |
| L2 | Cabo Primero / Cabo | Subjefe, radio, funciones técnicas |
| L1 | Soldado / Capellán Médico | Ejecución, combate, sanidad |

Tipos de escuadra: infantería (8), reconocimiento (2), artilleros (5). Ver `docs/premade-squads/conf-*` para composiciones específicas.

## Los Rojos

Fuerzas del sur. Resistencia bien pertrechada con estructura más horizontal y rangos informales. Cascos camuflados con vetas rojas, aspecto decontracturado.

| Nivel | Rango | Función |
|-------|-------|---------|
| HQ | Líder Revolucionario | Comandante Rojo (el jugador) |
| HQ | Oficial de Comunicaciones | Gestión de radio y cadena de mando del HQ |
| L5 | Teniente | Jefe de pelotón |
| L4 | Subteniente | Subjefe de pelotón |
| L3 | Sargento | Jefe de escuadra |
| L2 | Basto (Cabo Primero) | Segundo al mando de escuadra, comunicaciones |
| L2 | Cabo | Especialista / tercero al mando |
| L2 | Jefe de Partida | Escuadras especiales (Los Infernales) |
| L1 | Soldado | Combate directo |
| L1 | Infernal | Los Infernales — compatible solo con escuadras de caballería |

La escuadra roja estándar es deliberadamente plana: un Sargento, un Basto y seis soldados sin especialización. El Basto es el apodo del Cabo Primero — segundo al mando con funciones de radio y coordinación. El Cabo, en cambio, cubre roles de especialista o tercero al mando según la composición de la escuadra. Tiene una mecánica especial pendiente de definir.

Cuerpo especial: "Los Infernales" — caballería ligera en motos eléctricas todoterreno. Escuadra L2: sin radio, opera con órdenes genéricas. Jefe de Partida (L2) al mando; la tropa se llama directamente Infernal (L1). Ver `docs/premade-squads/rojos-*`.

## Diferencias entre facciones

<!-- Las facciones son mecánicamente simétricas o asimétricas?
     La Confederación tiene capellanes y estructura rígida.
     Los Rojos tienen estructura plana con mecánica especial pendiente.
     Definir si hay diferencias en stats, habilidades, o solo en composición. -->

## Presentación en pantalla

El líder Rojo aparece siempre a la izquierda. El Comandante verde siempre a la derecha. El jugador elige su facción — e incluso puede enfrentar Rojos contra Rojos o Confederación contra Confederación.
