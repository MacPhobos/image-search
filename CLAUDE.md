# Image Search Monorepo - Claude Code Guide

> **Last Updated**: 2026-01-05
> **Purpose**: Single-path coordination for AI-powered semantic image search system

---

## 🔴 CRITICAL: Monorepo Structure

This is a **monorepo** with two independent but coordinated subprojects:

```
image-search/
├── image-search-service/    # Python FastAPI backend
│   └── CLAUDE.md           # Backend-specific instructions
├── image-search-ui/         # SvelteKit 5 frontend
│   └── CLAUDE.md           # Frontend-specific instructions
├── docs/                    # Shared documentation
└── CLAUDE.md               # THIS FILE (coordination rules)
```

**Navigation Rules**:
- **Backend work** → Read `image-search-service/CLAUDE.md` FIRST
- **Frontend work** → Read `image-search-ui/CLAUDE.md` FIRST
- **Cross-project changes** → Read this file FIRST, then both subproject guides

---

## 🔴 Priority System

Instructions are marked with priority indicators:

- 🔴 **CRITICAL** - Must never be violated (API contracts, data safety, type safety)
- 🟡 **IMPORTANT** - Strong conventions (testing, code style, workflow)
- 🟢 **RECOMMENDED** - Best practices (performance, maintainability)
- ⚪ **OPTIONAL** - Nice-to-have improvements

---

## 🔴 The Golden Rule: API Contract is FROZEN

**Contract Location**: `docs/api-contract.md` exists in **BOTH** subprojects (must be identical)

### Contract Synchronization Workflow (ONE WAY ONLY)

```bash
# STEP 1: Update backend contract
cd image-search-service
# Edit docs/api-contract.md with new endpoint/model
# Update Pydantic models in src/image_search_service/api/schemas.py
# Run backend tests
make test

# STEP 2: Verify OpenAPI generation
make dev  # Start server
curl http://localhost:8000/openapi.json | jq '.paths' # Verify endpoint exists

# STEP 3: Copy contract to frontend
cp image-search-service/docs/api-contract.md image-search-ui/docs/api-contract.md

# STEP 4: Regenerate frontend types
cd image-search-ui
npm run gen:api  # Generates src/lib/api/generated.ts from OpenAPI spec

# STEP 5: Update frontend code to use new types
# Import from src/lib/api/generated.ts
# TypeScript will catch any mismatches

# STEP 6: Run frontend tests
npm run test
```

**🔴 NEVER**:
- Manually edit `image-search-ui/src/lib/api/generated.ts` (auto-generated)
- Change API without updating `docs/api-contract.md` in BOTH repos
- Deploy frontend without regenerating types after backend changes
- Break backward compatibility without version bump (v1 → v2)

**🟡 Contract Change Checklist**:
- [ ] Update `image-search-service/docs/api-contract.md`
- [ ] Update Pydantic models (backend)
- [ ] Backend tests pass (`make test`)
- [ ] Copy contract to `image-search-ui/docs/api-contract.md`
- [ ] Regenerate types (`npm run gen:api`)
- [ ] Update frontend code using new types
- [ ] Frontend tests pass (`npm run test`)

---

## 🔴 Development Workflow (Single Path)

### Initial Setup (First Time Only)

```bash
# 1. Clone repository
git clone <repo-url> image-search
cd image-search

# 2. Backend setup
cd image-search-service
uv sync --dev           # Install Python dependencies (fast!)
make db-up              # Start Postgres, Redis, Qdrant via Docker
make migrate            # Run database migrations
make bootstrap-qdrant   # Initialize Qdrant collections

# 3. Frontend setup (separate terminal)
cd ../image-search-ui
npm ci                  # Install Node dependencies (CI-safe)
npm run gen:api         # Generate TypeScript types from backend

# 4. Verify setup
# Terminal 1: Backend
cd image-search-service && make dev    # http://localhost:8000
# Terminal 2: Worker
cd image-search-service && make worker # Background job processor
# Terminal 3: Frontend
cd image-search-ui && npm run dev      # http://localhost:5173
```

### Daily Development Workflow

```bash
# Start all services (3 terminals)
# Terminal 1: Backend API
cd image-search-service && make dev

# Terminal 2: Background worker
cd image-search-service && make worker

# Terminal 3: Frontend
cd image-search-ui && npm run dev
```

**Access Points**:
- UI Dashboard: http://localhost:5173
- API Docs: http://localhost:8000/docs (interactive Swagger)
- API Health: http://localhost:8000/health
- OpenAPI Spec: http://localhost:8000/openapi.json

---

## 🟡 Quality Standards (Pre-Commit Requirements)

### Backend Quality Checks

```bash
cd image-search-service

make lint       # Ruff linting (PEP 8 + more)
make typecheck  # MyPy strict mode (100% coverage)
make test       # Pytest (must work WITHOUT external services)
make format     # Ruff auto-fix + format
```

**🔴 Backend Requirements**:
- All code under `src/` (src-layout)
- 100% type coverage (mypy strict mode)
- No import-time side effects (lazy initialization)
- Tests must pass without Postgres/Redis/Qdrant running
- Every feature change includes test updates in same PR

### Frontend Quality Checks

```bash
cd image-search-ui

npm run lint    # ESLint 9 + typescript-eslint
npm run test    # Vitest + Testing Library (happy-dom)
npm run format  # Prettier + prettier-plugin-svelte
```

**🔴 Frontend Requirements**:
- NEVER manually edit `src/lib/api/generated.ts`
- Use Testing Library queries (semantic, not test IDs)
- Centralized mocking via `tests/helpers/mockFetch.ts`
- Tests must pass without backend running
- Every feature change includes test updates in same PR

### Pre-Commit Validation (Both Projects)

```bash
# Backend
cd image-search-service
make lint && make typecheck && make test

# Frontend
cd image-search-ui
npm run lint && npm run test
```

---

## 🟡 Common Development Tasks

### Task: Add New Backend Endpoint

```bash
cd image-search-service

# 1. Update API contract
vim docs/api-contract.md  # Add endpoint spec with types

# 2. Create Pydantic schemas
vim src/image_search_service/api/schemas.py  # Add request/response models

# 3. Create route handler
vim src/image_search_service/api/routes/new_feature.py

# 4. Register router
vim src/image_search_service/main.py  # app.include_router(...)

# 5. Add tests
vim tests/api/test_new_feature.py

# 6. Run quality checks
make lint && make typecheck && make test

# 7. Sync to frontend (see "Contract Synchronization" above)
```

### Task: Add New Frontend Component

```bash
cd image-search-ui

# 1. Create component
vim src/lib/components/NewComponent.svelte

# 2. Add component test
vim src/tests/components/NewComponent.test.ts

# 3. Use in page/route
vim src/routes/+page.svelte  # Import and use component

# 4. Run quality checks
npm run lint && npm run test
```

### Task: Add Database Model Field

```bash
cd image-search-service

# 1. Update SQLAlchemy model
vim src/image_search_service/db/models.py

# 2. Generate migration
make makemigrations  # Prompts for migration message

# 3. Review migration SQL
ls db/migrations/versions/  # Check latest migration file

# 4. Apply migration
make migrate

# 5. Update Pydantic schemas if field is exposed via API
vim src/image_search_service/api/schemas.py

# 6. If API changed: Sync contract to frontend (see above)
```

### Task: Run Face Recognition Pipeline

```bash
cd image-search-service

# Full automated pipeline (detect → cluster → stats)
make faces-pipeline-dual

# Or run steps individually:
make faces-ensure-collection  # Setup Qdrant
make faces-backfill LIMIT=5000  # Detect faces (default: 5000 images)
make faces-cluster-dual  # Cluster unknowns + match to known people
make faces-stats  # Show statistics
```

**🟢 Face Pipeline Parameters**:
- `LIMIT=N` - Process N images (default: 5000)
- `BATCH_SIZE=N` - Batch size for detection (default: 8)
- `PERSON_THRESHOLD=0.7` - Similarity threshold for known people
- `UNKNOWN_METHOD=hdbscan` - Clustering algorithm (hdbscan/kmeans)

---

## 🟡 Project Architecture

### Tech Stack Summary

| Layer | Backend | Frontend |
|-------|---------|----------|
| **Language** | Python 3.12+ | TypeScript (strict) |
| **Framework** | FastAPI (async) | SvelteKit 2 + Svelte 5 |
| **Database** | PostgreSQL + SQLAlchemy 2.0 | N/A (API client) |
| **Vector DB** | Qdrant (similarity search) | N/A |
| **Queue** | Redis + RQ (4-tier priority) | N/A |
| **Testing** | Pytest (async) | Vitest + Testing Library |
| **Linting** | Ruff | ESLint 9 |
| **Type Check** | MyPy (strict) | TypeScript (strict) |
| **Package Mgr** | uv (fast!) | npm |

### Data Flow (High Level)

```
User uploads photo → FastAPI ingests → PostgreSQL metadata
                                    ↓
                              Background job (RQ)
                                    ↓
                    OpenCLIP embeddings + InsightFace faces
                                    ↓
                              Qdrant vectors
                                    ↓
User searches text → OpenCLIP embedding → Qdrant similarity search
                                         ↓
                                   PostgreSQL enrichment
                                         ↓
                                   API response → SvelteKit UI
```

### Key Engineering Patterns

**Backend**:
- **Lazy Initialization** - No DB/Redis/Qdrant connections at import time
- **Dependency Injection** - FastAPI `Depends()` for testability
- **Graceful Degradation** - Health check works without external services
- **Structured Logging** - Context-aware logs with asset_id, job_id
- **Async I/O** - All DB/network operations are async

**Frontend**:
- **Auto-Generated Types** - OpenAPI → TypeScript (compile-time safety)
- **Svelte 5 Runes** - `$state`, `$derived`, `$effect` (fine-grained reactivity)
- **Centralized Mocking** - `tests/helpers/mockFetch.ts` for consistent tests
- **Semantic Queries** - Testing Library (accessibility-focused)
- **No Global State** - Local page state (stores when needed)

---

## 🟢 Testing Philosophy

### Backend Testing

**Zero External Dependencies**:
- Tests use in-memory SQLite (not Postgres)
- Redis/Qdrant mocked via dependency injection
- OpenCLIP/InsightFace models mocked

**Test Structure**:
```
tests/
├── api/           # Route/endpoint tests
├── unit/          # Business logic tests
├── conftest.py    # Shared fixtures
└── test_*.py      # Test files
```

**Naming Convention**:
- Files: `test_{module}.py`
- Functions: `test_{behavior}_when_{condition}_then_{result}()`

### Frontend Testing

**No Backend Required**:
- API calls mocked via `mockFetch` helpers
- Uses `happy-dom` (lightweight DOM simulation)
- Fixtures provide consistent test data

**Test Structure**:
```
src/tests/
├── components/    # Component unit tests
├── routes/        # Page integration tests
├── helpers/       # Mock utilities + fixtures
└── api-client.test.ts
```

**Testing Rules**:
- Use Testing Library queries (`getByRole`, `getByLabelText`)
- Avoid `getByTestId` unless no semantic alternative
- Mock fetch via centralized `mockFetch` helpers
- Use fixtures from `helpers/fixtures.ts` for API types

---

## 🟢 Performance Considerations

### Backend Performance

- **Vector Search** - Sub-second on 10k+ images via Qdrant HNSW index
- **Face Detection** - Batch processing (8 images/batch default)
- **Background Jobs** - 4-tier priority queue (high → normal → low → default)
- **Database** - Async I/O via asyncpg (non-blocking)
- **Connection Pooling** - SQLAlchemy pool (5 connections default)

### Frontend Performance

- **Server-Side Rendering** - SvelteKit SSR for initial page load
- **Code Splitting** - Automatic route-based splitting
- **Type-Safe API** - Catch errors at compile time (no runtime overhead)
- **Hot Module Replacement** - Instant dev feedback

---

## 🟢 Observability and Debugging

### Backend Debugging

```bash
# View logs (structured JSON format)
cd image-search-service
make dev  # Logs to stdout

# Check background job queue
redis-cli
> LLEN rq:queue:default  # Queue length
> LLEN rq:queue:high     # High-priority queue

# Check database migrations
make migrate  # Apply pending migrations
alembic current  # Show current version
alembic history  # Show migration history

# Verify Qdrant collections
make verify-qdrant
```

### Frontend Debugging

```bash
# Component tests (watch mode)
npm run test:watch

# View API calls (browser DevTools Network tab)
npm run dev
# Open http://localhost:5173
# Check Network → Filter by "fetch"

# Type checking (watch mode)
npm run dev  # Vite shows TypeScript errors in browser
```

### Health Monitoring

```bash
# Backend health (works WITHOUT external services)
curl http://localhost:8000/health
# Returns: {"status": "healthy", "version": "1.9.0"}

# Detailed service status
curl http://localhost:8000/api/v1/health/detailed
# Returns: {"database": "ok", "redis": "ok", "qdrant": "ok"}
```

---

## 🔴 Security Considerations

### Current State (Development)

- **No authentication** - Optional API keys planned for production
- **CORS enabled** - Configured for `localhost:5173` in dev
- **Path validation** - All file paths validated against allowed roots
- **Input validation** - Pydantic models validate all API inputs

### Future (Production Roadmap)

- [ ] API key authentication (optional in dev, required in prod)
- [ ] Role-based access control (RBAC)
- [ ] Rate limiting (per-client quotas)
- [ ] HTTPS enforcement
- [ ] Environment variable for ALLOWED_ORIGINS

**🔴 Never Commit**:
- API keys, credentials, secrets
- `.env` files with real credentials
- Database dumps with personal photos

---

## 🟡 Memory System Integration (KuzuMemory)

This project uses **KuzuMemory** for AI context management.

### Memory Commands

```bash
# Enhance prompts with project context
kuzu-memory enhance "How do I add a new API endpoint?"

# Store learning asynchronously
kuzu-memory learn "Face clustering uses HDBSCAN with min_cluster_size=3"

# Query project memories
kuzu-memory recall "How does face recognition work?"

# View statistics
kuzu-memory stats
```

### MCP Tools (Claude Desktop)

When interacting via Claude Desktop, these tools are available:
- `kuzu_enhance` - Enhance prompts with project memories
- `kuzu_learn` - Store new learnings asynchronously
- `kuzu_recall` - Query specific memories
- `kuzu_stats` - Get memory system statistics

**🟢 What to Store**:
- Project-specific decisions (API contract changes, architecture choices)
- Technical specifications (embedding dimensions, clustering params)
- User preferences (preferred workflows, team conventions)
- Error solutions (how bugs were fixed)

**🟢 What NOT to Store**:
- Generic programming knowledge (Python syntax, TypeScript basics)
- Obvious information (README content, public docs)
- Temporary debugging notes

---

## 🟡 Contribution Guidelines

### Before Starting Work

1. Read this file (`CLAUDE.md` at root)
2. Read subproject guide (`image-search-service/CLAUDE.md` or `image-search-ui/CLAUDE.md`)
3. Check `docs/api-contract.md` if API changes needed
4. Run existing tests to verify setup

### Pull Request Checklist

- [ ] All quality checks pass (lint + typecheck + test)
- [ ] API contract updated if endpoints changed
- [ ] Frontend types regenerated if backend changed
- [ ] Tests added/updated in same PR as feature
- [ ] Both subproject guides updated if workflows changed
- [ ] Commit message follows conventional commits (`feat:`, `fix:`, `docs:`, etc.)

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`
**Scopes**: `api`, `ui`, `db`, `queue`, `faces`, `search`, `contract`

**Examples**:
- `feat(api): add temporal prototype endpoints for face recognition`
- `fix(ui): correct pagination offset calculation in search results`
- `docs(contract): update face cluster response schema for v1.10`
- `refactor(faces): extract HDBSCAN clustering to separate service`
- `test(api): add integration tests for /api/v1/search endpoint`

---

## 🟢 Troubleshooting Common Issues

### Backend Issues

**Problem**: `ModuleNotFoundError: No module named 'image_search_service'`
**Solution**: Install dependencies with `uv sync --dev`, activate venv with `source .venv/bin/activate`

**Problem**: `sqlalchemy.exc.OperationalError: could not connect to server`
**Solution**: Start Postgres with `make db-up`, check with `docker ps`

**Problem**: Migration fails with "target database is not empty"
**Solution**: `make db-down && make db-up && make migrate` (fresh start)

**Problem**: Qdrant connection refused
**Solution**: Check `docker-compose.dev.yml` includes Qdrant, verify port 6333 available

### Frontend Issues

**Problem**: TypeScript errors about missing API types
**Solution**: Regenerate types with `npm run gen:api` (requires backend running)

**Problem**: Tests fail with fetch mocking errors
**Solution**: Use `mockResponse()` from `tests/helpers/mockFetch.ts`, check `setup.ts` loaded

**Problem**: `VITE_API_BASE_URL` not working
**Solution**: Create `.env.local` with `VITE_API_BASE_URL=http://localhost:8000`

### Cross-Project Issues

**Problem**: Frontend shows 404 for new endpoint
**Solution**: Verify backend running, check `/openapi.json` includes endpoint, regenerate types

**Problem**: API response doesn't match TypeScript types
**Solution**: Check `docs/api-contract.md` matches Pydantic models, regenerate types

---

## 🟢 Useful References

### Documentation Files

- **This File**: Monorepo coordination and cross-project workflows
- **Backend Guide**: `image-search-service/CLAUDE.md` (Python/FastAPI specifics)
- **Frontend Guide**: `image-search-ui/CLAUDE.md` (SvelteKit/Svelte 5 specifics)
- **API Contract**: `docs/api-contract.md` (in both subprojects, must match)
- **Main README**: `README.md` (project overview, features, architecture)

### External Documentation

- **FastAPI**: https://fastapi.tiangolo.com/
- **SvelteKit**: https://kit.svelte.dev/docs
- **Svelte 5 Runes**: https://svelte.dev/docs/svelte/$state
- **Qdrant**: https://qdrant.tech/documentation/
- **OpenCLIP**: https://github.com/mlfoundations/open_clip
- **InsightFace**: https://github.com/deepinsight/insightface

### Quick Links (Localhost)

- UI: http://localhost:5173
- API Docs: http://localhost:8000/docs
- OpenAPI Spec: http://localhost:8000/openapi.json
- Health Check: http://localhost:8000/health

---

## 🟢 Roadmap and Future Work

**Planned Features** (See README.md for full roadmap):
- Authentication (API keys, JWT tokens)
- Advanced filters (date ranges, location-based)
- User corrections (feedback system)
- Mobile apps (React Native or PWA)
- Multi-user support (RBAC)

**Technical Improvements**:
- EXIF data extraction (taken_at timestamps)
- Duplicate detection (perceptual hashing)
- Video support (frame extraction)
- Performance optimization (100k+ photos)
- Monitoring (Prometheus + Grafana)

---

## 📋 Quick Reference Card

```bash
# INITIAL SETUP (once)
cd image-search-service && uv sync --dev && make db-up && make migrate
cd image-search-ui && npm ci && npm run gen:api

# DAILY WORKFLOW (3 terminals)
cd image-search-service && make dev     # Terminal 1: API
cd image-search-service && make worker  # Terminal 2: Jobs
cd image-search-ui && npm run dev       # Terminal 3: UI

# QUALITY CHECKS (before commit)
cd image-search-service && make lint && make typecheck && make test
cd image-search-ui && npm run lint && npm run test

# API CONTRACT SYNC (after backend changes)
cp image-search-service/docs/api-contract.md image-search-ui/docs/
cd image-search-ui && npm run gen:api

# COMMON TASKS
make faces-pipeline-dual  # Run face recognition
make ingest DIR=/path     # Import photos
make verify-qdrant        # Check vector DB
```

---

**Last Updated**: 2026-01-05
**Maintained By**: AI Assistants + Human Developers
**Questions?** Check subproject CLAUDE.md files or open an issue.
