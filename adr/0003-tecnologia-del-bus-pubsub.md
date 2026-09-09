# ADR-0003: Tecnología del bus Pub/Sub

- **Estado:** Propuesto
- **Fecha:** —
- **Decisores:** _(a completar)_
- **Servicios afectados:** todos

> **Este ADR está sin decidir y bloquea al resto del backend.** El bus es el mecanismo de comunicación por defecto ([ADR-0002](0002-api-gateway-y-publicacion-al-bus.md)): sin esta decisión no se puede escribir el primer consumidor.

## Contexto

La consigna exige Pub/Sub para propagar mensajes entre instancias del servicio de mensajería, y comunicación asíncrona por defecto entre todos los servicios. Además `metrics` se suscribe a los eventos de **todos** los servicios, lo que implica fan-out.

Restricciones a tener en cuenta:

- Tiene que correr en `docker-compose` local (task TP-107) y en la nube elegida en [ADR-0006](0006-proveedor-cloud-y-cicd.md).
- Clientes maduros en **Python y Go** (ver [ADR-0001](0001-stack-y-lenguajes-por-servicio.md)).
- El equipo tiene que aprenderlo dentro del sprint, no después.

## Criterios de evaluación

| Criterio | Peso |
|---|---|
| Curva de aprendizaje del equipo | Alto |
| Facilidad de correrlo local en docker-compose | Alto |
| Soporte de fan-out (un evento, N consumidores) | Alto — lo necesita `metrics` |
| Clientes en Python y Go | Alto |
| Costo / free tier en la nube | Medio |
| Durabilidad y reintentos | Medio |
| Ordenamiento de mensajes | Bajo — evaluar si mensajería lo necesita |

## Opciones consideradas

### Opción A — RabbitMQ

- A favor: el modelo exchange/queue expresa fan-out directo; UI de administración que ayuda a debuggear y a mostrar en la defensa; clientes maduros en ambos lenguajes; corre en un contenedor sin configuración.
- En contra: no retiene el historial de eventos (una vez consumido, se fue); operar reintentos y dead-letter es manual.

### Opción B — Apache Kafka / Redpanda

- A favor: log persistente, un consumidor nuevo puede releer desde el principio (útil si `metrics` se suma tarde); es lo que se usa en la industria.
- En contra: la curva más empinada de las tres; Kafka en docker-compose es pesado (Redpanda lo aliviana); mucho concepto (particiones, offsets, consumer groups) para lo que el TP necesita.

### Opción C — NATS / JetStream

- A favor: el más liviano de correr; cliente Go de primera clase; muy simple de arrancar.
- En contra: menos material y ejemplos en Python; menos conocido por correctores y por el equipo.

### Opción D — Servicio gestionado (Google Cloud Pub/Sub, AWS SNS+SQS)

- A favor: cero operación, escala sola, se integra con el deploy en la nube que pide la consigna.
- En contra: ata la decisión a [ADR-0006](0006-proveedor-cloud-y-cicd.md); necesita emulador o una cuenta compartida para desarrollo local; el free tier tiene límites y hay riesgo de costos.

## Decisión

_(a completar)_

## Consecuencias

_(a completar)_

## Referencias

- [ADR-0001](0001-stack-y-lenguajes-por-servicio.md), [ADR-0002](0002-api-gateway-y-publicacion-al-bus.md), [ADR-0006](0006-proveedor-cloud-y-cicd.md)
- Catálogo de eventos: [`arquitectura/eventos.md`](../arquitectura/eventos.md)
