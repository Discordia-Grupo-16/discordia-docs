# Entregas

El fuente de cada informe es **Markdown**. Una carpeta por checkpoint.

| Checkpoint | Fecha | Carpeta |
|---|---|---|
| CP0 | 2026-09-04 | [`cp0/`](cp0/) |
| CP1 | 2026-09-25 | [`cp1/`](cp1/) |
| CP2 | 2026-10-23 | [`cp2/`](cp2/) |
| CP3 | 2026-11-13 | [`cp3/`](cp3/) |
| Entrega final | 2026-12-04 | [`final/`](final/) |

## Convención

- Un archivo por informe: `cpN/informe-cpN.md`.
- Los diagramas no se duplican: se referencian desde [`arquitectura/`](../arquitectura/). Si un informe necesita una imagen, se exporta al momento de armar la entrega y se deja en `cpN/assets/`.
- El informe se manda por PR como cualquier otro cambio, con al menos un review. **No se escribe la noche anterior en un doc suelto.**

## Si piden PDF

```bash
pandoc informe-cp1.md -o informe-cp1.pdf \
  --toc --number-sections -V geometry:margin=2.5cm -V lang=es
```

Con diagramas Mermaid, exportarlos primero:

```bash
mmdc -i ../../arquitectura/arquitectura-discordia.mmd -o assets/arquitectura.png
```

Los PDF generados **no se versionan** (están en `.gitignore`), salvo el de la entrega final.

## Esqueleto sugerido de informe

1. Resumen del período
2. Alcance comprometido vs. alcance entregado
3. Arquitectura: qué cambió desde el checkpoint anterior y por qué (link a los ADR)
4. Decisiones tomadas en el período (tabla con link a cada ADR)
5. Métricas del proceso: historias cerradas, cobertura, estado del CI
6. Riesgos y deuda técnica
7. Plan del período siguiente
