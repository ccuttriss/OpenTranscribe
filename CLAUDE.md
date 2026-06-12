# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Architecture

OpenTranscribe is a containerized AI-powered transcription application with these core services:
- **Frontend**: Svelte/TypeScript SPA with Progressive Web App capabilities
- **Backend**: FastAPI with async support and OpenAPI documentation
- **Database**: PostgreSQL with SQLAlchemy ORM (no migrations during development)
- **Storage**: MinIO S3-compatible object storage
- **Search**: OpenSearch 3.3.1 (Apache Lucene 10) for full-text and vector search
- **Queue**: Celery with Redis for background AI processing
- **Monitoring**: Flower for task monitoring

### Key Technologies
- **AI Models**: WhisperX for transcription (100+ languages), PyAnnote for speaker diarization
- **Frontend**: Svelte, TypeScript, Vite, Plyr for media playback, i18n (7 UI languages)
- **Backend**: FastAPI, SQLAlchemy 2.0, Alembic, Celery
- **Infrastructure**: Docker Compose, NGINX for production

## Development Commands

### Primary Development Script
Use `./opentr.sh` for all development operations:

```bash
# Start development environment
./opentr.sh start dev

# Stop all services
./opentr.sh stop

# View logs (all or specific service)
./opentr.sh logs [backend|frontend|postgres|celery-worker]

# Reset environment (WARNING: deletes all data)
./opentr.sh reset dev

# Check service status
./opentr.sh status

# Access container shell
./opentr.sh shell [backend|frontend|postgres]

# Database backup/restore
./opentr.sh backup
./opentr.sh restore backups/backup_file.sql

# Multi-GPU scaling (optional - for high-throughput systems)
./opentr.sh start dev --gpu-scale
./opentr.sh reset dev --gpu-scale
```

### Frontend Development
```bash
# From frontend/ directory
npm run dev          # Start dev server
npm run build        # Production build
npm run check        # Type checking
```

### Backend Development
```bash
# From backend/ directory (or via container)
uvicorn app.main:app --reload --host 0.0.0.0 --port 8080
pytest tests/        # Run tests
alembic upgrade head # Apply migrations (production only)
```

### Python Virtual Environment

**IMPORTANT**: Always use `backend/venv` for all Python operations outside Docker
(pre-commit, mypy, ruff, bandit, pytest, any Python CLI tools):

```bash
source backend/venv/bin/activate
ruff check backend/ && ruff format backend/
mypy backend/app
pytest backend/tests/
```

Setup (only if missing): `python3.11 -m venv backend/venv`, then
`pip install -r backend/requirements.txt pre-commit mypy ruff bandit`.

## Database Management

**IMPORTANT**: During development, do NOT create Alembic migrations. Instead:
1. Update `database/init_db.sql` directly
2. Update SQLAlchemy models in `backend/app/models/`
3. Update Pydantic schemas in `backend/app/schemas/`
4. Reset the development database: `./opentr.sh reset dev`

For production deployments, the backend now includes an **automatic migration system** that runs on startup:
- Migrations are located in `backend/app/db/migrations.py`
- Fresh installs, upgrades from v0.1.0, and already-tracked databases are handled automatically
- No manual migration commands required for normal operation

## Code Organization Patterns

### Backend Structure
- `app/api/endpoints/` - REST API routes organized by resource
- `app/models/` - SQLAlchemy ORM models
- `app/schemas/` - Pydantic validation schemas
- `app/services/` - Business logic and external integrations
- `app/tasks/` - Celery background tasks
- `app/core/` - Configuration and security

### Frontend Structure
- `src/components/` - Reusable Svelte components
- `src/components/settings/` - Settings-related components
- `src/routes/` - Page components
- `src/stores/` - Svelte stores for state management
- `src/lib/` - Utilities and services
- `src/lib/i18n/` - Internationalization system (7 languages)
- `src/lib/i18n/locales/` - Translation JSON files (en, es, fr, de, pt, zh, ja)

### Key Patterns
- **Authentication**: Hybrid multi-method with JWT tokens and refresh token rotation
- **File Processing**: Upload to MinIO → Celery task → AI processing → Database storage
- **Real-time Updates**: WebSockets for task progress notifications
- **Error Handling**: Structured error responses with proper HTTP status codes

### Authentication System (summary)

Hybrid multi-method auth — local password, LDAP/AD, OIDC/Keycloak, and PKI/X.509
can be enabled simultaneously via `AUTH_TYPE` in `.env`. JWT access tokens +
refresh-token rotation; per-IP/user rate limiting, lockout, TOTP MFA, audit logging.
Modules live in `backend/app/auth/`. Full module map, DB models, and provider
setup: [docs/claude-reference.md](docs/claude-reference.md) and
`docs/KEYCLOAK_SETUP.md` / `docs/PKI_SETUP.md`.

## Development Guidelines

### Code Quality
- Keep files under 200-300 lines
- Use Google-style docstrings for Python code
- Follow existing patterns before creating new ones
- Always check for TypeScript errors
- Ensure light/dark mode compliance for frontend changes

### Docker and Services
- Use `docker compose` (not `docker-compose`)
- **Docker Compose Structure** (base + override pattern):
  - `docker-compose.yml` - Base configuration (all environments)
  - `docker-compose.override.yml` - Development overrides (auto-loaded)
  - `docker-compose.prod.yml` - Production overrides
  - `docker-compose.offline.yml` - Offline/airgapped overrides
- **Development**: Just run `docker compose up` (auto-loads override)
- **Production**: Use `-f docker-compose.yml -f docker-compose.prod.yml`
- **Offline**: Use `-f docker-compose.yml -f docker-compose.offline.yml`
- Always check container logs after starting services
- Kill existing servers before testing changes
- Layer Docker files for optimal caching

### Testing and Deployment
- Write thorough tests for major functionality
- No mocking data for dev/prod (tests only)
- Always restart/reset services after making changes
- Use appropriate opentr.sh commands for testing changes

## Service Endpoints

### Development URLs
- Frontend: http://localhost:5173
- Backend API: http://localhost:5174/api
- API Docs: http://localhost:5174/docs
- MinIO Console: http://localhost:5179
- Flower Dashboard: http://localhost:5175/flower
- OpenSearch: http://localhost:5180 (v3.3.1 with Lucene 10)

### Important File Locations
- Environment config: `.env` (never overwrite without confirmation)
- Environment template: `.env.example` (template for all installations)
- Database init: `database/init_db.sql`
- Docker base config: `docker-compose.yml` (common to all environments)
- Docker dev config: `docker-compose.override.yml` (auto-loaded in dev)
- Docker prod config: `docker-compose.prod.yml` (production overrides)
- Docker offline config: `docker-compose.offline.yml` (airgapped overrides)
- Frontend build: `frontend/vite.config.ts`

## AI Processing (summary)

Pipeline: upload → MinIO → Celery (GPU) → WhisperX transcription (100+ languages,
configurable source/translate) → PyAnnote diarization → optional LLM speaker
suggestions + BLUF summarization (12 output languages; provider via `LLM_PROVIDER`:
vllm/openai/ollama/anthropic/openrouter, or empty for transcription-only) →
OpenSearch indexing → WebSocket notify. yt-dlp ingestion from 1800+ platforms
(max 4h / 15GB). Models cache to `MODEL_CACHE_DIR` volumes (~2.6GB, persists
across restarts); diarization `MAX_SPEAKERS` raisable to 50+ for conferences.
Backend containers run as non-root `appuser` (UID 1000) — permission fixes via
`./scripts/fix-model-permissions.sh`.

Full detail (multi-GPU scaling, Docker build & push, LLM features, multilingual,
media-URL support, model cache layout, security): see
[docs/claude-reference.md](docs/claude-reference.md).

## Common Tasks

### Adding New API Endpoints
1. Create endpoint in `backend/app/api/endpoints/`
2. Add to router in `backend/app/api/router.py`
3. Create/update schemas in `backend/app/schemas/`
4. Update database models if needed
5. Test with `./opentr.sh restart-backend`

### Frontend Component Development
1. Create component in `src/components/`
2. Ensure light/dark mode support
3. Test responsive design
4. Update relevant routes/stores if needed
5. Test with `./opentr.sh restart-frontend`

### Database Changes
1. Modify `database/init_db.sql`
2. Update SQLAlchemy models
3. Update Pydantic schemas
4. Reset dev environment: `./opentr.sh reset dev`
