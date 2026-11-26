# ADR-0001: Monolito modular por dominios

- **Estado:** Aceptado
- **Fecha:** 2026-11-26

## Contexto

El sistema tiene muchos dominios (identidad, propiedades, inventario, precios, reservas, pagos, reseñas, notificaciones), pero todavía no tiene usuarios, carga medida ni equipo grande. Las operaciones más delicadas (retener inventario, confirmar una reserva tras un pago) necesitan transacciones que abarcan varios dominios.

## Decisión

Un solo servicio FastAPI con una sola base PostgreSQL, organizado en módulos por dominio dentro de `app/modules/`, con reglas explícitas de dependencia entre módulos ([architecture.md](../architecture.md)).

## Alternativas consideradas

- **Microservicios desde el inicio.** Descartado: obliga a resolver consistencia distribuida (sagas, outbox, compensaciones) antes de tener el núcleo funcionando, y multiplica la infraestructura.
- **Monolito sin módulos (por capas técnicas: `routers/`, `models/`, `services/`).** Descartado: los límites de dominio se diluyen y extraer un módulo después sería una reescritura.

## Consecuencias

- Las transacciones entre dominios son transacciones locales de PostgreSQL.
- Hay que respetar la regla de no acceder a tablas de otro módulo; si se rompe, el monolito se vuelve un bloque.
- Extraer un servicio en la versión 4 sigue siendo posible si los límites se respetaron.
