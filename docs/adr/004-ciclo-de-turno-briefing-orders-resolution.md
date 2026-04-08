# ADR-004: Ciclo de turno — Briefing, Orders, Resolution

## Estado

Aceptado — 2026-04-08

## Contexto

SyV es un juego de estrategia por turnos simultaneos: ambos jugadores planifican al mismo tiempo sin conocer las acciones del oponente. Este modelo requiere fases claramente separadas con responsabilidades distintas y actores definidos (servidor o cliente) en cada una. Sin una estructura formal de turno, la comunicacion entre cliente y servidor se vuelve ambigua y dificil de depurar.

## Decision

Cada turno sigue un ciclo de tres fases en orden estricto:

### 1. Briefing (servidor)

El servidor calcula el nuevo estado del juego y lo comunica a cada jugador con **scope individual** — cada cliente recibe unicamente la informacion que le corresponde conocer.

Responsabilidades:
- Distribucion de piezas y condiciones a cada jugador
- Comunicacion de cambios en el mapa
- Validacion de coherencia: verifica que la data enviada como eventos durante la Resolution anterior coincida con el estado actual del juego

### 2. Orders (cliente)

Cada jugador asigna ordenes a sus unidades desde su cliente. Esta fase es **enteramente local** — el backend no participa en absoluto.

Caracteristicas:
- El jugador dispone de undo y reset sin comunicacion con el servidor
- No hay limite de tiempo impuesto por el sistema (puede añadirse como regla de juego)
- Al confirmar, el cliente envia un unico mensaje con todas sus ordenes al servidor

### 3. Resolution (servidor)

El servidor recibe las ordenes de ambos jugadores y ejecuta la logica del juego en la secuencia que determina su logica de negocio. Comunica cada resolucion como **eventos secuenciales** a los clientes, para que presenten los efectos del turno de forma progresiva.

Al finalizar la Resolution, el ciclo regresa a Briefing, donde se verifica la coherencia y comienza un nuevo turno.

```
Briefing → Orders → Resolution → Briefing → ...
```

## Consecuencias

**Positivas:**
- La comunicacion entre cliente y servidor ocurre en momentos discretos y predecibles, no de forma continua
- Cada fase tiene un actor claro y responsabilidades acotadas, facilitando el debugging y el testing
- La fase de Orders no consume recursos del servidor — este permanece inactivo mientras los jugadores planifican
- El streaming de eventos en Resolution permite al cliente presentar la resolucion de forma progresiva y dramatica

**Negativas:**
- Rigidez del ciclo: toda interaccion entre jugador y juego debe encajar en estas tres fases
- El servidor debe esperar a que ambos jugadores confirmen sus ordenes antes de iniciar la Resolution

## Alternativas descartadas

- **Dos fases (sin Briefing separado)**: fusionar Briefing con el final de Resolution impide validar la coherencia entre turnos como paso independiente. Ademas, el calculo de scope individual es lo suficientemente complejo como para merecer su propia fase.
- **Resolution sin streaming de eventos**: si el servidor enviara unicamente el estado final (sin los eventos intermedios), el cliente no podria mostrar la resolucion de forma progresiva. El jugador veria solo el resultado, sin entender que ocurrio.
