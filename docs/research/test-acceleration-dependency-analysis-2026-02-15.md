# Test Acceleration: Dependency, Mocking, and Fixture Analysis

**Date**: 2026-02-15
**Scope**: `image-search-service/tests/` (86 Python test files)
**Type**: Informational research for test acceleration initiative

---

## Executive Summary

Analysis of the image-search-service test suite reveals **428 test functions** across **44 test files** (excluding helpers, conftest, and `__init__.py`). The test suite has significant optimization potential through:

1. **Fixture deduplication** - Same fixtures (`mock_image_asset`, `mock_person`, `mock_face_instance`) are copy-pasted across 8+ files
2. **Per-test app creation** - Every test using `test_client` creates a new FastAPI app instance via `create_app()`
3. **No fixture scoping** - All fixtures use default `function` scope; zero `session` or `module` scoped fixtures (except postgres testcontainers)
4. **Heavyweight autouse fixtures** - 4 autouse fixtures run on every single test, even pure unit tests that don't need them
5. **Massive fixture-in-test-file pattern** - 134 fixture definitions live inside test files instead of conftest

---

## 1. Conftest.py Inventory

### Root conftest (`tests/conftest.py`) - 663 lines

| Fixture | Scope | Autouse | Complexity | Description |
|---------|-------|---------|------------|-------------|
| `clear_settings_cache` | function | **YES** | Low | Clears `@lru_cache` before/after every test |
| `use_test_settings` | function | **YES** | Low | Sets env vars for test collection names |
| `clear_embedding_cache` | function | **YES** | Medium | Clears cache + monkeypatches `EmbeddingService` methods |
| `validate_embedding_dimensions` | function | **YES** | Low | Asserts mock dimensions match expectations |
| `db_engine` | function | No | **Heavy** | Creates async SQLite engine + `create_all` + `drop_all` |
| `db_session` | function | No | Medium | Creates session from `db_engine` + rollback |
| `sync_db_engine` | function | No | **Heavy** | Creates sync SQLite engine + `create_all` + `drop_all` |
| `sync_db_session` | function | No | Medium | Creates session from `sync_db_engine` + rollback |
| `qdrant_client` | function | No | **Heavy** | Creates in-memory Qdrant + 2 collections + 5 payload indexes |
| `mock_embedding_service` | function | No | Low | Instantiates `SemanticMockEmbeddingService` |
| `face_qdrant_client` | function | No | Medium | Wraps `qdrant_client` in `FaceQdrantClient` |
| `temp_image_factory` | function | No | Medium | Factory for creating test images (PIL) |
| `test_client` | function | No | **Very Heavy** | Calls `create_app()` + overrides 5 dependencies + creates AsyncClient |

Also imports 8 postgres fixtures from `conftest_postgres.py` (session-scoped).

**Key classes defined in conftest:**
- `SemanticMockEmbeddingService` (342 lines) - Full semantic embedding mock with concept clusters, numpy operations, L2 normalization
- `LegacyMockEmbeddingService` - Deprecated, still present (30 lines)

### Faces conftest (`tests/faces/conftest.py`) - 118 lines

| Fixture | Scope | Complexity | Description |
|---------|-------|------------|-------------|
| `mock_face_embedding` | function | Low | Random 512-dim numpy vector |
| `mock_detected_face` | function | Medium | Full `DetectedFace` object with numpy arrays |
| `mock_image_asset` | function | Medium | Creates `ImageAsset` in DB |
| `mock_face_instance` | function | Medium | Creates `FaceInstance` in DB (depends on `mock_image_asset`) |
| `mock_person` | function | Medium | Creates `Person` in DB |
| `mock_qdrant_client` | function | Medium | Patches `get_face_qdrant_client` with `MagicMock` |
| `mock_insightface` | function | Low | Patches `_ensure_model_loaded` |

**Total fixture definitions**: 184 across 55 files (14 in conftest files, 134 inline in test files, 36 in helpers)

---

## 2. Heaviest Fixtures (by setup complexity)

### Tier 1: Very Heavy (multiple subsystem initialization)

**`test_client`** (conftest.py:583-639) - **Used by 34 test files, 815 references**
- Creates a **new FastAPI app** via `create_app()` every invocation
- Sets up 5 dependency overrides (DB, sync DB, Qdrant, face Qdrant, embedding)
- Creates `ASGITransport` + `AsyncClient`
- Depends on: `db_session`, `sync_db_session`, `qdrant_client`, `face_qdrant_client`, `mock_embedding_service`
- **Impact**: This is the single most expensive fixture. Every API test creates a full app instance.

**`qdrant_client`** (conftest.py:460-510) - **Used by ~40 tests directly + indirectly via test_client**
- Creates in-memory `QdrantClient(":memory:")`
- Creates 2 collections (768-dim image + 512-dim face)
- Creates 5 payload indexes on face collection
- **Impact**: Collection creation + index setup runs per-test

**`db_engine`** (conftest.py:396-409) - **Foundation for all DB-using tests**
- Creates async SQLAlchemy engine with `sqlite+aiosqlite:///:memory:`
- Runs `Base.metadata.create_all()` (creates all tables)
- Runs `Base.metadata.drop_all()` on teardown
- **Impact**: Full schema DDL per-test (all models)

### Tier 2: Heavy (single subsystem, notable setup)

**`sync_db_engine`** (conftest.py:427-443)
- Same as `db_engine` but synchronous SQLite
- Also runs full `create_all` / `drop_all`
- **Used by**: Queue job tests, background job tests

**`SemanticMockEmbeddingService.__init__`** (conftest.py:246-249)
- Creates numpy RandomState
- Generates 5 cluster centers (each 768-dim vector)
- Used both directly via `mock_embedding_service` fixture AND indirectly via autouse `clear_embedding_cache`

### Tier 3: Medium (DB writes)

Multiple fixtures that create DB entities:
- `mock_image_asset` - Creates ImageAsset + commit + refresh
- `mock_face_instance` - Creates FaceInstance + commit + refresh (depends on mock_image_asset)
- `mock_person` - Creates Person + commit + refresh
- Various `test_category`, `test_assets`, `test_session` fixtures

---

## 3. Fixture Duplication Analysis (Redundant Setup)

### Critical: `mock_image_asset` defined in 8 locations

| File | Line | Identical? |
|------|------|-----------|
| `tests/faces/conftest.py` | 40 | Base definition |
| `tests/api/test_faces_routes.py` | 11 | **Copy-paste duplicate** |
| `tests/api/test_prototype_endpoints.py` | 10 | **Copy-paste duplicate** |
| `tests/api/test_person_name_regression.py` | 20 | **Copy-paste duplicate** |
| `tests/api/test_get_faces_for_asset.py` | 9 | **Copy-paste duplicate** |
| `tests/api/test_clusters_filtering.py` | 10 | **Copy-paste duplicate** |
| `tests/api/test_face_suggestions.py` | 12 | **Copy-paste duplicate** |
| `tests/api/test_face_session_suggestions.py` | 14 | **Copy-paste duplicate** |
| `tests/unit/test_suggestion_qdrant_sync.py` | 55 | **Copy-paste duplicate** |

### Critical: `mock_person` defined in 8 locations

| File | Line | Identical? |
|------|------|-----------|
| `tests/faces/conftest.py` | 81 | Base definition |
| `tests/api/test_faces_routes.py` | 52 | **Copy-paste duplicate** |
| `tests/api/test_prototype_endpoints.py` | 29 | **Copy-paste duplicate** |
| `tests/api/test_person_name_regression.py` | 39 | **Copy-paste duplicate** |
| `tests/api/test_get_faces_for_asset.py` | 28 | **Copy-paste duplicate** |
| `tests/api/test_face_suggestions.py` | 53 | **Copy-paste duplicate** |
| `tests/api/test_face_session_suggestions.py` | 55 | **Copy-paste duplicate** |
| `tests/unit/test_suggestion_qdrant_sync.py` | 27 | **Copy-paste duplicate** |

### Critical: `mock_face_instance` defined in 6 locations

Similar copy-paste across `faces/conftest.py`, `test_faces_routes.py`, `test_face_suggestions.py`, `test_suggestion_qdrant_sync.py`, `test_face_session_suggestions.py`, `test_prototype_endpoints.py`.

### Critical: `mock_qdrant_client` defined in 10 locations

Each with slightly different mock configurations but fundamentally the same pattern (MagicMock wrapping FaceQdrantClient methods). Found in:
- `tests/faces/conftest.py:97`
- `tests/api/test_faces_routes.py:68`
- `tests/api/test_clusters_filtering.py:29`
- `tests/api/test_prototype_endpoints.py:68`
- `tests/api/test_face_suggestion_cleanup.py:28`
- `tests/scripts/test_bootstrap_qdrant.py:32`
- `tests/unit/test_bootstrap_qdrant.py:22`
- `tests/unit/test_centroid_qdrant.py:29`
- `tests/unit/queue/test_face_jobs_coverage.py:64`
- `tests/integration/test_restart_workflows.py:315`

**Optimization opportunity**: Moving these 4 duplicated fixtures into `tests/conftest.py` or a shared `tests/fixtures/` package would eliminate ~50 redundant fixture definitions.

---

## 4. Mocking Patterns Analysis

### Mock import usage

| Import Pattern | File Count | Total References |
|----------------|-----------|-----------------|
| `from unittest.mock import` | 46 files | 46 |
| `@patch` / `with patch()` | 31 files | 310 occurrences |
| `monkeypatch.setattr/setenv` | 16 files | 111 occurrences |
| `MagicMock()` instantiations | 38 files | 422 occurrences |

### Mocking approach breakdown

**Pattern 1: `with patch()` context managers (310 occurrences in 31 files)**

Most common in API tests. Example from `test_faces_routes_extended.py:115`:
```python
with patch("redis.Redis"), \
     patch("rq.Queue") as mock_queue_cls:
    mock_job = MagicMock()
    ...
```
- **Inefficiency**: Many tests patch the same targets repeatedly within the same file
- **Example**: `test_core/test_device.py` uses 56 `@patch` decorators across its test methods, many patching the same `torch.cuda.is_available` target

**Pattern 2: Monkeypatch-based fixtures (111 occurrences in 16 files)**

Used primarily in conftest autouse fixtures and queue tests. Example:
```python
monkeypatch.setattr(EmbeddingService, "embed_text", mock_embed_text)
```
- **Better scoped** than `with patch()` - cleanup is automatic

**Pattern 3: FastAPI dependency overrides (in `test_client` fixture)**

The `test_client` fixture uses `app.dependency_overrides` dict to replace 5 dependencies. This is the most efficient pattern for API tests but creates the heaviest single fixture.

### Patch target analysis - most commonly patched modules

| Patch Target | Count | Files |
|-------------|-------|-------|
| `redis.Redis` / `rq.Queue` | ~25 | Most API tests with queue enqueuing |
| `image_search_service.vector.face_qdrant.*` | ~20 | Face-related tests |
| `torch.cuda.is_available` / `torch.backends.mps.*` | 56 | `test_device.py` alone |
| `image_search_service.faces.detector._ensure_model_loaded` | ~5 | Face pipeline tests |

**Key finding**: The `test_device.py` file accounts for 56 out of 310 total `@patch` usages (18%). Each test method stacks 2-3 `@patch` decorators, and `clear_device_cache()` is called in `setup_method` AND within each test.

---

## 5. Real I/O Analysis

### Tests doing file I/O

| Pattern | File Count | Occurrences | Details |
|---------|-----------|-------------|---------|
| `Image.new()` + `img.save()` | 8 files | 18 | PIL image creation to temp files |
| `tmp_path` / `tmpdir` usage | 8 files | 83 | pytest temp directory fixture |
| `open()` / `Path().read*` | 13 files | 25 | Various file read/write |

**Key files with real I/O**:
- `tests/unit/services/test_exif_service.py` (32 tests) - Creates test images with EXIF data via PIL, writes to `tmp_path`
- `tests/unit/test_perceptual_hash.py` (21 occurrences) - Creates many PIL images for perceptual hash testing
- `tests/unit/queue/test_training_jobs_coverage.py` - Creates test images for training job tests
- `tests/api/test_search.py` - 6 uses of `tmp_path` for test image creation

### Tests requiring Redis (real network I/O)

**`tests/integration/test_listener_worker.py`**:
- Line 42: `Redis()` - Creates real Redis connection
- Line 27: `time.sleep(0.1)` - Real sleep for job delay simulation
- Line 105-112: `torch.randn()` on GPU device - Real GPU tensor operations
- **Impact**: This file requires both Redis and potentially GPU, making it the heaviest integration test

### Tests using asyncio.sleep (simulated I/O delays)

| File | Line | Duration | Purpose |
|------|------|----------|---------|
| `tests/helpers/concurrency.py` | 44 | Variable | Stagger concurrent requests |
| `tests/helpers/concurrency.py` | 143, 149 | Variable | Simulated DB read/commit delays |
| `tests/api/test_race_conditions.py` | 75 | 10ms | Widen race window |
| `tests/integration/test_listener_worker.py` | 27 | 100ms | Simulate job processing |

---

## 6. Largest Test Files (Parallelization Candidates)

### Top 15 by line count and test count

| File | Lines | Tests | Classes | Uses test_client? | Parallelizable? |
|------|-------|-------|---------|-------------------|----------------|
| `api/test_search.py` | 1326 | 27 | 0 | Yes (53 refs) | **Yes** - stateless |
| `unit/test_admin_export_import.py` | 1217 | 30 | 0 | No | **Yes** - pure DB |
| `faces/test_dual_clusterer.py` | 1153 | 39 | 6 | No | **Yes** - unit tests |
| `api/test_faces_routes.py` | 1070 | ~50 | 9 | Yes (101 refs) | **Yes** - stateless |
| `unit/queue/test_face_jobs_coverage.py` | 1053 | 17 | 8 | No | **Yes** - mocked |
| `integration/test_restart_workflows.py` | 1047 | ~1 | 5 | No | **Caution** - shared fixtures |
| `api/test_training_routes.py` | 890 | 14 | 0 | Yes | **Yes** - stateless |
| `api/test_faces_routes_extended.py` | 873 | ~30 | 8 | Yes (59 refs) | **Yes** - stateless |
| `api/test_recompute_with_rescan.py` | 866 | ~15 | 1 | Yes (30 refs) | **Yes** - stateless |
| `unit/queue/test_training_jobs_coverage.py` | 779 | 17 | 0 | No | **Yes** - mocked |
| `api/test_vectors.py` | 707 | 28 | 0 | Yes (35 refs) | **Yes** - stateless |
| `unit/services/test_exif_service.py` | 702 | 32 | 0 | No | **Yes** - file I/O only |
| `unit/queue/test_multiproto_propagation.py` | 697 | ~10 | 1 | No | **Yes** - mocked |
| `api/test_face_suggestion_cleanup.py` | 686 | 10 | 0 | Yes (14 refs) | **Yes** - stateless |
| `api/test_queues.py` | 676 | 16 | 0 | Yes (32 refs) | **Yes** - mocked |

### Test count by directory

| Directory | Files | Tests | Notes |
|-----------|-------|-------|-------|
| `tests/api/` | 28 | ~250 | All use `test_client` fixture |
| `tests/unit/` | 27+ | ~140 | Mix of pure unit + DB-dependent |
| `tests/faces/` | 6 | ~45 | Face pipeline logic |
| `tests/integration/` | 5 | ~15 | Postgres + Redis dependent |
| `tests/scripts/` | 1 | ~8 | Bootstrap script tests |
| `tests/core/` | 1 | ~40 | Device detection tests |

---

## 7. Test Interdependency Analysis (Parallel Execution Blockers)

### Safe for parallelization (no shared mutable state)

**All API tests** (`tests/api/`) - Each test:
- Gets its own `test_client` with fresh app instance
- Gets its own `db_session` (SQLite in-memory, separate engine per test)
- Gets its own `qdrant_client` (in-memory, separate instance)
- No global state mutations (autouse fixtures reset caches)
- **Verdict: Safe for pytest-xdist**

**All unit tests** (`tests/unit/`) - Most tests:
- Use local mock objects
- Use function-scoped `db_session` or `sync_db_session`
- No file system side effects (except `test_exif_service`, `test_perceptual_hash`)
- **Verdict: Safe for pytest-xdist**

### Potentially unsafe for parallelization

1. **`tests/core/test_device.py`** (40 tests)
   - Uses `@patch.dict(os.environ, ...)` extensively
   - Patches `torch.cuda.is_available` at module level
   - Uses `clear_device_cache()` in `setup_method`
   - **Risk**: Environment variable mutations could leak between parallel workers
   - **Mitigation**: Already uses `clear=True` in some patches

2. **`tests/integration/test_listener_worker.py`**
   - Requires real Redis connection
   - Uses shared queue names (`"test-queue"`)
   - Uses `time.sleep()` for timing assertions
   - **Risk**: Queue name collisions if run in parallel
   - **Mitigation**: Already excluded by default (`-m "not postgres"` but this file isn't marked postgres - it would still run)

3. **`tests/unit/test_config_unknown_person.py`** (16 `monkeypatch.setenv` calls)
   - Modifies environment variables
   - Creates `Settings()` objects that read from env
   - **Risk**: Low - monkeypatch is function-scoped and auto-reverts

4. **`tests/unit/test_qdrant_wrapper.py`** and **`tests/unit/test_qdrant_safety.py`**
   - Modify `monkeypatch.setenv("QDRANT_COLLECTION", ...)` and similar
   - **Risk**: Low with monkeypatch, but the autouse `use_test_settings` might interact

### Key finding: No test ordering dependencies detected

No tests use `pytest.mark.order`, `@pytest.mark.dependency`, or `pytest-ordering`. Tests appear independent within files. The test class structure (`class TestXxx:`) groups related tests but doesn't create shared state between methods (no `cls`-level fixtures).

---

## 8. Heavy Module Imports in Tests

### Direct heavy imports

| Module | Files Importing | Impact |
|--------|----------------|--------|
| `numpy` | 17 files | Moderate - loaded at import time |
| `torch` | 1 file (`test_listener_worker.py`) | **Very Heavy** - ~2GB import |
| `PIL.Image` | 8 files | Moderate |
| `qdrant_client` | 12 files | Moderate |
| `sqlalchemy` | ~40 files | Moderate (already fast with aiosqlite) |

### Indirect heavy imports (through production code)

Most test files import from `image_search_service.db.models`, which transitively imports SQLAlchemy. However, the critical heavy imports (OpenCLIP, InsightFace, torch) are **successfully avoided** through:
1. The autouse `clear_embedding_cache` fixture monkeypatches methods before real models can load
2. The faces conftest mocks `_ensure_model_loaded`
3. Lazy initialization patterns in production code

**Exception**: `test_listener_worker.py` directly imports `torch` (line 9: `import torch`), which triggers the full PyTorch import chain (~2s on cold start).

### Import-time costs estimated

| Import | Estimated Cold Cost | Files Affected |
|--------|-------------------|----------------|
| `torch` | ~2s | 1 file |
| `numpy` | ~200ms | 17 files |
| `qdrant_client` | ~100ms | 12 files |
| `PIL` | ~100ms | 8 files |
| `sqlalchemy` | ~150ms | ~40 files |
| `pydantic` | ~100ms | ~30 files |

Note: Python caches imports, so these costs are paid once per process, not per test.

---

## 9. Parametrized Tests

**Finding: Zero parametrized tests found** (`@pytest.mark.parametrize` returns 0 matches).

This is a significant optimization opportunity. Many test files contain repetitive test functions that test the same logic with different inputs:

### Candidates for parametrization

1. **`tests/core/test_device.py`** - 40 tests, many testing `get_device()` with different env var combinations
   - `test_cuda_available`, `test_mps_fallback`, `test_cpu_fallback`, `test_device_env_override`, `test_device_env_auto` etc.
   - Could be consolidated into ~5 parametrized tests

2. **`tests/unit/test_config_unknown_person.py`** - Multiple validation tests following same pattern
   - `test_min_display_count_must_be_at_least_2`, `test_default_threshold_lower_bound`, etc.
   - Pattern: Set env var -> Create Settings -> Assert validation error

3. **`tests/unit/test_temporal_service.py`** - 8 test classes with repetitive pattern
   - Each class tests one function with multiple input/output pairs

4. **`tests/unit/api/test_face_schemas.py`** - Pydantic model validation tests
   - Many tests follow: create model with X -> assert field Y has value Z

**Estimated impact**: Parametrizing the top 4 candidates could reduce test function count by ~50-80 while maintaining identical coverage.

---

## 10. Sleep and Timeout Analysis

### time.sleep() usage (blocking)

| File | Line | Duration | Purpose |
|------|------|----------|---------|
| `integration/test_listener_worker.py` | 27 | 100ms | Job processing simulation |

### asyncio.sleep() usage

| File | Line | Duration | Purpose |
|------|------|----------|---------|
| `helpers/concurrency.py` | 44 | Variable | Stagger for race condition tests |
| `helpers/concurrency.py` | 143 | Variable | Simulated read delay |
| `helpers/concurrency.py` | 149 | Variable | Simulated commit delay |
| `api/test_race_conditions.py` | 75 | 10ms | Widen race window |

### Timing-dependent assertions

| File | Line | Details |
|------|------|---------|
| `integration/test_listener_worker.py` | 170-171 | `assert elapsed >= 0.25` - Tests sequential processing takes ~0.3s |

**Total impact**: Sleep usage is minimal and mostly confined to integration/race-condition tests. Not a major bottleneck for the fast test suite.

---

## 11. Autouse Fixture Overhead Analysis

The root conftest defines **4 autouse fixtures** that run on every single test:

### Per-test overhead from autouse fixtures

```
clear_settings_cache:        2x lru_cache.cache_clear()     ~0.01ms
use_test_settings:           3x cache_clear + 2x setenv     ~0.05ms
clear_embedding_cache:       2x cache_clear + 3x monkeypatch + SemanticMock init  ~0.5ms
validate_embedding_dimensions: 2x assertion + MockEmbedding instantiation  ~0.3ms
```

**Estimated total overhead per test**: ~1ms from autouse fixtures alone

**For 428 tests**: ~428ms of pure autouse overhead

**Issue with `clear_embedding_cache`**: Creates a new `SemanticMockEmbeddingService()` instance per test, which involves:
- `np.random.RandomState(42)` creation
- 5x `np.random.randn(768)` + L2 normalization for cluster centers

**Issue with `validate_embedding_dimensions`**: Creates a `MockEmbeddingService()` instance per test purely for validation assertion. This is redundant if `clear_embedding_cache` already sets up the mock.

### Optimization: Could reduce to 2 autouse fixtures

- Merge `clear_settings_cache` + `use_test_settings` into one fixture
- Merge `clear_embedding_cache` + `validate_embedding_dimensions` into one fixture
- Make the merged embedding fixture use a **module-scoped** cached instance
- **Savings**: ~50% reduction in autouse overhead

---

## 12. `test_client` Fixture Deep Dive

The `test_client` fixture is the most impactful target for optimization.

### Current cost per invocation:
1. `db_engine` creation + `Base.metadata.create_all()` - ~5-10ms
2. `db_session` creation from engine - ~1ms
3. `sync_db_engine` creation + `Base.metadata.create_all()` - ~5-10ms
4. `sync_db_session` creation - ~1ms
5. `qdrant_client` creation + 2 collections + 5 indexes - ~5ms
6. `mock_embedding_service` instantiation - ~0.5ms
7. `face_qdrant_client` wrapping - ~0.1ms
8. **`create_app()`** - Creates FastAPI app, registers all routers - ~10-20ms
9. 5x dependency overrides - ~0.1ms
10. `AsyncClient` + `ASGITransport` creation - ~1ms

**Estimated total**: ~30-50ms per test using `test_client`

**For ~250 API tests**: ~7.5-12.5 seconds just for test_client setup

### Optimization paths:

**Option A: Session-scoped app + module-scoped engine**
- Create app once per session, override deps per test
- Create engine once per module, use nested transactions for isolation
- **Estimated savings**: 60-70% of test_client overhead

**Option B: Module-scoped test_client with transaction rollback**
- Share client across all tests in a module
- Use `SAVEPOINT` / rollback for DB isolation
- **Estimated savings**: 80-90% of test_client overhead
- **Risk**: Tests that commit and need to verify committed state

---

## 13. Qdrant Client Recreation Pattern

The in-memory Qdrant client is recreated per-test with full collection setup:

```python
client = QdrantClient(":memory:")
client.create_collection("test_image_assets", VectorParams(size=768, ...))
client.create_collection("test_faces", VectorParams(size=512, ...))
# + 5 payload indexes
```

**Found in 5 separate locations** (some tests create their own instead of using the shared fixture):
- `conftest.py:466` - Shared fixture
- `unit/test_qdrant_wrapper.py:141, 170` - Own instances
- `unit/queue/test_face_jobs_coverage.py:69` - Own instance
- `faces/test_trainer.py:240` - Own instance

**Optimization**: Module-scoped Qdrant client with collection clearing between tests (instead of full recreation).

---

## 14. Database Operations Per Test

### DB write operations in test files

**Total**: 777 `db_session.add/commit/flush` calls across 47 test files

**Heaviest files by DB operations**:
| File | DB ops | Notes |
|------|--------|-------|
| `unit/test_admin_export_import.py` | 85 | Creates many entities for export/import testing |
| `api/test_training_routes.py` | 55 | Complex training session setup |
| `api/test_search.py` | 48 | Multiple search scenario setups |
| `faces/test_dual_clusterer.py` | 39 | Cluster test data setup |
| `api/test_faces_routes.py` | 39 | Face route test data setup |

**Pattern**: Most API tests follow:
1. Create entities in DB (3-10 `add/commit` calls)
2. Make HTTP request via `test_client`
3. Assert response

---

## Key Recommendations (Priority Ordered)

### P0: Fixture Consolidation (Low Risk, High Impact)
- Move `mock_image_asset`, `mock_person`, `mock_face_instance`, `mock_qdrant_client` into shared conftest
- Eliminates ~50 duplicate fixture definitions
- No behavior change needed

### P1: Scope `db_engine` to Module/Session (Medium Risk, High Impact)
- Schema DDL (`create_all`/`drop_all`) is identical across all tests
- Use nested transactions (SAVEPOINT) for test isolation instead
- **Estimated savings**: 5-10ms per test for 250+ API tests = 1.25-2.5s

### P2: Scope `test_client` App Creation (Medium Risk, High Impact)
- Cache `create_app()` at session level
- Only override dependencies per-test
- **Estimated savings**: 10-20ms per test for 250 tests = 2.5-5s

### P3: Merge Autouse Fixtures (Low Risk, Low-Medium Impact)
- Combine 4 autouse fixtures into 2
- Cache SemanticMockEmbeddingService at module level
- **Estimated savings**: ~0.5s across full suite

### P4: Parametrize Repetitive Tests (Low Risk, Medium Impact)
- Especially `test_device.py` (56 patch decorators) and `test_config_unknown_person.py`
- Reduces test function count, faster collection
- **Estimated savings**: Faster collection + less overhead

### P5: Scope `qdrant_client` to Module (Low Risk, Medium Impact)
- Clear collections between tests instead of recreating
- **Estimated savings**: ~5ms per test for 40+ tests = 200ms

---

## Appendix: File Inventory

### Test file count by directory
```
tests/api/           28 files
tests/unit/          18 files
tests/unit/api/       4 files
tests/unit/queue/     5 files
tests/unit/services/  4 files
tests/faces/          6 files
tests/integration/    5 files
tests/scripts/        1 file
tests/core/           1 file
tests/helpers/        2 files (+ __init__.py)
tests/               1 file (test_main.py)
```

### Total metrics
- **86 Python files** in tests directory
- **44 files with test functions** (excluding conftest, helpers, __init__)
- **428 test functions**
- **115+ test classes**
- **184 fixture definitions** (14 conftest + 134 inline + 36 helper)
- **310 @patch/@with-patch occurrences**
- **422 MagicMock() instantiations**
- **111 monkeypatch calls**
- **777 DB write operations**
- **0 parametrized tests**
