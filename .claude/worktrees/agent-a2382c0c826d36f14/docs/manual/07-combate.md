---
title: "Combate"
order: 7
section: combat
lang: es
---

# Combate

## Resolución de fuerzas

Combate probabilístico resuelto **por soldado individual (L1)**.

La resolución se realiza **a nivel de soldado individual**, y cada enfrentamiento es **estocástico**: el resultado no es determinado únicamente por la comparación directa de fuerzas, sino por una probabilidad modificada por la diferencia de potencia de combate.

**Modelo:**
- Cada soldado en combate (L1) realiza una tirada independiente
- Se calcula la potencia de combate de cada soldado basada en cuatro factores (ver más abajo)
- Se calcula la diferencia neta entre atacante y defensor
- Esta diferencia **modifica una probabilidad base** de que el defensor sea eliminado, que el atacante sea eliminado, o que ambos se mantengan (bounce)
- La resolución favorece al lado más fuerte, pero **no garantiza su victoria** — el resultado es siempre probabilístico

**Resultado:**
- El lado más fuerte tiene una probabilidad *mayor* de ganar, proporcional a su ventaja
- El lado más débil conserva una posibilidad *menor* pero real de infligir bajas o resistir
- Los empates de fuerza producen probabilidades equilibradas

La orden DEFEND otorga un **modificador positivo** a la probabilidad de defensa del defensor.

## Potencia de combate del soldado individual

La capacidad de un soldado individual para infligir daño o resistirlo depende de **cuatro factores**:

1. **Stats propios:** Estadísticas heredadas de la plantilla de clase + modificadores (Perks, Traits, Phobias). Nombres específicos y valores base: **Pendiente de definir**.

2. **Equipamiento:** Arma, accesorios, protección. El tipo y calidad del equipamiento afecta directamente la probabilidad. Un soldado con fusil de asalto tiene más potencia que uno con pistola.

3. **Estado de la escuadra:** Si la escuadra está en estado ROUTED o RETREAT, los soldados sufren penalizadores al combate (o pueden ser impedidos de combatir según el estado). Si está ACTIVE, combaten a potencia plena.

4. **Contexto táctico:** Factores del campo de batalla que modifican la probabilidad por soldado:
   - Elevación (terreno elevado favorece)
   - Terreno (cobertura, dificultad)
   - Flanqueo (atacante sin cobertura lateral)
   - Defensa de posición (defender otorga modificador positivo)
   - Otros modificadores tácticos (por definir)

**Fórmula de resolución:** La combinación exacta de estos cuatro factores y cómo mapean a una probabilidad final de éxito/fallo: **Pendiente de definir**.

## Modificadores

<!-- Pendiente: si hay modificadores adicionales no cubiertos en "Potencia de combate del soldado individual"
     - Otros contextos tácticos?
     - Ímpetu?
     - Orden DEFEND? (ya mencionada arriba)
     - Específicamente: fórmula exacta que mapea los cuatro factores → probabilidad -->

## Encuentros durante el movimiento

Cuando una unidad en movimiento detecta un enemigo, su comportamiento depende de la **orden que la rige**:

- **Orden MOVE (repositionamiento):** La unidad **se detiene en su hex actual** cuando detecta al enemigo. Cesa su desplazamiento y no avanza más. Puede disparar defensivamente si el enemigo está dentro de su línea de vista, pero la intención tácticaoriginal era moverse, no atacar.

- **Orden ATTACK (ofensiva):** La unidad **continúa avanzando** hacia el objetivo aunque encuentre resistencia. Avanza y se compromete a combatir. Si hay enemigos en el camino, los enfrenta como parte de la orden.

**Distancia de detección y mecanismos de reacción:** Pendiente de definir.

Esta distinción es tácticamente importante: MOVE es cautela (evita ambuscadas accidentales), ATTACK es compromiso (empuja hacia el objetivo).

## El combate dentro de la línea de tiempo

El combate no ocurre en un instante ni en un bloque separado. Cada enfrentamiento es un **evento con marca de tiempo** dentro de la línea de tiempo de cuatro horas que genera el servidor durante la Resolution.

Cuando una unidad alcanza su momento de acción en la línea de tiempo, el servidor calcula sus disparos, aplica la resolución probabilística por soldado, y emite los eventos resultantes con sus timestamps exactos. Si una unidad enemiga fue eliminada previamente en la misma línea de tiempo, ya no está disponible como objetivo.

Este modelo tiene consecuencias concretas: una unidad con alta iniciativa puede abatir a un enemigo **antes** de que ese enemigo llegue a disparar en el mismo turno. La diferencia de iniciativa puede ser decisiva.

## Iniciativa

La iniciativa determina **cuándo dentro de la ventana de cuatro horas actúa cada unidad**. No es un desempate de último recurso — es el mecanismo central que ordena la línea de tiempo.

- Mayor iniciativa = acción más temprana en la línea de tiempo
- Menor iniciativa = acción más tardía
- Unidades con baja iniciativa pueden enfrentar un campo ya modificado por las acciones previas

Los factores que componen la iniciativa de una unidad (stats, estado, contexto táctico, órdenes recibidas) y la fórmula exacta de cálculo: **Pendiente de definir**.

## Designación de objetivos por soldado

El fuego puede designarse **por soldado individual (L1)**. Cada soldado de una escuadra puede apuntar a un enemigo diferente, que puede estar en un hex distinto.

Esto significa que una orden de ATTACK para una escuadra puede especificar múltiples hexes objetivo — uno por soldado, o una distribución asignada por el oficial de escuadra según alguna lógica de distribución de fuego. La mecánica exacta de la interfaz de objetivos y la lógica de distribución automática: **Pendiente de definir**.

## Fuego indirecto (morteros y artillería)

A diferencia de la infantería que requiere **línea de vista** para disparar, los morteros y artillería son **puramente indirectos**:

- **No requieren línea de vista:** El mortero o pieza de artillería dispara a **coordenadas de hexes**, no a unidades visibles.
- **Fuego ciego:** La unidad apunta al hex objetivo por sus coordenadas. No necesita ver al enemigo que se encuentra en ese hex.
- **Impacto único:** El proyectil afecta **solo al hex objetivo**. Todas las unidades en ese hex se ven afectadas; las unidades en hexes adyacentes **no reciben daño**.
- **Sin área de efecto:** No hay concepto de "radio de explosión" o "splash". El daño se limita estrictamente al hex.
- **Observador y precisión:** Si la precisión de fuego indirecto mejora con un observador o spotter: **Pendiente de definir**.

Esta mecánica hace que el mortero sea fundamentalmente diferente de la infantería directa: es una herramienta de apoyo remoto, no una unidad de combate táctico cercano.

## Sin cuerpo a cuerpo

No existe un sistema de combate cuerpo a cuerpo separado. **Todo el combate se resuelve como fuego**, independientemente de la distancia. Un enfrentamiento a 0-1 hexes de distancia sigue siendo fuego — no hay fase ni mecánica diferenciada para el combate en contacto directo.

## Sin logística

Las unidades siempre tienen capacidad de disparar y actuar. No existe rastreo de munición ni sistema de suministros. Una unidad nunca queda sin la posibilidad de combatir por agotamiento de recursos materiales — su degradación ocurre exclusivamente a través de bajas y heridas.

## Secuencia de resolución

<!-- Pendiente: qué se resuelve primero
     1. Movimiento (todos simultáneamente?)
     2. Combate (resolución probabilística por soldado/grupo)
     3. Post-procesado (estados, Valor, disband, regla 5 hexes) -->

## Heridas y degradación del soldado

El sistema de heridas replica la fragilidad humana. A diferencia de otros juegos, **no existe una progresión fija de estados de herida** ("leve", "grave", etc.). En cambio, cada herida es un **modificador negativo acumulable** que afecta los stats del soldado, similar al sistema de Perks, Traits y Phobias.

### Heridas como modificadores

Cuando un soldado recibe daño sin ser eliminado, adquiere una o más heridas. Cada herida es:
- Un modificador negativo que se suma a su lista de modificadores activos
- Permanente durante toda la partida (las heridas no se curan)
- Acumulable — un soldado puede llevar múltiples heridas, cada una con su propia penalización

Por ejemplo, un soldado podría llevar "Herida en pierna (movimiento -15%)" y "Herida en brazo (ataque -10%)" simultáneamente. Sus stats base se actualizan restando ambos modificadores en cada cálculo de combate.

### Umbral de eliminación

Un soldado es eliminado (muere o es sacado de combate) cuando la **acumulación total de penalizadores de heridas cruza un umbral específico**. Este umbral: **Pendiente de definir**.

Esta aproximación es elegante porque:
1. Reutiliza completamente el sistema abierto de modificadores ya existente
2. Permite granularidad — heridas de diferente severidad tienen penalizadores distintos
3. Modela la erosión gradual del soldado sin estados discretos artificiales
4. Facilita la interacción con el sistema de Perks/Traits/Phobias — todos compiten por los mismos slots de modificadores

### El rol del Sanitario (Medic)

Un Sanitario (medic/corpsman) puede **estabilizar** un soldado herido para evitar que empeore, pero **nunca cura** heridas ni remueve modificadores.

**Capacidades del Sanitario:**
- Prevenir que un soldado herido desarrolle complicaciones adicionales (infección, shock, etc.) en turnos posteriores
- Mantener vivo a un soldado que de otro modo sería eliminado por umbral de heridas
- No restaurar stats perdidos ni remover heridas existentes

**Efecto en juego:**
El daño persiste. Un soldado estabilizado sigue llevando sus heridas y sus penalizadores, pero no se degradará más. El sanitario no es un sanador — es un preservador. Esta mecánica refuerza el tema central de **atrito y erosión lenta**: la victoria viene de presión sostenida, no de recuperación rápida de bajas.

Nombres y efectos concretos de las heridas: **Pendiente de definir**.

## El desgaste

La mayoría de los enfrentamientos no producen resultado. Las fuerzas chocan, rebotan, y vuelven a chocar. La victoria se construye acumulando pequeñas ventajas posicionales a lo largo de muchos turnos — presión sostenida, como olas erosionando un acantilado. Este desgaste se manifiesta también en el nivel individual: cada soldado que sobrevive a múltiples combates lleva heridas acumuladas que lo degradan gradualmente hasta su eliminación final.
