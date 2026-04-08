Subordinación y Valor ('syv') es un juego de estrategia por turnos simultáneos, multijugador, en un tablero de grilla hexagonal de varios niveles. Arquitectura de servidor autoritativo con Godot headless.

Está íntegramente programado en Godot.

## Motor

- **Godot Engine**: 4.4.1 stable
- **Renderer**: GL Compatibility
- **Binary**: `~/.local/bin/godot`

## Decisiones arquitectónicas (ADR)

Documentación completa en `docs/adr/`. Resumen:

| ADR | Decisión |
|-----|----------|
| 001 | **Nomenclatura y glosario** — vocabulario canónico del proyecto (Briefing, Orders, Resolution, scope, autoridad) |
| 002 | **Convenciones Godot** — reglas explícitas de uso del motor; documento vivo |
| 003 | **Backend autoritativo** — Godot headless con la misma base de código GDScript que el cliente |
| 004 | **Ciclo de turno** — tres fases: Briefing (servidor calcula scope), Orders (cliente planifica local), Resolution (servidor ejecuta y emite eventos) |
| 005 | **Seguridad por scope** — el cliente nunca recibe información fuera de su scope; fog-of-war por arquitectura |
| 006 | **Transporte** — ENet en prototipo, Steam Networking Sockets en producción; nomenclatura agnóstica |
| 007 | **Listen server** — prototipo con una instancia como server+client; separación por código, no por procesos |
| 008 | **Estructura** — shared/ (lógica), client/ (UI), server/ (árbitro), protocol/ (contrato de mensajes) |
