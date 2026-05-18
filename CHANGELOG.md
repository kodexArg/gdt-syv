# Changelog

All notable changes to Subordinación y Valor (SyV) will be documented here.

Format: [Semantic Versioning](https://semver.org/) — `MAJOR.MINOR.PATCH`

---

## [Unreleased]
- group: protocol-contract
  priority: critical
  commit: cd6238c
  changes:
    - feat(protocol): define network protocol contract + ADR-010

- group: combat-resolution-initiative
  priority: critical
  commit: 81908c6
  changes:
    - docs(adr): ADR-011 stochastic combat resolution per soldier

- group: combat-resolution-initiative
  priority: critical
  commit: 96b8a95
  changes:
    - docs(adr): ADR-012 initiative calculation and resolution order

- group: holistic-vision
  priority: high
  commit: c536876
  changes:
    - docs: game vision synthesis — narrative synthesis across all ADRs

- group: holistic-vision
  priority: high
  commit: 751a3e3
  changes:
    - docs: army structure and combat force — visual synthesis with diagrams

## [0.0.0] - 2026-04-08

### Setup
- Proyecto inicializado con Godot 4.4.1 stable (GL Compatibility)
- Dependencias de sistema verificadas: Vulkan (NVIDIA 580), Wayland, audio ALSA/PulseAudio
