# Plan de implementación — Fase 1 (esqueleto) de `rag-docs-api`

> Estado: **aprobado** (2026-11-18). Decisiones del usuario incorporadas (ver "Decisiones cerradas").

## Resumen

Esqueleto FastAPI gestionado con `uv` (layout `src/`), endpoint `GET /health` (liveness con
`status` y `version`), Postgres + pgvector en docker-compose, ruff + mypy strict + pytest con
cobertura ≥ 80 % y CI en GitHub Actions. **Sin** lógica RAG, modelos de BD ni Ollama; se dejan
seams limpios (settings inyectables, `create_app()`, routers por módulo, fixtures reutilizables).

## Decisiones cerradas

| # | Tema | Decisión |
|---|------|----------|
| 1 | Paquete | `rag_docs_api`. Se añade `**/rag_docs_api/**/*.py` a `paths` de `.claude/rules/ecc/python/fastapi.md`. |
| 2 | Ramas | `master` renombrada a `main`. Trabajo en `feat/phase-1-skeleton`. CI en `push` a `main` y en `pull_request`. |
| 3 | Python | 3.13 (`.python-version` = `3.13`, `requires-python = ">=3.13,<3.14"`). |
| 4 | Postgres | pg17. Tag verificado en Docker Hub el 2026-11-18: **`pgvector/pgvector:0.8.7-pg17`** (última 0.8.x para pg17; `0.8.1-pg17` también existe pero está desactualizada). Re-verificar antes del paso 5. |
| 5 | E2E | Fuera de la Fase 1. Llegará con el flujo RAG (ingesta → pregunta → respuesta). |
| 6 | Actions | Fijadas por versión mayor (`actions/checkout@v5`, `astral-sh/setup-uv@vN`), confirmando la mayor vigente al implementar. |
| 7 | Cobertura | `fail_under = 80`, cobertura de ramas activada. |
| 8 | Extras | Ninguno (sin pre-commit, task runner, logging estructurado ni Dockerfile/servicio `api`). |
| 9 | `/health` | Devuelve `{"status": "ok", "version": "<versión del paquete>"}`. |

## Decisiones de diseño

### Layout

```
rag-docs-api/
├── .github/workflows/ci.yml
├── docker/postgres/init/01-enable-pgvector.sql
├── docs/fase1-plan.md
├── src/rag_docs_api/
│   ├── __init__.py            # __version__ (importlib.metadata)
│   ├── py.typed
│   ├── config.py              # Settings + get_settings() (lru_cache)
│   ├── main.py                # create_app(settings=None) + app = create_app()
│   ├── api/routes/health.py   # APIRouter GET /health
│   └── schemas/health.py      # HealthResponse
├── tests/
│   ├── conftest.py            # fixtures: test_settings, app, client (httpx AsyncClient)
│   ├── unit/{test_package.py, test_config.py}
│   └── integration/test_health.py
├── .env.example
├── .python-version
├── compose.yaml
├── pyproject.toml
├── uv.lock
└── README.md
```

YAGNI: no se crean `db/`, `models/`, `services/`, `llm/` ni `api/deps.py` vacíos.

### Settings

- `Settings(BaseSettings)` con `env_prefix="RAG_"`, `env_file=".env"`, `extra="ignore"`, `frozen=True`.
- Campos: `app_name: str = "rag-docs-api"`, `environment: Literal["development","test","production"] = "development"`,
  `log_level: Literal["DEBUG","INFO","WARNING","ERROR"] = "INFO"`.
- `get_settings()` con `@lru_cache`; `create_app(settings: Settings | None = None)` para inyección en tests.
- `DATABASE_URL` / `OLLAMA_BASE_URL` sólo comentados en `.env.example` (Fase 2).

### `/health`

- `GET /health` → `200 {"status": "ok", "version": "x.y.z"}`, `response_model=HealthResponse`
  (`status: Literal["ok"]`, `version: str`, frozen). No toca dependencias externas.
- Fase 2: `GET /health/ready` con checks de BD (`SELECT 1`) y Ollama → `200`/`503`.

### docker-compose

- Sólo servicio `db`: `pgvector/pgvector:0.8.7-pg17`, puerto `${POSTGRES_PORT:-5432}:5432`,
  `POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD no definido}`, volumen con nombre
  `pgdata`, healthcheck `pg_isready`, init `01-enable-pgvector.sql` (`CREATE EXTENSION IF NOT EXISTS vector;`).
- La API corre en el host (`uv run uvicorn --reload`).

### Tooling

- Runtime: `fastapi`, `uvicorn[standard]`, `pydantic-settings`.
- Dev (`[dependency-groups] dev`): `pytest`, `pytest-asyncio`, `pytest-cov`, `httpx`, `ruff`, `mypy`.
- Build backend `uv_build`.
- ruff: `target-version = "py313"`, `line-length = 100`, `line-ending = "lf"`,
  `select = ["E","W","F","I","B","UP","SIM","N","S","ASYNC","PT","C4","RUF"]`, `tests/**` ignora `S101`.
  (Sustituye a black + isort + bandit.)
- mypy: `strict = true`, `files = ["src", "tests"]`, plugin `pydantic.mypy`.
- pytest: `addopts = "-ra --strict-markers --strict-config --cov=rag_docs_api --cov-report=term-missing"`,
  markers `unit`, `integration`, `asyncio_mode = "auto"`.
- coverage: `branch = true`, `fail_under = 80`.

## Pasos

Convención: commits convencionales; en pasos con código, commit RED → commit GREEN. Todo con `uv run`.

### Paso 0 — Prerequisitos (sin commit)
Verificar `uv --version`, `docker version`, `docker compose version`, `git config core.autocrlf`.

### Paso 1 — Proyecto uv — `chore: initialize uv project with src layout`
`pyproject.toml`, `.python-version`, `uv.lock`, `src/rag_docs_api/__init__.py`, `py.typed`, `.gitignore`.
Verificación: `uv sync`; `uv run python -c "import rag_docs_api"`; `uv lock --check`; `git status` limpio de artefactos.

### Paso 2 — Puertas de calidad — `test: ... (RED)` → `chore: configure ruff, mypy, pytest and coverage`
Test primero: `test_package_exposes_installed_version` (`__version__ == importlib.metadata.version("rag-docs-api")`).
Verificación: ruff check, ruff format --check, mypy, pytest verdes + verificaciones negativas (cada puerta falla cuando debe).

### Paso 3 — Settings — `test: add settings tests (RED)` → `feat: add environment-based settings`
Tests: defaults, lectura de `RAG_*`, rechazo de `environment` inválido, inmutabilidad, `get_settings()` cacheado
(`Settings(_env_file=None)`, `monkeypatch`, `cache_clear()`). Crear `.env.example`.

### Paso 4 — App factory y `/health` — `test: add health endpoint tests (RED)` → `feat: add app factory and liveness health endpoint`
Fixtures `test_settings`, `app`, `client` (httpx `ASGITransport`). Tests:
- `test_health_returns_200_with_status_and_version` → `{"status": "ok", "version": rag_docs_api.__version__}`
- `test_health_returns_json_content_type`
- `test_health_is_documented_in_openapi` (`/health` y `HealthResponse`)
- `test_create_app_uses_injected_settings` (`title`, `version`)
- `test_unknown_route_returns_404`
Verificación manual: `uv run uvicorn rag_docs_api.main:app --reload` + `Invoke-RestMethod http://127.0.0.1:8000/health`.

### Paso 5 — docker-compose — `chore: add docker compose with pgvector postgres`
Re-verificar el tag en Docker Hub. Verificación: `docker compose config -q` (falla sin `POSTGRES_PASSWORD`),
`up -d db` → healthy, `SELECT extversion FROM pg_extension WHERE extname='vector'` → `0.8.7`, persistencia tras `down`/`up`.

### Paso 6 — CI — `ci: add lint, format, typecheck and test workflow`
`.github/workflows/ci.yml`: `push` a `main` + `pull_request`, `permissions: contents: read`, `concurrency`.
Job: checkout → setup-uv (versión fijada, cache) → `uv python install` → `uv sync --locked` → ruff check
(`--output-format=github`) → ruff format --check → mypy → pytest → `docker compose config -q` (password ficticia).
Verificación: verde en el PR; rojo ante error de formato provocado.

### Paso 7 — README — `docs: add README with setup and development commands`
Verificación: seguirlo desde un clon limpio.

### Revisión de cierre
`code-reviewer`, `python-reviewer`, `fastapi-reviewer` (pasos 3–4), `security-reviewer` (compose, `.env.example`, CI).

## Gotchas de Windows

- Usar siempre `uv run`; no activar el venv (Execution Policy).
- `uv.lock` es universal: generado en Windows, válido en CI Linux con `--locked`.
- Docker Desktop con WSL2 debe estar **arrancado**; volúmenes con nombre.
- Puerto 5432 puede estar ocupado por un Postgres local → cambiar `POSTGRES_PORT`.
- LF: `.gitattributes` + ruff `line-ending = "lf"`; init en `.sql`, no `.sh`.
- No crear `.env` con `>`/`Out-File` en PowerShell 5.1 (UTF-16); usar `Copy-Item`.
- `curl` en PowerShell 5.1 es alias → `curl.exe` o `Invoke-RestMethod`.
- Scripts `docker-entrypoint-initdb.d` sólo corren con volumen vacío (`docker compose down -v` para re-ejecutar).

## Criterios de éxito

- [ ] `uv sync --locked` funciona en Windows y en CI Linux.
- [ ] ruff check, ruff format --check, mypy strict sin errores.
- [ ] pytest verde con cobertura de ramas ≥ 80 %.
- [ ] `GET /health` → `200 {"status":"ok","version":...}`, documentado en OpenAPI.
- [ ] `docker compose up -d db` → healthy, extensión `vector` 0.8.x.
- [ ] Sin secretos versionados; compose falla sin `POSTGRES_PASSWORD`.
- [ ] CI verde en el PR y rojo ante errores provocados.
- [ ] README validado desde clon limpio.
