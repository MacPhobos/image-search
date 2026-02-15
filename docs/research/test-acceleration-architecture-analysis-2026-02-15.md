# Test Architecture Analysis: image-search-service

**Date**: 2026-02-15
**Scope**: Test infrastructure, framework config, execution bottlenecks
**Project**: image-search-service (Python FastAPI backend)

---

## 1. Executive Summary

The image-search-service has **1,122 test functions** across **87 test files**. Tests use pytest with `asyncio_mode = "auto"`, SQLite in-memory for DB, and in-memory Qdrant for vector search. The current setup has several significant bottlenecks:

1. **Torch import at module level** costs ~1.8s at test session startup (via `core/device.py`)
2. **4 autouse fixtures** run for every single test, even pure unit tests that don't need them
3. **Function-scoped `test_client`** recreates FastAPI app + DB + Qdrant for each of 356 API tests
4. **Zero parametrize usage** despite many boundary/variant test patterns
5. **No parallel execution** (no pytest-xdist, pytest-forked, or similar)
6. **Heavy fixture duplication** - `mock_image_asset`, `mock_person`, `mock_face_instance` are copy-pasted across 10+ files

---

## 2. Pytest Configuration

**Source**: `pyproject.toml` lines 71-78

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "postgres: marks tests as requiring PostgreSQL (deselect with '-m \"not postgres\"')",
]
addopts = "-m 'not postgres'"
```

### Key observations:

| Config | Value | Impact |
|--------|-------|--------|
| `asyncio_mode` | `"auto"` | All async tests auto-detected; no explicit `@pytest.mark.asyncio` needed (though many files still use it redundantly — 476 occurrences) |
| `addopts` | `-m 'not postgres'` | Postgres tests excluded by default (15 tests skipped in `make test`) |
| Missing | No `--timeout`, `--tb`, `-x`, `--dist` | No timeouts, no parallel, no fail-fast by default |
| Missing | No `filterwarnings` | DeprecationWarning noise in output |

### Installed pytest plugins:

| Plugin | Version | Notes |
|--------|---------|-------|
| `pytest` | 9.0.2 | Base framework |
| `pytest-asyncio` | 1.3.0 | Async test support |
| `pytest-cov` | 7.0.0 | Listed in deps but coverage not configured in addopts |

**Not installed**: pytest-xdist, pytest-timeout, pytest-randomly, pytest-benchmark, pytest-sugar

---

## 3. Test Directory Structure & Counts

```
tests/                           1 file      4 tests    (test_main.py)
tests/api/                      32 files   394 tests    (API integration tests)
tests/core/                      1 file     31 tests    (device detection)
tests/faces/                     6 files    83 tests    (face pipeline unit tests)
tests/integration/               6 files    53 tests    (postgres + restart workflows)
tests/scripts/                   1 file     12 tests    (bootstrap scripts)
tests/unit/                     26 files   342 tests    (business logic)
tests/unit/api/                  4 files    64 tests    (schema validation)
tests/unit/queue/                5 files    73 tests    (job processing)
tests/unit/services/             5 files    66 tests    (service layer)
tests/helpers/                   2 files     0 tests    (concurrency + failure injection)
─────────────────────────────────────────────────────
TOTAL                           87 files  1,122 tests
```

### Collected at runtime:
- `pytest --co`: **1,096 collected, 15 deselected** (postgres marker) in **1.38s** collection time

### Top 10 largest test files:

| File | Tests | Category |
|------|-------|----------|
| `unit/test_temporal_service.py` | 57 | Pure computation |
| `api/test_faces_routes.py` | 49 | API + DB + Qdrant |
| `unit/services/test_exif_service.py` | 32 | Pure computation |
| `core/test_device.py` | 31 | Mocked torch |
| `unit/test_admin_export_import.py` | 30 | DB-dependent |
| `api/test_faces_routes_extended.py` | 30 | API + DB + Qdrant |
| `api/test_vectors.py` | 28 | API + DB + Qdrant |
| `api/test_search.py` | 27 | API + DB + Qdrant |
| `unit/api/test_face_schemas.py` | 24 | Pure Pydantic validation |
| `unit/queue/test_face_jobs_coverage.py` | 22 | Mocked queue jobs |

---

## 4. Test Type Distribution

### Async vs Sync

| Type | Count | % |
|------|-------|---|
| `async def test_*` | 545 | 48.5% |
| `def test_*` | 577 | 51.5% |
| **Total** | **1,122** | 100% |

### By dependency profile

| Profile | Count | Fixtures needed |
|---------|-------|-----------------|
| Pure unit (no DB, no test_client) | 426 | None beyond autouse |
| Unit with DB session | 119 | db_engine + db_session |
| API with test_client | 356 | db_engine + db_session + qdrant + app + AsyncClient |
| Integration (non-postgres) | 38 | Various; some need restart_workflows fixtures |
| Integration (postgres) | 15 | testcontainers PostgreSQL (session-scoped) |
| Face pipeline | 83 | faces/conftest.py fixtures + DB |
| Other | 85 | Mixed |

---

## 5. Fixture Analysis

### 5.1 Autouse Fixtures (Global — `tests/conftest.py`)

**4 autouse fixtures run for EVERY test** regardless of whether they're needed:

| Fixture | Scope | What it does | Cost per test |
|---------|-------|-------------|---------------|
| `clear_settings_cache` | function | Clears `@lru_cache` on `get_settings()` twice (before + after) | ~0.01ms |
| `use_test_settings` | function | Sets env vars for test collection names, clears settings cache 3x | ~0.1ms |
| `clear_embedding_cache` | function | Creates `SemanticMockEmbeddingService`, monkeypatches 4 methods | ~8ms (numpy RNG init) |
| `validate_embedding_dimensions` | function | Creates `MockEmbeddingService` instance, checks dim == 768 | ~8ms |

**Combined overhead per test**: ~16ms for autouse fixtures alone.
**Across 1,096 tests**: ~17.5 seconds of pure autouse fixture overhead.

**Critical issue**: `clear_embedding_cache` and `validate_embedding_dimensions` each create a new `SemanticMockEmbeddingService()` instance, which initializes numpy RandomState + generates cluster centers. This runs for pure unit tests that never touch embeddings.

### 5.2 Key Shared Fixtures (Function-Scoped)

| Fixture | Scope | Dependencies | Approx cost |
|---------|-------|-------------|-------------|
| `db_engine` | function | None | ~2-3ms (SQLite create_all) |
| `db_session` | function | db_engine | ~0.5ms |
| `sync_db_engine` | function | None | ~2-3ms |
| `sync_db_session` | function | sync_db_engine | ~0.5ms |
| `qdrant_client` | function | None | ~5ms (in-memory + create collections + indexes) |
| `face_qdrant_client` | function | qdrant_client | ~0.5ms |
| `mock_embedding_service` | function | None | ~8ms |
| `test_client` | function | db_session, sync_db_session, qdrant_client, face_qdrant_client, mock_embedding_service | ~20-30ms (create_app + dependency overrides + AsyncClient) |

**`test_client` cascade**: Each of the 356 API tests triggers:
1. Create SQLite in-memory async engine + create all tables
2. Create SQLite in-memory sync engine + create all tables
3. Create in-memory Qdrant client + 2 collections + 5 payload indexes
4. Create `MockEmbeddingService` instance
5. Call `create_app()` (FastAPI app factory, ~146ms first time, negligible after)
6. Set 5 dependency overrides
7. Create `AsyncClient` with ASGI transport

**Estimated per-test overhead for API tests**: ~30-50ms in fixture setup/teardown.

### 5.3 Faces-Specific Conftest (`tests/faces/conftest.py`)

7 fixtures, all function-scoped:
- `mock_face_embedding` — generates 512-dim numpy vector
- `mock_detected_face` — creates `DetectedFace` with numpy arrays
- `mock_image_asset` — DB insert + commit + refresh
- `mock_face_instance` — DB insert + commit + refresh
- `mock_person` — DB insert + commit + refresh
- `mock_qdrant_client` — MagicMock with 7 configured methods
- `mock_insightface` — patches model loading

### 5.4 Postgres Fixtures (`tests/conftest_postgres.py`)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `postgres_container` | **session** | Starts PostgresContainer once per session (Docker) |
| `pg_connection_url` | **session** | Cached async URL |
| `pg_sync_connection_url` | **session** | Cached sync URL |
| `pg_engine` | function | Async engine (function-scoped for event loop compat) |
| `pg_session` | function | Async session with rollback |
| `pg_sync_engine` | function | Sync engine |
| `pg_sync_session` | function | Sync session with rollback |
| `fresh_pg_database` | function | Creates fresh DB per migration test |

**Good practice**: Container is session-scoped, only DB setup is per-test.

### 5.5 Fixture Duplication (Anti-Pattern)

The following fixtures are **copy-pasted** across multiple files instead of being shared via conftest:

| Fixture | Files where duplicated |
|---------|----------------------|
| `mock_image_asset` | `faces/conftest.py`, `api/test_faces_routes.py`, `api/test_face_suggestions.py`, `api/test_prototype_endpoints.py`, `api/test_get_faces_for_asset.py`, `api/test_face_session_suggestions.py`, `api/test_clusters_filtering.py`, `api/test_get_faces_orphaned_person.py`, `api/test_person_name_regression.py`, `unit/test_suggestion_qdrant_sync.py` |
| `mock_person` | 8 files (same pattern as above) |
| `mock_face_instance` | 7 files |
| `mock_qdrant_client` | 2 files (faces/conftest.py + api/test_faces_routes.py) |

This means ~10 near-identical copies of `mock_image_asset`, each doing `db_session.add(); commit(); refresh()`.

---

## 6. Import Time Analysis

### Startup cost breakdown (measured):

| Import | Time | Cause |
|--------|------|-------|
| `torch` | **1,771ms** | PyTorch framework load |
| `open_clip` | 1,771ms | (Cached after torch) |
| `insightface` | 1,420ms | (Would add if imported) |
| `qdrant_client` | **912ms** | gRPC + protobuf |
| `fastapi` | **412ms** | Starlette + Pydantic |
| `sqlalchemy` | **146ms** | Core + async extensions |
| `image_search_service.core.device` | **1,609ms** | `import torch` at line 22 |
| `image_search_service.services.embedding` | **1,766ms** | Imports `core.device` |
| `image_search_service.db.models` | 130ms | SQLAlchemy model definitions |
| `image_search_service.main` (first import) | **4,171ms** | Cascading all above |
| `create_app()` (subsequent) | 146ms | App factory itself is fast |

### Critical path:
```
conftest.py
  └── from image_search_service.main import create_app
        └── from image_search_service.api.routes import api_v1_router
              └── (indirect) services.embedding → core.device → import torch  [1.8s]
        └── qdrant_client                                                     [0.9s]
        └── fastapi                                                           [0.4s]
        └── sqlalchemy                                                        [0.1s]
                                                                    Total:   ~3.2s
```

**Key finding**: `core/device.py` line 22 does `import torch` at module level. This is the single biggest import-time cost. The embedding service correctly uses lazy loading for models but still pays the torch import cost via device.py.

### Test collection time:
- `pytest --co`: 1.38s (including import overhead)
- Most of this is the torch/qdrant import chain

---

## 7. Async Test Patterns

### Configuration:
- `asyncio_mode = "auto"` — correct configuration for pytest-asyncio
- pytest-asyncio 1.3.0 installed

### Redundant markers:
- 476 tests have explicit `@pytest.mark.asyncio` despite `asyncio_mode = "auto"` making it unnecessary
- This is harmless but adds visual noise

### Pattern: Most API tests are async
```python
# Typical API test pattern (seen in 356 tests):
@pytest.mark.asyncio
async def test_something(test_client: AsyncClient, db_session: AsyncSession):
    # Setup: insert test data
    asset = ImageAsset(path="/test/photo.jpg", ...)
    db_session.add(asset)
    await db_session.commit()

    # Act: make API request
    response = await test_client.post("/api/v1/endpoint", json={...})

    # Assert
    assert response.status_code == 200
```

### Pattern: Many unit tests are sync but use async DB
```python
# Pattern in unit tests that could be pure sync:
async def test_compute_something(db_session):
    # Inserts data, then tests a computation
    # Could potentially be split: pure logic (sync) vs integration (async)
```

---

## 8. Mocking Patterns

### Mock method distribution:

| Method | Occurrences | Primary use |
|--------|-------------|-------------|
| `monkeypatch.*` | 94 | Env vars, method replacement |
| `unittest.mock.patch/MagicMock/Mock` | 94 | Module-level patching |
| Dependency injection overrides | 356 tests | Via `test_client` fixture |

### Mocking strategy:
- **FastAPI DI overrides** (primary): `app.dependency_overrides[get_db] = override_get_db`
  - Used for: DB sessions, Qdrant clients, embedding service
  - Clean pattern, properly isolated
- **`monkeypatch`** (secondary): Used for environment variables and method replacement
  - 4 autouse fixtures use monkeypatch for settings isolation
- **`unittest.mock.patch`** (tertiary): Used for module-level patching
  - Face detection model loading, queue functions

### Mock embedding service:
- `SemanticMockEmbeddingService`: 768-dim vectors with semantic clustering
- Includes concept clusters (nature, animal, food, urban, people)
- Uses numpy RandomState for deterministic output
- **Cost**: ~8ms per instantiation (numpy RNG + cluster center generation)
- Created in both `clear_embedding_cache` (autouse) AND `mock_embedding_service` fixture

---

## 9. Identified Bottlenecks

### B1: Torch Import at Module Level (HIGH IMPACT)
- **Where**: `src/image_search_service/core/device.py` line 22: `import torch`
- **Cost**: ~1.8s one-time at test session startup
- **Impact**: Every test session pays this cost
- **Fix**: Lazy-import torch inside functions that need it

### B2: Autouse Fixtures Running for All Tests (HIGH IMPACT)
- **Where**: `tests/conftest.py` lines 47-162
- **Cost**: ~16ms × 1,096 tests = ~17.5s total
- **Impact**: Pure unit tests like `test_temporal_service.py` (57 tests) pay for embedding cache clearing they never use
- **Fix**: Convert from autouse to explicit fixture usage, or use conftest scoping

### B3: Function-Scoped test_client (MEDIUM-HIGH IMPACT)
- **Where**: `tests/conftest.py` lines 583-639
- **Cost**: ~30-50ms × 356 API tests = ~10-18s total
- **Impact**: Each API test creates a complete app stack from scratch
- **Fix**: Module or session-scoped app with per-test transaction rollback

### B4: No Parallel Execution (HIGH IMPACT)
- **Where**: N/A — not configured
- **Cost**: All 1,096 tests run sequentially
- **Impact**: Linear scaling with test count
- **Fix**: Add pytest-xdist for multi-process execution

### B5: Fixture Duplication (LOW-MEDIUM IMPACT)
- **Where**: `mock_image_asset`, `mock_person`, `mock_face_instance` duplicated 7-10 times
- **Cost**: Maintenance burden, inconsistency risk
- **Impact**: Not a direct performance issue but indicates missing test infrastructure
- **Fix**: Consolidate into `tests/api/conftest.py` or `tests/conftest.py`

### B6: Zero Parametrize Usage (LOW IMPACT)
- **Where**: No `@pytest.mark.parametrize` anywhere in 87 test files
- **Cost**: Many test methods that differ only in input values are separate functions
- **Impact**: More functions = more fixture setup/teardown overhead
- **Example**: `test_temporal_service.py` has 10+ boundary tests that could be 2 parametrized tests

### B7: Redundant `@pytest.mark.asyncio` Markers (COSMETIC)
- **Where**: 476 occurrences across 46 files
- **Cost**: Negligible runtime cost
- **Impact**: Visual noise, minor maintenance burden
- **Fix**: Remove since `asyncio_mode = "auto"` handles detection

### B8: `time.sleep()` in Tests (LOW IMPACT)
- **Where**: `tests/integration/test_listener_worker.py` line 27: `time.sleep(0.1)`
- **Cost**: 0.1s per affected test (only ~10 tests in listener worker)
- **Impact**: Minor; these are integration tests that need real timing

---

## 10. Test Markers & Categorization

### Defined markers:
- `postgres` — 15 tests requiring PostgreSQL (excluded by default)

### Missing markers that would enable selective running:
- No `slow` marker for integration tests
- No `unit` marker for fast tests
- No `api` marker for API tests
- No `faces` marker for face pipeline tests

### Current Makefile targets:

| Target | Command | Selects |
|--------|---------|---------|
| `make test` | `pytest` | All non-postgres tests (1,096 tests) |
| `make test-postgres` | `pytest -m "postgres" -v` | Only postgres tests (15 tests) |
| `make test-all` | `pytest -m "" -v` | All tests (1,111 tests) |

---

## 11. Test Execution Configuration

### Current defaults:
```
asyncio_mode = "auto"
addopts = "-m 'not postgres'"
```

### What's missing:
- **No timeouts**: A hung test will block the entire suite forever
- **No fail-fast**: All tests run even if early failures indicate systemic issues
- **No coverage in CI**: `pytest-cov` is installed but not configured in addopts
- **No random ordering**: Tests may have hidden ordering dependencies
- **No output filtering**: DeprecationWarnings clutter output
- **No parallelism**: No pytest-xdist for multi-core execution
- **No test splitting**: No way to run "fast" vs "slow" subsets (beyond postgres)

---

## 12. Summary: Bottleneck Impact Ranking

| # | Bottleneck | Est. Impact | Effort to Fix | Priority |
|---|-----------|-------------|---------------|----------|
| B4 | No parallel execution | 60-75% speedup possible | Low (add xdist) | **P0** |
| B1 | Torch import at module level | ~1.8s startup | Low (lazy import) | **P1** |
| B2 | Autouse fixtures on all tests | ~17.5s total | Medium (refactor) | **P1** |
| B3 | Function-scoped test_client | ~10-18s total | Medium (scope change) | **P1** |
| B5 | Fixture duplication | Maintenance cost | Low (consolidate) | **P2** |
| B6 | No parametrize | Minor overhead | Low (refactor tests) | **P2** |
| B7 | Redundant asyncio markers | Cosmetic | Low (search-replace) | **P3** |
| B8 | time.sleep in tests | ~1s total | Low | **P3** |

---

## Appendix A: File-Level Test Counts (Complete)

<details>
<summary>All 87 test files with test counts</summary>

| File | Tests |
|------|-------|
| tests/test_main.py | 4 |
| tests/api/test_categories.py | 21 |
| tests/api/test_clusters_filtering.py | 5 |
| tests/api/test_face_session_suggestions.py | 7 |
| tests/api/test_face_suggestion_cleanup.py | 10 |
| tests/api/test_face_suggestions.py | 14 |
| tests/api/test_face_suggestions_list.py | 16 |
| tests/api/test_faces_cluster_confidence.py | 2 |
| tests/api/test_faces_routes.py | 49 |
| tests/api/test_faces_routes_extended.py | 30 |
| tests/api/test_get_faces_for_asset.py | 2 |
| tests/api/test_get_faces_orphaned_person.py | 1 |
| tests/api/test_health.py | 2 |
| tests/api/test_images.py | 7 |
| tests/api/test_ingest.py | 5 |
| tests/api/test_person_name_regression.py | 6 |
| tests/api/test_prototype_endpoints.py | 11 |
| tests/api/test_queues.py | 16 |
| tests/api/test_race_conditions.py | 4 |
| tests/api/test_recompute_with_rescan.py | 15 |
| tests/api/test_search.py | 27 |
| tests/api/test_suggestion_regeneration.py | 12 |
| tests/api/test_training_deduplication.py | 6 |
| tests/api/test_training_routes.py | 14 |
| tests/api/test_unified_people_endpoint.py | 22 |
| tests/api/test_unknown_persons_accept.py | 11 |
| tests/api/test_unknown_persons_candidates.py | 11 |
| tests/api/test_unknown_persons_detail.py | 8 |
| tests/api/test_unknown_persons_discover.py | 8 |
| tests/api/test_unknown_persons_dismiss.py | 11 |
| tests/api/test_unknown_persons_merge.py | 7 |
| tests/api/test_unknown_persons_stats.py | 6 |
| tests/api/test_vectors.py | 28 |
| tests/core/test_device.py | 31 |
| tests/faces/test_assigner.py | 10 |
| tests/faces/test_clusterer.py | 8 |
| tests/faces/test_detector.py | 12 |
| tests/faces/test_dual_clusterer.py | 19 |
| tests/faces/test_service.py | 13 |
| tests/faces/test_trainer.py | 21 |
| tests/integration/test_listener_worker.py | 10 |
| tests/integration/test_postgres_constraints.py | 5 |
| tests/integration/test_postgres_jsonb.py | 6 |
| tests/integration/test_postgres_migrations.py | 4 |
| tests/integration/test_restart_workflows.py | 19 |
| tests/integration/test_temporal_migration.py | 9 |
| tests/scripts/test_bootstrap_qdrant.py | 12 |
| tests/unit/api/test_config_post_training_suggestions.py | 13 |
| tests/unit/api/test_config_unknown_clustering.py | 8 |
| tests/unit/api/test_face_schemas.py | 24 |
| tests/unit/api/test_schemas.py | 19 |
| tests/unit/queue/test_centroid_post_training.py | 12 |
| tests/unit/queue/test_face_jobs_coverage.py | 22 |
| tests/unit/queue/test_multiproto_propagation.py | 12 |
| tests/unit/queue/test_post_training_suggestions_job.py | 10 |
| tests/unit/queue/test_training_jobs_coverage.py | 17 |
| tests/unit/services/test_centroid_race_conditions.py | 2 |
| tests/unit/services/test_exif_service.py | 32 |
| tests/unit/services/test_face_clustering_service.py | 15 |
| tests/unit/services/test_person_service.py | 15 |
| tests/unit/services/test_training_race_conditions.py | 2 |
| tests/unit/test_admin_export_import.py | 30 |
| tests/unit/test_bootstrap_qdrant.py | 8 |
| tests/unit/test_centroid_qdrant.py | 16 |
| tests/unit/test_cluster_confidence.py | 12 |
| tests/unit/test_cluster_labeling_service.py | 11 |
| tests/unit/test_config_keys_sync.py | 10 |
| tests/unit/test_config_unknown_person.py | 21 |
| tests/unit/test_discover_unknown_persons_job.py | 10 |
| tests/unit/test_dual_clusterer_batch.py | 11 |
| tests/unit/test_embedding_router.py | 7 |
| tests/unit/test_embedding_service.py | 6 |
| tests/unit/test_face_jobs_payload.py | 6 |
| tests/unit/test_face_qdrant_unlabeled.py | 15 |
| tests/unit/test_faces_cli.py | 4 |
| tests/unit/test_fusion.py | 12 |
| tests/unit/test_perceptual_hash.py | 18 |
| tests/unit/test_person_birth_date.py | 15 |
| tests/unit/test_prototype_selection.py | 7 |
| tests/unit/test_qdrant_safety.py | 6 |
| tests/unit/test_qdrant_wrapper.py | 17 |
| tests/unit/test_siglip_embedding_service.py | 9 |
| tests/unit/test_suggestion_qdrant_sync.py | 5 |
| tests/unit/test_temporal_service.py | 57 |
| tests/unit/test_unknown_person_schemas.py | 12 |
| tests/unit/test_unknown_person_service.py | 15 |
| tests/unit/test_validate_qdrant_collections.py | 5 |

</details>

---

## Appendix B: Conftest Fixture Inventory

### tests/conftest.py (14 fixtures)
- `clear_settings_cache` (autouse, function) — clears lru_cache
- `use_test_settings` (autouse, function) — sets test env vars
- `clear_embedding_cache` (autouse, function) — monkeypatches embedding methods
- `validate_embedding_dimensions` (autouse, function) — asserts dim == 768
- `db_engine` (function) — SQLite async engine + create_all
- `db_session` (function) — async session with rollback
- `sync_db_engine` (function) — SQLite sync engine + create_all
- `sync_db_session` (function) — sync session with rollback
- `qdrant_client` (function) — in-memory Qdrant + 2 collections + 5 indexes
- `mock_embedding_service` (function) — SemanticMockEmbeddingService instance
- `face_qdrant_client` (function) — wrapper around qdrant_client
- `temp_image_factory` (function) — creates PIL images in tmp_path
- `test_client` (function) — full FastAPI app + DI overrides + AsyncClient
- `mock_queue` (function) — mock RQ queue

### tests/conftest_postgres.py (8 fixtures)
- `postgres_container` (session) — PostgresContainer
- `pg_connection_url` (session) — async URL
- `pg_sync_connection_url` (session) — sync URL
- `pg_engine` (function) — async engine
- `pg_session` (function) — async session
- `pg_sync_engine` (function) — sync engine
- `pg_sync_session` (function) — sync session
- `fresh_pg_database` (function) — fresh DB per test

### tests/faces/conftest.py (7 fixtures)
- `mock_face_embedding` (function) — 512-dim random vector
- `mock_detected_face` (function) — DetectedFace object
- `mock_image_asset` (function) — DB insert
- `mock_face_instance` (function) — DB insert
- `mock_person` (function) — DB insert
- `mock_qdrant_client` (function) — MagicMock
- `mock_insightface` (function) — patches model loading

### Per-file fixtures: ~146 additional fixtures defined in individual test files
