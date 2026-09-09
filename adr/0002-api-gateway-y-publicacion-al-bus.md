# ADR-0002: API Gateway como punto único de entrada que publica al bus

- **Estado:** Aceptado
- **Fecha:** 2026-09-04
- **Decisores:** equipo completo (reunión de kickoff)
- **Servicios afectados:** todos, `api-gateway` en particular

## Contexto

La consigna exige un **API Gateway como punto único de entrada** y **comunicación asíncrona por defecto** entre servicios, con Pub/Sub obligatorio para la mensajería. Toda comunicación sincrónica tiene que estar justificada en un ADR.

Los tres artefactos cliente (`web-app`, `backoffice`, `mobile`) no deben conocer la topología interna de servicios: si hablaran directo con cada uno, cualquier cambio de límites de servicio rompería a los tres.

## Opciones consideradas

### Opción A — Gateway que hace proxy HTTP a cada servicio

- A favor: modelo request/response directo, trivial de implementar y de debuggear.
- En contra: cada llamada del front es una llamada sincrónica a un servicio, que es exactamente lo que la consigna pide evitar como default.

### Opción B — Gateway que publica al bus y los servicios consumen

- A favor: cumple el "async por defecto" de la consigna de punta a punta; desacopla el gateway de la disponibilidad de cada servicio; los servicios escalan y se despliegan independientemente.
- En contra: el front espera respuestas y el bus no las devuelve. Hay que definir un mecanismo de respuesta.

## Decisión

Elegimos la **Opción B**.

- Los tres artefactos cliente hablan **solamente** con `api-gateway`.
- El gateway **publica al bus Pub/Sub**; los microservicios consumen de ahí. El gateway no llama directo a los servicios.
- El gateway concentra autenticación (validación de JWT), rate limiting y ruteo.

## Consecuencias

**Positivas**

- Un solo punto de entrada para autenticar y auditar.
- Los servicios no exponen puertos al exterior.
- La comunicación por defecto queda asíncrona sin excepciones que justificar.

**Negativas / costo que aceptamos**

- El gateway pasa a ser un punto único de falla y el componente más crítico del sistema.
- **Problema abierto e importante:** un `POST /servers` desde el front necesita una respuesta, y publicar al bus no la devuelve. Hay que elegir un mecanismo y documentarlo:
  1. **Correlation ID + espera:** el gateway publica con un `correlationId`, se suscribe a un tópico de respuesta y bloquea hasta el timeout. Simple para el front, pero vuelve sincrónico el gateway y hay que manejar timeouts.
  2. **202 + polling:** el gateway responde `202 Accepted` con un id de operación y el front consulta el estado. Honesto con el modelo asíncrono, pero cambia el diseño de todas las pantallas.
  3. **202 + WebSocket/SSE:** el gateway acepta y notifica el resultado por el canal de tiempo real que ya hace falta para la mensajería. Mejor UX, más infraestructura.
- Este punto **debe resolverse antes de empezar las historias de CP1**, porque define la forma de todos los endpoints.

**Qué queda pendiente por esta decisión**

- ADR sobre el mecanismo de respuesta del gateway (candidato a ADR-0008).
- La tecnología concreta del bus: [ADR-0003](0003-tecnologia-del-bus-pubsub.md).

## Referencias

- [ADR-0003](0003-tecnologia-del-bus-pubsub.md), [ADR-0004](0004-comunicaciones-sincronicas.md)
