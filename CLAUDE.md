# StayFinder Backend

API de reservas de alojamientos (estilo Booking) con FastAPI y PostgreSQL. Este repositorio es solo el backend; el frontend vive en otro repositorio.

## Al empezar cada sesión

1. Recuperá el contexto de Engram (`mem_context`).
2. Revisá `docs/change-map.md` y el change activo en `openspec/changes/`, si hay uno.
3. Resumí en dos o tres líneas en qué quedó el trabajo y qué sigue, y esperá confirmación.

Abrí el doc específico de `docs/` que la tarea necesite; el índice está en `docs/README.md`.

## Fuentes de verdad

Si dos fuentes se contradicen, manda la de más arriba:

1. `openspec/specs/`
2. `docs/adr/` con estado **Aceptado**
3. El resto de `docs/`
4. `docs/reference/plan-inicial.md` (insumo, no normativo)
5. Engram (para recordar, nunca para decidir)

- Si el código contradice una spec, es un bug o falta un change. No ajustes una spec al código sin preguntar.
- Un ADR **Propuesto** no es vinculante: si la tarea depende de él, preguntá antes de implementar.
- Si encontrás una contradicción entre fuentes, avisá; no la resuelvas en silencio.

## Flujo de trabajo

- Todo cambio de comportamiento pasa por OpenSpec (`/opsx:propose` → `/opsx:apply` → `/opsx:archive`). Detalle en `docs/workflow.md`.
- Un change activo por vez, en el orden de `docs/change-map.md`.
- Arreglos que restauran comportamiento ya especificado, typos y formato: directo, sin change.
- Specs en español con la estructura de OpenSpec en inglés (`### Requirement:`, `#### Scenario:`, `SHALL`/`MUST`, `**WHEN**`/`**THEN**`).
- Antes de dar por terminado un artefacto: `openspec validate <change-id> --strict`.

## Cómo trabajamos

Lucas quiere programar a la par del agente, no delegarle todo:

- Antes de codear, explicitá supuestos y preguntá si algo es ambiguo.
- Las decisiones de diseño (esquema de tablas, límites de transacción, dependencias nuevas) se proponen y esperan confirmación.
- Cambios mínimos y acotados a la tarea; nada especulativo.
- Explicá en dos o tres líneas el porqué de lo que no es obvio (locks, idempotencia, concurrencia).

## Reglas no negociables

- PostgreSQL es la fuente de verdad de inventario y reservas; Redis nunca garantiza disponibilidad.
- Retener inventario y crear el hold ocurren en la misma transacción.
- Nunca `float` para dinero.
- Un pago se confirma solo con webhook verificado o consulta al proveedor, nunca por lo que diga el cliente.
- Autorización por recurso en cada operación, no solo autenticación.
- El estado de una reserva cambia solo por transiciones válidas ejecutadas por el backend.
- Nada de secretos, tokens ni datos de tarjeta en código, logs o Engram.
- Los comandos del proyecto (tests, migraciones, lint) se ejecutan dentro del contenedor, no en el host (ADR-0010).

## Engram

- Guardá decisiones de la sesión, descubrimientos, bugs resueltos con su causa y el estado al cerrar.
- No guardes lo que ya está en `docs/` u `openspec/` (guardá la ruta), ni secretos.
- Si una decisión tiene que sobrevivir, proponé un ADR: nada importante vive solo en Engram.
- Al terminar la sesión: `mem_session_summary` con objetivo, descubrimientos, lo hecho, próximos pasos y archivos tocados.

## Comandos

Se definen con `docker compose` en el change de bootstrap (levantar el entorno, correr tests, lint, migraciones).
