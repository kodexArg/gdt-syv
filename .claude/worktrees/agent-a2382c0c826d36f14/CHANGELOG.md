# Changelog

All notable changes to Subordinación y Valor (SyV) will be documented here.

Format: [Semantic Versioning](https://semver.org/) — `MAJOR.MINOR.PATCH`

---

## [Unreleased]

## [0.1.0] - 2026-05-18

- group: adr-batch-013-022-design-closure
  priority: critical
  commit: ae2809d
  changes:
    - feat(adr): ADR-013 stats canonicos AIM/TGH/RCT/NRV/MOV/VAL + catalogs Perks/Traits/Phobias + mutacion
    - feat(adr): ADR-014 escenario/presupuesto (escala PF)/despliegue/victoria
    - feat(adr): ADR-015 mando, comunicaciones, triangulacion, perdida de HQ
    - feat(adr): ADR-016 terreno, elevacion, puntos de movimiento, fortificacion
    - feat(adr): ADR-017 degradacion/recuperacion/umbrales de Valor y disband
    - feat(adr): ADR-018 pool de ordenes por rango, genericas, comportamiento de unidad
    - feat(adr): ADR-019 secuencia de resolucion, deteccion en movimiento, designacion de objetivos
    - feat(adr): ADR-020 formula de revelacion y niebla de guerra
    - feat(adr): ADR-021 agregacion de escuadra (strength), heridas por zona, schema de Equipment
    - feat(adr): ADR-022 escuadras especiales L2 (caballeria, zapadores, Cabo Rojo)

- group: data-reconciliation
  priority: high
  commit: 5e78dfb
  changes:
    - fix(data): reconciliar Tropa de Comunicaciones en premade-squads.yaml (manual/04b match)
    - fix(data): definir Sanitario como L1 (manual/04b reconciliation)

- group: documentation-consolidation
  priority: high
  commit: ae2809d
  changes:
    - docs(adr): consolidate Wave B (ADR-017–022), unify index, reconcile moral aggregation under ADR-017
    - docs: rewrite docs/adr/README.md with unified catalog (ADR-000–022)

## [0.0.0] - 2026-04-08

### Setup
- Proyecto inicializado con Godot 4.4.1 stable (GL Compatibility)
- Dependencias de sistema verificadas: Vulkan (NVIDIA 580), Wayland, audio ALSA/PulseAudio
