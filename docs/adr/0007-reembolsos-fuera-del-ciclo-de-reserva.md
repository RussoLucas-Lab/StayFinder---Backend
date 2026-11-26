# ADR-0007: Reembolsos como entidad propia, fuera del ciclo de la reserva

- **Estado:** Propuesto
- **Fecha:** 2026-11-26

## Contexto

El plan inicial sugiere, como opción, estados `REFUND_PENDING`, `PARTIALLY_REFUNDED` y `REFUNDED` dentro del ciclo de la reserva, y a la vez exige que "el estado de la reserva no implique falsamente que el dinero ya fue devuelto".

## Decisión

- La reserva modela el ciclo de **negocio** (`PENDING_PAYMENT`, `CONFIRMED`, `CANCELLED`, `EXPIRED`, `COMPLETED`, `NO_SHOW`).
- El módulo `payments` modela el ciclo **financiero**: `Payment` y `Refund`, cada uno con su propio estado. Un reembolso tiene importe, motivo, actor, ID del proveedor y estado (`PENDING`, `SUCCEEDED`, `FAILED`).
- La reserva expone un estado financiero **derivado** (sin pagar, pagada, parcialmente reembolsada, reembolsada), calculado a partir de sus pagos y reembolsos, nunca guardado a mano.

## Alternativas consideradas

- **Estados de reembolso dentro de la reserva.** Descartado: una reserva cancelada con reembolso pendiente necesitaría dos estados a la vez, y un reembolso parcial sin cancelación (por ejemplo, una compensación) no tendría lugar en el ciclo.

## Consecuencias

- Cancelar una reserva y pedir el reembolso son dos pasos: el servicio de cancelación calcula el importe según la política y solicita el reembolso a `payments`.
- Un reembolso fallido no revierte la cancelación; queda visible para reintentar o para revisión del admin.
- Los reembolsos parciales múltiples son posibles sin cambiar el modelo.
