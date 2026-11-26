# ADR-0009: Acceso async a la base de datos

- **Estado:** Propuesto
- **Fecha:** 2026-11-26

## Contexto

FastAPI admite endpoints async y sync. SQLAlchemy 2.x tiene API sync y async. Celery ejecuta tareas sync. La interfaz `PaymentProvider` del plan inicial es async y los tests de ejemplo usan `pytest-asyncio`. Esta decisión se toma en el change de bootstrap porque cambiarla después afecta a todos los repositorios y servicios.

## Decisión

- La API usa **SQLAlchemy asyncio con `asyncpg`**: sesiones `AsyncSession`, endpoints y servicios `async`.
- Las relaciones se cargan de forma explícita (`selectinload`, `joinedload`). La carga perezosa queda deshabilitada (`lazy="raise"`) para que un acceso implícito falle en los tests y no en producción.
- Las tareas de Celery reutilizan los mismos servicios async ejecutándolos con `asyncio.run(...)`, con un engine creado dentro de ese event loop (o `NullPool`), porque las conexiones de `asyncpg` quedan atadas a su loop.

## Alternativas consideradas

- **Todo sync** (endpoints `def`, `Session`, `psycopg`). Más simple de razonar y nativo para Celery; FastAPI ejecuta los endpoints sync en un pool de hilos. Es la alternativa real si se prioriza la simplicidad.
- **API async y repositorios sync aparte para los workers.** Descartado: duplica la capa de persistencia.
- **Reemplazar Celery por una cola async** (por ejemplo arq o taskiq). Elimina el problema del event loop, pero cambia el stack elegido en el plan inicial. Se puede reevaluar en la versión 2.

## Consecuencias

- Las llamadas externas (proveedor de pagos, email) no bloquean el proceso de la API.
- Hay que aprender y respetar las reglas de SQLAlchemy async (sin lazy loading, una sesión por tarea).
- Los tests de integración y de API son async.
