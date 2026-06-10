# ADR-008: Estructura del proyecto

## Estado

Aceptado — 2026-04-08

## Contexto

La arquitectura de SyV separa responsabilidades entre un servidor autoritativo y clientes de presentacion (ver ADR-003). Existe logica compartida — reglas de juego, grilla hexagonal, modelos de datos — que ambos lados necesitan, y un contrato de comunicacion (mensajes y RPCs) que debe estar definido en un solo lugar. La estructura de directorios debe reflejar esta division y facilitar la extraccion futura del servidor a un proceso independiente.

## Decision

El proyecto se organiza en cuatro carpetas principales dentro de la raiz del proyecto Godot:

```
gdt-syv/
├── shared/          # logica de juego compartida
├── client/          # interfaz y presentacion
├── server/          # arbitro y resolucion
├── protocol/        # contrato de comunicacion
├── assets/          # recursos graficos y sonoros
│   ├── art/
│   └── audio/
├── addons/          # plugins de Godot
└── docs/            # documentacion (incluye ADRs)
```

### shared/

Logica de juego compartida: reglas, grilla hexagonal, unidades, modelos de datos, facciones. Todo lo que ambos lados necesitan para entender el estado del juego. El servidor la usa para computar. El cliente la usa para presentar.

### client/

Escenas de interfaz, rendering, input del jugador. Todo lo que solo tiene sentido con una pantalla delante. Escena de entrada: `client/main/main.tscn`.

### server/

Escena headless, validacion de ordenes, calculo de scope, resolucion de turnos. Todo lo que solo tiene sentido en el arbitro. Escena de entrada: `server/server_main/server_main.tscn`.

### protocol/

Definicion del contrato entre client y server: que mensajes se envian, que contienen, que RPCs existen. Un solo lugar donde vive la especificacion de la comunicacion. Cambiar un mensaje se hace aqui y se refleja en ambos lados.

## Consecuencias

**Positivas:**
- Un modulo en `shared/` lo usan ambos lados sin duplicar codigo
- El contrato vive en `protocol/` — cualquier cambio en la comunicacion se hace en un solo lugar
- La separacion por rol (client/server/shared) facilita extraer `server/` a un build independiente en el futuro
- Claridad inmediata: cualquier persona puede entender que es de cliente, que de servidor y que es compartido

**Negativas:**
- Requiere disciplina para no poner logica compartida en `client/` o `server/` por conveniencia
- Algunos modulos pueden ser ambiguos sobre donde pertenecen — el criterio es: si ambos lados lo necesitan, va en `shared/`

## Alternativas descartadas

- **Estructura flat sin separacion**: mezcla responsabilidades. A medida que el proyecto crece, se vuelve imposible distinguir que codigo es de cliente, que de servidor y que es compartido.
- **Separacion por feature** (ej: `hexgrid/` con sus subcarpetas `client/`, `server/`, `shared/` dentro): dificulta la extraccion del servidor como unidad independiente, ya que la logica de server esta dispersa por todo el arbol de directorios.
