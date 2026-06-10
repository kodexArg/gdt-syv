# Arquitectura — Subordinación y Valor

## Visión general

SyV es un juego de estrategia por turnos simultáneos, multijugador, en grilla hexagonal multi-nivel. La arquitectura sigue un modelo de **servidor autoritativo**: los clientes Godot son interfaces de presentación y entrada; toda la lógica de juego se ejecuta en un backend.

```
[Cliente A] —órdenes→ [Backend] ←órdenes— [Cliente B]
                           ↓
                    ejecuta la lógica
                           ↓
[Cliente A] ←estado→  [Backend]  →estado→ [Cliente B]
```

El cliente no tiene autoridad sobre ningún estado de juego. Solo confía en lo que el backend le comunica.

## Backend: Godot headless

El backend corre **Godot 4.4.1 en modo `--headless`**, con el mismo proyecto y la misma base de código GDScript que el cliente. La diferencia es la escena de entrada: el servidor carga una escena sin rendering que solo ejecuta lógica de juego.

### Por qué Godot headless

- **Una sola base de código.** La grilla hexagonal, unidades, reglas y facciones existen una sola vez en GDScript. El servidor carga los mismos Resources y Scenes que el cliente, sin renderizar.
- **Infraestructura nativa.** Godot 4.x provee `MultiplayerAPI`, `@rpc`, sistema de autoridad por nodo y branch con `multiplayer.is_server()`.
- **Cómputo mínimo.** Un juego por turnos no necesita ticks continuos. El servidor trabaja en momentos discretos.

### Despliegue

- **Prototipo (actual)**: Godot headless corriendo en localhost junto a los clientes
- **Producción (futuro)**: contenedor Linux + Godot headless + `.pck`, publicado en Steam

## Ciclo de turno: tres fases

Cada turno sigue un ciclo de tres fases:

### 1. Briefing

El servidor calcula el nuevo estado del juego y lo comunica a cada jugador **con scope individual** — cada cliente solo recibe la información que le corresponde conocer.

- Distribución de piezas y condiciones
- Cambios en el mapa
- Validación de coherencia con el turno anterior

Objetivos: imposibilitar el acceso a información ajena (seguridad) y reducir el payload.

### 2. Orders

Cada jugador, desde su cliente, asigna órdenes a sus unidades. Esta fase es **enteramente local** — el backend no participa.

- El jugador puede hacer undo/reset libremente
- No hay comunicación con el servidor hasta confirmar
- Al confirmar, el cliente envía un único mensaje con todas sus órdenes

### 3. Resolution

El servidor recibe las órdenes de ambos jugadores y ejecuta la lógica de juego en la secuencia que determina su lógica de negocio. Va comunicando cada resolución como eventos secuenciales a los clientes, para que presenten los efectos del turno.

Al finalizar, el ciclo regresa a **Briefing**, donde se verifica coherencia y comienza un nuevo turno.

```
Briefing → Orders → Resolution → Briefing → ...
```

## Seguridad

- El cliente nunca recibe información fuera de su scope
- Las órdenes se validan server-side contra el estado real
- No hay estado de juego "hackeable" en el cliente porque la información sensible nunca llega a él
- El Briefing con scope individual es el mecanismo central de fog-of-war

## Estructura del proyecto

```
gdt-syv/
├── shared/              # lógica de juego compartida (reglas, hex grid, unidades, modelos)
├── client/              # escenas de UI, rendering, input del jugador
├── server/              # escena headless, validación, autoridad, resolución de turnos
├── protocol/            # definición de mensajes/RPCs entre client y server
├── assets/
│   ├── art/
│   └── audio/
├── addons/
└── docs/
```

- `shared/` es la clave: contiene todo lo que ambos lados necesitan. El servidor la usa para computar. El cliente la usa para presentar.
- `protocol/` define el contrato entre ambos — qué mensajes se envían y qué contienen.

## Transporte

- **Prototipo**: ENet sobre localhost (nativo de Godot, sin configuración)
- **Producción**: Steam Networking Sockets (relay gratuito de Valve, NAT resuelto, IP oculta, protección DDoS)

La nomenclatura de interfaces, señales y clases de red debe ser agnóstica al transporte, anticipando la migración a Steam Networking Sockets sin renombramientos.

## Fases del proyecto

### Fase actual: Prototipo local
- Servidor y clientes corren en la misma máquina
- ENet localhost como transporte
- Sin Steam SDK, sin cloud, sin contenedores
- La arquitectura client/server se respeta desde el día uno

### Futuro: Producción
- Steam como plataforma de distribución
- Steam Networking Sockets como transporte
- Hosting del servidor por definir (AWS u alternativa económica)
- Matchmaking y autenticación por definir
- Persistencia de partidas por definir
