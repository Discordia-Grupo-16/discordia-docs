# Flujo de trabajo con Git

Aplica a los 8 repos de servicio, los 3 de front y a este repo de documentación.

## Repos

**Un repo por microservicio** (más los de front y este). No hay monorepo.

## Ramas

| Rama | Para qué |
|---|---|
| `master` | Lo que está desplegado / lo que se entrega en cada checkpoint |
| `dev` | Integración. De acá salen y acá vuelven todas las features |
| `feature/<TP-XXX>-<slug>` | Una rama por historia o task de Jira |

- **Sin commits directos a `dev` ni a `master`.**
- `dev` → `master` solo en los checkpoints.

## Pull requests

- **Una historia = un PR.** Si el PR toca tres historias, está mal armado.
- Un **encargado** (autor) y un **reviewer** que aprueba. El reviewer no puede ser el autor.
- **Squash merge** hacia `dev`: un commit por historia deja un historial legible y hace fácil revertir.
- El PR referencia el ticket de Jira en el título: `TP-95: FastAPI base de identity`.

## Convención de commits (propuesta)

```
<tipo>(<scope>): <descripción en imperativo>

TP-95
```

Tipos: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`.

> Propuesta a validar con el equipo. Si nadie la banca, alcanza con que el mensaje diga qué se hizo y referencie el ticket.

## Antes de pedir review

- [ ] CI en verde (la consigna marca CI roto como red line)
- [ ] Sin `.env`, credenciales ni claves en el diff
- [ ] Tests de lo nuevo, con la cobertura del repo por encima del 70%
- [ ] Si cambia un contrato entre servicios, el evento está en [`arquitectura/eventos.md`](../arquitectura/eventos.md)
- [ ] Si es una decisión de diseño, hay un ADR (ver [CONTRIBUTING](../CONTRIBUTING.md))
