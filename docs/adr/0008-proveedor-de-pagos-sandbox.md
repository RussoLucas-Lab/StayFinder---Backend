# ADR-0008: Stripe como primer proveedor de pagos

- **Estado:** Propuesto
- **Fecha:** 2026-11-26

## Contexto

La versión 1 necesita pagos en sandbox con webhooks verificados. El plan inicial menciona Mercado Pago o Stripe, detrás de una interfaz `PaymentProvider` que desacopla el dominio del proveedor.

## Decisión

Implementar primero un adaptador de **Stripe** en modo test. Mercado Pago queda como segundo adaptador posible, que además demostraría que la abstracción funciona.

Motivos:

- Modo test completo y documentación extensa.
- Su CLI reenvía webhooks a `localhost`, así que se puede probar el flujo real sin exponer el entorno local.
- Firma de webhooks verificable con un secreto por endpoint.
- Trabaja en unidades menores y acepta claves de idempotencia, en línea con [ADR-0005](0005-montos-en-unidades-menores.md) y las invariantes del dominio.

## Alternativas consideradas

- **Mercado Pago primero.** Más relevante si el producto apunta al mercado argentino o a cobros en ARS.
- **Ambos desde el inicio.** Descartado: duplica trabajo antes de que el núcleo funcione.
- **Solo un proveedor falso.** Descartado como única opción: no permite probar verificación real de webhooks. Igual se usa un `FakePaymentProvider` en los tests unitarios y de API.

## Consecuencias

- **Riesgo a verificar antes de aceptar:** que se pueda crear una cuenta de Stripe en modo test desde el país de registro de la cuenta.
- Los tipos de Stripe no salen de `app/integrations/payments/stripe/`; el dominio solo ve `PaymentProvider`.
- Los tests nunca llaman a Stripe real.
