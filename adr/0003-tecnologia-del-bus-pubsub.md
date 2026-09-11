# ADR-0003 — Tecnología de Pub/Sub

- **Estado:** Propuesto
- **Fecha:** 2026-09-09
- **Decisores:** Grupo 16
- **Tarea relacionada:** INF-03 (Sprint 1 — CP1)
- **Bloquea a:** INF-05 (docker-compose), INF-06 (cliente compartido de Pub/Sub), INF-10 (API Gateway), historia #15 (Enviar mensaje en un canal)

---

## Contexto

El enunciado impone tres requisitos que condicionan esta decisión:

1. **Comunicación asíncrona por defecto entre servicios backend.** Toda llamada sincrónica entre dos servicios debe justificarse en un ADR aparte. Los consumidores de eventos deben ser idempotentes y el sistema debe tolerar mensajes duplicados o fuera de orden.
2. **Propagación de mensajes entre instancias del servicio de mensajería mediante pub/sub**, de forma que el sistema funcione correctamente con múltiples instancias corriendo en paralelo.
3. **Manejo de errores distribuidos**: diferenciar fallos transitorios (reintentar) de permanentes (compensar o notificar), sin silenciar errores.

A esto se suman restricciones propias del proyecto:

- Dos lenguajes de backend en uso (**Python + FastAPI** y **Go**), por lo que se necesita un cliente maduro en ambos.
- El entorno local debe levantarse completo con `docker-compose`.
- El despliegue debe usar un plan gratuito o de bajo costo.
- El equipo tiene ~13 semanas y ninguna experiencia previa operando un broker en producción.

### Dos casos de uso, no uno

El sistema necesita el broker para dos cosas con semántica opuesta, y esto es central para la decisión:

| | Caso A — Eventos de dominio | Caso B — Fan-out de mensajería |
| --- | --- | --- |
| Ejemplo | `user.banned`, `server.created`, `payment.confirmed` | Un mensaje enviado en un canal debe llegar a todos los WebSockets conectados |
| Emisor | Cualquier servicio | Una instancia de `chat-and-real-time` |
| Consumidores | Un servicio (aunque tenga N réplicas, procesa **una** sola) | **Todas** las instancias de `chat-and-real-time` |
| Pérdida aceptable | No. Debe persistir y reintentarse | Tolerable: si no hay nadie conectado, el mensaje ya está en MongoDB y se recupera por historial |
| Latencia | Segundos | Milisegundos |

Una solución que resuelve bien A puede resolver mal B, y viceversa. La decisión debe cubrir ambos.

---

## Decisión

**Se adopta RabbitMQ como único broker de mensajería del sistema**, usado con dos patrones de topología distintos según el caso de uso.

### Topología para el Caso A — Eventos de dominio

- Un **topic exchange** durable: `discordia.events`.
- Routing keys jerárquicas: `<dominio>.<entidad>.<acción>` (ej. `mod.member.banned`, `community.server.created`).
- **Una cola durable por servicio consumidor** (ej. `notifications.mod-events`), compartida por todas sus réplicas. RabbitMQ reparte round-robin: cada evento lo procesa exactamente una réplica.
- `ack` manual después de procesar, `nack` sin requeue hacia una **Dead Letter Queue** tras N reintentos.
- Reintentos con backoff exponencial vía TTL + DLX.
- Todo evento lleva `message_id` (UUID v4), `occurred_at` y `version` en el envelope (ver INF-02). Los consumidores persisten los `message_id` procesados para ser idempotentes.

### Topología para el Caso B — Fan-out de mensajería en tiempo real

- Un **topic exchange** separado: `discordia.realtime`.
- **Una cola exclusiva y `auto-delete` por instancia** de `chat-and-real-time`, con nombre generado por el broker, bindeada al exchange.
- Sin durabilidad: si la instancia muere, su cola desaparece; los clientes se reconectan a otra instancia y recuperan lo perdido por el historial (historia #16).

> **Trampa a evitar:** si el Caso B usara una cola compartida como el Caso A, RabbitMQ haría round-robin y el mensaje llegaría a **una sola instancia**. Los clientes conectados a las otras no verían nada. Ese es exactamente el escenario que la demo del CP1 debe probar, así que la separación de topologías es obligatoria, no un detalle de tuning.

---

## Alternativas evaluadas

### Redis Pub/Sub — descartada

Es la opción más simple y la de menor latencia, y resolvería el Caso B casi sin código.

Se descarta porque es *fire-and-forget*: no hay persistencia, ni `ack`, ni reintentos. Un consumidor caído durante 5 segundos pierde definitivamente todo lo publicado en esa ventana. Eso es inaceptable para el Caso A, donde el enunciado exige consistencia ante fallos parciales y compensación de flujos de pago. Se evaluó **Redis Streams** como variante con persistencia, pero obliga a implementar a mano consumer groups, reintentos y DLQ — trabajo que RabbitMQ ya trae resuelto.

### Apache Kafka — descartada

Técnicamente superior para el Caso A: log persistente, replay desde cualquier offset, particionado con orden garantizado por partición, throughput muy alto.

Se descarta por costo operativo desproporcionado al proyecto. Requiere gestionar el cluster (o pagar un servicio gestionado sin free tier razonable), y el modelo de offsets, consumer groups y rebalanceos es una curva de aprendizaje que competiría con las 16 pts de la épica de Voz, que ya es el riesgo #1 del cuatrimestre. Además, el Caso B es un antipatrón en Kafka: crear un topic o consumer group efímero por instancia de WebSocket es caro.

### NATS / NATS JetStream — descartada

Es probablemente el mejor ajuste técnico puro: un binario liviano, latencia muy baja, y core NATS resuelve el Caso B de forma nativa mientras JetStream cubre el Caso A con persistencia.

Se descarta por razones de equipo, no técnicas: nadie del grupo lo usó, la cátedra tiene menos expertise para acompañar, y el ecosistema de tutoriales y respuestas es más chico que el de RabbitMQ. Con 18 días de sprint, la familiaridad pesa más que el rendimiento marginal.

### RabbitMQ — elegida

- Cubre **ambos** casos de uso con una sola pieza de infraestructura, cambiando solo la topología de colas.
- Trae de fábrica lo que el enunciado pide: `ack` manual, colas durables, DLQ, TTL para backoff, routing por topic.
- Clientes maduros en los dos lenguajes: `aio-pika` (Python) y `amqp091-go` (Go).
- Se levanta en local con una sola entrada en `docker-compose`, con management UI en el 15672 — muy útil para depurar durante la demo.
- Existe free tier gestionado (CloudAMQP) para el entorno de nube, evitando operar el broker nosotros.
- Recomendado por la cátedra, lo que asegura acompañamiento docente durante las weeklies.

---

## Consecuencias

### Positivas

- Una sola dependencia de infraestructura para toda la comunicación asíncrona.
- El cliente compartido de INF-06 puede exponer una API única (`publish` / `subscribe`) con un flag que seleccione la topología A o B, ocultando la diferencia al resto del equipo.
- La management UI permite demostrar visualmente el flujo de eventos al corrector.

### Negativas y riesgos asumidos

- **No hay replay histórico.** A diferencia de Kafka, un evento consumido y ackeado no se puede volver a leer. Si `metrics` (épica optativa) necesitara reconstruir su estado, debería hacerlo desde cero. Mitigación: `metrics` persiste rollups incrementales en PostgreSQL desde el día uno, en vez de depender de reprocesar el log.
- **El orden no está garantizado end-to-end.** RabbitMQ preserva el orden dentro de una cola con un solo consumidor, pero con prefetch o múltiples réplicas se pierde. Por eso el envelope de INF-02 incluye `occurred_at` y los consumidores deben ser idempotentes y tolerantes al desorden.
- **El broker es un punto único de falla.** Si RabbitMQ cae, la mensajería en tiempo real deja de propagarse entre instancias. Mitigación para CP1: el cliente de INF-06 implementa reconexión con backoff, y los mensajes se persisten en MongoDB *antes* de publicarse, para que el historial nunca dependa del broker.
- Queda pendiente decidir la política de `prefetch` por consumidor y el número máximo de reintentos antes de DLQ. Se define al implementar INF-06 y se documenta en su README.

---

## Referencias

- Enunciado 2026C2 — Discordia, sección *Requisitos No Funcionales → Mensajería en tiempo real y Voz*, e *Integridad y Flujo de Datos*.
- ADR-0002 — Corte de microservicios por dominio de negocio.
- INF-02 — Contrato de eventos del bus (envelope, naming, versionado, idempotencia).