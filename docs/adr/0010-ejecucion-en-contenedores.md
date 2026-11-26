# ADR-0010: Ejecución en contenedores

- **Estado:** Aceptado
- **Fecha:** 2026-11-26

## Contexto

El sistema depende de varios procesos (API, PostgreSQL y, desde la versión 2, Redis y workers de Celery). Si cada máquina instala sus propias versiones de Python, dependencias y servicios, el entorno deja de ser reproducible y aparecen diferencias entre desarrollo, tests y despliegue.

## Decisión

La aplicación corre en contenedores Docker, tanto en desarrollo como en despliegue.

- **Desarrollo:** Docker Compose levanta los servicios de la aplicación; `api` corre uvicorn con recarga y el código montado como volumen. La lista de servicios por versión está en [architecture.md](../architecture.md#servicios-de-compose).
- **Comandos del proyecto:** todos corren dentro del contenedor: tests, migraciones y lint (por ejemplo `docker compose run --rm api pytest`). En el host solo hacen falta Git, Docker y Node (para OpenSpec).
- **Tests de integración:** usan una base PostgreSQL de Compose separada de la de desarrollo.
- **Despliegue:** la misma imagen, con un Dockerfile multi-stage y un usuario no root.

## Alternativas consideradas

- **Correr la API en el host con solo la base en Docker.** Descartado: el entorno pasa a depender de cada máquina (versión de Python, dependencias, sistema operativo).

## Consecuencias

- El entorno es reproducible y el mismo para todos; Python en el host es opcional (solo para el editor).
- En Windows, la recarga automática necesita `WATCHFILES_FORCE_POLLING=true` o el repo dentro del sistema de archivos de WSL2.
- Los comandos del proyecto se definen con `docker compose` en el change de bootstrap.

## Notas

- **2026-11-26:** la lista de servicios de Compose se movió a [architecture.md](../architecture.md#servicios-de-compose). Era un detalle de implementación: la versión 1 suma `hold-sweeper` (ver [change-map.md](../change-map.md)). La decisión de correr todo en contenedores no cambia.
