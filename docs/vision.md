# Visión

## Qué es

StayFinder es una plataforma de reservas de alojamientos inspirada en Booking.com, enfocada en **alojamientos independientes y hoteles boutique**: hostels, cabañas, departamentos y hoteles chicos. Combina un motor de reservas para viajeros con herramientas de gestión para propietarios.

Este repositorio es el **backend** (API REST). El frontend vive en un repositorio separado ([ADR-0004](adr/0004-repos-separados-front-back.md)).

## Objetivo del proyecto

No es un CRUD de hoteles. El objetivo es un backend con reglas de negocio reales y consistentes bajo carga:

- disponibilidad por fecha con inventario limitado;
- reservas temporales (holds) que vencen;
- prevención de doble reserva bajo concurrencia;
- precios con snapshot inmutable;
- pagos confirmados por webhook verificado, con idempotencia;
- cancelaciones y reembolsos según políticas;
- auditoría y observabilidad.

**Criterio principal:** el sistema mantiene la consistencia de reservas e inventario ante solicitudes simultáneas, reintentos, webhooks duplicados y caídas de dependencias.

## Usuarios

| Rol | Qué hace |
|---|---|
| **Traveler** (viajero) | Busca, compara, reserva, paga, cancela según la política y reseña estadías verificadas |
| **Host** (propietario) | Publica propiedades, gestiona habitaciones, tarifas, inventario y reservas; invita empleados con permisos limitados |
| **Admin** (plataforma) | Modera propiedades y reseñas; supervisa reservas, pagos, disputas y auditoría |

## Alcance

**Incluido:** alojamientos (hoteles, hostels, cabañas, departamentos).

**Fuera de alcance (no-objetivos):**

- Vuelos y otros productos de viaje.
- Microservicios antes de que haya una necesidad operativa medida.
- Integración con channel managers y motores de búsqueda especializados (versión 4, si se justifica).
- Multi-moneda y multi-idioma en la versión 1.
- Almacenar datos de tarjeta: el proveedor de pagos los gestiona.

## Definición de terminado (versión 1)

La versión 1 está terminada cuando se puede demostrar este recorrido de punta a punta, con tests que cubren tanto el camino feliz como los errores, duplicados y solicitudes concurrentes:

1. Un propietario registra una propiedad y configura habitaciones, inventario y tarifas.
2. Un viajero busca un destino para un rango de fechas.
3. El sistema devuelve alojamientos disponibles con precios calculados.
4. El viajero pide una cotización.
5. El backend crea un hold transaccional sin permitir sobreventa.
6. El viajero inicia el pago en el sandbox del proveedor.
7. El proveedor notifica el resultado con un webhook verificado.
8. El backend confirma la reserva de forma idempotente.
9. Se notifica al viajero y al propietario.
10. El viajero consulta o cancela la reserva según la política.
11. El propietario ve el cambio reflejado en su calendario e inventario.
