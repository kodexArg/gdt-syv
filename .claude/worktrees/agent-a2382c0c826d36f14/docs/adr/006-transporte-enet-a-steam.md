# ADR-006: Transporte — ENet para prototipo, Steam Networking Sockets para produccion

## Estado

Aceptado — 2026-04-08

## Contexto

SyV se distribuira en Steam como plataforma final. Sin embargo, la fase actual es un prototipo local donde servidor y clientes corren en la misma maquina. Se necesita elegir el protocolo de transporte para la comunicacion client-server, equilibrando las necesidades inmediatas de desarrollo con la migracion futura a la infraestructura de Valve.

## Decision

### Prototipo (fase actual)

**ENet sobre localhost.** Es el protocolo nativo de Godot, funciona sin configuracion ni dependencias externas. Ideal para iterar rapidamente en una sola maquina.

### Produccion (futuro)

**Steam Networking Sockets** a traves del addon GodotSteam. La infraestructura de Valve ofrece:
- Relay gratuito (los clientes se conectan a traves de servidores de Valve)
- NAT traversal resuelto
- IP del servidor de juego oculta
- Proteccion DDoS integrada

### Regla critica: nomenclatura agnostica

La nomenclatura de interfaces, señales, clases y funciones de red debe ser **agnostica al transporte**. Nada debe llamarse `enet_enviar_ordenes` ni `steam_conectar`. Los nombres deben reflejar la funcion (`enviar_ordenes`, `conectar_a_partida`), no la implementacion.

Esto garantiza que la migracion de ENet a Steam Networking Sockets no requiera renombramientos en toda la base de codigo.

## Consecuencias

**Positivas:**
- Iteracion rapida en prototipo sin dependencias externas
- Migracion a Steam sin renombramientos gracias a la nomenclatura agnostica
- En produccion, Valve provee infraestructura de red gratuita que de otro modo habria que pagar y mantener

**Negativas:**
- ENet en prototipo no ofrece relay ni proteccion — aceptable en localhost
- La migracion a Steam Networking Sockets requerira integrar GodotSteam y adaptar la capa de transporte
- Dos tecnologias de transporte a lo largo del proyecto, aunque la interfaz sea la misma

## Alternativas descartadas

- **WebSocket**: su ventaja principal es la compatibilidad con navegadores. Para un juego de Steam con clientes nativos exclusivamente, esta ventaja es irrelevante. Ademas, WebSocket opera sobre TCP, con mayor overhead que ENet (UDP).
- **Implementar Steam SDK desde el inicio**: añade una dependencia externa y complejidad de integracion sin beneficio tangible en un prototipo local. Es prematuro.
- **ENet permanente sin migrar a Steam**: renuncia a la infraestructura gratuita de Valve (relay, proteccion DDoS, NAT traversal). En produccion, estas capacidades tendrian que construirse o pagarse por separado.
