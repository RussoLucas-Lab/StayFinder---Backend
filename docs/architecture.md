# Arquitectura

## Estilo

**Monolito modular** organizado por dominios ([ADR-0001](adr/0001-monolito-modular.md)). Un solo proceso de API, un solo esquema de base de datos y límites claros entre módulos, para que extraer un módulo en el futuro no obligue a reescribir el sistema.

## Stack

| Área | Tecnología |
|---|---|
| Lenguaje | Python 3.12 o superior |
| API | FastAPI + Pydantic v2 |
| Persistencia | SQLAlchemy 2.x + Alembic |
| Base de datos | PostgreSQL: fuente de verdad ([ADR-0002](adr/0002-postgresql-fuente-de-verdad.md)) |
| Cache, rate limiting, broker | Redis |
| Tareas asíncronas | Celery + Celery Beat (desde la versión 2) |
| Autenticación | OAuth2 Password Bearer + JWT (access de vida corta, refresh rotado y revocable) |
| Contraseñas | Argon2id |
| Archivos | Almacenamiento compatible con S3 |
| Pagos | Adaptador `PaymentProvider` ([ADR-0008](adr/0008-proveedor-de-pagos-sandbox.md), propuesto) |
| Tests | pytest, pytest-asyncio, HTTPX, PostgreSQL real para integración |
| Entorno local | Docker Compose; la aplicación corre en contenedores en desarrollo y despliegue ([ADR-0010](adr/0010-ejecucion-en-contenedores.md)) |
| CI | GitHub Actions |
| Observabilidad | Logs estructurados en JSON; OpenTelemetry, Prometheus y Grafana en la versión 3 |

El modo de acceso a la base (async o sync) lo fija [ADR-0009](adr/0009-acceso-async-a-base-de-datos.md) (propuesto).

## Servicios de Compose

Todo corre en contenedores ([ADR-0010](adr/0010-ejecucion-en-contenedores.md)).

| Versión | Servicios |
|---|---|
| 1 | `api` (FastAPI), `db` (PostgreSQL), `hold-sweeper` (corre `expire-holds` periódicamente; cada minuto por defecto, configurable) |
| 2 | Se suman `redis`, `worker` y `beat` (Celery). `beat` reemplaza a `hold-sweeper` e invoca el mismo servicio de expiración. |

Los tests de integración usan una base PostgreSQL de Compose separada de la de desarrollo.

## Módulos

| Módulo | Responsabilidad |
|---|---|
| `identity` | Registro, login, tokens, recuperación de contraseña, roles |
| `users` | Perfil del usuario |
| `properties` | Propiedades, servicios, reglas, imágenes, estado de publicación, staff |
| `rooms` | Tipos de habitación (y unidades físicas si se necesitan) |
| `inventory` | Inventario diario, bloqueos, restricciones, retención y liberación |
| `pricing` | Tarifas, planes, impuestos, descuentos, cálculo de cotizaciones |
| `search` | Búsqueda por destino, fechas, ocupación y filtros |
| `reservations` | Holds, reservas, snapshot, transiciones de estado, cancelaciones |
| `payments` | Pagos, reembolsos, webhooks, adaptadores de proveedores |
| `reviews` | Reseñas, respuestas, reportes |
| `notifications` | Emails y avisos (vía workers) |
| `admin` | Endpoints de supervisión y moderación |
| `audit` | Registro de acciones críticas |

Dependencias esperadas (quién llama a quién):

```text
search ──────────▶ properties, rooms, inventory, pricing
reservations ────▶ inventory, pricing, payments, audit
payments ────────▶ reservations (solo para notificar resultados), audit
reviews ─────────▶ reservations
notifications ◀── eventos de reservations, payments
admin ───────────▶ todos (solo lectura y operaciones de moderación)
```

Si un change necesita una dependencia que no está en este diagrama, la propone en su `design.md`.

## Estructura de carpetas

```text
stayfinder-backend/
├── app/
│   ├── main.py                 # crea la app y monta routers
│   ├── api/
│   │   └── v1/router.py        # solo agrega los routers de los módulos
│   ├── core/                   # config, security, logging, errors, dependencies
│   ├── db/                     # base, session
│   ├── modules/
│   │   └── <modulo>/
│   │       ├── router.py       # HTTP ↔ casos de uso
│   │       ├── schemas.py      # Pydantic de entrada y salida
│   │       ├── models.py       # modelos SQLAlchemy
│   │       ├── service.py      # reglas de negocio y orquestación
│   │       ├── repository.py   # consultas y persistencia
│   │       └── permissions.py  # autorización por recurso, si hace falta
│   ├── workers/                # celery_app.py, tasks/
│   └── integrations/           # payments/, email/, storage/
├── migrations/                 # Alembic
├── tests/                      # unit/, integration/, api/, e2e/, factories/
├── docs/
├── openspec/
├── compose.yaml
├── Dockerfile
├── pyproject.toml
└── .env.example
```

Los routers viven en cada módulo; `app/api/v1/router.py` solo los agrega. No se crean archivos vacíos "por si acaso": cada archivo aparece cuando tiene contenido.

## Reglas

1. **Routers delgados.** Validan entrada, resuelven dependencias (usuario actual, sesión), llaman a un servicio y traducen el resultado a HTTP. Sin reglas de negocio.
2. **Servicios coordinan.** Cada caso de uso es una función o método de servicio con límites de transacción explícitos.
3. **Repositorios encapsulan consultas** cuando eso mejora la claridad o la testabilidad; no son obligatorios para consultas triviales.
4. **Sin acceso cruzado a tablas.** Un módulo no lee ni escribe tablas de otro módulo directamente: usa su servicio. La excepción son las consultas de solo lectura de `search` y `admin`, que se documentan en el `design.md` del change que las introduce.
5. **Proveedores externos detrás de adaptadores** (`integrations/`), con timeouts y sin filtrar tipos del proveedor al dominio.
6. **Tareas asíncronas idempotentes y observables:** toleran reintentos y duplicados, y registran su resultado.
7. **Errores de dominio, no HTTP, en los servicios.** Los servicios lanzan excepciones de dominio; un manejador global las traduce al formato de error de la API ([conventions.md](conventions.md#errores)).

## Transacciones y concurrencia

- Las operaciones que escriben inventario, reservas o pagos definen su transacción en el servicio, no en el router.
- La estrategia anti doble reserva (bloqueo de filas, actualización condicional, restricciones) se detalla en el `design.md` del change de holds, a partir de [ADR-0006](adr/0006-inventario-por-tipo-de-habitacion.md).
- Cuando una transacción bloquea varias filas de inventario, las bloquea siempre en el mismo orden (por fecha) para evitar deadlocks.
- Ante conflictos transitorios (deadlock, serialization failure) se reintenta de forma acotada; nunca se reintenta a ciegas una llamada externa no idempotente.

## Operación

- `/health`: el proceso está vivo. `/ready`: puede atender tráfico (base de datos accesible y, si están configuradas, las dependencias esenciales). Ninguno expone detalles internos.
- Cada request lleva un `request_id` que aparece en los logs y en las respuestas de error.
