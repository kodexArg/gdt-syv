---
title: "Valor"
order: 9
section: valor
lang: es
---

# Valor

## Qué es el Valor

El Valor **no es una tirada de salvación ni un recurso que se agota**. Es un **modificador pasivo** que representa la moral y la disciplina de un soldado. Un Valor más bajo degrada el desempeño en todas las acciones: combate, órdenes, resistencia. La degradación moral se manifiesta como que todo se vuelve más difícil, no como un mecanismo separado.

## Nivel de Valor: Por Soldado (L1)

El Valor es **individual**:
- Cada soldado tiene su propio stat de Valor
- Coherente con el modelo de stats individuales (plantilla de clase + perks/traits/phobias + wounds)
- El Valor de un soldado puede degradarse por:
  - **Heridas sufridas** — daño físico que afecta la moral
  - **Pérdida de compañeros** — ver a miembros de la escuadra eliminados
  - **Pérdida de contacto** — estar fuera del radio de mando (regla de los 5 hexes)
  - **Presencia enemiga** — flanqueado, rodeado, bajo fuego
  - **Traits/phobias negativas** — características específicas que degradan Valor
  
*(Triggers exactos y cantidades de degradación: Pendiente de definir)*

## Nivel de Valor: Por Escuadra (L3)

El Valor de la escuadra es un **promedio ponderado** de sus miembros:
- Se calcula agregando el Valor de todos los soldados en la escuadra
- Fórmula de ponderación: Pendiente de definir (puede que el líder cuente más, o promedio simple)
- Este valor derivado **determina los efectos a nivel escuadra** (transiciones de estado ROUTED, RETREAT)

## Cómo funciona: Valor como Modificador

1. El Valor **modifica otros tiradas y rolls**, no dispara rolls propios:
   - Modificador en tiradas de combate
   - Modificador en tiradas de órdenes
   - Modificador en resistencia/supervivencia

2. El Valor **no se gasta ni se recupera por roll**:
   - Solo se degrada por los triggers mencionados arriba
   - Solo se recupera por descanso, refuerzo de moral, ausencia de amenaza
   - *(Mecánica de recuperación: Pendiente de definir)*

## Transiciones de Estado por Valor de Escuadra

El estado de la escuadra transiciona de forma **progresiva** según el Valor ponderado:

### Estados (orden de degradación)

1. **ACTIVE** — Normal operation. Escuadra operativa, ejecuta órdenes sin restricción. Acceso a todas las capacidades (MOVE, DEFEND, ATTACK).

2. **RETREAT** — Primer umbral. La escuadra intenta retirarse controladamente hacia la retaguardia. Recibe órdenes **limitadas**:
   - ✓ MOVE (solo hacia retaguardia)
   - ✓ DEFEND
   - ✗ ATTACK prohibido
   - Deterioro controlado del desempeño; mantiene algo de cohesión.

3. **ROUTED** — Segundo umbral (más bajo). Pérdida total de cohesión. La escuadra se retira **automáticamente** hacia la retaguardia, ignora todas las órdenes, no puede combatir. Completamente demoralizada.

4. **ELIMINATED** — Todos los miembros muertos o incapacitados permanentemente. Removida del juego.

### Progresión y recuperación

**Degradación:** ACTIVE → RETREAT → ROUTED (según cae el Valor)

**Recuperación:** ROUTED → RETREAT → ACTIVE (según sube el Valor)

- Las transiciones son **reversibles**: una escuadra puede recuperarse gradualmente si **pasa tiempo sin estar amenazada**.
- Recuperación de ROUTED a RETREAT requiere que la escuadra esté:
  - No bajo fuego
  - Sin sufrir bajas
  - Posiblemente dentro del radio de mando (mejora pero tal vez no sea obligatorio)
- El Valor aumenta lentamente durante estos turnos seguros, hasta superar el umbral de ROUTED, luego el de RETREAT.
- **Velocidad de recuperación y condiciones exactas**: Pendiente de definir

### Umbrales de Valor

- **Umbral RETREAT**: Por definir (primer punto donde cae el Valor)
- **Umbral ROUTED**: Por definir (segundo, más bajo)

## Resumen de Decisiones Arquitectónicas

| Aspecto | Decisión |
|---------|----------|
| **Tipo** | Modificador pasivo, no tirada ni recurso |
| **Scope** | Por soldado (L1), agregado a escuadra (L3) |
| **Función** | Degrada performance en todas las acciones |
| **Triggers** | Heridas, pérdidas, aislamiento, amenaza, traits |
| **Efecto escuadra** | Transiciones ROUTED/RETREAT por umbral |
| **Reversibilidad** | Sí, por recuperación de moral |
