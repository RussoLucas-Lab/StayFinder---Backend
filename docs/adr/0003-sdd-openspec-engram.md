# ADR-0003: SDD con OpenSpec, artefactos en archivos y Engram como memoria

- **Estado:** Aceptado
- **Fecha:** 2026-11-26

## Contexto

El proyecto se desarrolla con un agente de IA (Claude Code) usando Spec-Driven Development. Hay tres tipos de conocimiento que tienden a mezclarse: qué hace el sistema, por qué se decidió así y qué pasó en cada sesión de trabajo. Si se mezclan, aparecen versiones contradictorias.

## Decisión

- **OpenSpec** gestiona el comportamiento: `openspec/specs/` es lo que el sistema debe hacer y `openspec/changes/` lo que está cambiando. Los artefactos (proposal, specs, design, tasks) se guardan **como archivos en el repositorio**.
- **`docs/`** guarda el porqué: visión, dominio, arquitectura, convenciones y ADRs.
- **Engram** es la memoria entre sesiones del agente. No guarda artefactos de SDD ni es fuente de verdad.
- La precedencia entre fuentes está definida en [docs/README.md](../README.md#fuentes-de-verdad-y-precedencia).
- El estilo de trabajo del agente sigue la skill `karpathy-guidelines`: pensar antes de codear, simplicidad, cambios acotados y criterios verificables.

## Alternativas consideradas

- **Artefactos de SDD guardados en Engram.** Descartado: el historial de changes no quedaría en el repositorio, donde lo puede revisar cualquiera que lo clone.
- **Solo `docs/`, sin OpenSpec.** Descartado: sin un ciclo de cambios con deltas, la documentación de comportamiento se desactualiza.
- **Sin memoria entre sesiones.** Descartado: cada sesión empezaría sin saber en qué quedó la anterior.

## Consecuencias

- `openspec/changes/archive/` funciona como historial revisable de cómo evolucionó el sistema.
- Toda decisión importante que aparezca en una sesión se promueve a ADR o spec.
- Hay que mantener `CLAUDE.md` corto y apuntando a `docs/`, para no duplicar contenido.
