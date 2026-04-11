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

## Backlog Agent (always active)

This project uses a Godot-specific Notion backlog managed by `/gdt-notion`. The agent must be **proactive** about the backlog during all work in this project.

### When to suggest adding to the Backlog

During any conversation about this project, if any of the following happen:
- A **decision** is made (architecture, convention, tool choice)
- A **bug** is found or fixed (the fix is done, but related issues surfaced)
- A **feature** is discussed, requested, or implied ("we should...", "eventually...", "we need...")
- A **scope change** happens (something gets cut, deferred, or added)
- A **blocked dependency** is identified ("this needs X first")
- A **TODO/FIXME** is found in code during work
- The user says something that implies future work

### How to ask

Match the language the user is currently using:

- Spanish: **Deberia poner esto en el Backlog?**
- English: **Should I add this to the Backlog?**

Always propose a compact entry so the user can approve or adjust:

```
Deberia poner esto en el Backlog?
  Task: "Implement fog-of-war per scope"
  Category: Core Mechanic | Priority: High | Scope: Prototype | Effort: Large
```

If the user confirms ("dale", "si", "yes", "go"), add it immediately via `/gdt-notion`.
If they adjust ("but make it Medium"), apply the adjustment.
If they decline ("no", "not yet"), drop it.

### What NOT to do

- Never silently add items to the backlog
- Never add items without asking first
- Never ask for fields you can infer (Effort, Category) — propose your best guess
- Never break the flow of work to insist on backlog items — ask at natural pauses
