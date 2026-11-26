# Architecture Decision Records

Cada ADR registra una decisión transversal: el contexto, la decisión, las alternativas y sus consecuencias.

## Estados

| Estado | Significado |
|---|---|
| **Propuesto** | Recomendación pendiente de aprobación. No es vinculante: las tareas que dependen de él esperan. |
| **Aceptado** | Vigente. Los changes y el código lo respetan. |
| **Reemplazado por ADR-XXXX** | Dejó de valer; se conserva como historia. |
| **Descartado** | Se evaluó y no se adoptó. |

Un ADR aceptado no se edita para cambiar la decisión: se escribe uno nuevo que lo reemplaza y se actualiza el estado del anterior.

Un ADR aceptado sí admite aclaraciones que no cambian la decisión (sacar un detalle de implementación, corregir una referencia). Cada aclaración se registra con una nota fechada en una sección `## Notas` al final del ADR.

## Índice

| # | Decisión | Estado |
|---|---|---|
| [0001](0001-monolito-modular.md) | Monolito modular por dominios | Aceptado |
| [0002](0002-postgresql-fuente-de-verdad.md) | PostgreSQL como fuente de verdad; Redis como complemento | Aceptado |
| [0003](0003-sdd-openspec-engram.md) | SDD con OpenSpec, artefactos en archivos y Engram como memoria | Aceptado |
| [0004](0004-repos-separados-front-back.md) | Repositorios separados para frontend y backend | Aceptado |
| [0005](0005-montos-en-unidades-menores.md) | Montos en unidades menores de la moneda | Propuesto |
| [0006](0006-inventario-por-tipo-de-habitacion.md) | Inventario por tipo de habitación y fecha | Propuesto |
| [0007](0007-reembolsos-fuera-del-ciclo-de-reserva.md) | Reembolsos como entidad propia, fuera del ciclo de la reserva | Propuesto |
| [0008](0008-proveedor-de-pagos-sandbox.md) | Stripe como primer proveedor de pagos | Propuesto |
| [0009](0009-acceso-async-a-base-de-datos.md) | Acceso async a la base de datos | Propuesto |
| [0010](0010-ejecucion-en-contenedores.md) | Ejecución en contenedores (desarrollo y despliegue) | Aceptado |

## Plantilla

```markdown
# ADR-XXXX: <título>

- **Estado:** Propuesto
- **Fecha:** AAAA-MM-DD

## Contexto
Qué problema hay y qué fuerzas están en juego.

## Decisión
Qué se decide, en una o dos oraciones claras.

## Alternativas consideradas
Cada alternativa con su motivo de descarte.

## Consecuencias
Qué se vuelve más fácil, qué se vuelve más difícil y qué hay que hacer a partir de ahora.
```
