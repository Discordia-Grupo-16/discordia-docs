# ADR-0005: Tecnología del canal de voz

- **Estado:** Propuesto
- **Fecha:** —
- **Decisores:** _(a completar — épica de Milton)_
- **Servicios afectados:** `chat-and-real-time`, `web-app`, `mobile`

## Contexto

El canal de voz vive dentro de `chat-and-real-time` (Go, MongoDB). Es la funcionalidad con más riesgo técnico del TP: WebRTC tiene señalización, NAT traversal (STUN/TURN), codecs y permisos de dispositivo, y hay que hacerlo funcionar en **web y en React Native** a la vez.

El equipo no tiene experiencia previa declarada en tiempo real, y esta épica compite en calendario con el resto del sprint.

## Opciones consideradas

### Opción A — Servicio gestionado (LiveKit Cloud, Agora, Daily, Twilio)

- A favor: SDK para web y React Native, TURN incluido, sale funcionando en días; el riesgo técnico se terceriza y el equipo se concentra en el producto.
- En contra: dependencia de un tercero y de su free tier; hace falta cuenta y credenciales (ver [ADR-0007](0007-gestion-de-secretos.md)); menos "mérito arquitectónico" si la cátedra valora la implementación propia.

### Opción B — SFU propio self-hosted (LiveKit OSS, mediasoup, Janus)

- A favor: control total, corre en la nube propia, demuestra manejo de la tecnología; LiveKit OSS mantiene los mismos SDK de cliente que la opción A.
- En contra: hay que operarlo (contenedor, puertos UDP, TURN propio); los puertos UDP son un problema real en varios PaaS baratos; sube el costo de la infra y del deploy.

### Opción C — WebRTC P2P en malla, sin servidor de medios

- A favor: nada que operar más allá de la señalización, que ya la puede hacer `chat-and-real-time`.
- En contra: no escala más allá de 3–4 participantes (cada cliente sube N-1 streams); problemas de NAT sin TURN; en la práctica no sirve para un canal de voz tipo Discord.

## Criterio sugerido para decidir

La pregunta que ordena todo: **¿la cátedra evalúa el canal de voz por funcionar o por estar implementado a bajo nivel?** Si es lo primero, la Opción A libera semanas. Si es lo segundo, la Opción B con LiveKit OSS es el punto medio razonable.

**Conviene confirmarlo con la cátedra antes de decidir.**

## Decisión

_(a completar)_

## Consecuencias

_(a completar)_

## Referencias

- [ADR-0006](0006-proveedor-cloud-y-cicd.md) — dónde correría el SFU si se elige la Opción B
- [ADR-0007](0007-gestion-de-secretos.md) — API keys si se elige la Opción A
