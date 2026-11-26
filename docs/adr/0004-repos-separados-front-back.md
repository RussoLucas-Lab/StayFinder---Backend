# ADR-0004: Repositorios separados para frontend y backend

- **Estado:** Aceptado
- **Fecha:** 2026-11-26

## Contexto

StayFinder va a tener un frontend en React + TypeScript y un backend en FastAPI. Se desarrolla primero el backend.

## Decisión

Dos repositorios: este (`stayfinder-backend`) y uno de frontend que se creará más adelante. El contrato entre ambos es la API REST versionada, descripta por el OpenAPI que genera FastAPI.

## Alternativas consideradas

- **Monorepo.** Descartado por ahora: el frontend no existe todavía y cada repo puede tener su propio ciclo de OpenSpec, CI y herramientas sin mezclarse.

## Consecuencias

- Este repo no contiene código de UI. Las skills de diseño de interfaz se instalan en el repo de frontend.
- El esquema OpenAPI es un entregable: los cambios incompatibles de la API requieren versión nueva o política de deprecación.
- CORS se configura con los orígenes explícitos del frontend.
- Las specs de este repo describen el comportamiento de la API; las del frontend describirán la experiencia de usuario y referenciarán estos endpoints.
