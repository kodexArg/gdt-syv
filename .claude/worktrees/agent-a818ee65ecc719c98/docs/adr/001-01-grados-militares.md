# ADR-001-01: Grados militares por faccion

> Documento vivo. Referenciado por ADR-001.

## Estado

Aceptado — 2026-04-11

**Direccion canonica del sistema L1-L5: bottom-up. L1 = tropa basica (Soldado); L5 = mando maximo de peloton (Teniente).**
Esta direccion es la estandar en diseno de juegos (nivel 1 = basico, nivel 5 = avanzado) y produce condiciones de codigo legibles:
`if level >= 3: has_radio = true`. Todos los documentos y YAMLs del proyecto usan esta convencion.

## Contexto

Las dos facciones tienen jerarquias militares propias con identidades distintas:

- **Confederacion**: estructura militar formal e institucional, herencia del Ejercito Argentino.
- **Los Rojos**: organizacion de tinte comunista y patriotico, herencia de las Fuerzas Armadas
  Revolucionarias (FAR, modelo cubano/sovietico). Excepcion: "Los Infernales" (caballeria)
  conservan nomenclatura de milicia irregular gaucha.

Los grados se mapean sobre un sistema de cinco niveles (L1 a L5). L1 es la tropa basica;
L5 es el mando maximo de peloton. Los niveles determinan autoridad, radio y autonomia.
Ambas facciones comparten el mismo sistema de niveles; solo difieren los nombres de grado.

## Sistema de niveles (comun a ambas facciones)

| Nivel | Descripcion | Radio | Autonomia |
|-------|-------------|-------|-----------|
| L5 | Mando de Peloton | Si | Alta — puede recibir ordenes genericas del HQ |
| L4 | Subjefe de Peloton | Si | Alta |
| L3 | Jefe de Escuadra | Si (5 hex) | Media — lider operativo de la escuadra |
| L2 | Sub-lider de Grupo / Lider de escuadra especial | No | Limitada — ver nota |
| L1 | Tropa basica | No | Ninguna — se desbanda sin mando superior |

**Nota L2**: El nivel L2 tiene dos usos:
1. Sub-lider de un Grupo dentro de una escuadra estandar (Cabo, Cabo Primero).
   No porta radio pero organiza internamente 2-5 tropas.
2. Lider de una escuadra de tipo especial (Los Infernales, commandos, zapadores).
   La escuadra completa opera sin radio; recibe ordenes genericas y actua con autonomia.
   Puede activar reglas especiales propias del tipo de escuadra.

**Tropa de Comunicaciones (L1 especial)**: Porta radio de alcance global. Por si sola no
extiende la cadena de mando. Cuando esta asignada a una escuadra con oficial L3/L4/L5,
amplifica el radio de ese oficial (el alcance exacto esta pendiente de definicion).
**Patron de uso**: Es la unica forma correcta de agregar capacidad de radio a una escuadra
sin elevar su nivel de mando. Un HQ o escuadra especial que necesite comunicaciones debe
incluir una Tropa de Comunicaciones L1 — nunca asignar radio a un miembro L2.

> **Nota**: Los nombres de grado específicos de cada facción (tablas a continuación) son **provisorios** — no están cerrados. Los niveles L1–L5 y sus reglas de radio/autonomía sí son canónicos y permanentes. Los grados de cada facción pueden cambiar sin afectar la lógica del juego.

## Confederacion

### Jerarquia lineal

| Nivel | Grado |
|-------|-------|
| L5 | Teniente |
| L4 | Sargento Primero |
| L3 | Sargento |
| L2 | Cabo Primero / Cabo |
| L1 | Soldado |

### Variantes de grado (mismo nivel L, tipo especial)

| Nivel | Grado | Tipo |
|-------|-------|------|
| L5 | Inquisidor | Linea eclesiastica — mando espiritual y militar |
| L2 | Lider de escuadra especial | Commandos, zapadores (grado segun tipo) |
| L1 | Artillero | Especialista de fuego (mortero, MG) |
| L1 | Capellan | Sanidad y moral (agregado eclesiastico, no combate) |
| L1 | Medico de Combate | Sanidad de campo |
| L1 | Comunicaciones | Radio global; amplifica rango del oficial |

## Los Rojos

### Jerarquia lineal

| Nivel | Grado |
|-------|-------|
| L5 | Teniente |
| L4 | Sargento de Primera |
| L3 | Sargento de Segunda |
| L2 | Sargento de Tercera / Cabo |
| L1 | Soldado |

### Variantes de grado (mismo nivel L, tipo especial)

| Nivel | Grado | Tipo |
|-------|-------|------|
| L5 | Lider Revolucionario | Linea politica — comisario con mando efectivo |
| L2 | Jefe de Partida | Los Infernales (escuadra especial L2, sin radio) |
| L2 | Baqueano | Los Infernales (sub-lider, conocimiento del terreno) |
| L1 | Jinete | Los Infernales (tropa montada) |
| L1 | Comunicaciones | Radio global; amplifica rango del oficial |

## Nota sobre Los Infernales

Los Infernales son una escuadra de tipo especial L2 dentro de Los Rojos. Su nomenclatura
es de origen gaucho-irregular, a diferencia del resto de Los Rojos que sigue el modelo
comunista/patriotico. Sus miembros no portan radio. Operan con ordenes genericas y reglas
especiales de caballeria. Son la excepcion, no la regla, dentro de Los Rojos.
