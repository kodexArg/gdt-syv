# ADR-007: Listen server como modelo de prototipado

## Estado

Aceptado — 2026-04-08

## Contexto

La arquitectura de SyV requiere un servidor autoritativo separado del cliente (ver ADR-003). Sin embargo, en fase de prototipo, ejecutar multiples procesos — un servidor headless mas clientes independientes — introduce friccion innecesaria: multiples ventanas, multiples instancias de depuracion, mayor complejidad para iterar rapidamente sobre las mecanicas del juego.

Se necesita un modelo de desarrollo que respete la separacion client/server desde el primer dia sin pagar el costo operativo de multiples procesos.

## Decision

En fase de prototipo, una unica instancia de Godot actua como **servidor y cliente simultaneamente** (listen server). La separacion de responsabilidades se implementa en el codigo, no en los procesos, mediante:

- `multiplayer.is_server()` para bifurcar logica segun el rol
- `@rpc` para marcar funciones invocables remotamente
- `set_multiplayer_authority()` para establecer la autoridad por nodo

La `MultiplayerAPI` de Godot se utiliza desde el primer commit, incluso cuando todo corre en un solo proceso. Esto significa que el codigo de prototipo **es** el codigo de produccion — solo cambia quien lo ejecuta.

### Camino de iteracion

1. **Fase actual**: una instancia = servidor + jugador (listen server)
2. **Test multijugador**: se lanza una segunda instancia de Godot que conecta a la primera
3. **Produccion**: se extrae la escena server a un proceso headless dedicado; los clientes conectan remotamente

Cada paso es incremental. No hay reescritura entre fases.

## Consecuencias

**Positivas:**
- Iteracion rapida con una sola ventana y un solo depurador
- La arquitectura multiplayer se respeta desde el dia uno — no hay deuda tecnica de networking
- El mismo codigo escala a servidor dedicado sin modificaciones
- Facilita el testing de las tres fases (Briefing, Orders, Resolution) en un solo proceso

**Negativas:**
- En listen server, el jugador local tiene acceso al proceso servidor (comparte memoria). Esto es aceptable en prototipo pero invalida la seguridad por scope (ADR-005) — razon por la cual produccion requiere procesos separados.
- Puede generar falsa confianza: algunas condiciones de red (latencia, perdida de paquetes) no se manifiestan en localhost.

## Alternativas descartadas

- **Single-player puro sin MultiplayerAPI**: facil al inicio, pero genera friccion significativa al retrofitear toda la comunicacion de red mas adelante. Cada llamada directa a la logica de juego deberia reemplazarse por un RPC, y la separacion de autoridad deberia añadirse retroactivamente.
- **Procesos separados desde el inicio**: overhead de desarrollo innecesario en fase de prototipo. Ralentiza la iteracion sin aportar beneficio tangible cuando todo corre en la misma maquina.
