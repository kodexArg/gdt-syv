---
title: "Comunicaciones"
order: 12
section: communications
lang: es
---

# Comunicaciones

## El problema de ser escuchado

En Subordinación y Valor, el verdadero riesgo de la radio no es la distancia sino
la interceptación. El universo de SyV transcurre en un contexto donde **El Fin de los Secretos**
volvió irreversible la transparencia de las comunicaciones: todo puede ser escuchado,
todo puede ser descifrado eventualmente.

Por esta razón, la solución adoptada por ambas facciones es la **E-UHF**:
una variante encriptada de UHF con cifrado de alta rotación. No es perfecta — en el mundo
de SyV nada lo es — pero es lo suficientemente segura para comunicaciones tácticas cortas.

Los soldados de tropa básica (L1 y L2) **no portan radios personales PRR-UHF** precisamente
porque una señal abierta sería detectada e interceptada de inmediato. Solo los rangos L3 y
superiores operan con E-UHF.

## Tipos de comunicación

### E-UHF táctica (L3, L4, L5)

Los niveles L3, L4 y L5 portan receptores E-UHF individuales.

- **Alcance**: 500 metros (= 5 hexes)
- **Tipo**: omnidireccional — la señal se irradia en todas direcciones
- **Uso**: el oficial transmite órdenes; la respuesta del receptor es mínima

Las respuestas están reducidas a códigos cortos para minimizar la exposición de la señal:

| Código | Significado |
|--------|-------------|
| `Recibido` | Orden recibida, se ejecutará |
| `Copiado` | Entendido, confirmado |
| `Negativo — imposible` | No puede ejecutarse, razón no especificada |
| `En movimiento` | Confirmación de ejecución en curso |

No se transmite información táctica compleja por este canal. La señal corta reduce el
riesgo de intercepción.

> La relación entre el alcance de 500 m y la **regla de los 5 hexes** de mando es directa:
> el límite de la cadena de mando es el límite físico del radio E-UHF.

### Radio de largo alcance (HQ y radio-operadores)

El HQ del Comandante y las escuadras que cuentan con un **radio-operador dedicado**
tienen acceso a comunicación de largo alcance:

- **Alcance**: ilimitado dentro de la grilla
- **Tipo**: **direccionada** — apunta con precisión a un receptor E-UHF específico
- **Seguridad**: más segura que la E-UHF táctica por su precisión, pero no invulnerable

Esta comunicación es la que permite al Comandante transmitir órdenes a los oficiales
de pelotón (L5) sin importar la posición en el tablero. Es también la que genera
el **log de radio** visible en la interfaz del jugador.

### La Tropa de Comunicaciones (L1 especial)

Existe un tipo de Tropa L1 especializada en comunicaciones. Porta una radio de largo
alcance (equivalente a la del HQ) pero **no extiende la cadena de mando por sí sola**.

Su función es amplificar el radio del oficial L3 o superior al que está asignada.
Sin un oficial cercano que la dirija, la radio que porta no tiene efecto en la cadena
de mando. Es un multiplicador, no un origen.

## Interceptación y triangulación

Toda señal puede ser triangulada. El **Oficial Criptógrafo** es la unidad especializada
en interceptar señales E-UHF en tránsito y desencriptarlas.

Las reglas completas de triangulación, stealth y exposición de señal están en:
[Triangulación y Sigilo](12b-triangulacion-y-sigilo.md).

## Resumen

| Tipo | Nivel | Alcance | Dirección | Riesgo interceptación |
|------|-------|---------|-----------|----------------------|
| Sin radio | L1, L2 | — | — | — |
| E-UHF táctica | L3, L4, L5 | 500 m (5 hex) | Omni | Medio |
| Radio larga distancia | HQ, radio-operador | Ilimitado | Dirigida | Bajo |
| Tropa Comunicaciones | L1 especial | Ilimitado (amplifica) | Dirigida | Bajo |
