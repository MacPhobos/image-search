# Production Isolation Safety Audit - image-search-service Test Suite

**Date**: 2026-02-06
**Auditor**: Claude Opus 4.6 (Research Agent)
**Scope**: All test infrastructure, fixtures, plans, and identified risk vectors
**Verdict**: CONDITIONALLY SAFE with 2 actionable risks requiring remediation

---

## 1. Executive Summary

The image-search-service test suite has **strong production isolation** across its three external dependency boundaries (PostgreSQL, Qdrant, Redis). The architecture uses FastAPI dependency injection overrides, in-memory substitutes, and autouse safety fixtures to prevent test code from touching production data.

However, this audit identified **two concrete risk vectors** that weaken the safety posture:

| Risk | Severity | Status | Remediation |
|------|----------|--------|-------------|
| R1: Two test files bypass dependency overrides via `TestClient(app)` | **MEDIUM** | Active | Refactor to use `test_client` fixture |
| R2: `.env` file with real service URLs loaded at import time | **LOW** | Mitigated by fixtures | Document explicitly; consider `pytest.ini` env override |

The remaining isolation mechanisms are well-designed and defense-in-depth:
- SQLite in-memory database substitution (complete DB isolation)
- Qdrant in-memory client substitution (complete vector DB isolation)
- Redis/RQ mock substitution (complete queue isolation)
- Autouse fixtures that clear settings cache and override collection names
- Production safety guards that block destructive operations during tests
- Dedicated regression tests validating the safety guards themselves

---

## 2. Database Isolation Assessment

### 2.1 Mechanism

**Primary isolation**: The test suite uses SQLite in-memory databases instead of PostgreSQL.

```python
# tests/conftest.py, lines 29-30
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
TEST_SYNC_DATABASE_URL = "sqlite:///:memory:"
```

**Injection point**: The `test_client` fixture (lines 272-328) overrides both `get_db` and `get_sync_db` FastAPI dependencies:

```python
app.dependency_overrides[get_db] = override_get_db       # Async SQLite session
app.dependency_overrides[get_sync_db] = override_get_sync_db  # Sync SQLite session
```

### 2.2 Strengths

1. **Complete protocol swap**: Uses `sqlite+aiosqlite://` instead of `postgresql+asyncpg://`. Even if settings leak, the SQLAlchemy engine URL is overridden at the fixture level.
2. **Per-test isolation**: Each test gets a fresh session with rollback (`await session.rollback()` in `db_session` fixture, line 139).
3. **Schema creation**: Tables are created from SQLAlchemy metadata at test start (`Base.metadata.create_all`), then dropped at teardown.
4. **Both async and sync paths covered**: Separate fixtures for async (`db_session`) and sync (`sync_db_session`) sessions, matching the dual-session architecture of the production code.

### 2.3 Weaknesses

1. **SQLite/PostgreSQL behavioral differences**: SQLite lacks PostgreSQL-specific features (JSON operators, array types, `SELECT FOR UPDATE`, partial indexes). Tests may pass on SQLite but fail on PostgreSQL. This is a *test fidelity* issue, not a *production safety* issue.
2. **The `test_client` fixture must be explicitly used**: Any test that creates its own client (e.g., `TestClient(app)`) bypasses the database override. See Risk Vector R1.

### 2.4 Verdict: SAFE

Database isolation is strong. Production PostgreSQL is never contacted during normal test execution. The SQLite substitution is injected at the dependency level, making it impossible for properly-written tests to accidentally connect to production.

---

## 3. Qdrant Isolation Assessment

### 3.1 Mechanism

**Primary isolation**: Three-layer defense:

**Layer 1 - Autouse settings override** (lines 47-65):
```python
@pytest.fixture(autouse=True)
def use_test_settings(monkeypatch):
    get_settings.cache_clear()
    monkeypatch.setenv("QDRANT_COLLECTION", "test_image_assets")
    monkeypatch.setenv("QDRANT_FACE_COLLECTION", "test_faces")
    get_settings.cache_clear()
```

This ensures that even if a Qdrant client is accidentally initialized from settings, it uses collection names prefixed with `test_` instead of production names (`image_assets`, `faces`).

**Layer 2 - In-memory Qdrant client** (lines 177-199):
```python
@pytest.fixture
def qdrant_client() -> QdrantClient:
    client = QdrantClient(":memory:")
    # Creates test_image_assets and test_faces collections
```

The `test_client` fixture injects this in-memory client via dependency override:
```python
app.dependency_overrides[get_qdrant_client] = override_get_qdrant
app.dependency_overrides[get_face_qdrant_client] = override_get_face_qdrant
```

**Layer 3 - Production safety guards** (source code):

In `src/image_search_service/vector/qdrant.py` (line 668):
```python
if os.getenv("PYTEST_CURRENT_TEST") and collection_name in ["image_assets", "faces"]:
    raise RuntimeError(
        f"Refusing to delete production collection '{collection_name}' during tests."
    )
```

In `src/image_search_service/vector/face_qdrant.py` (line 814):
```python
if os.getenv("PYTEST_CURRENT_TEST"):
    if collection_name == "faces":
        raise RuntimeError(
            "SAFETY GUARD: Refusing to delete production 'faces' collection during tests."
        )
```

### 3.2 Strengths

1. **Defense-in-depth**: Three independent layers must all fail simultaneously for production data to be at risk.
2. **Autouse fixtures**: The `clear_settings_cache` and `use_test_settings` fixtures run for EVERY test, even those that do not explicitly request them. This prevents cached production settings from leaking.
3. **PYTEST_CURRENT_TEST sentinel**: Pytest automatically sets this environment variable. The production code checks it before destructive operations, providing a last-resort safety net.
4. **Dedicated regression tests**: `tests/unit/test_qdrant_safety.py` (161 lines) explicitly validates that:
   - Collection names come from environment variables, not hardcoded values
   - Production collection reset is blocked during tests
   - Autouse fixtures correctly set `test_` prefixed names

### 3.3 Weaknesses

1. **Safety guards only protect `reset_collection()`**: The `PYTEST_CURRENT_TEST` check is on the `reset_collection` / destructive path. Other operations (upsert, delete points, scroll) do not have safety guards. However, since the in-memory client is injected via dependency override, these operations target the in-memory store, not production.
2. **`FaceQdrantClient` is a singleton with lazy init**: If a test somehow triggers `FaceQdrantClient.__init__` without the fixture override, it would connect to the real Qdrant URL from settings. The `use_test_settings` fixture mitigates collection name risk but not connection risk.
3. **Hardcoded collection name in `dual_clusterer.py`**: Plan P2-06 notes that `_get_face_embedding()` in `dual_clusterer.py` hardcodes `collection_name="faces"` instead of using `_get_face_collection_name()`. If this code path is exercised during tests with a real Qdrant client, it would target the production collection name.

### 3.4 Verdict: SAFE (with defense-in-depth)

Qdrant isolation is the strongest of the three boundaries. The three-layer defense (settings override + in-memory client + production safety guards) makes accidental production data modification extremely unlikely. The regression tests in `test_qdrant_safety.py` provide ongoing validation.

---

## 4. Redis/RQ Isolation Assessment

### 4.1 Mechanism

**Primary isolation**: Mock queue via `MockQueue` class (lines 331-351):
```python
class MockQueue:
    def enqueue(self, func, *args, **kwargs):
        jobs.append({"func": func, "args": args, "kwargs": kwargs})
```

Tests that need queue interaction use `monkeypatch` to replace `get_queue_service` with the mock.

### 4.2 Strengths

1. **No real Redis connection**: The `MockQueue` is a pure Python object. No Redis URL is resolved, no TCP connection is established.
2. **Job tracking**: Enqueued jobs are recorded in a list, allowing tests to assert that the correct jobs were enqueued without actually executing them.
3. **Background job tests use sync fixtures**: Face and training job tests use `sync_db_session` with SQLite, not production PostgreSQL. Redis progress tracking is mocked.

### 4.3 Weaknesses

1. **MockQueue does not validate function signatures**: An enqueued function reference could be misspelled or have the wrong signature, and the mock would silently record it. This is a fidelity issue, not a safety issue.
2. **Some tests may not mock Redis at all**: If a test creates a `TestClient(app)` without overrides (see R1), and the tested endpoint enqueues a job, the code will attempt to connect to the real Redis instance from `.env`. This would typically fail with a `ConnectionRefusedError` rather than silently touching production data.

### 4.4 Verdict: SAFE

Redis/RQ isolation is adequate. The mock queue prevents any real Redis interaction. Even in the worst case (unmocked code attempting Redis), the result would be a connection error, not silent data corruption.

---

## 5. Test Run Safety Verification

### 5.1 Test Execution Analysis

From `docs/research/service-testing/test-execution-analysis.md`:
- **795 total tests**, 744 passed (93.6%), 35 failed, 5 errors, 11 skipped
- **48% code coverage** (6001 missed of 11440 statements)

### 5.2 Safety-Relevant Failure Categories

Of the 40 non-passing tests, the following have production isolation implications:

**Category D (5 failures)**: `tests/unit/api/test_config_post_training_suggestions.py` uses `TestClient(app)` bypassing dependency overrides. These tests attempt to connect to production-configured services. They FAIL rather than silently connecting, because the services are typically not running during test execution.

**Category C (8 failures)**: `tests/integration/test_restart_workflows.py` hits `MissingGreenlet` errors due to async/sync session mismatch. The `FaceQdrantClient.get_instance()` singleton is mocked but the mocking approach has gaps. These tests FAIL rather than touching production data.

**Category B (2 failures)**: `tests/faces/test_service.py` loads the real InsightFace model (`buffalo_l`). This is a resource/performance concern, not a production data safety issue. The model processes local test images, not production data.

### 5.3 Verdict: SAFE

No test failures indicate actual production data access. Failures are due to missing mocks, stale test code, or environment mismatches. The failure mode is "test crashes" not "test silently touches production."

---

## 6. Review of 7 Service Testing Plans for Safety Gaps

### 6.1 P0-01: Enable CI Gating

**Safety Assessment**: MINOR GAP

The CI workflow defines service containers for PostgreSQL and Redis:
```yaml
services:
  postgres: ...
  redis: ...
```

Environment variables are set to point to these CI-local services:
```yaml
env:
  DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test_image_search
  REDIS_URL: redis://localhost:6379
  QDRANT_URL: http://localhost:6333
```

**Gap**: `QDRANT_URL` is set but no Qdrant service container is defined. If any test bypasses the in-memory Qdrant fixture and tries to connect to `localhost:6333`, it would fail with connection refused (safe) in CI. However, if a developer runs these env vars locally where a real Qdrant instance IS running on port 6333, the test could connect to it.

**Mitigation**: The autouse `use_test_settings` fixture overrides collection names to `test_*` prefixes, so even if a connection is established, it would target test collections, not production ones.

**Recommendation**: Add `QDRANT_COLLECTION=test_image_assets` and `QDRANT_FACE_COLLECTION=test_faces` to CI env vars explicitly, duplicating the autouse fixture protection.

### 6.2 P0-02: Fix 35 Failing Tests

**Safety Assessment**: SAFE, with one important note

**Category D fix** (lines 396-464): This plan correctly identifies the `TestClient(app)` bypass issue and prescribes refactoring to use the `test_client` fixture. Implementing this fix would eliminate Risk Vector R1 for the post-training suggestions tests.

**Category H fix** (lines 499-568): This plan creates a new Alembic migration. This is the only plan that modifies production code. The migration uses `ON CONFLICT (key) DO NOTHING` which is idempotent and safe.

**No safety regressions introduced**: All fixes are test-only changes (except the migration) that improve isolation, not weaken it.

### 6.3 P1-03: Race Condition and Concurrency Tests

**Safety Assessment**: SAFE

All proposed tests use mocked database sessions (`AsyncMock`) and patched Qdrant clients. The race condition tests simulate concurrent requests using `asyncio.gather` with mock objects, never connecting to real services.

The plan explicitly states: "Since the existing test infrastructure uses mocked database sessions (in-memory SQLite, not real PostgreSQL), true concurrent transactions are not possible."

**No safety concerns**: These are pure mock-based tests documenting existing vulnerabilities.

### 6.4 P1-04: Security Testing

**Safety Assessment**: SAFE, with a CORS test caveat

The path traversal tests use `pytest.tmp_path` for filesystem operations (safe -- temporary directories cleaned up by pytest).

The CORS tests create fresh `app = create_app()` instances with mocked settings. **Caveat**: The `TestClient(app)` usage in CORS tests (line 679) does NOT inject database/Qdrant overrides. However, the CORS tests only make OPTIONS requests to `/api/v1/health`, which does not require database access.

The information leakage tests use fully mocked database sessions (`AsyncMock`).

**No safety concerns for production data**.

### 6.5 P2-05: PostgreSQL Test Tier

**Safety Assessment**: SAFE

This plan uses `testcontainers-python` to spin up isolated PostgreSQL containers with random ports:
```python
@pytest.fixture(scope="session")
def pg_container():
    with PostgresContainer("postgres:16-alpine") as postgres:
        yield postgres
```

**Strengths**:
- Containers are ephemeral (destroyed after tests)
- Random ports prevent collision with production databases
- Per-test rollback via `SAVEPOINT` mechanism
- Fresh database creation for migration tests

**No risk of touching production PostgreSQL**: The containers are completely independent Docker instances.

### 6.6 P2-06: Critical Path Coverage

**Safety Assessment**: SAFE, with monkeypatch dependency

All proposed tests use `monkeypatch.setattr` to inject in-memory Qdrant clients and mock dependencies:
```python
monkeypatch.setattr(
    "image_search_service.faces.dual_clusterer.get_face_qdrant_client",
    lambda: face_qdrant,  # In-memory client
)
```

**Dependency on correct patch paths**: If the patch path is wrong, the mock is not applied and the real singleton might be used. However, the autouse `use_test_settings` fixture still protects collection names, and the in-memory Qdrant client fixture provides a second layer.

**Note on hardcoded collection name**: The plan identifies that `dual_clusterer.py` hardcodes `collection_name="faces"` instead of using the configurable name. If a test exercises this code path with a real Qdrant connection, it would target the production `faces` collection. The in-memory client injection prevents this, but the hardcoded name is a latent risk.

### 6.7 P3-07: Fix Mock Accuracy

**Safety Assessment**: SAFE

This plan modifies only test infrastructure (mock classes, fixtures, assertions). It changes embedding dimensions from 512 to 768 for image search collections and improves mock fidelity.

**No production code changes**. No new external connections. Qdrant collection dimensions change only in the in-memory test client.

---

## 7. Specific Risk Vectors

### R1: TestClient(app) Bypasses Dependency Overrides

**Severity**: MEDIUM
**Status**: Active (2 test files affected)
**Files**:
- `tests/unit/api/test_config_unknown_clustering.py` (line 7)
- `tests/unit/api/test_config_post_training_suggestions.py` (line 12)

**Mechanism**: These files import the module-level `app` from `main.py` and create a `TestClient(app)` without dependency overrides:

```python
from image_search_service.main import app
client = TestClient(app)
```

This bypasses ALL five dependency overrides from the `test_client` fixture:
1. `get_db` -- would try production PostgreSQL
2. `get_sync_db` -- would try production PostgreSQL
3. `get_qdrant_client` -- would try production Qdrant
4. `get_face_qdrant_client` -- would try production Qdrant
5. `get_embedding_service` -- would try loading OpenCLIP

**Why this is MEDIUM not HIGH**: The config endpoints tested by these files operate on in-memory `Settings` objects, not database records. The `TestClient(app)` instantiation triggers `create_app()` which calls `get_settings()` (reading `.env`), but the actual test requests go to `/api/v1/config/*` endpoints that read/modify the in-memory settings singleton. These endpoints do NOT access the database or Qdrant.

However, the `create_app()` call triggers the `lifespan` context manager which may attempt to preload the embedding model. Additionally, if any middleware or exception handler attempts database access, it would fail.

**Current behavior**: These tests either pass (by operating on in-memory settings) or fail with connection errors (when database access is attempted). They do NOT silently modify production data.

**Remediation**: Refactor both files to use the `test_client` fixture from conftest.py, as prescribed in P0-02 Category D.

### R2: .env File with Real Service URLs

**Severity**: LOW
**Status**: Mitigated by autouse fixtures

**Mechanism**: The `.env` file contains real service URLs:
```
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:15432/image_search
REDIS_URL=redis://localhost:6379/0
QDRANT_URL=http://localhost:6333
```

The `Settings` class loads from `.env` via `SettingsConfigDict(env_file=".env")`. When `get_settings()` is called (including during `create_app()`), these URLs are loaded.

**Why this is LOW**: The autouse `clear_settings_cache` and `use_test_settings` fixtures clear and override settings for every test. The `test_client` fixture overrides all dependency functions, preventing the loaded URLs from being used to establish connections.

**Remaining concern**: The R1 risk vector (TestClient bypassing overrides) combined with R2 (real URLs in settings) creates a scenario where test code COULD establish connections to local development services. The impact is limited to the developer's local environment (not production), and the connection would typically fail if services are not running.

**Remediation**: Consider adding to `pyproject.toml` or `conftest.py`:
```python
# In conftest.py as autouse fixture:
monkeypatch.setenv("DATABASE_URL", "sqlite+aiosqlite:///:memory:")
monkeypatch.setenv("QDRANT_URL", "http://localhost:99999")  # Unreachable port
```

### R3: InsightFace Model Loading (Non-Data Risk)

**Severity**: LOW (environment risk, not data risk)
**Status**: Active

Two tests in `tests/faces/test_service.py` load the real InsightFace model (`buffalo_l`). This:
- Requires GPU model files on disk
- Takes ~7 seconds per test
- May download models from the internet

This does NOT risk production data. The model processes local test images and returns face detection results. However, it violates the project rule "Tests must pass without Postgres/Redis/Qdrant running" and is an environment dependency issue.

### R4: Hardcoded Collection Name in dual_clusterer.py (Latent Risk)

**Severity**: LOW (latent, requires multiple failures to exploit)
**Status**: Latent

Plan P2-06 identifies that `_get_face_embedding()` in `dual_clusterer.py` hardcodes `collection_name="faces"` instead of using the configurable `_get_face_collection_name()`. If:
1. A test exercises this code path, AND
2. The in-memory Qdrant override fails to inject, AND
3. A real Qdrant instance is running on `localhost:6333`

Then the test would read from the production `faces` collection. The autouse `use_test_settings` fixture sets `QDRANT_FACE_COLLECTION=test_faces`, but this hardcoded path bypasses that setting.

**Impact**: Read-only (the `_get_face_embedding` function only retrieves vectors, it does not modify them). No data corruption risk.

---

## 8. Recommendations

### 8.1 Immediate Actions (Priority: High)

1. **Fix R1**: Refactor `test_config_unknown_clustering.py` and `test_config_post_training_suggestions.py` to use the `test_client` fixture. This is already prescribed in P0-02 Category D and should be prioritized.

2. **Add test infrastructure validation**: Create a simple smoke test that verifies no real service connections are made during test execution:
   ```python
   def test_no_production_connections(monkeypatch):
       """Verify tests cannot accidentally connect to production services."""
       import socket
       original_connect = socket.socket.connect

       connections = []
       def tracking_connect(self, address):
           connections.append(address)
           return original_connect(self, address)

       monkeypatch.setattr(socket.socket, "connect", tracking_connect)
       # ... run a representative test ...
       # Assert no connections to production ports
   ```

### 8.2 Short-Term Improvements (Priority: Medium)

3. **Add CI environment variable guards**: In P0-01 CI workflow, explicitly set:
   ```yaml
   QDRANT_COLLECTION: test_image_assets
   QDRANT_FACE_COLLECTION: test_faces
   ```

4. **Fix InsightFace model loading** (P3-07 Task 4): Correct the mock path in `test_service.py` so the two batch tests do not load the real GPU model.

5. **Fix hardcoded collection name**: Change `dual_clusterer.py` to use `_get_face_collection_name()` instead of hardcoded `"faces"`.

### 8.3 Long-Term Improvements (Priority: Low)

6. **Consider `pytest.ini` environment overrides**: Set safe default environment variables in `pyproject.toml` under `[tool.pytest.ini_options]`:
   ```toml
   [tool.pytest.ini_options]
   env = [
       "DATABASE_URL=sqlite+aiosqlite:///:memory:",
       "QDRANT_URL=http://localhost:99999",
       "REDIS_URL=redis://localhost:99999",
   ]
   ```
   This requires the `pytest-env` plugin but provides belt-and-suspenders protection.

7. **Add a pre-test safety assertion**: In `conftest.py`, add a session-scoped fixture that validates settings are not pointing to production:
   ```python
   @pytest.fixture(scope="session", autouse=True)
   def verify_test_environment():
       """Fail fast if settings somehow point to production."""
       settings = get_settings()
       assert "test_" in settings.qdrant_collection, "SAFETY: Qdrant collection must have test_ prefix"
       assert "test_" in settings.qdrant_face_collection, "SAFETY: Face collection must have test_ prefix"
   ```

---

## Appendix A: Files Examined

| File | Purpose | Safety Relevance |
|------|---------|-----------------|
| `tests/conftest.py` | Root test fixtures | PRIMARY: All isolation mechanisms defined here |
| `tests/faces/conftest.py` | Face-specific fixtures | Mock InsightFace model |
| `tests/unit/test_qdrant_safety.py` | Safety regression tests | Validates Qdrant isolation guards |
| `tests/unit/test_bootstrap_qdrant.py` | Bootstrap tests | Qdrant collection creation |
| `tests/unit/api/test_config_unknown_clustering.py` | Config tests | RISK: Uses TestClient(app) |
| `tests/unit/api/test_config_post_training_suggestions.py` | Config tests | RISK: Uses TestClient(app) |
| `tests/api/test_search.py` | Search API tests | Uses proper overrides |
| `tests/faces/test_detector.py` | Face detection tests | Uses mock InsightFace |
| `src/image_search_service/core/config.py` | Settings class | Loads from .env file |
| `src/image_search_service/db/session.py` | Database session management | Lazy initialization |
| `src/image_search_service/main.py` | App factory | Module-level app creation |
| `src/image_search_service/vector/qdrant.py` | Qdrant client | Safety guard on line 668 |
| `src/image_search_service/vector/face_qdrant.py` | Face Qdrant client | Safety guard on line 814 |
| `.env` | Environment variables | Real service URLs |
| `pyproject.toml` | Project config | Pytest settings |

## Appendix B: Plans Reviewed

| Plan | Safety Assessment | Key Finding |
|------|-------------------|-------------|
| P0-01: CI Gating | Minor gap | Missing Qdrant service container; no collection name env vars |
| P0-02: Fix Failing Tests | Safe | Correctly prescribes fixing TestClient bypass |
| P1-03: Race Conditions | Safe | Pure mock-based tests |
| P1-04: Security Testing | Safe | Uses tmp_path and mocks |
| P2-05: PostgreSQL Tier | Safe | Uses testcontainers (isolated Docker) |
| P2-06: Critical Coverage | Safe | Uses monkeypatch injection |
| P3-07: Mock Accuracy | Safe | Test infrastructure only |

## Appendix C: Safety Guard Inventory

| Guard | Location | Trigger | Action |
|-------|----------|---------|--------|
| Settings cache clear | `tests/conftest.py:33-44` | Every test (autouse) | Prevents production settings from caching |
| Collection name override | `tests/conftest.py:47-65` | Every test (autouse) | Forces `test_*` collection names |
| In-memory SQLite | `tests/conftest.py:29-30` | DB fixture use | Substitutes PostgreSQL entirely |
| In-memory Qdrant | `tests/conftest.py:177-199` | Qdrant fixture use | Substitutes real Qdrant server |
| Dependency overrides | `tests/conftest.py:316-320` | test_client fixture use | Injects all test doubles |
| PYTEST_CURRENT_TEST check | `src/.../qdrant.py:668` | `reset_collection()` call | Blocks production collection deletion |
| PYTEST_CURRENT_TEST check | `src/.../face_qdrant.py:814` | `reset_collection()` call | Blocks production face collection deletion |
| Mock embedding service | `tests/conftest.py:68-108` | Embedding fixture use | Prevents OpenCLIP model loading |
| Mock queue | `tests/conftest.py:331-351` | Queue fixture use | Prevents Redis connection |
| Safety regression tests | `tests/unit/test_qdrant_safety.py` | CI test suite | Validates guards still work |
