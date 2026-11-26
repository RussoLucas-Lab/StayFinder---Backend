# Dominio

Lenguaje común del proyecto e invariantes que ningún change puede romper. Los requisitos detallados de cada capacidad viven en `openspec/specs/`.

## Glosario

| Término | Nombre en código | Definición |
|---|---|---|
| Propiedad | `Property` | Alojamiento publicable (hotel, hostel, cabaña, departamento). Tiene estado de publicación: `DRAFT`, `PENDING_REVIEW`, `PUBLISHED`, `SUSPENDED`. |
| Tipo de habitación | `RoomType` | Categoría vendible de una propiedad (Standard, Deluxe, Suite) con capacidad de adultos y niños, camas y servicios. Es la unidad de venta ([ADR-0006](adr/0006-inventario-por-tipo-de-habitacion.md), propuesto). |
| Unidad física | `Unit` | Habitación concreta (número 204). Solo existe si el negocio necesita asignarla; no participa del cálculo de disponibilidad en la versión 1. |
| Inventario diario | `InventoryDay` | Para un tipo de habitación y una fecha: capacidad, unidades reservadas, retenidas y bloqueadas. |
| Noche | — | Unidad de consumo de inventario. Una estadía consume las noches del intervalo `[check_in, check_out)`. |
| Bloqueo | `InventoryBlock` | Cierre manual de inventario por el propietario (mantenimiento, uso propio). |
| Restricción | `StayRestriction` | Estadía mínima o máxima, cierre a la llegada o a la salida, para un tipo y rango de fechas. |
| Plan tarifario | `RatePlan` | Conjunto de precio, condiciones (reembolsable o no, desayuno incluido) y política de cancelación. |
| Cotización | `Quote` | Cálculo de precio con desglose para una selección concreta. No retiene inventario. |
| Hold | `Hold` | Retención temporal de inventario con vencimiento (`expires_at`), creada al iniciar el checkout. |
| Reserva | `Reservation` | Acuerdo entre viajero y propiedad. Nace como `PENDING_PAYMENT` junto con su hold. |
| Snapshot | `ReservationSnapshot` | Copia inmutable de lo acordado al reservar: precios por noche, impuestos, cargos, descuentos, moneda, plan tarifario y política de cancelación. |
| Política de cancelación | `CancellationPolicy` | Fecha límite sin cargo y penalidad posterior. Se copia al snapshot. |
| Pago | `Payment` | Intento de cobro ante un proveedor externo, con estado propio. |
| Reembolso | `Refund` | Devolución total o parcial de un pago, con estado propio ([ADR-0007](adr/0007-reembolsos-fuera-del-ciclo-de-reserva.md), propuesto). |
| Evento de webhook | `WebhookEvent` | Notificación recibida de un proveedor, registrada con su ID único para procesarla una sola vez. |
| Clave de idempotencia | `IdempotencyKey` | Clave enviada por el cliente (header `Idempotency-Key`) para que repetir una operación crítica no duplique su efecto. |
| Reseña | `Review` | Opinión de un viajero sobre una estadía completada. Una por estadía. |
| Empleado | `PropertyStaff` | Usuario invitado por un host con permisos limitados sobre ciertas propiedades. |

### Indicadores comerciales

| Indicador | Definición |
|---|---|
| Occupancy Rate | Noches vendidas ÷ noches disponibles en el período |
| ADR (Average Daily Rate) | Ingreso por habitaciones ÷ noches vendidas. **No confundir con los ADR de `docs/adr/`** (Architecture Decision Records). |
| RevPAR | Ingreso por habitaciones ÷ noches disponibles (= Occupancy × ADR) |

Las fórmulas exactas (qué cuenta como "disponible", si los bloqueos restan) se fijan en la spec de métricas, en la versión 3.

## Invariantes

Reglas que valen siempre, en todos los módulos. Una spec puede detallarlas pero nunca contradecirlas.

1. **Sin sobreventa.** Para cada tipo de habitación y fecha: `reservadas + retenidas + bloqueadas <= capacidad`. La garantía la da PostgreSQL (restricción y transacción), no la aplicación ni Redis. La sobreventa solo se habilitaría con una decisión explícita futura.
2. **Disponibilidad única.** `disponible = capacidad − reservadas − retenidas − bloqueadas`. Todo el sistema usa esta fórmula; nadie calcula disponibilidad por su cuenta.
3. **Hold atómico.** Crear el hold y retener el inventario de todas sus noches ocurre en una misma transacción: o se retienen todas las noches, o ninguna.
4. **Liberación única.** El inventario de un hold vencido o cancelado se libera exactamente una vez, aunque el proceso se ejecute varias veces.
5. **Snapshot inmutable.** Cambiar tarifas, políticas o datos de la propiedad nunca altera reservas existentes.
6. **Transiciones controladas.** El estado de una reserva cambia solo por las transiciones de la tabla siguiente, ejecutadas por el backend. Ningún cliente fija un estado directamente.
7. **Pago verificado.** Una reserva que exige pago se confirma solo con evidencia del proveedor: webhook con firma verificada o consulta directa a su API. Nunca por lo que diga el frontend.
8. **Cancelar no es reembolsar.** El estado de la reserva nunca implica que el dinero ya se devolvió.
9. **Efecto único.** Repetir una operación con la misma clave de idempotencia, o recibir dos veces el mismo webhook, produce un único efecto.
10. **Autorización por recurso.** Cada operación verifica que el actor tenga permiso sobre ese recurso concreto, no solo que esté autenticado.
11. **Dinero exacto.** Nunca `float` para importes ([ADR-0005](adr/0005-montos-en-unidades-menores.md), propuesto).
12. **Reseña verificada.** Solo reseña quien tuvo una estadía completada, y una sola vez por estadía.

## Fechas y horas

- Las fechas de estadía (`check_in`, `check_out`) son fechas locales de la propiedad, sin hora ni zona: tipo `date`.
- Los instantes (`expires_at`, `created_at`, fechas límite de cancelación) se guardan en UTC (`timestamptz`).
- La fecha límite de cancelación se calcula en la zona horaria de la propiedad y se guarda en UTC en el snapshot.

## Ciclo de vida de la reserva

```text
              hold creado
                  │
                  ▼
          PENDING_PAYMENT ──── vence sin pago ────▶ EXPIRED
             │        │
   pago      │        └── el viajero abandona ───▶ CANCELLED
 verificado  │
             ▼
         CONFIRMED ──── cancelación según política ─▶ CANCELLED
          │      │
 check-out│      └── el huésped no llega ─────────▶ NO_SHOW
          ▼
      COMPLETED
```

| Desde | Hacia | Disparador | Quién | Efecto sobre inventario |
|---|---|---|---|---|
| — | `PENDING_PAYMENT` | Crear hold | Viajero | Retiene las noches |
| `PENDING_PAYMENT` | `CONFIRMED` | Pago verificado con el hold vigente | Sistema | Retenidas pasan a reservadas |
| `PENDING_PAYMENT` | `EXPIRED` | Vence el hold sin pago confirmado | Sistema | Libera retenidas |
| `PENDING_PAYMENT` | `CANCELLED` | El viajero abandona el checkout | Viajero | Libera retenidas |
| `CONFIRMED` | `CANCELLED` | Cancelación según política | Viajero, host o admin | Libera reservadas de noches futuras |
| `CONFIRMED` | `COMPLETED` | Pasa la fecha de check-out | Sistema | Ninguno |
| `CONFIRMED` | `NO_SHOW` | El huésped no se presentó | Host o admin | A definir (ver preguntas abiertas) |

`EXPIRED`, `CANCELLED`, `COMPLETED` y `NO_SHOW` son estados finales. Toda transición queda auditada con actor, motivo y fecha.

Los estados de reembolso no forman parte de este ciclo ([ADR-0007](adr/0007-reembolsos-fuera-del-ciclo-de-reserva.md), propuesto).

## Preguntas abiertas

Se resuelven en el change que las necesita (en `design.md` o con un ADR), no antes:

1. **Carrera entre pago y vencimiento.** Si el webhook de pago aprobado llega después de que el hold venció: ¿se intenta retener de nuevo y confirmar si hay disponibilidad, o se reembolsa automáticamente? Se decide en el change de pagos.
2. **Liberación en `NO_SHOW`.** ¿Se liberan las noches restantes de una estadía larga cuando el huésped no llega?
3. **Penalidad de `NO_SHOW`.** ¿Se cobra según la política de cancelación o según otra regla?
4. **Duración del hold.** ¿Fija (por ejemplo 15 minutos) o configurable por propiedad?
5. **Reservas de varias habitaciones.** ¿Una reserva con varios tipos de habitación o una reserva por tipo?
