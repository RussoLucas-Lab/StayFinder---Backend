> **Documento de referencia, no normativo.** Es el plan original del proyecto y sirve como insumo para escribir specs, ADRs y el change map. Cuando una sección ya se convirtió en spec (`openspec/specs/`) o en ADR (`docs/adr/`), manda esa fuente. Ver la precedencia en [docs/README.md](../README.md#fuentes-de-verdad-y-precedencia).

# StayFinder — Booking Clone API

**Plan de producto, arquitectura y desarrollo**  
**Tipo de proyecto:** plataforma de reservas de alojamientos y gestión de propiedades (OTA / Booking Engine)  
**Backend:** Python + FastAPI  
**Base de datos principal:** PostgreSQL  
**Cache y colas:** Redis + Celery  
**Testing:** pytest

---

## 1. Visión del proyecto

Construir una plataforma inspirada en Booking.com, enfocada inicialmente en **hoteles y alojamientos**. Los viajeros podrán buscar propiedades, consultar disponibilidad, comparar tarifas, reservar, pagar y administrar sus estadías. Los propietarios podrán publicar y administrar propiedades, habitaciones, tarifas, inventario y reservas. Los administradores tendrán herramientas de supervisión y operación.

El objetivo no es hacer solamente un CRUD de hoteles. El objetivo es desarrollar un backend con reglas de negocio reales: disponibilidad por fecha, inventario limitado, reservas temporales, prevención de doble reserva, precios, pagos, cancelaciones, reembolsos, notificaciones, auditoría y observabilidad.

### Alcance inicial: alojamientos, no vuelos

Se recomienda comenzar por hoteles, hostels, cabañas, departamentos y otros alojamientos. Los vuelos agregan una complejidad considerable: inventario de asientos, clases tarifarias, emisión de tickets, cambios de itinerario y proveedores globales. Se pueden considerar como una expansión futura, no como parte del MVP.

### Propuesta de valor

Para diferenciar el proyecto de un clon genérico, StayFinder puede enfocarse en alojamientos independientes y hoteles boutique, ofreciendo tanto un motor de reservas para viajeros como herramientas de administración para propietarios.

---

## 2. Tipos de usuario y permisos

### Traveler — Viajero

- Buscar y comparar alojamientos.
- Consultar disponibilidad y precios.
- Crear y administrar reservas.
- Pagar mediante un proveedor integrado.
- Consultar comprobantes e historial.
- Cancelar reservas según las políticas aplicables.
- Publicar reseñas después de una estadía verificada.

### Host — Propietario o administrador de alojamiento

- Crear y editar propiedades.
- Administrar habitaciones o unidades.
- Gestionar tarifas, restricciones y disponibilidad.
- Consultar y administrar reservas de sus propiedades.
- Bloquear fechas y cerrar inventario.
- Configurar políticas de cancelación y servicios.
- Consultar ocupación, ingresos e indicadores.
- Invitar empleados con permisos limitados, sin compartir la cuenta principal.

### Admin — Administrador de la plataforma

- Administrar usuarios, propietarios y propiedades.
- Revisar, aprobar, suspender o moderar alojamientos.
- Supervisar reservas, pagos, reembolsos y disputas.
- Consultar auditorías y métricas de negocio.
- Gestionar reportes de contenido y reseñas.

**Regla esencial:** no alcanza con validar que el usuario esté autenticado. Cada operación debe comprobar que el usuario tenga permiso sobre el recurso específico. Por ejemplo, un propietario solo puede modificar propiedades que administra y un viajero solo puede consultar sus propias reservas.

---

## 3. Módulos funcionales

### 3.1. Identidad y autenticación (`identity`)

- Registro e inicio de sesión.
- Hash seguro de contraseñas.
- Access tokens y refresh tokens.
- Cierre de sesión e invalidación o rotación de refresh tokens.
- Recuperación y restablecimiento de contraseña.
- Verificación de email, si corresponde.
- Roles y permisos.
- Gestión de sesiones y dispositivos, como mejora posterior.

### 3.2. Propiedades (`properties`)

- Crear, editar y consultar alojamientos.
- Datos generales: nombre, descripción, dirección y ubicación geográfica.
- Tipo de alojamiento: hotel, hostel, departamento, cabaña, etc.
- Servicios y comodidades.
- Galería de imágenes.
- Reglas del alojamiento: check-in, check-out, mascotas, fumar, edades, etc.
- Estado de publicación: borrador, pendiente de revisión, publicado, suspendido.
- Asociación entre propiedades y propietarios/empleados.

### 3.3. Habitaciones y unidades (`rooms`)

- Tipos de habitación y unidades físicas, según el modelo elegido.
- Capacidad de adultos y niños.
- Cantidad de camas y configuración.
- Servicios específicos.
- Imágenes.
- Estado activo/inactivo.

Conviene distinguir entre **tipo de habitación** y **unidad física**. Por ejemplo, un hotel puede vender diez habitaciones del tipo Standard sin que el huésped elija un número de habitación concreto. En ese caso se administra inventario por tipo y fecha. Si el negocio requiere asignar una unidad física específica, el modelo debe controlar cada unidad individualmente.

### 3.4. Búsqueda (`search`)

El buscador debe permitir combinar:

- Destino: ciudad, región, país o coordenadas.
- Check-in y check-out.
- Número de habitaciones.
- Adultos y niños.
- Precio mínimo y máximo.
- Tipo de alojamiento.
- Servicios: Wi-Fi, pileta, estacionamiento, desayuno, etc.
- Calificación.
- Política de cancelación.
- Distancia a puntos de interés, si existe información geográfica suficiente.
- Ordenamiento por precio, popularidad, valoración o relevancia.

El endpoint de búsqueda debe devolver resultados que cumplan las fechas y ocupación solicitadas; no solamente propiedades que coincidan con el texto del destino.

### 3.5. Inventario y disponibilidad (`inventory`)

- Inventario disponible por tipo de habitación y fecha.
- Bloqueos manuales de fechas.
- Cierre temporal de ventas.
- Reservas pendientes y confirmadas.
- Liberación del inventario cuando una reserva temporal vence.
- Restricciones de estadía mínima y máxima.
- Restricciones de llegada y salida.
- Control de sobreventa únicamente si el negocio la habilita explícitamente.

Ejemplo conceptual de disponibilidad:

| Fecha | Standard | Deluxe | Suite |
|---|---:|---:|---:|
| 26/11 | 5 | 3 | 2 |
| 27/11 | 4 | 3 | 2 |
| 28/11 | 2 | 2 | 1 |
| 29/11 | 0 | 2 | 1 |

La disponibilidad debe ser consistente ante solicitudes concurrentes. Si queda una sola habitación y llegan cincuenta solicitudes al mismo tiempo, solamente una debe poder adquirirla.

### 3.6. Tarifas y precios (`pricing`)

- Precio por noche y tipo de habitación.
- Tarifas diferentes por fecha.
- Temporadas altas y bajas.
- Tarifas reembolsables y no reembolsables.
- Descuentos por cantidad de noches.
- Descuentos por anticipación.
- Planes con desayuno u otros servicios incluidos.
- Impuestos y cargos adicionales.
- Cupones y promociones.
- Moneda y reglas de redondeo.

Ejemplo de desglose de tarifa:

```json
{
  "room_type": "DELUXE",
  "nights": 3,
  "currency": "USD",
  "breakdown": {
    "nightly_rates": [120, 150, 135],
    "subtotal": 405,
    "taxes": 40.5,
    "discount": 20,
    "total": 425.5
  }
}
```

Al crear una reserva debe guardarse una **fotografía o snapshot del precio acordado**: precio por noche, impuestos, cargos, descuentos, moneda, plan tarifario y políticas relevantes. Una modificación posterior de las tarifas no debe cambiar el precio de una reserva ya confirmada.

### 3.7. Cotización y reservas (`reservations`)

Flujo esperado:

1. El viajero selecciona propiedad, tipo de habitación y fechas.
2. El backend valida ocupación, restricciones y disponibilidad.
3. El backend calcula una cotización con desglose.
4. El viajero solicita una reserva temporal (*hold*).
5. El sistema vuelve a comprobar el inventario y lo retiene de manera atómica.
6. Se crea una sesión o intento de pago.
7. El proveedor de pagos informa el resultado mediante un webhook verificado.
8. Si el pago y las reglas de la reserva se cumplen, la reserva se confirma.
9. Se envían comprobantes y notificaciones.
10. Si el hold vence o el pago no se completa, el sistema libera el inventario de forma segura.

Las reservas deben tener estados controlados y transiciones válidas. No se debe permitir que el cliente cambie arbitrariamente el estado enviando un `PATCH`.

Estados sugeridos:

- `PENDING_PAYMENT`: reserva temporal esperando el pago.
- `CONFIRMED`: reserva confirmada.
- `CANCELLED`: reserva cancelada.
- `EXPIRED`: reserva temporal vencida.
- `COMPLETED`: estadía finalizada.
- `NO_SHOW`: el huésped no se presentó.
- `REFUND_PENDING` / `PARTIALLY_REFUNDED` / `REFUNDED`: estados financieros adicionales, si se decide modelarlos dentro del ciclo de reserva.

Las transiciones deben definirse explícitamente. Por ejemplo, `PENDING_PAYMENT -> CONFIRMED` solo debería ocurrir después de validar el resultado del pago y las condiciones de negocio. `PENDING_PAYMENT -> EXPIRED` ocurre al vencer el hold si no se completó el flujo.

### 3.8. Pagos (`payments`)

- Adaptador para uno o más proveedores.
- Creación de checkout o intención de pago.
- Consulta del estado de una transacción.
- Confirmación mediante webhooks.
- Verificación de firma/autenticidad de webhooks.
- Registro de eventos recibidos.
- Reembolsos totales y parciales.
- Historial de transacciones.
- Idempotencia.
- Conciliación de pagos como funcionalidad avanzada.

Se puede comenzar con un entorno sandbox de Mercado Pago o Stripe. La confirmación del pago debe venir del proveedor y verificarse en el backend; nunca se debe confiar únicamente en una respuesta enviada por el frontend. No almacenar datos completos de tarjetas.

Interfaz conceptual para desacoplar el proveedor:

```python
from typing import Protocol


class PaymentProvider(Protocol):
    async def create_payment(self, *, amount: int, currency: str, reference: str): ...
    async def get_payment_status(self, provider_payment_id: str): ...
    async def refund_payment(self, provider_payment_id: str, *, amount: int | None = None): ...
    async def verify_webhook(self, headers: dict, payload: bytes) -> bool: ...
```

Los montos deben manejarse con precisión decimal o unidades menores de la moneda (por ejemplo, centavos), nunca con `float` para importes financieros.

### 3.9. Cancelaciones y políticas

- Política de cancelación asociada a una tarifa o reserva.
- Fecha límite para cancelar sin cargo.
- Penalidad configurable después del límite.
- Cancelación por parte del huésped o del propietario, con permisos y auditoría.
- Reembolso total o parcial, según las reglas.
- Liberación del inventario cuando corresponda.
- Registro del motivo y actor que ejecutó la cancelación.

La cancelación de la reserva y el reembolso son procesos relacionados, pero no idénticos. El estado de la reserva no debe implicar falsamente que el dinero ya fue devuelto.

### 3.10. Reseñas y reputación (`reviews`)

- Solo viajeros con una estadía elegible/verificada pueden reseñar.
- Calificación general y, opcionalmente, por categorías.
- Comentarios y fotografías.
- Respuesta del propietario.
- Reporte de contenido inapropiado.
- Moderación administrativa.
- Prevención de reseñas duplicadas para la misma estadía.

### 3.11. Notificaciones (`notifications`)

Eventos posibles:

| Evento | Acción |
|---|---|
| Reserva confirmada | Enviar comprobante al viajero y aviso al propietario |
| Reserva próxima | Enviar recordatorio antes del check-in |
| Pago rechazado | Informar al viajero |
| Reserva cancelada | Notificar a las partes y comunicar el estado del reembolso |
| Nueva reserva | Notificar al propietario |
| Hold vencido | Liberar inventario y registrar el resultado |
| Estadía finalizada | Invitar al viajero a dejar una reseña |

Los correos y tareas demoradas deben ejecutarse mediante workers, no dentro del ciclo HTTP principal.

### 3.12. Administración, auditoría y métricas (`admin`, `audit`)

- Panel de actividad general.
- Gestión de usuarios y propiedades.
- Supervisión de reservas y transacciones.
- Registro de cambios críticos.
- Registro de actor, acción, recurso, fecha y resultado.
- Reportes de incidentes y disputas.
- Métricas operativas y de negocio.

### 3.13. Indicadores comerciales

- **Occupancy Rate:** porcentaje de inventario disponible que se ocupó durante un período.
- **ADR (Average Daily Rate):** ingreso promedio por habitación vendida.
- **RevPAR (Revenue Per Available Room):** ingreso por habitación disponible.
- Ingresos brutos y netos.
- Tasa de cancelación.
- Tasa de conversión de búsqueda a reserva.
- Valor promedio de reserva.

Las fórmulas deben documentarse y definirse según el tipo de inventario que maneje cada propiedad.

---

## 4. Stack tecnológico recomendado

| Área | Tecnología | Responsabilidad |
|---|---|---|
| Lenguaje | Python | Lenguaje principal |
| Framework | FastAPI | API HTTP, routing, dependencias y OpenAPI |
| Validación | Pydantic | Validación y serialización de datos |
| ORM | SQLAlchemy 2.x | Persistencia y consultas |
| Migraciones | Alembic | Evolución del esquema de base de datos |
| Base de datos | PostgreSQL | Datos transaccionales y relaciones |
| Cache | Redis | Cache y datos temporales no críticos |
| Background jobs | Celery + Redis | Tareas asíncronas y programadas |
| Autenticación | JWT/OAuth2 Password Bearer, según el flujo | Autenticación y autorización |
| Hash de contraseñas | Argon2id o bcrypt | Almacenamiento seguro de contraseñas |
| Archivos | S3 compatible | Imágenes y documentos |
| Documentación | OpenAPI / Swagger UI de FastAPI | Contrato y documentación de la API |
| Testing | pytest | Tests unitarios, integración y API |
| Cliente HTTP para tests | HTTPX | Pruebas de endpoints ASGI |
| Infraestructura local | Docker / Docker Compose | Entorno reproducible |
| CI/CD | GitHub Actions | Calidad y despliegue automatizado |
| Logging | structlog o logging estructurado | Logs consultables |
| Observabilidad | OpenTelemetry + Prometheus + Grafana | Trazas, métricas y dashboards |
| Errores | Sentry, opcional | Seguimiento de excepciones |

### Decisiones importantes

- **PostgreSQL como fuente de verdad:** no usar Redis como garantía principal de inventario o reservas.
- **SQLAlchemy + Alembic:** mantener modelos, consultas y migraciones explícitas.
- **Redis como complemento:** cache, rate limiting y broker/backend de tareas; su caída no debe permitir sobreventa.
- **Celery para trabajos asíncronos:** expiración de holds, notificaciones, recordatorios y conciliaciones.
- **Monolito modular al inicio:** no dividir en microservicios antes de tener una necesidad operativa real.
- **FastAPI:** aprovechar los esquemas Pydantic y la documentación OpenAPI generada, manteniendo el contrato HTTP bajo control.

---

## 5. Arquitectura propuesta

### 5.1. Estilo arquitectónico

Comenzar con un **monolito modular** organizado por dominios y capas. Los límites entre módulos deben ser claros para que, si en el futuro aparece una necesidad real de escalado independiente, algunos módulos puedan extraerse sin reescribir todo el sistema.

Módulos sugeridos:

- `identity`
- `users`
- `properties`
- `rooms`
- `search`
- `inventory`
- `pricing`
- `reservations`
- `payments`
- `reviews`
- `notifications`
- `admin`
- `audit`

### 5.2. Estructura inicial de carpetas

```text
stayfinder/
├── app/
│   ├── main.py
│   ├── api/
│   │   └── v1/
│   │       ├── router.py
│   │       └── endpoints/
│   ├── core/
│   │   ├── config.py
│   │   ├── security.py
│   │   ├── logging.py
│   │   ├── errors.py
│   │   └── dependencies.py
│   ├── db/
│   │   ├── base.py
│   │   ├── session.py
│   │   └── migrations/
│   ├── modules/
│   │   ├── identity/
│   │   │   ├── router.py
│   │   │   ├── schemas.py
│   │   │   ├── models.py
│   │   │   ├── service.py
│   │   │   └── repository.py
│   │   ├── properties/
│   │   ├── rooms/
│   │   ├── search/
│   │   ├── inventory/
│   │   ├── pricing/
│   │   ├── reservations/
│   │   ├── payments/
│   │   ├── reviews/
│   │   ├── notifications/
│   │   ├── admin/
│   │   └── audit/
│   ├── workers/
│   │   ├── celery_app.py
│   │   └── tasks/
│   └── integrations/
│       ├── payments/
│       ├── email/
│       └── storage/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── api/
│   ├── e2e/
│   ├── factories/
│   └── conftest.py
├── alembic.ini
├── pyproject.toml
├── Dockerfile
├── compose.yaml
├── .env.example
└── README.md
```

Cada módulo puede seguir un patrón similar:

- `router.py`: endpoints HTTP y traducción entre HTTP y casos de uso.
- `schemas.py`: esquemas Pydantic de entrada y salida.
- `models.py`: modelos SQLAlchemy.
- `service.py`: reglas del negocio y orquestación.
- `repository.py`: consultas y persistencia.
- `permissions.py` o políticas equivalentes: autorización sobre recursos, si el módulo lo requiere.

No es necesario crear capas vacías para cada archivo desde el día uno. La separación debe aportar claridad y evitar duplicación, no convertirse en ceremonia.

### 5.3. Diagrama lógico

```text
                 WEB / MOBILE CLIENTS
                          |
                          v
                    FASTAPI APP
                          |
                AUTH / VALIDATION / RBAC
                          |
                          v
                    API ROUTERS
                          |
                          v
                  APPLICATION SERVICES
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
       IDENTITY         SEARCH         RESERVATIONS
          |               |               |
          +---------------+---------------+
                          |
                          v
                    REPOSITORIES
                          |
                 +--------+--------+
                 |                 |
                 v                 v
             PostgreSQL          Redis
                 |                 |
                 |             Celery Broker
                 |                 |
                 v                 v
       Transactional Data      Celery Workers
                                   |
                         +---------+---------+
                         |         |         |
                      Emails    Expiry    Payments
                                 Holds   / Webhooks
```

### 5.4. Reglas de arquitectura

- Los routers no deben contener reglas de negocio complejas.
- Los servicios deben coordinar los casos de uso.
- Los repositorios deben encapsular las consultas cuando eso mejore la claridad y testabilidad.
- Las transacciones críticas deben tener límites explícitos.
- Los módulos no deben modificar directamente las tablas de otros dominios sin una interfaz clara.
- Las llamadas a proveedores externos deben estar detrás de adaptadores.
- Las operaciones asíncronas deben ser idempotentes y observables.

---

## 6. Diseño REST y endpoints iniciales

Prefijo y versionado:

```http
/api/v1
```

### 6.1. Autenticación y usuarios

```http
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password

GET    /api/v1/users/me
PATCH  /api/v1/users/me
```

### 6.2. Propiedades y búsqueda

```http
GET    /api/v1/search
GET    /api/v1/properties
GET    /api/v1/properties/{property_id}
GET    /api/v1/properties/{property_id}/availability
GET    /api/v1/properties/{property_id}/rooms
GET    /api/v1/properties/{property_id}/reviews
```

Ejemplo de búsqueda:

```http
GET /api/v1/search?destination=Mendoza&check_in=2027-01-10&check_out=2027-01-14&adults=2&rooms=1&min_price=50&max_price=200&sort=price_asc&page=1&limit=20
```

### 6.3. Gestión del propietario

```http
POST   /api/v1/host/properties
PATCH  /api/v1/host/properties/{property_id}
POST   /api/v1/host/properties/{property_id}/rooms

GET    /api/v1/host/reservations
GET    /api/v1/host/calendar

PUT    /api/v1/host/inventory
PUT    /api/v1/host/pricing
```

### 6.4. Reservas

```http
POST   /api/v1/reservations/quote
POST   /api/v1/reservations/holds

GET    /api/v1/reservations/{reservation_id}
GET    /api/v1/users/me/reservations

POST   /api/v1/reservations/{reservation_id}/cancel
```

La confirmación definitiva debe ocurrir como resultado del flujo validado de pagos y reglas de negocio. Si se expone un endpoint de confirmación manual, debe estar restringido a actores y casos internos autorizados; no debe permitir que un cliente confirme una reserva sin evidencia de pago cuando el plan tarifario exige pago.

### 6.5. Pagos y webhooks

```http
POST   /api/v1/payments/checkout
GET    /api/v1/payments/{payment_id}
POST   /api/v1/webhooks/payments/{provider}
POST   /api/v1/payments/{payment_id}/refund
```

### 6.6. Reseñas

```http
POST   /api/v1/reservations/{reservation_id}/review
GET    /api/v1/properties/{property_id}/reviews
PATCH  /api/v1/reviews/{review_id}
DELETE /api/v1/reviews/{review_id}
POST   /api/v1/reviews/{review_id}/report
```

La autorización y elegibilidad deben comprobarse en el backend: una reseña solo puede publicarse si existe una estadía válida y el usuario tiene derecho a reseñarla.

### 6.7. Administración

```http
GET    /api/v1/admin/dashboard
GET    /api/v1/admin/users
GET    /api/v1/admin/properties
GET    /api/v1/admin/reservations
GET    /api/v1/admin/transactions
GET    /api/v1/admin/audit-logs
```

### 6.8. Operación y salud

```http
GET    /health
GET    /ready
```

- `/health`: comprueba que el proceso esté vivo.
- `/ready`: comprueba si la instancia puede atender tráfico, incluyendo dependencias esenciales configuradas.

No conviene exponer detalles internos sensibles en estas respuestas.

---

## 7. Reglas críticas del motor de reservas

Esta es la parte más importante del proyecto desde el punto de vista técnico.

### 7.1. Evitar doble reserva

No alcanza con consultar disponibilidad y después insertar una reserva en otra operación sin protección. Dos solicitudes concurrentes podrían leer el mismo último cupo y reservarlo simultáneamente.

La operación que retiene inventario debe ser atómica. Algunas opciones de diseño:

- Transacción de PostgreSQL con bloqueo de filas (`SELECT ... FOR UPDATE`) sobre los registros diarios de inventario.
- Actualización condicional que reduzca inventario solo si la cantidad disponible es suficiente.
- Restricciones de base de datos para evitar solapamientos cuando se asigna una unidad física concreta.
- Reintentos controlados ante conflictos transitorios.

Para inventario por tipo de habitación, una tabla diaria puede guardar capacidad, unidades reservadas, unidades retenidas y unidades bloqueadas. El sistema debe definir una única fórmula consistente para calcular disponibilidad.

### 7.2. Hold temporal

- El hold debe tener fecha de expiración.
- La retención de inventario y la creación del hold deben realizarse en una misma transacción.
- El worker de expiración debe procesar holds vencidos de forma idempotente.
- La liberación del inventario debe ocurrir una sola vez.
- Si un webhook de pago llega cerca de la expiración, el flujo debe resolver explícitamente la carrera entre pago y vencimiento.
- No depender exclusivamente de un TTL de Redis para liberar inventario: PostgreSQL debe conservar el estado autoritativo.

### 7.3. Idempotencia

Para operaciones críticas, aceptar una clave como:

```http
Idempotency-Key: 8c3b7f4d-...
```

El servidor debe guardar la clave, el ámbito de usuario/operación, el resultado y su estado. Repetir la misma solicitud con la misma clave debe devolver un resultado consistente, sin crear una segunda reserva o un segundo cobro.

Definir también el comportamiento cuando una misma clave se reutiliza con un payload diferente: normalmente debe rechazarse como conflicto.

### 7.4. Webhooks duplicados o fuera de orden

- Verificar la firma del proveedor.
- Guardar el identificador único del evento.
- Tener una restricción única para evitar procesarlo dos veces.
- Procesar eventos mediante una transición de estado válida.
- No asumir que los webhooks llegan una sola vez ni en el orden esperado.
- Consultar al proveedor si el evento recibido no permite determinar de manera segura el estado final.

### 7.5. Snapshot de reserva

La reserva debe conservar los datos relevantes acordados en el momento de compra: alojamiento, habitación/tipo, fechas, ocupación, tarifa, moneda, impuestos, descuentos, cargos, política de cancelación y datos mínimos necesarios para el comprobante. Las modificaciones futuras del catálogo no deben alterar el historial.

---

## 8. Seguridad y calidad de API

### Seguridad

- Hash de contraseñas con Argon2id o bcrypt.
- Access tokens de vida corta y refresh tokens rotados/revocables.
- Autorización basada en roles y permisos (RBAC) y validación de propiedad del recurso.
- Validación de entrada mediante Pydantic.
- Rate limiting por IP, usuario o credencial según el endpoint.
- CORS con orígenes explícitos.
- HTTPS en despliegues.
- Gestión segura de secretos y variables de entorno.
- Protección de endpoints administrativos.
- Verificación de webhooks.
- Límites de tamaño y tipo para archivos subidos.
- No registrar contraseñas, tokens, datos de tarjeta ni información sensible.
- Auditoría de operaciones críticas.
- Revisión de riesgos OWASP API Security Top 10.

### Contrato y consistencia HTTP

- Versionado de API.
- Paginación, filtros y ordenamiento.
- Esquemas de respuesta consistentes.
- Códigos HTTP correctos.
- Errores uniformes con código, mensaje y detalles seguros.
- Documentación OpenAPI.
- Compatibilidad y política de deprecación para cambios futuros.

Ejemplo de error:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Los datos enviados no son válidos",
    "details": [
      {
        "field": "check_out",
        "message": "Debe ser posterior a check_in"
      }
    ]
  },
  "request_id": "req_01..."
}
```

Los detalles internos, trazas de excepciones y consultas SQL no deben enviarse al cliente.

---

## 9. Cache, tareas asíncronas y resiliencia

### Redis

Usos potenciales:

- Cache de búsquedas y datos públicos con expiración corta.
- Rate limiting.
- Coordinación auxiliar de tareas.
- Broker/backend de Celery.
- Datos temporales que puedan reconstruirse.

La cache debe invalidarse o expirar de manera coherente cuando cambien inventario, precios o publicación de una propiedad. Los resultados cacheados nunca deben ser la garantía definitiva de una reserva.

### Celery

Workers y tareas sugeridas:

- Expirar holds vencidos.
- Enviar emails y notificaciones.
- Recordatorios de check-in.
- Solicitudes de reseña tras una estadía.
- Procesar imágenes o generar miniaturas.
- Conciliación de transacciones.
- Reportes y agregaciones no críticas.

Las tareas deben tolerar reintentos, duplicados y fallas parciales. Deben tener identificadores, logs y métricas. Para trabajos periódicos se puede utilizar Celery Beat.

### Resiliencia

- Timeouts en llamadas externas.
- Reintentos con backoff exponencial y jitter donde corresponda.
- Evitar reintentos ciegos de operaciones no idempotentes.
- Circuit breaker cuando sea apropiado.
- Manejo explícito de fallas parciales.
- Dead-letter o mecanismo de inspección para tareas que agoten reintentos.
- No confirmar reservas basándose en una dependencia que no haya sido verificada.

---

## 10. Testing con pytest

El testing no debe limitarse a comprobar que cada endpoint devuelve `200`. Debe verificar reglas de negocio, seguridad, consistencia y concurrencia.

### Herramientas

- `pytest`: runner y aserciones.
- `pytest-asyncio`: tests asíncronos cuando sean necesarios.
- `httpx`: cliente para probar la aplicación ASGI.
- `pytest-cov`: cobertura.
- `factory-boy` o factories propias: generación de datos de prueba.
- PostgreSQL de test o Testcontainers: pruebas de integración con un motor real.
- Redis de test: pruebas de cache/colas cuando corresponda.

### Capas de testing

**Unitarios (`tests/unit/`)**

- Reglas de precios.
- Cálculo de noches e impuestos.
- Políticas de cancelación.
- Transiciones de estados.
- Validaciones de ocupación.
- Cálculo de indicadores.

**Integración (`tests/integration/`)**

- Repositorios SQLAlchemy.
- Transacciones y bloqueos.
- Constraints e índices.
- Migraciones.
- Integración con Redis y Celery, cuando se prueben componentes reales.

**API (`tests/api/`)**

- Registro e inicio de sesión.
- Autorización por roles y recursos.
- Búsqueda y filtros.
- Creación de holds y reservas.
- Cancelaciones.
- Webhooks.
- Respuestas de error y validaciones.

**End-to-end (`tests/e2e/`)**

- Flujo completo de búsqueda, cotización, hold, pago sandbox, confirmación y cancelación.
- Flujo de expiración de hold.
- Flujo de estadía completada y reseña.

### Escenarios críticos obligatorios

| Escenario | Resultado esperado |
|---|---|
| 50 usuarios intentan reservar el último cupo | Solo se confirma una retención válida |
| El cliente repite la solicitud de pago | No se genera un segundo cobro |
| El proveedor envía dos veces el mismo webhook | El evento produce un único efecto |
| Los webhooks llegan fuera de orden | Se aplican solo transiciones válidas |
| Redis deja de funcionar | No se pierde la garantía transaccional del inventario |
| El proceso de pago se interrumpe | El hold queda pendiente y luego se resuelve sin inconsistencias |
| Un usuario consulta una reserva ajena | Se deniega el acceso según la política definida |
| Un propietario modifica los precios | No cambia reservas ya confirmadas |
| Un hold vence dos veces por reintentos del worker | El inventario se libera una sola vez |
| Dos procesos modifican el último cupo concurrentemente | La base de datos conserva la consistencia |
| Se cancela una reserva con penalidad | El importe y el inventario siguen la política configurada |

### Criterios de calidad

- Tests deterministas, sin depender de servicios externos reales.
- Proveedores externos simulados en unitarios y pruebas aisladas.
- Pruebas de integración con PostgreSQL real para transacciones críticas.
- Cobertura medida con `pytest-cov`, sin perseguir un porcentaje artificial a costa de tests inútiles.
- Pruebas de migraciones desde una base vacía y desde versiones anteriores.
- Tests explícitos de permisos y acceso a objetos.
- Pruebas de concurrencia para inventario y reservas.

Ejemplo de comando:

```bash
pytest
pytest tests/unit -q
pytest tests/integration -q
pytest --cov=app --cov-report=term-missing
```

Ejemplo básico de test de endpoint con FastAPI y HTTPX:

```python
import pytest
from httpx import ASGITransport, AsyncClient

from app.main import app


@pytest.mark.asyncio
async def test_health_endpoint():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/health")

    assert response.status_code == 200
    assert response.json()["status"] == "ok"
```

La configuración exacta de fixtures dependerá de cómo se gestionen la sesión de base de datos y las dependencias de FastAPI.

---

## 11. Observabilidad y operación

Una API de producción debe permitir diagnosticar qué ocurrió sin depender de reproducir manualmente cada incidente.

### Logs estructurados

Registrar, como mínimo:

- `request_id` o trace ID.
- Método y ruta.
- Código HTTP.
- Duración.
- Usuario o actor interno, sin exponer secretos.
- Módulo o caso de uso.
- Identificador de reserva o pago cuando corresponda.
- Resultado y clase de error.

### Métricas técnicas

- Solicitudes por segundo.
- Latencia p50, p95 y p99.
- Tasa de errores por endpoint.
- Conexiones y tiempos de PostgreSQL.
- Uso de CPU y memoria.
- Longitud de colas y edad del trabajo más antiguo.
- Fallas y reintentos de tareas.
- Disponibilidad de dependencias.

### Métricas de negocio

- Búsquedas realizadas.
- Cotizaciones generadas.
- Holds creados y expirados.
- Reservas confirmadas y canceladas.
- Tasa de conversión.
- Pagos aprobados y rechazados.
- Ocupación e ingresos.

### Herramientas

- OpenTelemetry para instrumentación y trazas.
- Prometheus para métricas.
- Grafana para dashboards.
- Sentry para seguimiento de excepciones, si se requiere.
- Logs estructurados con `structlog` o el módulo `logging` configurado en formato JSON.

### Endpoints operativos

- `/health`: proceso vivo.
- `/ready`: capacidad para atender tráfico y estado de dependencias requeridas.

No incluir secretos, URLs internas sensibles ni detalles de infraestructura en respuestas públicas.

---

## 12. Infraestructura y despliegue

### Desarrollo local

Usar Docker Compose para levantar:

- API FastAPI.
- PostgreSQL.
- Redis.
- Celery worker.
- Celery Beat, si hay tareas programadas.
- Herramientas opcionales de observabilidad.

### CI/CD con GitHub Actions

Pipeline sugerido en cada pull request:

1. Instalar dependencias.
2. Ejecutar lint y formateo/verificación de tipos.
3. Ejecutar tests unitarios.
4. Levantar servicios de integración y ejecutar tests de integración.
5. Medir cobertura.
6. Construir imagen Docker.
7. Ejecutar escaneo de dependencias e imagen.

En la rama principal, después de pasar los controles:

1. Construir y etiquetar imagen.
2. Publicar en registry.
3. Ejecutar migraciones de forma controlada.
4. Desplegar.
5. Verificar readiness y smoke tests.
6. Tener estrategia de rollback.

### Producción

- HTTPS obligatorio.
- Secretos mediante un gestor seguro, no dentro del repositorio.
- PostgreSQL con backups automatizados y pruebas periódicas de restauración.
- Redis configurado según su rol y tolerancia a fallas.
- Workers separados de los procesos HTTP.
- Límites de recursos y escalado medido.
- Migraciones compatibles con despliegues graduales cuando sea necesario.
- Monitoreo y alertas.
- Plan documentado de recuperación ante incidentes.

---

## 13. Roadmap de desarrollo

### Versión 1 — Core Booking

Objetivo: producto mínimo funcional con un flujo de reserva coherente.

- Configuración de FastAPI y estructura modular.
- PostgreSQL, SQLAlchemy y Alembic.
- Registro, login y permisos básicos.
- Gestión de propiedades.
- Gestión de tipos de habitación.
- Búsqueda con filtros.
- Inventario y calendario de disponibilidad.
- Cotización de reservas.
- Holds temporales.
- Prevención de doble reserva.
- Integración de pago sandbox.
- Webhook de pago.
- Confirmación y cancelación.
- Tests con pytest para los flujos principales.
- Docker Compose y documentación OpenAPI.

### Versión 2 — Advanced Booking

- Tarifas por fecha y temporada.
- Políticas de cancelación configurables.
- Cupones y promociones.
- Reseñas verificadas.
- Redis y cache selectiva.
- Celery y workers.
- Notificaciones por email.
- Expiración automática de holds.
- Reembolsos.
- Panel/API del propietario.
- Auditoría de operaciones críticas.
- Tests de integración y concurrencia más completos.

### Versión 3 — Production & Business

- Métricas de ocupación, ADR y RevPAR.
- Comisiones de plataforma.
- Liquidaciones a propietarios, si el modelo comercial las requiere.
- Multi-moneda con reglas explícitas de conversión y redondeo.
- Multi-idioma.
- Integraciones externas.
- Observabilidad con OpenTelemetry, Prometheus y Grafana.
- Pruebas de carga.
- CI/CD y despliegue automatizado.
- Backups y recuperación probada.
- Políticas de seguridad y operación documentadas.

### Versión 4 — Enterprise / Escalabilidad

Solo cuando las métricas y necesidades del negocio lo justifiquen:

- Arquitectura orientada a eventos.
- Bus de mensajes dedicado.
- Extracción selectiva de servicios.
- Motor de búsqueda especializado.
- Recomendaciones personalizadas.
- Integración con channel managers.
- Alta disponibilidad y estrategias avanzadas de resiliencia.
- Separación de cargas de lectura y escritura, si hace falta.

---

## 14. Diferenciación comercial: alojamientos independientes

Una posible especialización es ofrecer una plataforma para hoteles boutique, hostels, cabañas, departamentos y alojamientos independientes.

Además de permitir reservar, StayFinder puede ofrecer a los propietarios:

- Gestión centralizada de reservas.
- Calendario de ocupación.
- Administración de múltiples propiedades.
- Tarifas configurables.
- Bloqueos de disponibilidad.
- Panel de ingresos.
- Reportes por temporada.
- Comparación del rendimiento entre propiedades.
- Gestión de personal y permisos.

Esto le da al producto una propuesta de valor más clara que simplemente copiar la experiencia pública de Booking.com.

---

## 15. Checklist de aceptación de nivel producción

La plataforma no se considera lista para producción solo porque los endpoints funcionen. La siguiente checklist sirve como criterio de aceptación y debe adaptarse al riesgo y alcance del despliegue.

### Arquitectura y API

- [ ] Arquitectura modular y responsabilidades separadas.
- [ ] Versionado de API.
- [ ] Validación de entrada en todos los endpoints relevantes.
- [ ] Manejo global y uniforme de errores.
- [ ] Paginación, filtros y ordenamiento.
- [ ] Documentación OpenAPI actualizada.
- [ ] Política de compatibilidad y deprecación.

### Seguridad

- [ ] Hash seguro de contraseñas.
- [ ] Access y refresh tokens con expiración y estrategia de revocación.
- [ ] Autorización por rol y por recurso.
- [ ] Rate limiting.
- [ ] CORS y security headers configurados.
- [ ] Secretos fuera del repositorio.
- [ ] Webhooks autenticados/verificados.
- [ ] Auditoría de operaciones críticas.
- [ ] Revisión de riesgos OWASP API Security.

### Datos y negocio

- [ ] Migraciones reproducibles.
- [ ] Restricciones e índices adecuados.
- [ ] Transacciones para operaciones críticas.
- [ ] Integridad del inventario bajo concurrencia.
- [ ] Prevención de doble reserva.
- [ ] Idempotencia en pagos y operaciones críticas.
- [ ] Snapshot de precios y políticas en cada reserva.
- [ ] Cancelaciones y reembolsos consistentes.
- [ ] Backups y restauración probada.

### Procesamiento y resiliencia

- [ ] Cache con estrategia de expiración e invalidación.
- [ ] Workers para procesos asíncronos.
- [ ] Jobs idempotentes.
- [ ] Reintentos y timeouts definidos.
- [ ] Manejo de fallas de Redis y proveedores externos.
- [ ] Holds vencidos liberados de forma confiable.

### Testing

- [ ] Tests unitarios con pytest.
- [ ] Tests de integración con PostgreSQL.
- [ ] Tests de API.
- [ ] Tests end-to-end de reserva y pago.
- [ ] Tests de permisos y acceso a objetos.
- [ ] Tests de concurrencia del inventario.
- [ ] Tests de webhooks duplicados y fuera de orden.
- [ ] Tests de migraciones.
- [ ] Cobertura medida y revisada.
- [ ] Pruebas de carga y rendimiento.

### Operación

- [ ] Logs estructurados y correlación de requests.
- [ ] Métricas técnicas y de negocio.
- [ ] Trazas distribuidas donde corresponda.
- [ ] Health y readiness checks.
- [ ] Alertas operativas.
- [ ] Docker e infraestructura reproducible.
- [ ] CI/CD con controles de calidad.
- [ ] Estrategia de rollback.
- [ ] Plan de incidentes y recuperación.

---

## 16. Orden recomendado para empezar

Para evitar diseñar demasiadas cosas antes de validar el núcleo, seguir este orden:

1. **Definir el modelo de dominio:** propiedades, tipos de habitación, inventario diario, tarifas, reservas, pagos y usuarios.
2. **Diseñar el esquema PostgreSQL:** claves, relaciones, restricciones, índices y migraciones iniciales.
3. **Crear el esqueleto FastAPI:** configuración, routers versionados, dependencias, errores y documentación.
4. **Implementar identidad y permisos:** usuarios, roles y autorización por propiedad/recurso.
5. **Implementar propiedades y habitaciones:** alta, edición, publicación e imágenes.
6. **Implementar inventario y disponibilidad:** primero reglas y transacciones; luego optimizaciones.
7. **Implementar cotización y holds:** controlar concurrencia y expiración.
8. **Integrar pagos sandbox:** checkout, webhooks e idempotencia.
9. **Completar cancelaciones y reembolsos:** aplicar políticas y actualizar inventario.
10. **Agregar Celery, Redis y notificaciones:** trabajos asíncronos y expiraciones.
11. **Completar pytest:** unitarios, integración, API, E2E y concurrencia.
12. **Preparar producción:** observabilidad, Docker, CI/CD, backups, seguridad y pruebas de carga.

## 17. Definición de terminado

Una primera versión sólida de StayFinder debería permitir demostrar el siguiente recorrido de punta a punta:

1. Un propietario registra una propiedad y configura sus habitaciones, inventario y tarifas.
2. Un viajero busca un destino para un rango de fechas.
3. El sistema devuelve alojamientos disponibles con precios calculados.
4. El viajero solicita una cotización.
5. El backend crea un hold transaccional sin permitir sobreventa.
6. El viajero inicia el pago en el entorno sandbox.
7. El proveedor notifica el resultado mediante un webhook verificado.
8. El backend confirma la reserva de forma idempotente.
9. Se notifican el viajero y el propietario.
10. El viajero puede consultar o cancelar la reserva según la política.
11. El propietario ve el cambio reflejado en su calendario e inventario.
12. Los tests con pytest verifican tanto el camino exitoso como los errores, duplicados y solicitudes concurrentes.

**Criterio principal:** el sistema debe mantener la consistencia de las reservas y del inventario incluso ante solicitudes simultáneas, reintentos, webhooks duplicados y fallas de dependencias. Esa es la diferencia entre una demo CRUD y un backend de reservas serio.

---

## 18. Recursos oficiales para consultar durante el desarrollo

- FastAPI: <https://fastapi.tiangolo.com/>
- Pydantic: <https://docs.pydantic.dev/>
- SQLAlchemy: <https://docs.sqlalchemy.org/>
- Alembic: <https://alembic.sqlalchemy.org/>
- PostgreSQL: <https://www.postgresql.org/docs/>
- Redis: <https://redis.io/docs/>
- Celery: <https://docs.celeryq.dev/>
- pytest: <https://docs.pytest.org/>
- HTTPX: <https://www.python-httpx.org/>
- OpenTelemetry: <https://opentelemetry.io/docs/>
- OWASP API Security Top 10: <https://owasp.org/www-project-api-security/>

