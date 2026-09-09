# ADR-0006: Proveedor cloud y pipeline de CI/CD

- **Estado:** Propuesto
- **Fecha:** —
- **Decisores:** _(a completar)_
- **Servicios afectados:** todos

## Contexto

La consigna exige **deploy en la nube con CI/CD** y marca como red line que **un CI roto bloquea la evaluación**. Hoy no hay tasks de CI ni de deploy en el sprint (riesgo ya señalado en la planificación de CP1).

Lo que hay que desplegar: 1 gateway + 7 microservicios + 3 artefactos front + al menos 2 PostgreSQL + 1 MongoDB + el bus de [ADR-0003](0003-tecnologia-del-bus-pubsub.md). Es mucha infraestructura para un proyecto sin presupuesto.

## Opciones consideradas

### Opción A — Un cloud grande (GCP / AWS / Azure)

- A favor: créditos educativos; servicios gestionados para el bus y las bases; es lo que se ve en la industria.
- En contra: la curva de configuración (IAM, redes, permisos) puede comerse un sprint entero; riesgo de costos si algo queda prendido.

### Opción B — PaaS simple (Railway, Render, Fly.io)

- A favor: deploy desde el repo con casi cero configuración; bases gestionadas incluidas; el equipo puede tener todo arriba en una tarde.
- En contra: free tiers ajustados para 11 componentes; puertos UDP limitados si el canal de voz va self-hosted ([ADR-0005](0005-tecnologia-del-canal-de-voz.md)); menos control.

### Opción C — Una VM con docker-compose

- A favor: es el mismo `docker-compose` del desarrollo local (TP-107), así que no hay nada nuevo que aprender; costo predecible.
- En contra: "deploy en la nube con CI/CD" queda pobre si es copiar archivos por SSH; no hay aislamiento entre servicios; un reinicio se lleva todo.

## CI (separable de la decisión de cloud)

**GitHub Actions** es la elección por defecto: los repos ya están en GitHub, no agrega una herramienta más y el free tier alcanza de sobra.

Pipeline mínimo por repo de servicio:

1. Lint
2. Tests con reporte de cobertura — **falla el build si baja de 70%** (requisito de la consigna)
3. Build de la imagen Docker
4. Deploy a la nube solo desde `master`

Pipeline de este repo de documentación: opcionalmente un chequeo de links rotos. **No agregar CI que pueda romperse sin aportar valor** — la red line penaliza el CI roto.

## Decisión

_(a completar — se puede decidir CI ahora y cloud después)_

## Consecuencias

_(a completar)_

## Referencias

- [ADR-0003](0003-tecnologia-del-bus-pubsub.md), [ADR-0005](0005-tecnologia-del-canal-de-voz.md), [ADR-0007](0007-gestion-de-secretos.md)
