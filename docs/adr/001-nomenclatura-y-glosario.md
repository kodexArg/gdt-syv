# ADR-001: Nomenclatura y glosario

> Este ADR es un documento vivo. Se actualiza conforme se agregan conceptos al proyecto.

## Estado

Aceptado — 2026-04-08

## Contexto

Un proyecto de juego involucra terminologia de multiples dominios: mecanicas de juego, arquitectura de software, networking, motor grafico. Sin un glosario compartido, cada conversacion, documento o linea de codigo puede usar terminos distintos para referirse a lo mismo, generando ambiguedad y errores.

## Decision

Se adopta la siguiente terminologia como vocabulario canonico del proyecto. Todo documento, comentario en codigo, nombre de clase, señal o mensaje debe usar estos terminos con el significado aqui definido.

### Glosario

| Termino | Significado |
|---------|------------|
| **SyV** | Subordinacion y Valor — nombre del juego |
| **Briefing** | Primera fase del turno. El servidor calcula el estado y envia a cada jugador unicamente su scope individual |
| **Orders** | Segunda fase del turno. Cada jugador asigna ordenes a sus unidades de forma local, sin participacion del servidor |
| **Resolution** | Tercera fase del turno. El servidor ejecuta la logica de negocio y emite eventos secuenciales a los clientes |
| **Scope** | La porcion del estado del juego que un jugador tiene derecho a conocer. Lo que no esta en su scope, no existe para el |
| **Autoridad** | Quien posee la verdad sobre un dato. En SyV, la autoridad es siempre del servidor |
| **shared** | Division del codigo que contiene logica de juego compartida: reglas, grilla hexagonal, unidades, modelos de datos |
| **client** | Division del codigo para interfaz, rendering e input del jugador |
| **server** | Division del codigo para la escena headless, validacion de ordenes y resolucion de turnos |
| **protocol** | Division del codigo que define los mensajes y RPCs entre client y server |
| **Listen server** | Modelo de prototipado donde una unica instancia de Godot actua como servidor y cliente simultaneamente |

## Consecuencias

**Positivas:**
- Comunicacion univoca entre documentos, codigo y conversaciones
- Los nombres en el codigo reflejan directamente los conceptos del juego
- Cualquier persona nueva en el proyecto puede consultar este glosario como referencia

**Negativas:**
- Requiere disciplina para mantener el glosario actualizado conforme crece el proyecto
- Terminos nuevos deben formalizarse aqui antes de adoptarse en el codigo

## Alternativas descartadas

- **No formalizar la terminologia**: en un proyecto con arquitectura client/server, tres fases de turno y multiples capas de codigo, la ambiguedad terminologica es un riesgo real. El costo de mantener un glosario es minimo comparado con el costo de malentendidos.
