# Convenciones

> **Estado: propuesta.** Ninguna de estas convenciones fue acordada todavía; están acá para discutirlas en una reunión y dejarlas fijas antes de que cada uno arranque su servicio con criterio propio.

## Repos

Ver [`git-workflow.md`](git-workflow.md#repos).

## Servicios

Minúscula, guion medio, en inglés y en singular salvo que el plural sea el dominio: `identity`, `community`, `mod`, `monetization`, `chat-and-real-time`, `notifications`, `metrics`.

## HTTP

- Todo cuelga de `/api/v1/` en el gateway.
- Recursos en **plural**: `/api/v1/servers`, `/api/v1/servers/{serverId}/channels`.
- Sin verbos en la URL: la acción la da el método HTTP.
- `camelCase` en los JSON (el front es JavaScript en los tres artefactos).
- Errores con la misma forma en todos los servicios:

```json
{ "error": { "code": "SERVER_NOT_FOUND", "message": "...", "correlationId": "uuid" } }
```

## Tópicos y eventos

Ver [`arquitectura/eventos.md`](../arquitectura/eventos.md).

## Bases de datos

- PostgreSQL: tablas en plural y `snake_case` (`server_members`); migraciones con Alembic en los servicios Python.
- MongoDB: colecciones en plural y `camelCase` en los documentos.
- **Ningún servicio se conecta a la base de otro.** Nunca.

## Identificadores

UUID v4 para todas las entidades de dominio. No exponer ids autoincrementales en la API.

## Fechas

UTC en ISO-8601 con `Z` (`2026-09-08T14:32:00Z`), tanto en la API como en los eventos. La conversión a hora local es responsabilidad del front.

## Idioma

- Código, nombres de variables, eventos y endpoints: inglés. 
- Ramas y commits: español (ver git-workflow.md). 
- Documentación, ADR e informes: español.

## Ramas, commits y PRs

Ver [`git-workflow.md`](git-workflow.md).