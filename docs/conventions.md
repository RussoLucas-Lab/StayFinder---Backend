# Convenciones

## Idioma

| Qué | Idioma |
|---|---|
| Código, identificadores, nombres de tablas y columnas | Inglés |
| Mensajes de commit y títulos de PR | Inglés |
| Docs, ADRs y artefactos de OpenSpec | Español (con las palabras clave de OpenSpec en inglés, ver [workflow.md](workflow.md#formato-de-specs)) |
| `code` de error de la API | Inglés, `UPPER_SNAKE_CASE` |
| `message` de error de la API | Español (multi-idioma recién en la versión 3) |

## Python

- Python 3.12 o superior, con type hints en todo el código público.
- Gestión de dependencias, linter, formateador y verificador de tipos: se fijan en el change de bootstrap y se documentan acá. Valores sugeridos: `uv`, `ruff` (lint + formato) y `mypy`.
- Configuración por variables de entorno con Pydantic Settings; `.env.example` documenta cada variable sin valores reales.
- Sin lógica en `__init__.py`.

## API

- Prefijo `/api/v1`. Los cambios incompatibles van a una versión nueva o se anuncian con una política de deprecación.
- JSON en `snake_case`.
- Fechas de estadía en `YYYY-MM-DD`; instantes en ISO 8601 UTC con `Z`.
- Paginación con `page` y `limit` (máximo 100); la respuesta incluye `total`, `page` y `limit`.
- Orden con `sort=<campo>_<asc|desc>`, por ejemplo `sort=price_asc`.
- Operaciones críticas (crear hold, crear checkout, reembolsar) aceptan el header `Idempotency-Key`. Repetir la clave con el mismo payload devuelve el mismo resultado; con otro payload devuelve `409 Conflict`.
- IDs públicos no secuenciales (UUID) para no permitir enumerar recursos.
- Los códigos HTTP se usan según su significado: `201` al crear, `404` cuando el recurso no existe **o el usuario no tiene permiso para saber que existe**, `409` para conflictos de estado o inventario, `422` para validación.

### Errores

Toda respuesta de error usa este formato:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Los datos enviados no son válidos",
    "details": [
      { "field": "check_out", "message": "Debe ser posterior a check_in" }
    ]
  },
  "request_id": "req_01..."
}
```

Nunca se envían al cliente trazas, consultas SQL ni detalles de infraestructura.

## Base de datos

- Tablas en plural y `snake_case`; columnas en `snake_case`.
- Clave primaria UUID.
- `created_at` y `updated_at` en `timestamptz` (UTC) en todas las tablas de negocio.
- Las invariantes se garantizan en la base (restricciones `CHECK`, `UNIQUE`, claves foráneas) además de en la aplicación.
- Una migración de Alembic por change que modifique el esquema, con `downgrade` siempre que sea posible.
- Las migraciones se prueban desde una base vacía en CI.

## Dinero

Según [ADR-0005](adr/0005-montos-en-unidades-menores.md) (propuesto): nunca `float`; la moneda siempre acompaña al importe.

## Testing

- Capas: `tests/unit` (reglas puras), `tests/integration` (repositorios, transacciones, constraints, migraciones con PostgreSQL real), `tests/api` (endpoints con HTTPX), `tests/e2e` (flujos completos).
- **Cada escenario de una spec tiene al menos un test.** El docstring del test nombra la spec y el escenario, por ejemplo: `"""spec: reservations/holds — Scenario: last unit under concurrency"""`.
- Tests deterministas: sin servicios externos reales; el reloj y los IDs se inyectan cuando el resultado depende de ellos.
- Inventario, holds, pagos y webhooks requieren tests de concurrencia o de duplicados, según corresponda (ver la lista de escenarios críticos en [reference/plan-inicial.md](reference/plan-inicial.md#escenarios-críticos-obligatorios)).
- La cobertura se mide (`pytest-cov`) y se revisa; no se persigue un porcentaje con tests vacíos.

## Git

- Rama principal `main`, siempre desplegable. Una rama por change: `change/<change-id>`.
- [Conventional Commits](https://www.conventionalcommits.org/) en inglés, con el change en el scope cuando aplica: `feat(holds): retain inventory atomically`.
- Un PR por change. El PR enlaza su carpeta en `openspec/changes/`.

## Seguridad

- Ningún secreto en el repositorio; `.env` en `.gitignore`.
- Los logs nunca incluyen contraseñas, tokens, datos de tarjeta ni el cuerpo completo de webhooks.
- CORS con orígenes explícitos (el frontend vive en otro dominio, [ADR-0004](adr/0004-repos-separados-front-back.md)).
- Archivos subidos con límite de tamaño y tipo validado.
