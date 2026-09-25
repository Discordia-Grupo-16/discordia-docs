# Estructura de Microservicios FastAPI (Plantilla Base)

Este documento forma parte del repositorio `discordia-docs` y describe la arquitectura y estructura estándar recomendada para los microservicios de la plataforma Discordia que utilicen **FastAPI**.

El código base de este esqueleto se encuentra disponible como referencia en el repositorio **`Identity`**, en la rama **`feat/initial-setup`**.

---

## 1. Estructura del Proyecto

```text
├── alembic/            # Migraciones de base de datos
├── app/                # Código fuente de la aplicación
│   ├── core/           # Configuración base, DB, errores y logs
│   ├── models/         # Modelos de base de datos (SQLModel)
│   ├── repositories/   # Capa de acceso a datos y consultas
│   ├── routers/        # Controladores HTTP y endpoints (FastAPI)
│   ├── schemas/        # Esquemas de validación y DTOs (Pydantic)
│   ├── services/       # Lógica de negocio
│   └── main.py         # Punto de entrada y ciclo de vida de la app
├── tests/              # Pruebas automatizadas (pytest)
├── .coveragerc         # Configuración de cobertura de tests (mínimo 70%)
├── .env.example        # Plantilla de variables de entorno
├── Dockerfile          # Definición de la imagen del contenedor
├── docker-compose.yml  # Orquestación local (App + PostgreSQL)
├── pytest.ini          # Configuración de pytest y cobertura
└── requirements.txt    # Dependencias del servicio
```

---

## 2. Responsabilidades por Capa

- **`routers/`**: Recibe las peticiones HTTP, valida los datos de entrada usando *schemas* y delega el procesamiento a la capa de *services*.
- **`services/`**: Implementa la lógica de negocio y reglas del dominio. No depende directamente de detalles de FastAPI (como `Request` o `Response`).
- **`repositories/`**: Aísla el acceso a la base de datos (consultas SQLModel/SQLAlchemy) para no acoplar la lógica de negocio al motor de persistencia.
- **`models/`**: Define las tablas y relaciones de la base de datos mediante clases `SQLModel` (`table=True`).
- **`schemas/`**: DTOs de Pydantic para validar entradas y formatear respuestas de los endpoints.
- **`core/`**:
  - `config.py`: Carga y validación de variables de entorno (`pydantic-settings`).
  - `database.py`: Conexión con PostgreSQL y proveedor de dependencias de sesión (`SessionDep`).
  - `error.py`: Manejo global de excepciones bajo el estándar RFC 7807 (`application/problem+json`).
  - `logging.py`: Configuración del logger estructurado del servicio.

---

## 3. Componentes Transversales (`app/core/`)

### 3.1. Configuración (`app/core/config.py`)
- Utiliza `pydantic-settings` para validar los tipos en tiempo de inicio.
- Si falta una variable de entorno obligatoria, el servicio **no arranca**, evitando fallas silenciosas en runtime.
- La función `get_settings()` está cacheada con `@lru_cache` para evitar re-leer el entorno en cada llamada.

### 3.2. Conexión a Base de Datos (`app/core/database.py`)
- Emplea `SQLModel` (que combina Pydantic y SQLAlchemy) con el driver `psycopg` (v3).
- Provee un generador `get_session()` que gestiona el contexto de la transacción:
  ```python
  SessionDep = Annotated[Session, Depends(get_session)]
  ```
- Al inyectar `SessionDep` en routers o servicios, FastAPI garantiza la apertura y cierre correcto de la sesión.

### 3.3. Manejo de Errores RFC 7807 (`app/core/error.py`)
- Cada error retornado por la API utiliza el tipo de medio `application/problem+json` con la estructura:
  ```json
  {
    "type": "about:blank",
    "title": "Bad Request",
    "status": 400,
    "detail": "email: value is not a valid email address",
    "instance": "/api/v1/resource"
  }
  ```
- Se capturan y formatean automáticamente:
  - Excepciones personalizadas (`ProblemException`).
  - Errores de validación de esquemas Pydantic (`RequestValidationError`).
  - Errores HTTP estándar de Starlette/FastAPI (`StarletteHTTPException`).
  - Errores 500 no capturados (`Exception`), logueando el stacktrace completo pero sin exponer detalles internos al cliente.

### 3.4. Logging Estructurado (`app/core/logging.py`)
- Configurado con formato estándar con fecha, nivel de log y nombre del servicio.
- El nivel (`DEBUG`, `INFO`, `WARN`, `ERROR`) es configurable desde la variable de entorno `LOG_LEVEL`.

---

## 4. Pasos para Crear un Nuevo Servicio

Para crear un nuevo servicio (por ejemplo, `channels`, `messages`, `voice` o `notifications`) utilizando este esqueleto:

### Paso 1: Clonar o copiar la estructura desde Identity
Copiar la estructura base desde el repositorio `Identity` (rama `feat/initial-setup`), omitiendo `.git`, carpetas `.venv` y archivos generados (`.coverage`, caches):
```bash
cp -r identity/ nuevo_servicio/
cd nuevo_servicio/
```

### Paso 2: Personalizar variables y configuración
1. Modificar `.env.example` y crear el `.env`:
   - Cambiar nombres de base de datos (`DATABASE_NAME=nuevo_servicio_db`).
   - Ajustar el puerto si se corren varios servicios en local (ej. `PORT=8081`).
2. En `docker-compose.yml`:
   - Actualizar los nombres de contenedores (`identity-app` -> `nuevo-servicio-app`, `identity_postgres` -> `nuevo_servicio_postgres`).
   - Actualizar puertos mapeados si es necesario.
3. En `app/main.py`:
   - Actualizar `title` y `description` en `FastAPI(...)`.
   - Modificar el logger name en `app/core/logging.py` y mensajes de startup/shutdown.

### Paso 3: Definir el Dominio del Servicio
1. **Modelos (`app/models/`)**:
   Crear las tablas heredando de `SQLModel, table=True`.
2. **Migraciones (`alembic/`)**:
   Importar los nuevos modelos en `alembic/env.py` (en `target_metadata = SQLModel.metadata`) y generar la migración inicial:
   ```bash
   docker compose run --rm app alembic revision --autogenerate -m "create initial tables"
   docker compose run --rm app alembic upgrade head
   ```
3. **Esquemas (`app/schemas/`)**:
   Definir DTOs de Request (validación de entrada) y Response (serialización de salida).
4. **Repositorios (`app/repositories/`)**:
   Crear clases o funciones para aislar consultas SQL (`select`, `insert`, `update`).
5. **Servicios (`app/services/`)**:
   Implementar la lógica de negocio consumiendo los repositorios.
6. **Routers (`app/routers/`)**:
   Exponer endpoints y registrarlos en `app/main.py` con `app.include_router(...)`.

### Paso 4: Escribir Pruebas y Validar Cobertura
1. Agregar pruebas en `tests/` para cada nuevo router y servicio.
2. Mantener `test_monitoring.py` y `test_errors.py` que validan el contrato de infraestructura.
3. Validar que la cobertura supere el **70%**:
   ```bash
   docker compose run --rm app pytest
   ```

---

## 5. Comandos Frecuentes

| Acción | Comando |
| :--- | :--- |
| **Levantar entorno completo** | `docker compose up --build` |
| **Levantar en segundo plano** | `docker compose up -d` |
| **Detener contenedores** | `docker compose down` |
| **Correr suite de tests** | `docker compose run --rm app pytest` |
| **Ver reporte de cobertura** | `docker compose run --rm app coverage report` |
| **Aplicar migraciones** | `docker compose run --rm app alembic upgrade head` |
| **Crear nueva migración** | `docker compose run --rm app alembic revision --autogenerate -m "descripcion"` |
| **Verificar formato y linter** | `docker compose run --rm app ruff check .` |
| **Autofix de linter** | `docker compose run --rm app ruff check --fix .` |
