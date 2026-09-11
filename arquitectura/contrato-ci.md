# Contrato de repositorio para el CI

Un solo pipeline sirve a los 10 repos siempre que todos ubiquen los mismos archivos en
los mismos lugares. Esto es lo que el CI espera encontrar. Si un repo no lo cumple, su
pipeline falla y hay que escribirle uno a medida.

---

## Servicios Python

`identity` · `community` · `mod` · `monetization`

| Archivo / carpeta | Obligatorio | Nota |
| --- | --- | --- |
| `requirements.txt` | sí | solo deps de runtime |
| `requirements-dev.txt` | sí | `ruff`, `pytest-cov`. Herramientas que no van a la imagen |
| `app/` | sí | código de la aplicación. Es lo que se mide para cobertura |
| `tests/` | sí | al menos un test desde el primer commit |
| `pytest.ini` | sí | config de pytest y coverage |
| `ruff.toml` | sí | config de lint y formato |
| `Dockerfile` | sí | en la raíz, contexto de build `.` |
| `alembic.ini` + `migrations/` | si el servicio tiene esquema | el CI **no** corre migraciones: los tests crean sus tablas |
| `.env.example` | sí | todas las variables que el servicio lee, con valores de ejemplo |

## Servicios Go

`api-gateway` · `chat-and-real-time` · `notifications` · `metrics`

| Archivo / carpeta | Obligatorio | Nota |
| --- | --- | --- |
| `go.mod` | sí | en la raíz |
| `internal/`, `pkg/` | sí | código testeable. **Es lo único que cuenta para cobertura** |
| `cmd/` | sí | entrypoints. Queda fuera de la medición a propósito |
| `.golangci.yml` | sí | config de lint |
| `Dockerfile` | sí | en la raíz |
| `.env.example` | sí | idem Python |

Si el código de negocio queda en `cmd/`, no se mide y el gate no sirve de nada.

## Front

`web-app` · `backoffice` · `mobile`

| Requisito | Nota |
| --- | --- |
| scripts `lint`, `test`, `build` en `package.json` | `mobile` puede no tener `build` |
| vitest con provider `v8` | |
| `tsconfig.json` | el CI corre `tsc --noEmit`: `npm run build` de Vite **no** typechequea |

---

## Lo que trae cada repo desde el primer commit

1. **`.github/workflows/ci.yml`** — el caller de ~20 líneas que apunta a `discordia-ci`.
   Lo provee INF-07; va en el mismo PR que la plantilla base del servicio.
2. **Endpoint `/livez` con su test.** Es RNF obligatorio del enunciado igual. Sin al menos
   un test, `pytest` sale con código 5 ("no tests collected") y el repo nace con el CI en
   rojo, que es red line.
3. **`.env` en `.gitignore`.** Junto con `coverage.xml`, `.coverage`, `.ruff_cache/`.

## Cobertura

El gate es **70% y bloquea el merge**, como pide el enunciado para los servicios backend.

- No se baja el número: si un servicio no llega, se resuelve con tests.
- En el front arranca informativo (no bloquea). El enunciado exige el 70% en backend.
- Escribir los tests junto con la feature, no después. El gate corre desde el primer PR.
- **Pendiente:** si el 70% se mide por servicio o global del backend. Consulta abierta al
  corrector. Con repo por servicio, el gate por servicio es el único implementable sin
  agregación externa, y es *más estricto* que el global (que se puede compensar con un
  servicio muy testeado tapando otro sin tests). Propuesta del grupo: 70% por servicio.

## Ramas y merge

- `develop` como rama de integración, `main` protegida.
- Ramas: `chore/SCRUM-XXX-descripcion`, `feat/`, `fix/`, `docs/`.
- Squash merge, 1 approval, sin commits directos a `develop` ni `main`.
- Los required status checks se llaman `ci / lint`, `ci / test`, `ci / build`,
  `security / gitleaks`. Aparecen en la lista de GitHub recién después de la primera
  corrida.

Esto se solapa con INF-04 (convenciones de repo). La versión canónica es la de INF-04;
acá está solo lo que el CI necesita para funcionar.

## Seguridad

`gitleaks` corre en los 10 repos con `fetch-depth: 0`, o sea escaneando el historial y no
solo el último commit. Cubre la red line de secretos del enunciado. Se activa en repos
vacíos también: el momento de mayor riesgo es el primer commit.

CodeQL (SAST, "se recomienda" en el enunciado) queda diferido a CP2 y solo en los
servicios que tocan auth y permisos: `api-gateway`, `identity`, `community`.