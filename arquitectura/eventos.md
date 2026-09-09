# Catálogo de eventos del bus

Todo lo que viaja por el bus se declara acá. **Un evento que no está en esta tabla no existe**: si un servicio publica algo que nadie documentó, el consumidor se entera cuando se rompe.

> Estado: plantilla vacía. Se llena a medida que cada épica define sus eventos, en el mismo PR que los implementa.

## Convención de nombres

```
<servicio>.<agregado>.<evento-en-pasado>
```

Ejemplos: `identity.user.registered`, `community.server.created`, `mod.member.banned`, `chat.message.sent`.

Reglas:

- El verbo va **en pasado**: el evento describe algo que ya ocurrió, no una orden.
- El servicio que publica es el dueño del nombre. Nadie publica en el namespace de otro.
- Un evento nuevo se agrega; **un evento existente no cambia de forma**. Si el payload tiene que cambiar de manera incompatible, se publica `v2` en paralelo y se deprecia el anterior.

## Envelope común

Todos los eventos comparten la misma estructura externa; lo específico va en `data`.

```json
{
  "eventId": "uuid",
  "eventType": "identity.user.registered",
  "eventVersion": 1,
  "occurredAt": "2026-09-08T14:32:00Z",
  "correlationId": "uuid",
  "producer": "identity",
  "data": { }
}
```

- `correlationId` viaja desde el gateway y se propaga a todos los eventos derivados: es lo único que permite seguir un flujo completo entre 8 servicios cuando algo falla.
- `eventVersion` permite convivir dos versiones durante una migración.

## Eventos

| Evento | Publica | Consumen | Payload (`data`) | ADR / historia |
|---|---|---|---|---|
| _(vacío)_ | | | | |

## Nota sobre los dos lenguajes

Los tipos del payload se escriben dos veces: structs en Go y modelos Pydantic en Python ([ADR-0001](../adr/0001-stack-y-lenguajes-por-servicio.md)). Esta tabla es la fuente de verdad que los mantiene alineados. Si el equipo prefiere generar los tipos desde un esquema (JSON Schema, Avro, Protobuf), es un ADR nuevo.
