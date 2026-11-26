# Roadmap

Objetivos por versión. El detalle ordenado y ejecutable vive en `docs/change-map.md`, que se genera a partir de este documento y de [reference/plan-inicial.md](reference/plan-inicial.md).

## Versión 1: Core Booking

Producto mínimo con un flujo de reserva coherente y consistente bajo concurrencia.

- Esqueleto de FastAPI, configuración, errores uniformes, `/health` y `/ready`, Docker Compose.
- PostgreSQL, SQLAlchemy y Alembic.
- Registro, login, tokens y permisos básicos por rol y recurso.
- Propiedades y tipos de habitación (alta, edición, publicación).
- Inventario diario y calendario de disponibilidad.
- Búsqueda con fechas, ocupación y filtros básicos.
- Cotización con desglose.
- Holds temporales con prevención de doble reserva.
- Pago en sandbox, webhook verificado, confirmación idempotente.
- Cancelación con liberación de inventario.
- Tests de los flujos principales, incluidos los de concurrencia del último cupo.

## Versión 2: Advanced Booking

- Tarifas por fecha y temporada; políticas de cancelación configurables.
- Cupones y promociones.
- Redis con cache selectiva; Celery y workers.
- Expiración automática de holds; notificaciones por email.
- Reembolsos totales y parciales.
- Reseñas verificadas.
- Panel y API completos del propietario (calendario, staff con permisos).
- Auditoría de operaciones críticas.
- Tests de integración y concurrencia más completos.

## Versión 3: Production & Business

- Métricas de ocupación, ADR y RevPAR.
- Comisiones de plataforma; liquidaciones a propietarios si el modelo lo requiere.
- Multi-moneda con reglas explícitas; multi-idioma.
- OpenTelemetry, Prometheus y Grafana.
- Pruebas de carga; CI/CD con despliegue; backups con restauración probada.

## Versión 4: Escalabilidad

Solo si las métricas lo justifican: eventos y bus de mensajes, extracción selectiva de servicios, motor de búsqueda especializado, recomendaciones, channel managers, alta disponibilidad.

## Puntos que el change map tiene que resolver

El plan inicial tiene estas tensiones; el change map debe tomar una posición explícita sobre cada una:

1. **Holds sin expiración en la versión 1.** Los holds están en la versión 1, pero la expiración automática con Celery está en la 2. Sin un mecanismo de liberación, los holds vencidos retienen inventario para siempre. Hay que decidir cómo se liberan en la versión 1.
2. **Notificaciones.** La definición de terminado de la versión 1 incluye notificar al viajero y al propietario, pero las notificaciones por email están en la versión 2. Hay que mover una de las dos o definir una notificación mínima.
3. **Decisiones pendientes.** Los changes de inventario, pagos y reembolsos dependen de ADRs en estado Propuesto (0005 a 0009). El change map indica qué ADR debe aceptarse antes de cada change.

## Formato del change map

`docs/change-map.md` contiene:

1. Una tabla con una fila por change, en orden de ejecución:

   | # | Change ID | Objetivo | Capacidades (nuevas / modificadas) | Depende de | ADRs requeridos | Versión | Estado |
   |---|---|---|---|---|---|---|---|

   - **Change ID:** kebab-case con verbo al inicio (`add-property-management`, `add-inventory-holds`). Es el nombre de la carpeta en `openspec/changes/`.
   - **Objetivo:** una oración con el resultado observable.
   - **Estado:** `pendiente`, `en curso`, `archivado`.

2. Un diagrama de dependencias entre changes.
3. Para cada change, de tres a cinco líneas: alcance, lo que queda afuera y los escenarios críticos que deben tener test.

Reglas para armarlo:

- El primer change es chico y valida el flujo completo (OpenSpec, Engram, tests, Docker): esqueleto de la API con `/health` y `/ready`.
- Cada change entrega algo verificable con tests; nada de changes que solo crean modelos sin comportamiento.
- Un change no debería tocar más de dos o tres módulos.
- Los changes críticos (holds, pagos, webhooks) son changes propios, no partes de uno más grande.
