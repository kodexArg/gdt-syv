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
| **Seccion** | Fuerza completa de un jugador. Nivel organizacional maximo en el juego. Contiene pelotones. |
| **Peloton** | Agrupamiento intermedio de escuadras bajo un mando L3 (Sargento). Nivel entre Seccion y Escuadra. |
| **Escuadra** | Unidad organizacional leaf. Ocupa un unico hex. Es la entidad que computa el combate. |
| **L1–L5** | Niveles jerarquicos de la fuerza militar. L1 = tropa basica (soldado). L5 = mando de peloton (Teniente). L3, L4 y L5 portan radio de 5 hex. L2 y L1 no portan radio. Ver ADR-001-01. |
| **Tropa** | Entidad individual de combate. Nivel L1 a L5, con faction, compatible_squads y equipment. La unidad atomica de la fuerza. |
| **Grupo** | Agrupacion interna de 2 a 5 Tropas dentro de una Escuadra. Liderada por un L2. No es una entidad de combate; es una unidad de organizacion. |

| **E-UHF** | Variante encriptada de UHF usada por ambas facciones para comunicacion tactica. Alcance 500 m (5 hex). Solo L3 a L5 la portan. Ver manual de Comunicaciones. |
| **Fin de los Secretos** | Evento historico del universo SyV que volvio irreversible la transparencia de las comunicaciones. Razon por la que L1/L2 no portan radio personal. |
| **Confederación Argentina** | Facción norte. Ejército regular del estado teocrático-militar. Estructura jerárquica formal heredada del Ejército Argentino con presencia eclesiástica integrada en la cadena de mando. Nombre corto de juego: "La Confederación". ID en código y YAML: `confederacion`. |
| **Los Rojos** | Facción sur. Movimiento de resistencia de raíz comunista-patriótica con estructura más horizontal. Incluye el cuerpo especial "Los Infernales" — caballería irregular con nomenclatura gaucha. ID en código y YAML: `rojos`. |
| **Invariante de bandos** | Azul = izquierda (q mínimo) / Rojo = derecha (q máximo). HQ Azul fijo en `(-20, 10)`; HQ Rojo fijo en `(20, -10)`. Eje de simetría = columna q=0. Color y lado son posicionales, no elegibles por el jugador. Ver ADR-024. |

Los grados militares especificos de cada faccion (rangos de Confederacion y Los Rojos) se documentan en ADR-001-01.

> Las facciones actuales son Confederación Argentina y Los Rojos. El sistema de niveles L1–L5 es agnóstico respecto a la facción — cualquier nueva facción puede incorporarse definiendo sus propios nombres de rango sobre los mismos niveles. Ver `docs/premade-squads/_template.yaml` para el esquema de escuadras.

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
