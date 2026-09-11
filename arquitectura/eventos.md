# Catálogo de eventos del bus

Todo lo que viaja por el bus se declara acá. **Un evento que no está en esta tabla no existe**: si un servicio publica algo que nadie documentó, el consumidor se entera cuando se rompe.

> Estado: envelope, naming, versionado, reglas de idempotencia/orden/reintentos y topología de colas ya definidos (SCRUM-122 a SCRUM-125, INF-02). Catálogo del camino crítico del CP1 completo; el payload de los eventos de `community` está propuesto por `chat` y pendiente de confirmación del dueño de `community`. Cada épica nueva suma sus eventos a la tabla en el mismo PR que los implementa.

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

## Topología de colas (RabbitMQ)

> [ADR-0003](../adr/0003-tecnologia-del-bus-pubsub.md) todavía figura como "Propuesto". Esta sección da por tomada la decisión de RabbitMQ porque bloquea a `chat` desde ya; si la weekly la cambia, se actualiza junto con el ADR.

Un único exchange **topic**, durable: `discordia.events`. La routing key es el `eventType` completo (`community.member.joined`). Cada consumidor elige su patrón de binding (`community.#`, `chat.message.*`, `#` para fan-out total como hace `metrics`).

Sobre ese exchange hay dos formas de declarar una cola, y **no son intercambiables**:

| Semántica | Cómo se declara | Para qué |
|---|---|---|
| **Cola compartida por servicio** | Durable, nombre fijo (`chat.community-projection`) | Las N instancias del servicio compiten por la cola: cada evento lo procesa **una sola**. Default para todo lo que termina escribiendo en una base |
| **Cola por instancia** | `exclusive` + `auto-delete`, nombre generado por el broker al conectar | Cada instancia recibe **todos** los eventos, no compite con las demás. Único modo que hace funcionar el fan-out multiinstancia |

`chat` necesita las dos, para cosas distintas:

- **Cola compartida** para consumir los eventos de `community` que alimentan su proyección de autorización: el Mongo de `chat` es uno solo, no tiene sentido que tres instancias apliquen el mismo `member.joined` tres veces.
- **Cola por instancia** para `chat.message.sent`: cada instancia tiene un conjunto distinto de clientes WebSocket conectados, así que cada una necesita enterarse de **todos** los mensajes para reenviárselos a los suyos.

**Riesgo a tener presente:** si `chat.message.sent` se declara por error como cola compartida, el mensaje le llega a una sola instancia — es decir, a una fracción de los usuarios conectados. No tira error, solo un cliente que nunca recibe nada, y es muy difícil de diagnosticar sin saber que la topología estaba mal desde el arranque. Por eso queda escrito acá y no solo en la cabeza de quien lo implementa.

Las reglas de ack, prefetch, backoff y dead-letter (sección anterior) aplican igual a las dos semánticas.

## Eventos

| Evento | Publica | Consumen | Payload (`data`) | ADR / historia |
|---|---|---|---|---|
| `identity.user.registered` | `identity` | `metrics` (fan-out) | *A definir por `identity`* | Historia de registro — dueño `identity` |
| `community.server.created` | `community` | `metrics` (fan-out) | *A definir por `community`* | Historia de creación de servidor — dueño `community` |
| `community.channel.created` | `community` | `chat` (proyección de autorización), `metrics` (fan-out) | `{ channelId, serverId, name, type }` ¹ | SCRUM-137 |
| `community.channel.updated` | `community` | `chat` (proyección de autorización), `metrics` (fan-out) | `{ channelId, name?, type? }` ¹ | SCRUM-137 |
| `community.channel.deleted` | `community` | `chat` (proyección de autorización), `metrics` (fan-out) | `{ channelId }` ¹ | SCRUM-137 |
| `community.member.joined` | `community` | `chat` (proyección de autorización), `metrics` (fan-out) | `{ serverId, userId }` ¹ | SCRUM-137 |
| `community.member.left` | `community` | `chat` (proyección de autorización), `metrics` (fan-out) | `{ serverId, userId }` ¹ | SCRUM-137 |
| `chat.message.sent` | `chat` | `chat` (fan-out multiinstancia, cola por instancia), `metrics` (fan-out) | `{ messageId, channelId, serverId, authorId, content, createdAt, clientMessageId }` | SCRUM-41 |

¹ Payload de los cinco eventos de `community` propuesto por `chat`, que es quien primero los necesita (proyección local de autorización). Falta la confirmación del dueño de `community`. Para el CP1 solo son obligatorios `member.joined` y `channel.created` (la demo depende de ellos); `member.left`, `channel.updated` y `channel.deleted` se consumen recién en CP2, pero se catalogan ahora para que el nombre y el payload no cambien cuando se implementen.

## Nota sobre los dos lenguajes

Los tipos del payload se escriben dos veces: structs en Go y modelos Pydantic en Python ([ADR-0001](../adr/0001-stack-y-lenguajes-por-servicio.md)). Esta tabla es la fuente de verdad que los mantiene alineados. Si el equipo prefiere generar los tipos desde un esquema (JSON Schema, Avro, Protobuf), es un ADR nuevo.
