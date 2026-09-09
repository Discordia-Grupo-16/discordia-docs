# ADR-0004: Comunicaciones sincrónicas permitidas

- **Estado:** Propuesto
- **Fecha:** —
- **Decisores:** _(a completar)_
- **Servicios afectados:** `identity`, `community`, `mod`, `monetization`

> La consigna exige que **toda comunicación sincrónica esté justificada en un ADR**. Este es ese ADR. Mientras esté en `Propuesto`, esas llamadas no deberían implementarse.

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

_(a completar — puede ser una decisión distinta por cada uno de los tres casos)_

Si se aprueba alguna llamada sincrónica, definir para cada una:

- Protocolo (HTTP/JSON o gRPC) y quién publica el contrato.
- Timeout y política de reintentos.
- **Comportamiento en caso de falla:** ¿el flujo falla, degrada o encola?
- Si pasa por el gateway o es tráfico interno directo.

## Consecuencias

_(a completar)_

## Referencias

- [ADR-0002](0002-api-gateway-y-publicacion-al-bus.md)
- [`arquitectura/contexto.md`](../arquitectura/contexto.md)
