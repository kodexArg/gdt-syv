# ADR-000: Plantilla

Este archivo no es un registro de decisiones arquitectonicas. Es la plantilla que define
el formato y las convenciones de todos los ADRs del proyecto Subordinacion y Valor.

## Convenciones generales

- **Idioma**: todos los ADRs se escriben en castellano.
- **Nomenclatura de archivos**: `NNN-nombre-en-kebab-case.md`, donde `NNN` es un numero
  secuencial de tres digitos comenzando en 001.
- **Un ADR por decision**: cada archivo captura una unica decision arquitectonica.
  Si una decision reemplaza a otra, el nuevo ADR referencia al anterior.
- **Inmutabilidad**: un ADR aceptado no se modifica en su esencia. Si la decision cambia,
  se escribe un nuevo ADR y el anterior pasa a estado "Reemplazado" con referencia al nuevo.
- **Excepcion**: los ADRs 001 (nomenclatura) y 002 (convenciones Godot) son documentos
  vivos que crecen con el proyecto.

## Formato

Todo ADR sigue esta estructura:

---

```markdown
# ADR-NNN: Titulo

## Estado

Aceptado | Reemplazado | Obsoleto — AAAA-MM-DD

Si fue reemplazado, indicar: "Reemplazado por ADR-NNN — AAAA-MM-DD"

## Contexto

Que problema, necesidad o circunstancia motiva esta decision.
Describir la situacion tal como se presenta, sin anticipar la solucion.

## Decision

Que se decidio, de forma concreta y sin ambiguedad.
Debe ser posible leer solo esta seccion y entender que se hara.

## Consecuencias

Que implica esta decision, tanto lo positivo como lo negativo.
Incluir impacto en el desarrollo, la arquitectura y las restricciones que introduce.

## Alternativas descartadas

Que otras opciones se evaluaron y por que no se eligieron.
Cada alternativa con una breve justificacion de su descarte.
```

---

## Estados posibles

| Estado | Significado |
|--------|------------|
| Aceptado | La decision esta vigente y se aplica |
| Reemplazado | Otra decision posterior la sustituye (indicar cual) |
| Obsoleto | Ya no aplica por cambios en el contexto del proyecto |
