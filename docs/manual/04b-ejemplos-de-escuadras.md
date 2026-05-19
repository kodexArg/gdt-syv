---
title: "Ejemplos de Escuadras"
order: 4
section: units
lang: es
---

# Ejemplos de Escuadras

Este documento presenta ejemplos concretos de composición de escuadras. Cada ejemplo muestra quiénes la integran, cómo funcionan en juego, y qué sucede cuando pierden miembros clave.

Para la mecánica de niveles, radio y cadena de mando, ver [Unidades](04-unidades.md) y [Mando y Subordinación](08-mando-y-subordinacion.md).

---

## 1. Escuadra de Infantería Estándar (Confederación)

**Tipo:** INFANTRY | **Squad level:** L3 | **Facción:** Confederación

La escuadra de infantería es la columna vertebral de la Confederación. Versátil, robusta, y pensada para mantener posiciones y presionar.

### Composición

| Tropa | Nivel | Rol |
|-------|-------|-----|
| Sargento | L3 | Jefe de escuadra — porta radio, comanda la escuadra |
| Cabo Primero | L2 | Líder del Grupo de Fuego |
| Soldado × 4 | L1 | Miembros del Grupo de Fuego (incluye un artillero con MG) |
| Cabo | L2 | Líder del Grupo Scout |
| Soldado × 3 | L1 | Miembros del Grupo Scout (incluye un sniper y un oteador) |
| Comunicaciones | L1 | Porta radio global; amplifica el rango del Sargento |

**Total:** 15 tropas. Dos grupos internos bajo dos L2; la Tropa de Comunicaciones como miembro directo bajo el Sargento.

> **Nota reconciliación #62 (2026-05-18):** La Tropa de Comunicaciones estaba listada en esta tabla pero ausente en `conf-infanteria-estandar.yaml`. Se añadió al YAML como `direct_member` L1 bajo el Sargento. El total pasa de 14 a 15.

### Mecánica en juego

La escuadra ocupa un hex. Su fuerza suma los niveles de todos sus miembros vivos. El Sargento porta radio y actúa como relay en la cadena de mando. La Tropa de Comunicaciones amplifica su alcance.

El Grupo de Fuego concentra potencia de fuego directa; el Grupo Scout aporta capacidades de visión especiales.

<!-- Pendiente: efectos concretos del Grupo Scout en niebla de guerra -->

#### Nota sobre Sanitarios

> **Decisión #63 (2026-05-18):** Nivel definido como **L1**; lugar: **miembro directo** de la escuadra, fuera de grupo, bajo el Sargento — igual que la Tropa de Comunicaciones. Coherente con ADR-009 (L1 = tropa básica sin autoridad de mando) y con `conf-sanitarios.yaml` donde los Médicos son L1. El Sanitario no lidera ni organiza sub-grupos; su rol es exclusivamente de soporte.

Si la escuadra incluye un Sanitario (tropa opcional, **L1**, miembro directo bajo el Sargento), este puede **estabilizar** soldados heridos durante el combate o entre turnos. Un Sanitario **no cura heridas** — es decir, no remueve los modificadores negativos acumulados. Su rol es **prevenir que empeoren**: mantiene a un soldado herido en su estado actual, evitando complicaciones que lo acercarían al umbral de eliminación. Este enfoque refuerza el tema de atrito: el daño es permanente, y la supervivencia depende de presión táctica sostenida, no de recuperación mágica.

### Bajas clave

- **Muere el Sargento (L3):** La escuadra pierde su líder con radio. Si ningún otro miembro porta radio, `has_radio` pasa a `false`. La escuadra puede dejar de actuar como relay. Los L2 no asumen el mando operativo por sí solos.
- **Muere la Tropa de Comunicaciones:** La escuadra vuelve al radio estándar de 5 hexes del Sargento. Si eso la deja fuera de mando, puede desencadenar una tirada de Valor.
- **Muere el Cabo Primero (L2):** El Grupo de Fuego pierde su organización interna. Las tropas del grupo siguen vivas pero sin mecánicas de grupo.
- **Bajas entre soldados:** La fuerza decrece proporcionalmente. Una escuadra con 14 miembros que pierde 5 sigue siendo funcional, pero con menos capacidad de combate.

---

## 2. Escuadra de Mortero (Confederación)

**Tipo:** MORTARS | **Squad level:** L3 | **Facción:** Confederación

Una escuadra pequeña, especializada, que opera como pieza de apoyo de fuego. No tiene grupos internos: todos los miembros se organizan directamente bajo el Sargento.

### Composición

| Tropa | Nivel | Rol |
|-------|-------|-----|
| Sargento | L3 | Jefe de escuadra — porta radio |
| Artillero | L1 | Operador del mortero |
| Artillero | L1 | Asistente de carga |
| Soldado | L1 | Protección directa |

**Total:** 4 tropas. Sin grupos internos.

### Mecánica en juego

El mortero es una pieza que requiere al menos dos artilleros para operar. La escuadra ocupa un hex y no suele avanzar al frente: se posiciona en la retaguardia y aporta fuego indirecto.

**Fuego indirecto:** El mortero dispara a **coordenadas de hexes**, sin necesidad de línea de vista. El oficial de escuadra designa el hex objetivo (por coordenadas) durante la fase Orders, y el mortero impacta en ese hex específico durante la Resolution. El proyectil afecta **solo al hex objetivo** — no hay daño de área ni "splash" a hexes adyacentes. Todas las unidades en el hex impactado se ven afectadas; el resto no.

El Sargento porta la radio y mantiene la escuadra en la cadena de mando. Sin él, la escuadra pierde radio inmediatamente: son 3 tropas L1 sin ningún portador.

### Bajas clave

- **Muere el Sargento (L3):** La escuadra queda con tres L1 sin radio. `has_radio = false`. Sin relay, sin mando. Los tres soldados están expuestos a disband si la tirada de Valor falla.
- **Muere un Artillero:** La pieza puede quedar sin operador. Si ambos artilleros caen, el mortero deja de funcionar aunque la escuadra siga existiendo.
- **Mueren 2 de 4 tropas:** La fuerza cae a la mitad. La escuadra sigue siendo una posición en el tablero, pero apenas puede defenderse.

---

## 3. Los Infernales (Los Rojos)

**Tipo:** CAVALRY | **Squad level:** L2 | **Facción:** Los Rojos

Los Infernales son caballería ligera en motos eléctricas todoterreno. Su nomenclatura es de milicia gaucha irregular, a diferencia del resto de Los Rojos. Son una escuadra especial L2: operan sin radio, con órdenes genéricas, y tienen reglas propias de caballería.

### Composición

| Tropa | Nivel | Rol |
|-------|-------|-----|
| Jefe de Partida | L2 | Líder de la escuadra — sin radio |
| Infernal × 4–6 | L1 | Tropa montada |

**Total:** 5–7 tropas. Sin grupos internos formales. Sin portadores de radio.

Solo Infernales pueden integrar esta escuadra (`compatible_squads = ["infernales"]`). Un soldado de infantería no puede ser asignado.

### Mecánica en juego

Los Infernales operan fuera de la cadena de mando convencional. No reciben órdenes directas del HQ durante la fase Orders: su Teniente de Pelotón les asigna una orden genérica antes de que entren en operación ("Flanqueen por el este", "Hostiguen y retiren"). Con esa directiva, actúan con autonomía limitada según sus reglas de caballería.

No actúan como relay. No pueden ampliar la red de mando de otras escuadras.

<!-- Pendiente: reglas especiales de caballería — ¿movimiento extra? ¿restricciones de terreno? ¿bonificación de carga? -->

### Bajas clave

- **Muere el Jefe de Partida (L2):** Los Infernales L1 quedan sin mando. Sin L2 vivo, están expuestos a disband.
- **Bajas entre Infernales:** La fuerza decrece. Son una escuadra rápida y de hostigamiento, no de resistencia prolongada: las bajas los sacan del juego más rápido que a una escuadra de infantería numerosa.
- **Sin radio en ningún momento:** Los Infernales nunca tuvieron radio. Su disband no viene de perder portadores sino de perder al Jefe de Partida o de que la orden genérica deje de ser ejecutable.

---

## 4. Escuadra de Zapadores

**Tipo:** ENGINEERS | **Squad level:** L2 | **Facción:** cualquiera

Los zapadores son especialistas de ingeniería de combate. Liderados por un L2 (sin radio), operan con órdenes genéricas y tienen reglas especiales de ingeniería: colocación de explosivos, apertura de brechas, destrucción de infraestructura.

### Composición

| Tropa | Nivel | Rol |
|-------|-------|-----|
| Cabo Primero (Conf.) / Cabo (Rojos) | L2 | Líder de la escuadra — sin radio |
| Soldado × 2 | L1 | Grupo de Ingeniería: operadores de carga |
| Soldado × 2 | L1 | Escolta de combate |

**Total:** 5 tropas. El Grupo de Ingeniería puede formalizarse como un grupo interno con el Cabo como líder.

### Mecánica en juego

Los zapadores se despliegan con una orden genérica táctica ("Destruyan el puente central", "Preparen el perímetro sur para minado"). No reciben nuevas instrucciones durante esa operación.

Su valor no está en la fuerza de combate —son pocos y su fuerza es baja— sino en la capacidad de modificar el tablero o negar posiciones al enemigo.

<!-- Pendiente: reglas especiales de ENGINEERS — ¿qué estructuras pueden afectar? ¿cómo interactúa con el sistema de terreno y hexes? -->

### Bajas clave

- **Muere el Cabo L2:** Los cuatro soldados L1 quedan sin mando. Sin L2 ni L3 ni superior, todos están expuestos a disband. Una escuadra de zapadores sin líder es un grupo de soldados dispersos con explosivos.
- **Mueren los operadores de carga:** La escuadra pierde capacidad de ejecutar su función especial, aunque siga existiendo como unidad de combate débil.
- **La orden genérica ya no es ejecutable:** Si el objetivo fue destruido por otro medio, o el terreno cambió, los zapadores quedan en limbo hasta el siguiente turno con nueva directiva.

<!-- Pendiente: mecánica para reasignar órdenes genéricas a escuadras L2 durante la partida -->
