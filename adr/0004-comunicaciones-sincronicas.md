# ADR-0004: Comunicaciones sincrónicas permitidas

- **Estado:** Rechazado
- **Fecha:** 2026-09-20
- **Decisores:** equipo completo
- **Servicios afectados:** `identity`, `community`, `mod`, `monetization`

> **Rechazado.** No se habilita ninguna llamada sincrónica entre servicios backend. Se conserva el documento para no volver a discutirlo; el porqué está en [Decisión](#decisión).
>
> Lo que rige en su lugar: **un servicio que necesita un dato ajeno lo recibe por evento y lo guarda en una proyección local**, como ya hace `chat` con los eventos de `community` ([`arquitectura/eventos.md`](../arquitectura/eventos.md)). Si una historia futura demuestra que eso no alcanza, se escribe un ADR nuevo **para ese caso concreto**; no se reabre este.

## Contexto

El default del sistema es asíncrono vía bus ([ADR-0002](0002-api-gateway-y-publicacion-al-bus.md)). En el kickoff se identificaron tres comunicaciones donde el flujo necesita una respuesta antes de continuar:

| # | Comunicación | Caso de uso | Por qué no puede ser asíncrona |
|---|---|---|---|
| 1 | `identity` ↔ `community` | _(a completar)_ | _(a completar)_ |
| 2 | `identity` ↔ `mod` | _(a completar)_ | _(a completar)_ |
| 3 | `monetization` → `identity` | _(a completar)_ | _(a completar)_ |

**Cada renglón hay que justificarlo individualmente.** La justificación válida es del tipo "el usuario no puede continuar sin el resultado y el resultado no se puede precalcular ni cachear", no "es más fácil".

## Opciones consideradas

### Opción A — Llamada HTTP sincrónica directa entre servicios

- A favor: simple, inmediato, fácil de testear.
- En contra: acopla la disponibilidad de los dos servicios; si el llamado se cae, el llamador se cae; hay que definir timeouts, reintentos y qué pasa cuando falla.

### Opción B — Eliminar la necesidad replicando el dato vía eventos

- A favor: mantiene el default asíncrono; cada servicio consulta su propia copia; cero acoplamiento en runtime.
- En contra: consistencia eventual (una copia puede estar desactualizada por milisegundos o segundos); más código de proyección y más superficie de bugs.

### Opción C — Sincrónico solo para lectura, asíncrono para escritura

- A favor: acota el acoplamiento a consultas idempotentes y baratas de reintentar.
- En contra: hay que analizar caso por caso si la lectura alcanza.

## Decisión

**Rechazado.** No se aprueba ninguna de las tres comunicaciones, y queda como regla que **ningún servicio backend llama sincrónicamente a otro**. Equivale a adoptar la Opción B como norma general, sin excepciones que enumerar.

Tres razones, en orden de peso:

1. **Los tres casos nunca se justificaron.** La tabla quedó con los tres "por qué no puede ser asíncrona" en `(a completar)` desde el kickoff hasta hoy, y ninguna historia del CP1 los necesitó. Un ADR que lista tres llamadas sin caso de uso, sin justificación y sin implementación no es una decisión: es una lista de pendientes disfrazada de decisión.
2. **El problema que las empujaba ya está resuelto en otro lado.** La presión por llamar a otro servicio venía de tener que contestarle algo al usuario. Desde el [ADR-0009](0009-gateway-proxy-sincronico-para-requests-del-cliente.md) el cliente recibe su respuesta del gateway, que hace de proxy al servicio dueño del recurso. Ningún servicio necesita llamar a otro para poder contestar.
3. **La alternativa ya está implementada y funcionando.** `chat` mantiene una proyección local de autorización alimentada por los eventos de `community` ([`arquitectura/eventos.md`](../arquitectura/eventos.md)), con las reglas de idempotencia y desorden ya escritas. Es la Opción B de este mismo ADR, que dejó de ser una opción para ser el patrón por defecto del sistema.

Como no se aprueba ninguna llamada, no hay nada que definir sobre protocolo, timeouts, reintentos ni comportamiento ante falla.

**Si en el futuro hace falta una llamada sincrónica entre servicios**, se escribe un ADR nuevo para ese caso puntual, con la justificación concreta ("el usuario no puede continuar sin el resultado y el resultado no se puede precalcular ni cachear") y las cuatro definiciones que este ADR pedía: protocolo y dueño del contrato, timeout y reintentos, comportamiento ante falla, y si pasa por el gateway o es tráfico interno.

## Consecuencias

**Positivas**

- La comunicación entre servicios backend queda **100% asincrónica, sin excepciones que justificar** ante la consigna. Es la afirmación más fuerte que el proyecto puede hacer sobre este requisito, y no depende de que nadie complete una tabla.
- Se elimina el acoplamiento de disponibilidad entre servicios: ninguno se cae porque otro esté caído.
- Un solo patrón para "necesito un dato de otro servicio": suscribirse al evento y proyectarlo localmente. Nadie tiene que decidir caso por caso.

**Negativas / costo que aceptamos**

- **Consistencia eventual en todas las proyecciones locales.** Una copia puede estar desactualizada por milisegundos o segundos, y el sistema tiene que tolerarlo. Las reglas de idempotencia, orden y `lastEventAt` de [`arquitectura/eventos.md`](../arquitectura/eventos.md) son obligatorias, no una recomendación.
- **Más código de proyección** en cada servicio que consume datos ajenos, y más superficie de bugs que una llamada directa.
- Si una proyección se corrompe o se pierde, **no se puede reconstruir releyendo el bus**: RabbitMQ no retiene historial ([ADR-0003](0003-tecnologia-del-bus-pubsub.md)). Hace falta un endpoint de re-sincronización en el productor o recrearla desde un seed.

**Qué queda pendiente por esta decisión**

- `identity.yaml` expone `GET /users/{userId}` marcado como *"uso interno / entre servicios"*. Ese endpoint existía para habilitar una de las llamadas de este ADR y hoy **no tiene ADR que lo respalde**: no lo consume nadie y el gateway no lo reexpone. Hay que sacarlo del contrato o escribir el ADR que lo justifique — decisión del dueño de `identity`.

## Referencias

- [ADR-0002](0002-api-gateway-y-publicacion-al-bus.md)
- [ADR-0009](0009-gateway-proxy-sincronico-para-requests-del-cliente.md) — resolvió el problema que empujaba estas llamadas
- [ADR-0003](0003-tecnologia-del-bus-pubsub.md) — el bus no retiene historial, lo que condiciona las proyecciones locales
- [`arquitectura/eventos.md`](../arquitectura/eventos.md) — proyecciones locales, idempotencia y orden
- [`arquitectura/contexto.md`](../arquitectura/contexto.md)
