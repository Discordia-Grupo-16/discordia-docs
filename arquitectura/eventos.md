# Catálogo de eventos del bus

Todo lo que viaja por el bus se declara acá. **Un evento que no está en esta tabla no existe**: si un servicio publica algo que nadie documentó, el consumidor se entera cuando se rompe.

> Estado: envelope, naming, versionado y reglas de idempotencia/orden/reintentos ya definidos (SCRUM-122, SCRUM-123). El catálogo de eventos del camino crítico y la topología de colas (SCRUM-124, SCRUM-125) se agregan a continuación. Cada épica suma sus propios eventos a la tabla, en el mismo PR que los implementa.

**Alcance de este documento: eventos, no comandos.** Un evento describe algo que ya ocurrió y nadie lo puede rechazar (`chat.message.sent`). Un comando es una orden que puede fallar (por ejemplo, lo que el gateway publica para `POST /servers` según [ADR-0002](../adr/0002-api-gateway-y-publicacion-al-bus.md)). El envelope de comandos y su mecanismo de respuesta se definen en el ADR del mecanismo de respuesta del gateway (candidato a ADR-0008), no acá.

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
  "causationId": "uuid",
  "producer": "identity",
  "data": { }
}
```

- `eventId`: identifica esta instancia del evento. Es la clave para deduplicar en el consumidor (ver "Idempotencia" más abajo).
- `correlationId` viaja desde el gateway y se propaga a todos los eventos derivados: es lo único que permite seguir un flujo completo entre 8 servicios cuando algo falla.
- `causationId`: el `eventId` (o id del comando) que causó directamente este evento. Distinto de `correlationId`, que identifica el flujo completo; `causationId` reconstruye la cadena causal paso a paso dentro de ese flujo.
- `eventVersion` permite convivir dos versiones durante una migración.
- `occurredAt` en UTC ISO-8601 con `Z`, igual que el resto de las fechas del sistema ([convenciones](../procesos/convenciones.md)).

## Idempotencia, orden y reintentos

El bus da entrega **at-least-once**: todo consumidor puede recibir el mismo evento más de una vez y tiene que tolerarlo. Estas reglas son de cumplimiento obligatorio, no una recomendación:

### Idempotencia

- Todo handler de evento es idempotente: procesar el mismo `eventId` dos veces produce el mismo resultado que procesarlo una vez.
- Para escrituras directas (inserts), usar el `eventId` o una clave de negocio como restricción única en la base y tratar el conflicto de duplicado como éxito, no como error.
- Para **proyecciones locales** (un servicio que mantiene una copia derivada del estado de otro, alimentada por eventos), la clave `{eventId}` no alcanza porque además hay que resolver el desorden: cada entidad de la proyección guarda un `lastEventAt`, y un evento entrante se aplica **solo si su `occurredAt` es más nuevo que el `lastEventAt` guardado**. Sin esto, un evento duplicado o reordenado puede pisar un estado más reciente con uno viejo.

### Orden

- El bus **no garantiza orden global** entre eventos de agregados distintos, y con cola compartida (varias instancias compitiendo por la misma cola; la topología se detalla en la próxima sección) tampoco lo garantiza entre eventos del mismo agregado.
- Ningún consumidor asume que los eventos le llegan en el orden en que ocurrieron. El mecanismo para tolerar desorden es el mismo `lastEventAt` de la regla de idempotencia: es la única fuente de verdad sobre "qué es más nuevo", no el orden de llegada.
- Si una historia necesita orden estricto para algo puntual (por ejemplo, los mensajes de un mismo emisor), ese orden se consigue por diseño del lado del productor/consumidor —una sola conexión, una sola goroutine—, no pidiéndoselo al bus.

### Reintentos y dead-letter

- Ack manual **después** de procesar el evento con éxito, nunca al recibirlo. Si el proceso se cae a mitad de camino, el evento no se pierde: vuelve a la cola.
- `prefetch` acotado por consumidor, para no acumular en memoria más de lo que se puede procesar.
- Reintento con backoff exponencial dentro del proceso ante un error transitorio, antes de dar el evento por fallido.
- Agotados los reintentos, el evento va a una dead-letter queue (`x-dead-letter-exchange`) en vez de perderse o bloquear la cola principal. Un evento en la DLQ requiere intervención manual; no hay reproceso automático.
- **Límite conocido de RabbitMQ:** no retiene historial una vez consumido. Si una proyección local se corrompe o se pierde, no se puede reconstruir releyendo el bus — hace falta un endpoint de re-sincronización en el servicio productor (fuera de alcance de CP1) o recrearla desde un seed.

## Eventos

| Evento | Publica | Consumen | Payload (`data`) | ADR / historia |
|---|---|---|---|---|
| _(vacío)_ | | | | |

## Nota sobre los dos lenguajes

Los tipos del payload se escriben dos veces: structs en Go y modelos Pydantic en Python ([ADR-0001](../adr/0001-stack-y-lenguajes-por-servicio.md)). Esta tabla es la fuente de verdad que los mantiene alineados. Si el equipo prefiere generar los tipos desde un esquema (JSON Schema, Avro, Protobuf), es un ADR nuevo.
