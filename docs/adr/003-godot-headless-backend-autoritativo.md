# ADR-003: Godot headless como backend autoritativo

## Estado

Aceptado — 2026-04-08

## Contexto

SyV es un juego multijugador de estrategia donde la informacion es un recurso competitivo. El cliente no debe tener autoridad sobre el estado del juego: si la logica corriera en el cliente, seria posible hacer trampa manipulando la memoria, interceptando paquetes o modificando el ejecutable.

Se necesita un servidor autoritativo — un arbitro central que ejecute toda la logica del juego y comunique a cada cliente unicamente lo que le corresponde. La pregunta es: que tecnologia ejecuta esa logica.

## Decision

El backend corre **Godot 4.4.1 en modo `--headless`**, con la misma base de codigo GDScript que el cliente. La unica diferencia es la escena de entrada: el servidor carga una escena sin rendering que solo ejecuta logica de juego.

### Fundamentos

- **Una sola base de codigo.** La grilla hexagonal, las unidades, las reglas y las facciones existen una unica vez en GDScript. No hay dos implementaciones que puedan divergir.
- **Infraestructura nativa.** Godot 4.x provee `MultiplayerAPI`, decoradores `@rpc`, sistema de autoridad por nodo y bifurcacion con `multiplayer.is_server()`. No se necesita construir un framework de comunicacion desde cero.
- **Computo proporcionado.** Un juego por turnos no necesita ticks continuos ni simulacion en tiempo real. El servidor trabaja en momentos discretos: calcula durante Briefing y Resolution, y permanece inactivo durante Orders.
- **Coherencia con la premisa del proyecto.** SyV esta "integramente programado en Godot". Un backend en otro lenguaje contradice esa premisa.

## Consecuencias

**Positivas:**
- Una sola implementacion de las reglas del juego, sin riesgo de divergencia
- Recursos de Godot (Resources, Scenes, Nodes) compartidos entre client y server
- El equipo solo necesita dominar un lenguaje y un motor
- Prototipado rapido: el servidor puede correr en la misma maquina que el cliente

**Negativas:**
- Godot no es un servidor web: no tiene HTTP routing, middleware ni autenticacion nativos
- La imagen del contenedor es mas pesada que un servidor minimo (Godot headless pesa ~40 MB)
- Debugging del servidor requiere familiaridad con las herramientas de Godot, no con herramientas estandar de backend

## Alternativas descartadas

- **Backend en otro lenguaje (Python, Rust, Go)**: obliga a mantener dos implementaciones de las reglas del juego. Cada cambio en una mecanica se escribe dos veces, en dos lenguajes. El riesgo de divergencia es alto y el costo de mantenimiento se duplica.
- **Hibrido con proxy (FastAPI + Godot headless)**: un servicio ligero para autenticacion y matchmaking que delega a Godot para la logica. Arquitectura valida pero prematura — introduce complejidad de infraestructura que no se justifica en la fase actual del proyecto.
