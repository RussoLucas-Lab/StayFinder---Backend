# ADR-0002: PostgreSQL como fuente de verdad; Redis como complemento

- **Estado:** Aceptado
- **Fecha:** 2026-11-26

## Contexto

La garantía central del sistema es no vender dos veces el mismo cupo. Redis es rápido y ofrece TTL, lo que tienta a usarlo para holds o contadores de disponibilidad, pero su persistencia y sus garantías transaccionales son más débiles que las de PostgreSQL.

## Decisión

PostgreSQL guarda el estado autoritativo de inventario, holds, reservas, pagos e idempotencia. Redis se usa solo para datos reconstruibles: cache de búsquedas y datos públicos, rate limiting y broker de Celery.

## Alternativas consideradas

- **Holds en Redis con TTL.** Descartado: si Redis se cae o pierde datos, se pierde la retención y puede haber sobreventa; además el TTL libera inventario sin dejar registro auditable.
- **Contadores de disponibilidad en Redis con sincronización a PostgreSQL.** Descartado: dos fuentes de verdad que hay que reconciliar.

## Consecuencias

- Si Redis no está disponible, el sistema puede degradarse (sin cache, sin rate limiting), pero nunca vende de más.
- La expiración de holds se apoya en `expires_at` guardado en PostgreSQL, no en un TTL.
- Las invariantes de inventario se expresan como restricciones de base de datos.
- Hay un escenario de test obligatorio: con Redis caído, el inventario sigue siendo consistente.
