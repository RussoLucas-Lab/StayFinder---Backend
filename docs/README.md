# Documentación de StayFinder Backend

Esta carpeta guarda el **porqué** del proyecto: visión, dominio, arquitectura, convenciones y decisiones. El **qué hace el sistema** (requisitos y escenarios) vive en `openspec/specs/`, no acá.

## Índice

| Documento | Contenido | Cuándo leerlo |
|---|---|---|
| [vision.md](vision.md) | Producto, usuarios, alcance, no-objetivos, definición de terminado | Para entender qué se construye y qué no |
| [domain.md](domain.md) | Glosario, invariantes de negocio, ciclo de vida de la reserva, preguntas abiertas | Antes de escribir specs o tocar inventario, reservas o pagos |
| [architecture.md](architecture.md) | Monolito modular, módulos, capas, estructura de carpetas, reglas | Antes de crear módulos o mover código |
| [conventions.md](conventions.md) | Código, API, base de datos, testing, git, seguridad | Antes de escribir código |
| [workflow.md](workflow.md) | Setup de herramientas, ciclo SDD con OpenSpec, uso de Engram | Al empezar a trabajar o al dudar del proceso |
| [roadmap.md](roadmap.md) | Versiones 1 a 4 y formato del change map | Para planificar changes |
| `change-map.md` | Ruta ordenada de changes. Todavía no existe: se genera a partir de [roadmap.md](roadmap.md#formato-del-change-map) | Para elegir el próximo change |
| [adr/](adr/README.md) | Decisiones de arquitectura con contexto y alternativas | Cuando una tarea depende de una decisión transversal |
| [reference/plan-inicial.md](reference/plan-inicial.md) | Plan original completo | Como insumo para specs y change map; **no es normativo** |

## Fuentes de verdad y precedencia

Si dos fuentes se contradicen, manda la de más arriba:

1. `openspec/specs/`: lo que el sistema debe hacer hoy.
2. `docs/adr/` con estado **Aceptado**.
3. El resto de `docs/`.
4. `docs/reference/plan-inicial.md`.
5. Engram (memoria entre sesiones): sirve para recordar, nunca para decidir.

- Si el código contradice una spec, es un bug o falta un change.
- Un ADR en estado **Propuesto** no es vinculante. Las tareas que dependan de él esperan a que se acepte.
- Una contradicción entre fuentes se reporta; no se resuelve en silencio.

## Qué va en cada lugar

| Si tenés... | Va en |
|---|---|
| Un comportamiento observable (endpoint, regla, error, permiso) | Spec de OpenSpec, vía un change |
| Una decisión transversal con alternativas reales | ADR nuevo en `docs/adr/` |
| Un término nuevo del dominio o una invariante | `domain.md` |
| Una regla de estilo, herramienta o formato | `conventions.md` |
| Un hallazgo de sesión, un gotcha, el estado del trabajo | Engram |
| La forma concreta de implementar un change (tablas, locks, clases) | `design.md` del change |

## Mantenimiento

- Los docs cambian poco y por decisión explícita. Si un change altera el dominio, la arquitectura o las convenciones, la actualización del doc es una tarea más de ese change.
- Estos docs no listan endpoints ni payloads: eso lo documentan las specs y el OpenAPI generado por FastAPI.
- Un ADR aceptado no se edita para cambiar la decisión: se crea uno nuevo que lo reemplaza.
