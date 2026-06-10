# ADR-002: Convenciones Godot

> Este ADR es un documento vivo. Se actualiza conforme se definen nuevas convenciones.

## Estado

Aceptado — 2026-04-08

## Contexto

Godot ofrece multiples formas de resolver cada problema: distintos tipos de nodos, patrones de señales, formas de organizar escenas, lenguajes de scripting. Sin convenciones explicitas, el codigo tiende a la inconsistencia conforme crece el proyecto. Es preferible formalizar las decisiones de motor desde el inicio, aunque el documento empiece con lo minimo.

## Decision

### Motor y renderer

- **Godot Engine**: 4.4.1 stable
- **Renderer**: GL Compatibility
- **Lenguaje**: GDScript exclusivamente

### Escenas de entrada

- **Cliente**: `client/main/main.tscn`
- **Servidor**: `server/server_main/server_main.tscn`
- `run/main_scene` apunta a la escena del cliente por defecto

### Separacion de autoridad

- La logica que solo debe correr en el servidor se bifurca con `multiplayer.is_server()`
- Las funciones invocables remotamente se marcan con `@rpc`
- La autoridad por nodo se establece con `set_multiplayer_authority()`

### Convenciones pendientes de definir

Las siguientes areas se formalizaran conforme se construya el proyecto:

- Estructura y nomenclatura de nodos
- Patrones de señales
- Organizacion de escenas dentro de cada carpeta
- Convenciones de nombrado para scripts
- Uso de Resources y su organizacion
- Patrones de autoload

## Consecuencias

**Positivas:**
- Decisiones de motor explicitas y consultables
- Consistencia desde el primer script
- El documento crece organicamente con el proyecto

**Negativas:**
- Requiere actualizacion disciplinada cada vez que se establece un nuevo patron

## Alternativas descartadas

- **No formalizar convenciones**: en Godot, la flexibilidad del motor invita a la inconsistencia. Documentar las decisiones evita que cada parte del proyecto adopte patrones distintos.
- **Definir todas las convenciones de antemano**: imposible sin codigo real. Mejor empezar con lo conocido y crecer con el proyecto.
