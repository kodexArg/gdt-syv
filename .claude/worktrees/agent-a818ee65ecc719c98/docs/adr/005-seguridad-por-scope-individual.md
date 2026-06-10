# ADR-005: Seguridad por scope individual

## Estado

Aceptado — 2026-04-08

## Contexto

En un juego de estrategia, la informacion es poder. Conocer la posicion de las unidades enemigas, sus recursos o sus movimientos planificados otorga una ventaja decisiva. Si la niebla de guerra se implementa unicamente como filtro visual en el cliente, un jugador con conocimientos tecnicos podria acceder a la informacion oculta inspeccionando la memoria del proceso, interceptando paquetes de red o modificando el ejecutable. Esto destruye la integridad competitiva del juego.

## Decision

El servidor solo envia a cada cliente la porcion del estado del juego que le corresponde conocer — su **scope**. La informacion del oponente nunca transita hacia el cliente en ningun momento. No se oculta: directamente no existe en el cliente.

### Principios

- **El cliente nunca recibe informacion fuera de su scope.** Lo que el jugador no debe ver, no llega a su maquina.
- **Las ordenes se validan server-side contra el estado real completo.** El servidor conoce todo; el cliente, solo su porcion.
- **No hay estado de juego "hackeable" en el cliente** porque la informacion sensible nunca llega a el. No se puede extraer lo que no esta.
- **El Briefing con scope individual es el mecanismo central de fog-of-war.** La niebla de guerra no es un efecto visual: es una propiedad de la arquitectura.

### Implicaciones para el Briefing

Durante la fase de Briefing, el servidor calcula y serializa un estado diferente para cada jugador. Esto implica que el Briefing no es un broadcast de un mismo paquete, sino una emision individualizada.

## Consecuencias

**Positivas:**
- La niebla de guerra es invulnerable desde el cliente por construccion, no por ofuscacion
- Los payloads son mas pequeños al transmitir solo lo necesario
- La integridad competitiva esta garantizada por la arquitectura, no por la confianza en el cliente
- Simplifica el cliente: no necesita logica de filtrado ni ocultamiento

**Negativas:**
- El servidor debe calcular y serializar un estado diferente para cada jugador, lo que añade complejidad a la logica de Briefing
- Debugging mas complejo: para reproducir lo que ve un jugador, hay que reconstruir su scope especifico
- Cualquier nueva mecanica que exponga informacion debe pasar por el filtro de scope en el servidor

## Alternativas descartadas

- **Enviar el estado completo y filtrar en el cliente**: inseguro por definicion. Cualquier modificacion del cliente, inspeccion de memoria o analisis de trafico expone toda la informacion. Es seguridad por conveniencia, no por diseño.
- **Encriptar el estado completo en el cliente**: seguridad por oscuridad. La clave de desencriptacion debe estar en el cliente para que pueda usarla, lo que la hace vulnerable a ingenieria inversa. Ademas, no reduce el payload.
