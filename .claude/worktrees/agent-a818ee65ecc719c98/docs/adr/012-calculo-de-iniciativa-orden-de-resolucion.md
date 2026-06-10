# ADR-012: Cálculo de iniciativa y orden de resolución

## Estado

Aceptado — 2026-05-18

## Contexto

ADR-004 establece que la Resolution es una simulación de tiempo continuo: el
servidor recibe las órdenes simultáneas de ambos jugadores y produce una línea
de tiempo de eventos discretos dentro de la ventana de cuatro horas del turno.
El capítulo 05 (El Turno) y el capítulo 07 (Combate) afirman que esa línea de
tiempo **no es aleatoria en su orden**: la *iniciativa* de cada unidad determina
cuándo, dentro de las cuatro horas, comienza a ejecutar su orden. Las unidades
con mayor iniciativa actúan antes y pueden modificar el campo (abatir enemigos,
tomar posiciones) antes de que las unidades lentas lleguen a actuar.

Hasta ahora la fórmula de iniciativa estaba marcada como *Pendiente de definir*
en ambos capítulos. Era el bloqueante #3: sin un cálculo de iniciativa concreto,
el servidor no puede ordenar la resolución simultánea y los capítulos 05 y 07
quedan sin cerrar en su mecánica central.

Esta decisión define **únicamente el ORDEN de resolución** (cuándo arranca cada
unidad su acción). La matemática de daño, puntería y heridas es responsabilidad
del capítulo 07 y de otra decisión; aquí no se toca.

Restricciones que la decisión debe respetar:

- Stats individuales por soldado: plantilla de clase + perks/traits/phobias +
  heridas (cap. 04). El nivel L1–L5 **no** es poder de combate.
- Estado de la escuadra ACTIVE / RETREAT / ROUTED (cap. 04, cap. 09).
- Mando: regla de los 5 hexes encadenada y aislamiento por pérdida de radio
  (cap. 08, cap. 12). Una escuadra `in_command` reacciona coordinada; una
  aislada ejecuta su última orden con reacción degradada.
- Tipos de orden ATTACK / MOVE / DEFEND (cap. 06).
- La escuadra es la unidad leaf de combate y ocupa un solo hex (cap. 04): la
  resolución se ordena a nivel **escuadra**, no de tropa ni de grupo.
- Debe haber azar acotado: dos planes idénticos no deben resolver siempre igual,
  pero el azar no debe dominar sobre stats y contexto.

## Decisión

### 1. Granularidad

La iniciativa se calcula y la línea de tiempo se ordena **por escuadra**. La
escuadra es la entidad que ocupa un hex, recibe una orden y computa combate
(cap. 04); ordenar por tropa o por grupo no tiene sentido físico porque todas
las tropas de una escuadra comparten hex y orden. Los grupos internos (p. ej.
ametralladora apuntador+asistente) afectan la resolución de combate dentro del
evento de esa escuadra, no su posición en la línea de tiempo.

El HQ estratégico es estático y no genera acción de campo: no entra en el
ordenamiento.

### 2. Qué entra en la iniciativa

La iniciativa de una escuadra `S` es un puntaje `INIT(S)`. Mayor `INIT` ⇒
acción más temprana en la ventana de 4 h. Se compone de cinco factores:

| Factor | Símbolo | Qué representa | Fuente |
|--------|---------|----------------|--------|
| Aptitud de la escuadra | `A` | Stats de reacción de las tropas vivas | cap. 04 |
| Bono por tipo de orden | `O` | Una orden de asalto arranca antes que una de reposicionamiento | cap. 06 |
| Estado de mando | `C` | En mando coordinado vs. aislado | cap. 08, 12 |
| Estado moral de la escuadra | `M` | ACTIVE / RETREAT / ROUTED | cap. 04, 09 |
| Azar acotado | `R` | Fricción / "niebla" táctica | diseño |

#### 2.1 Aptitud de la escuadra `A`

Cada plantilla de clase de tropa expone un stat de **Reacción** (`reaccion`),
un stat base de clase más perks/traits/phobias/heridas como cualquier otro
(cap. 04). Nombre canónico del stat sujeto a la nomenclatura de stats todavía
pendiente en cap. 04; aquí se lo asume normalizado a un rango **0–100** por
soldado vivo.

`A` es la **Reacción del líder operativo de la escuadra** (el L3 en escuadra
estándar, el L2 en escuadra especial) **más** una contribución menor del resto:

```
A = reaccion(lider) + 0.25 * media( reaccion(t) para t en tropas_vivas, t != lider )
```

Se pondera fuerte al líder porque la iniciativa modela *capacidad de mando para
hacer reaccionar a la unidad*, no la suma de reflejos individuales. Si el líder
murió y la escuadra sigue operando aislada, `lider` pasa a ser la tropa viva de
mayor nivel; ver §3 (penalización de aislamiento la cubre).

Rango efectivo de `A`: ~0–125.

#### 2.2 Bono por tipo de orden `O`

| Orden | `O` | Racional |
|-------|-----|----------|
| ATTACK | +20 | Intención ofensiva: la unidad empuja, busca el contacto temprano. |
| MOVE | +5 | Reposicionamiento; avanza pero se detiene ante contacto. |
| DEFEND | −10 | No busca acción; "actúa" tarde porque solo reacciona si la alcanzan. |
| Sin orden / orden vieja (aislada) | −15 | Ejecuta comportamiento por defecto, sin impulso de mando reciente. |

DEFEND con `O` negativo es deliberado: una escuadra que defiende no necesita
actuar temprano en la línea de tiempo; su evento se inserta cuando un atacante
la alcanza (el atacante, con `O=+20`, llega antes). El bonus de fortificación
de DEFEND (cap. 06) es ajeno a la iniciativa y no se modela aquí.

#### 2.3 Estado de mando `C`

Calculado a partir de `in_command` (BFS 5 hexes desde HQ, cap. 08) y de
`has_radio` (cap. 12):

| Situación | `C` | Racional |
|-----------|-----|----------|
| `in_command = true` | +15 | Recibe la orden de este turno, reacciona coordinada y a tiempo. |
| Fuera de mando pero con radio (`has_radio`, fuera de BFS) | −5 | Tiene medios pero no recibió orden nueva; actúa con retraso. |
| Aislada (`has_radio = false`, p. ej. L3 muerto, o escuadra especial L2) | −20 | Sin orden nueva ni coordinación; ejecuta su última orden con demora. |

Excepción de diseño para escuadras especiales L2 (Los Infernales, commandos):
operan sin radio por diseño pero con autonomía entrenada, así que **no** sufren
el −20 de aislada; reciben `C = +5` (autonomía limitada pero deliberada).
Sujeto a playtest.

#### 2.4 Estado moral de la escuadra `M`

Derivado del estado operativo (cap. 04, cap. 09):

| Estado | `M` |
|--------|-----|
| ACTIVE | +10 |
| RETREAT | −10 |
| ROUTED | −25 |

Una escuadra ROUTED se mueve automáticamente y descoordinada: actúa tarde y
mal. ELIMINATED no participa.

#### 2.5 Azar acotado `R`

```
R = uniform(-15, +15)   # tirada por escuadra, por turno, semilla del servidor
```

Rango de ±15 elegido para que el azar pueda invertir un duelo entre escuadras
de iniciativa parecida (diferencia < 30) pero **no** entre una escuadra
claramente superior y una inferior. La semilla es del servidor y forma parte
de la verificación de coherencia del Briefing (ADR-004): determinista dado el
estado + órdenes + semilla del turno.

### 3. Fórmula

```
INIT(S) = A + O + C + M + R
```

Todos los términos ya están en la misma escala (puntos de iniciativa). No se
normaliza a 0–100: lo único que importa es el **orden relativo** de los `INIT`
de todas las escuadras de ambos bandos en el turno.

### 4. De `INIT` a marca de tiempo

La ventana del turno es de 4 h (240 min). Sea `INIT_max` e `INIT_min` el mayor
y menor puntaje entre **todas** las escuadras activas del turno. El instante de
arranque `t(S)` dentro de la ventana se interpola linealmente:

```
t(S) = T_inicio + (240 min) * (INIT_max - INIT(S)) / (INIT_max - INIT_min)
```

- La escuadra de mayor `INIT` arranca su acción en `t = 0:00` de la ventana.
- La de menor `INIT` arranca cerca del final.
- El resto se distribuye proporcionalmente.

`t(S)` es solo el **instante de inicio** de la acción de la escuadra; la
duración de moverse/disparar y los eventos subsiguientes los produce la
simulación de combate (cap. 07), fuera del alcance de este ADR. Si todas las
escuadras quedaran con el mismo `INIT` (denominador 0), todas arrancan en
`t = 0:00` y se aplica la regla de empates.

### 5. Empates

Cuando dos o más escuadras tienen `INIT` idéntico (o `t(S)` que colapsa al
mismo tick de la simulación), el orden se decide por esta cascada determinista:

1. Mayor `A` (la unidad intrínsecamente más reactiva actúa primero).
2. Si persiste: orden ATTACK > MOVE > DEFEND > sin orden.
3. Si persiste: mayor `strength` de escuadra (cap. 04).
4. Si persiste: menor `squad_id` (desempate determinista final, estable entre
   re-simulaciones para la verificación de coherencia del Briefing).

No se hace una segunda tirada de azar para empates: `R` ya inyectó la
aleatoriedad y el desempate debe ser reproducible.

### 6. Ejemplo trabajado

Turno con tres escuadras. Stats de Reacción ya normalizados 0–100.

**Escuadra Alfa** — infantería estándar, líder L3 vivo (Reacción 70), 9 tropas
más (media Reacción 55). Orden ATTACK. `in_command = true`. ACTIVE.
Tirada `R = +8`.

```
A = 70 + 0.25 * 55              = 83.75
O (ATTACK)                      = +20
C (in_command)                  = +15
M (ACTIVE)                      = +10
R                               = +8
INIT(Alfa) = 83.75+20+15+10+8   = 136.75
```

**Escuadra Bravo** — infantería estándar, L3 **muerto** (líder pasa a L2,
Reacción 50), 6 tropas más (media 48). Última orden conocida MOVE. Aislada
(`has_radio = false`). ACTIVE. `R = -3`.

```
A = 50 + 0.25 * 48              = 62.0
O (MOVE)                        = +5
C (aislada)                     = -20
M (ACTIVE)                      = +10
R                               = -3
INIT(Bravo) = 62+5-20+10-3      = 54.0
```

**Escuadra Charlie** — Los Infernales (especial L2), líder L2 vivo
(Reacción 80), 4 tropas más (media 60). Orden genérica tipo ATTACK. Sin radio
**por diseño** (excepción §2.3 ⇒ C=+5). ACTIVE. `R = +12`.

```
A = 80 + 0.25 * 60              = 95.0
O (ATTACK)                      = +20
C (especial L2)                 = +5
M (ACTIVE)                      = +10
R                               = +12
INIT(Charlie) = 95+20+5+10+12   = 142.0
```

Ordenamiento por `INIT` descendente: **Charlie (142.0) > Alfa (136.75) >
Bravo (54.0)**.

Interpolación a la ventana de 240 min (`INIT_max=142.0`, `INIT_min=54.0`):

```
t(Charlie) = 0   + 240 * (142.0-142.0)/(142.0-54.0)  = 0:00
t(Alfa)    = 0   + 240 * (142.0-136.75)/(88.0)       ≈ 14.3 min  ≈ 0:14
t(Bravo)   = 0   + 240 * (142.0-54.0)/(88.0)          = 240 min   ≈ 4:00
```

Lectura táctica: Los Infernales (especiales, líder muy reactivo, asalto)
golpean al inicio del turno; Alfa los sigue 14 min después; Bravo, aislada por
la muerte de su Sargento y solo reposicionándose, actúa al final del turno
sobre un campo ya alterado por las dos primeras. Esto es exactamente la
consecuencia táctica descrita en cap. 07: la iniciativa puede ser decisiva.

### 7. Parámetros tuneables (sujetos a playtest)

| Parámetro | Valor inicial | Notas |
|-----------|---------------|-------|
| Peso del no-líder en `A` | 0.25 | Subir si las escuadras decapitadas reaccionan demasiado bien. |
| `O` ATTACK / MOVE / DEFEND / sin-orden | +20 / +5 / −10 / −15 | Brecha ATTACK–DEFEND = 30; controla cuán dominante es la iniciativa ofensiva. |
| `C` in_command / sin-mando-c-radio / aislada | +15 / −5 / −20 | |
| `C` excepción escuadra especial L2 | +5 | Decisión de diseño; podría igualarse a in_command. |
| `M` ACTIVE / RETREAT / ROUTED | +10 / −10 / −25 | |
| Amplitud de `R` | ±15 | Define cuándo el azar puede invertir un duelo (Δ INIT < 30). |
| Ventana del turno | 240 min | Fijada por ADR-004; no es tuneable aquí. |

Todos los valores numéricos son hipótesis de diseño iniciales y deben
calibrarse con playtest. La estructura de la fórmula (`A + O + C + M + R`,
ordenamiento por escuadra, interpolación lineal a la ventana) es la decisión
estable; los coeficientes son ajustables sin reabrir este ADR.

## Consecuencias

**Positivas:**

- Cierra el bloqueante #3: el servidor tiene un algoritmo determinista
  (dado estado + órdenes + semilla) para ordenar la Resolution, compatible con
  la verificación de coherencia del Briefing (ADR-004).
- La iniciativa premia el mando intacto y conectado: perder al L3 o quedar
  fuera de la cadena de 5 hexes no solo afecta órdenes (cap. 06/08) sino que
  también retrasa la reacción de la escuadra — refuerza la tesis del juego.
- El azar acotado evita resultados idénticos en planes idénticos sin permitir
  que la suerte aplaste la diferencia de competencia.
- Ordenar por escuadra mantiene la coherencia con "la escuadra es la unidad
  leaf de combate" y evita estado por-tropa en la línea de tiempo.

**Negativas:**

- Introduce un stat nuevo de clase (Reacción) que el cap. 04 aún no lista
  nominalmente; queda atado a la nomenclatura de stats pendiente. Si esa
  nomenclatura nombra el stat de otra forma, este ADR sigue válido (mapea por
  rol, no por nombre).
- La interpolación lineal `INIT → tiempo` puede agrupar muchas escuadras al
  inicio si la mayoría ataca; aceptable para el prototipo, revisable si la
  línea de tiempo se siente comprimida.
- Coeficientes sin calibrar: la sensación táctica real depende de playtest.

## Alternativas descartadas

- **Iniciativa por tropa, agregada a la escuadra al final:** mayor costo de
  cómputo y estado, sin beneficio: todas las tropas de una escuadra comparten
  hex y orden, así que el orden relevante es el de la escuadra.
- **Orden puramente aleatorio (cola barajada):** contradice cap. 05/07, que
  afirman explícitamente que la línea de tiempo *no es aleatoria en su orden*.
- **Iniciativa solo por stats (sin contexto de mando/estado):** desperdicia el
  pilar del juego (subordinación). Quedar aislado debe tener costo táctico
  visible, y la línea de tiempo es el lugar natural para expresarlo.
- **Iniciativa como puro desempate de último recurso:** rechazado por cap. 07,
  que la define como "el mecanismo central que ordena la línea de tiempo",
  no un desempate.
- **Sistema de tiempo por puntos de acción acumulables (estilo ATB):** más
  expresivo pero introduce una segunda economía temporal que choca con el
  modelo de ventana fija de 4 h de ADR-004; sobredimensionado para el
  prototipo.
