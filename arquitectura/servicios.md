# Servicios

Referencia rápida de qué hace cada componente. El detalle de por qué el stack es este está en [ADR-0001](../adr/0001-stack-y-lenguajes-por-servicio.md).

## Backend

| Servicio | Lenguaje | Base de datos | Responsabilidad | Repo |
|---|---|---|---|---|
| `api-gateway` | Go | — | Punto único de entrada: valida JWT, rutea, publica al bus | `discordia-api-gateway` |
| `identity` | Python + FastAPI | PostgreSQL | Registro, login, sesión, perfil de usuario | `discordia-identity` |
| `community` | Python + FastAPI | PostgreSQL | Servidores, canales, invitaciones, membresías | `discordia-community` |
| `mod` | Python + FastAPI | PostgreSQL | Roles, permisos, baneos, suspensiones | `discordia-mod` |
| `monetization` | Python + FastAPI | PostgreSQL | Suscripciones y pagos | `discordia-monetization` |
| `chat-and-real-time` | Go | MongoDB | Mensajería en tiempo real y canal de voz | `discordia-chat` |
| `notifications` | Go | sin DB propia | Envío de notificaciones vía Firebase Cloud Messaging | `discordia-notifications` |
| `metrics` | Go | PostgreSQL | Consume eventos de todos los servicios y agrega métricas | `discordia-metrics` |

> Los nombres de repo son una propuesta; ajustar cuando estén creados en la organización.

## Front-end

| Artefacto | Stack | Notas |
|---|---|---|
| `web-app` | React | Aplicación de usuario final |
| `backoffice` | React | Administración de plataforma; comparte base con `web-app` |
| `mobile` | React Native | Cliente móvil |

## Dependencias externas

| Servicio externo                     | Lo usa               | ADR                                                    |
| ------------------------------------ | -------------------- | ------------------------------------------------------ |
| Firebase Cloud Messaging             | `notifications`      | —                                                      |
| Bus Pub/Sub (sin definir)            | todos                | [ADR-0003](../adr/0003-tecnologia-del-bus-pubsub.md)   |
| Infraestructura de voz (sin definir) | `chat-and-real-time` | [ADR-0005](../adr/0005-tecnologia-del-canal-de-voz.md) |
| Pasarela de pagos (sin definir)      | `monetization`       | —                                                      |
