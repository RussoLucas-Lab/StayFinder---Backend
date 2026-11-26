# ADR-0006: Inventario por tipo de habitación y fecha

- **Estado:** Propuesto
- **Fecha:** 2026-11-26

## Contexto

El plan inicial deja abierto si se vende por **tipo de habitación** ("10 Standard", sin número) o por **unidad física** ("habitación 204"). La elección define el modelo de inventario, la consulta de disponibilidad de la búsqueda y la técnica para evitar la doble reserva.

## Decisión

Se vende por tipo de habitación. El inventario es una tabla diaria por tipo:

```text
inventory_days(room_type_id, date, capacity, booked, held, blocked)
PRIMARY KEY (room_type_id, date)
CHECK (booked >= 0 AND held >= 0 AND blocked >= 0)
CHECK (booked + held + blocked <= capacity)
```

- Retener un hold incrementa `held` en todas las noches dentro de una transacción. Si alguna noche no tiene cupo, se revierte todo.
- La restricción `CHECK` es la última red de seguridad: aunque la aplicación se equivoque, la base rechaza la sobreventa.
- La técnica concreta (actualización condicional o `SELECT ... FOR UPDATE` en orden de fecha) se decide en el `design.md` del change de holds, con un test de concurrencia que la valide.
- Un alojamiento de una sola unidad (cabaña, departamento) es un tipo con `capacity = 1`.

## Alternativas consideradas

- **Por unidad física, con rangos de fechas y restricción de exclusión** (`daterange` + `btree_gist`). Exacta para asignar habitaciones concretas, pero obliga a elegir la unidad al reservar y hace más costosa la búsqueda "hay al menos N Standard libres". Queda como posible extensión operativa (asignar la unidad al hacer check-in) sin cambiar el modelo de venta.
- **Disponibilidad calculada en el momento** contando reservas que se solapan, sin contadores. Sin datos duplicados, pero exige aislamiento `SERIALIZABLE` o locks amplios, y la búsqueda se vuelve más lenta.

## Consecuencias

- La búsqueda por fechas consulta una sola tabla con índice por `(room_type_id, date)`.
- `held` solo vuelve a cero si algo libera los holds vencidos. La versión 1 necesita un mecanismo de liberación aunque Celery llegue en la versión 2 (ver [roadmap.md](../roadmap.md#puntos-que-el-change-map-tiene-que-resolver)).
- Hay que generar filas de inventario para las fechas futuras de cada tipo (al crear el tipo o con un horizonte móvil).
- Los bloqueos manuales del propietario incrementan `blocked` y respetan la misma restricción.
