# Test Execution Analysis - image-search-service

**Date**: 2026-02-06
**Environment**: Python 3.12.3, pytest 9.0.2, pytest-asyncio 1.3.0, pytest-cov 7.0.0
**Runner**: `uv run pytest tests/ -v --tb=long --durations=0`
**Total Duration**: 103.91s (1m 43s) without coverage; 201.56s (3m 21s) with coverage

---

## 1. Summary Results

| Metric | Count |
|--------|-------|
| **Total Tests** | 795 |
| **Passed** | 744 (93.6%) |
| **Failed** | 35 (4.4%) |
| **Errors** | 5 (0.6%) |
| **Skipped** | 11 (1.4%) |
| **Warnings** | 4 |

**Overall Coverage**: 48% (6001 missed of 11440 statements)

---

## 2. Failure Analysis (35 failures, 5 errors)

### Category A: `FaceClusterer` Constructor Signature Mismatch (7 failures)

**Files**: `tests/faces/test_clusterer.py` (all 7 tests)
**Root Cause**: **Code bug (breaking API change not reflected in tests)**

The `FaceClusterer.__init__()` now requires a `qdrant_client` positional argument, but all tests still use the old signature: `FaceClusterer(mock_session, ...)`. The tests pass only `session` as the first positional arg and keyword args for clustering params.

**Error**: `TypeError: FaceClusterer.__init__() missing 1 required positional argument: 'qdrant_client'`

**Fix**: Update all test instantiations to pass a mock qdrant_client:
```python
clusterer = FaceClusterer(mock_session, mock_qdrant_client, min_cluster_size=5)
```

**Severity**: HIGH - All clusterer tests are broken. Zero coverage for clustering logic.

---

### Category B: `FaceProcessingService.process_assets_batch()` Mock Mismatch (2 failures)

**Files**: `tests/faces/test_service.py` (test_process_assets_batch, test_process_assets_batch_parallel_loading)
**Root Cause**: **Test bug (stale mock patching)**

The tests mock `_load_image` and `detect_faces` but the actual InsightFace model is being loaded (visible in stdout: "Loaded InsightFace model (buffalo_l)"). The mock for `image_search_service.faces.detector.detect_faces` doesn't intercept the call path used by `process_assets_batch()`. The batch method likely invokes detection through a different code path (possibly via the loaded model directly, not via the module-level `detect_faces` function).

**Error**: `assert result["total_faces"] == 1` fails because `total_faces` is 0 (mocked detection returns empty because real model path is used).

**Fix**: Trace the actual call path in `service.py` for `process_assets_batch` and patch at the correct location. The mock for `_load_image` may also not be intercepting correctly since the real InsightFace model loads.

**Severity**: MEDIUM - Tests work for single-asset processing but batch mode coverage is broken.

---

### Category C: Restart Workflow Integration Tests (8 failures)

**Files**: `tests/integration/test_restart_workflows.py` (all 8 tests in TestFaceDetectionRestart, TestClusteringRestart, TestRestartWorkflowIntegration)
**Root Cause**: **Code/Environment bug (async/sync session mismatch)**

The restart services internally use synchronous DB operations (via `restart_service_base`) but the test fixtures provide async SQLAlchemy sessions. When the service tries to query the DB after the API call, it hits:

**Error**: `sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called; can't call await_only() here.`

The `FaceDetectionRestartService` also references `FaceQdrantClient.get_instance()` which the tests try to mock via `monkeypatch.setattr(face_detection_restart_service.FaceQdrantClient, "get_instance", ...)`, but the mocking approach doesn't fully prevent sync DB access outside the greenlet context.

For TestClusteringRestart tests, the mock target `image_search_service.services.face_clustering_service.cluster_unlabeled_faces` likely doesn't match the actual import path used by the restart workflow.

For TestRestartWorkflowIntegration tests, the `TestNormalWorkflowUnchanged` failure is due to the mock RQ queue not being injected at the right level (the training start endpoint tries to connect to Redis directly).

**Fix**:
1. Restart services need sync DB session injection (similar to `get_sync_db` dependency)
2. Or, convert restart services to fully async
3. Mock targets need to match actual import paths in the restart service chain

**Severity**: HIGH - All restart workflow integration tests are broken.

---

### Category D: Config Post-Training Suggestions Tests (5 failures)

**Files**: `tests/unit/api/test_config_post_training_suggestions.py` (5 of 12 tests)
**Root Cause**: **Test bug (using `TestClient(app)` bypasses dependency overrides)**

These tests use `from fastapi.testclient import TestClient; client = TestClient(app)` directly instead of the shared `test_client` fixture. This means they connect to the real database (or fail trying) instead of using the in-memory SQLite test database. Some tests pass when run in isolation (state from previous tests persists in the app singleton), but fail when run as part of the full suite.

**Error**: Tests that modify state (PUT endpoints) fail because the real DB isn't available, or because test ordering causes state pollution. The GET test passes in isolation but fails in suite context.

**Fix**: Refactor these tests to use the async `test_client` fixture from conftest.py instead of raw `TestClient(app)`.

**Severity**: MEDIUM - Tests are architecturally flawed but the underlying code is likely correct.

---

### Category E: Multi-Prototype Propagation Job Tests (8 failures)

**Files**: `tests/unit/queue/test_multiproto_propagation.py` (8 of 12 tests)
**Root Cause**: **Test bug (Qdrant mock not properly intercepted)**

The tests mock `image_search_service.vector.face_qdrant.get_face_qdrant_client` but the job function (`propagate_person_label_multiproto_job`) likely acquires the qdrant client through a different path (e.g., instantiates it directly or caches it). The log output confirms: "Aggregated 0 candidate faces from all prototypes" despite the mock returning candidates.

**Error**: `assert result["suggestions_created"] == 1` fails because `suggestions_created` is 0.

**Fix**: Trace how `propagate_person_label_multiproto_job` obtains its Qdrant client and mock at the correct location. May need to patch a cached instance or a different import path.

**Severity**: HIGH - 8 of 12 multi-proto tests broken, critical path for face suggestion generation.

---

### Category F: Prototype Endpoint Route Mismatch (2 failures)

**Files**: `tests/api/test_faces_routes.py` (TestPrototypeEndpoints: test_create_prototype_person_not_found, test_list_prototypes_person_not_found)
**Root Cause**: **Code change (route removed or changed)**

Tests POST to `/api/v1/faces/persons/{id}/prototypes` but get `405 Method Not Allowed`. The route either:
- Was removed/renamed
- Changed HTTP method (POST -> PUT)
- Moved to a different URL pattern

**Error**: `assert response.status_code == 404` fails because actual is `405`.

**Fix**: Check current route registration in `api/routes/faces.py` for the prototype creation endpoint. Update test URLs to match current API.

**Severity**: LOW - Tests reference deprecated/changed API routes.

---

### Category G: Pin Prototype Quota Assertion (1 failure)

**Files**: `tests/api/test_prototype_endpoints.py` (TestPinPrototype::test_pin_quota_exceeded)
**Root Cause**: **Code change (quota logic updated)**

The test asserts `"Maximum 3 PRIMARY prototypes"` but the actual response is `"Maximum 2 PRIMARY prototypes already pinned (based on max_exemplars=5)"`. The quota calculation was changed to derive the PRIMARY limit from `max_exemplars` config.

**Error**: `assert 'Maximum 3 PRIMARY prototypes' in response.json()["detail"]` - string mismatch.

**Fix**: Update test assertion to match new error message format. Consider using a less brittle assertion (e.g., assert response status 400 + "Maximum" in detail).

**Severity**: LOW - Simple assertion update needed.

---

### Category H: Config Keys Sync Tests (2 failures)

**Files**: `tests/unit/test_config_keys_sync.py`
**Root Cause**: **Missing migration for new config keys**

Four new config keys exist in `ConfigService.DEFAULTS` but don't have corresponding INSERT statements in migration files:
- `post_training_suggestions_top_n_count`
- `post_training_suggestions_mode`
- `post_training_use_centroids`
- `centroid_min_faces_for_suggestions`

**Error**: `AssertionError: Config keys in DEFAULTS without migration descriptions: {'post_training_suggestions_top_n_count', 'post_training_suggestions_mode', ...}`

**Fix**: Create a new Alembic migration that INSERTs these config keys with descriptions into the `system_configs` table.

**Severity**: MEDIUM - Missing migration means these defaults won't exist in production DB.

---

### Category I: Unified Progress Endpoint Errors (5 errors)

**Files**: `tests/api/test_training_unified_progress.py`
**Root Cause**: **Test bug (missing fixture)**

All tests reference fixture `async_client` which doesn't exist. The project uses `test_client` as the fixture name (defined in conftest.py).

**Error**: `fixture 'async_client' not found`

**Fix**: Rename `async_client` to `test_client` in all test function parameters. However, note that all test bodies are `pass` (TODO stubs), so they would just pass without testing anything.

**Severity**: LOW - These are unimplemented test stubs. The fixture name error should be fixed, and ideally the tests should be implemented.

---

## 3. Performance Analysis

### Slowest Tests (Top 20)

| Duration | Test | Notes |
|----------|------|-------|
| 7.34s | `test_process_assets_batch` (faces/service) | Loads real InsightFace model (GPU!) |
| 3.69s | `test_process_assets_batch_parallel_loading` (faces/service) | Same issue |
| 1.62s | `test_fusion_scoring_with_face_results` (unit/test_fusion) | Complex query |
| 0.63s | `test_unified_people_endpoint_pagination` | Multiple DB queries |
| 0.56s | `test_search_by_text_only` | Embedding + vector search |
| 0.42s | `test_search_by_image_path` | Embedding + search |
| 0.42s | `test_search_returns_results` | Similar to above |
| 0.39s | `test_search_face_boosted_score_calculation` | Complex fusion |
| 0.35s | `test_fusion_scoring_multiple_matches` | Complex fusion |
| 0.30s | `test_face_detection_restart_preserves_persons` | Restart workflow |

### Performance Observations

1. **GPU Model Loading**: The 2 `process_assets_batch` tests load the real InsightFace model (~3s each) because mocking isn't intercepting the model initialization. This violates the "no external dependencies" rule.

2. **Fast Test Suite**: Most tests run in <50ms. The 95th percentile is ~0.3s.

3. **Test Isolation**: The `clear_settings_cache` and `use_test_settings` autouse fixtures add minimal overhead but ensure test isolation.

4. **Database Fixtures**: In-memory SQLite setup/teardown is fast (~5ms per test).

---

## 4. Coverage Analysis

### Well-Covered Areas (>80%)

| Module | Coverage | Notes |
|--------|----------|-------|
| `api/schemas.py` | 100% | All Pydantic models tested |
| `api/face_schemas.py` | 100% | Complete schema coverage |
| `api/training_schemas.py` | 100% | Complete schema coverage |
| `api/vector_schemas.py` | 100% | Complete schema coverage |
| `api/admin_schemas.py` | 100% | Complete schema coverage |
| `api/queue_schemas.py` | 100% | Complete schema coverage |
| `api/routes/__init__.py` | 100% | Router registration |
| `core/config.py` | 92% | Settings well tested |
| `core/device.py` | 95% | Device detection tested |
| `db/models.py` | 97% | Models exercised by other tests |
| `services/temporal_service.py` | 99% | Excellent coverage |
| `services/perceptual_hash.py` | 100% | Complete |
| `services/fusion.py` | 97% | Search fusion tested |
| `services/embedding_router.py` | 100% | Routing tested |
| `faces/detector.py` | 96% | Detection well tested |
| `vector/qdrant.py` | 76% | Reasonable coverage |
| `vector/centroid_qdrant.py` | 76% | Reasonable coverage |

### Poorly-Covered Areas (<30%)

| Module | Coverage | Notes |
|--------|----------|-------|
| `faces/dual_clusterer.py` | 0% | Entire file untested |
| `faces/trainer.py` | 0% | Entire file untested |
| `services/periodic_scanner.py` | 0% | Entire file untested |
| `queue/training_jobs.py` | 10% | Critical queue jobs barely tested |
| `queue/auto_detection_jobs.py` | 11% | Auto-detection untested |
| `queue/face_jobs.py` | 13% | Face jobs minimally tested |
| `api/routes/face_centroids.py` | 17% | Centroid routes untested |
| `services/face_clustering_restart_service.py` | 18% | Restart service untested |
| `services/face_detection_restart_service.py` | 20% | Restart service untested |
| `services/evidence_service.py` | 20% | Evidence service untested |
| `services/directory_service.py` | 21% | Directory service untested |
| `services/queue_service.py` | 23% | Queue orchestration untested |
| `api/routes/images.py` | 24% | Image routes untested |
| `queue/jobs.py` | 25% | Base jobs untested |
| `services/thumbnail_service.py` | 25% | Thumbnail service untested |
| `api/routes/training.py` | 26% | Training routes minimal |
| `db/sync_operations.py` | 27% | Sync ops untested |
| `services/file_watcher.py` | 27% | File watcher untested |
| `api/routes/face_sessions.py` | 27% | Face sessions untested |
| `services/siglip_embedding.py` | 29% | SigLIP service untested |
| `api/routes/faces.py` | 30% | 839 statements, 590 missed |

### Critical Coverage Gaps

1. **`faces/dual_clusterer.py` (0%)**: The dual clustering algorithm (known+unknown) has zero test coverage. This is the primary production clustering path.

2. **`faces/trainer.py` (0%)**: The face training pipeline has zero test coverage.

3. **`queue/face_jobs.py` (13%)**: 752 statements with only 96 covered. This contains all background face processing jobs.

4. **`queue/training_jobs.py` (10%)**: 347 statements with only 35 covered. Critical training pipeline.

5. **`api/routes/faces.py` (30%)**: The largest route file (839 stmts) with 590 missed lines.

---

## 5. Test Architecture Assessment

### Strengths

1. **Excellent fixture design**: `conftest.py` properly isolates tests with in-memory SQLite, mock Qdrant, and mock embedding service.
2. **Settings safety**: `clear_settings_cache` + `use_test_settings` autouse fixtures prevent production data access.
3. **Schema coverage**: 100% coverage on all Pydantic schemas ensures API contract stability.
4. **Deterministic mocks**: `MockEmbeddingService` uses hash-based vectors for reproducibility.
5. **Good test structure**: Clear separation of api/, unit/, integration/, and faces/ test directories.

### Weaknesses

1. **Stale tests**: 35 failures (4.4%) indicate tests weren't updated when code changed. This is a CI/CD gap.
2. **Missing test client pattern**: Some tests use `TestClient(app)` directly instead of the shared fixture, bypassing dependency injection.
3. **Mock depth issues**: Several test categories fail because mocks don't intercept at the correct depth (especially for Qdrant client access).
4. **Unimplemented stubs**: 5 test stubs (`test_training_unified_progress.py`) with `pass` bodies reference non-existent fixtures.
5. **Real model loading**: Two tests accidentally load InsightFace GPU models, adding ~7s to the suite and violating the no-external-deps rule.

---

## 6. Warnings

1. **DeprecationWarning**: `regex` parameter in FastAPI `Query()` is deprecated, use `pattern` instead. (2 occurrences in `api/routes/faces.py:509,514`)
2. **DeprecationWarning**: `job.result` in rq is deprecated, use `job.return_value` instead.
3. **RuntimeWarning**: Coroutine `Connection._cancel` never awaited in `test_fills_exemplars_after_temporal`.

---

## 7. Recommendations (Priority Ordered)

### P0 - Fix Broken Tests (to restore CI green)

1. **Fix FaceClusterer tests**: Add `qdrant_client` parameter to all 7 test instantiations
2. **Fix restart workflow tests**: Either inject sync session properly or convert to async
3. **Fix multiproto propagation tests**: Trace and correct Qdrant client mock path
4. **Fix config post-training tests**: Use shared `test_client` fixture instead of `TestClient(app)`
5. **Fix unified progress tests**: Rename `async_client` fixture to `test_client`

### P1 - Improve Coverage

1. **Add tests for `dual_clusterer.py`**: 0% coverage on production clustering
2. **Add tests for `trainer.py`**: 0% coverage on face training pipeline
3. **Add tests for queue jobs**: `face_jobs.py` and `training_jobs.py` are critically under-tested
4. **Add tests for route handlers**: `faces.py`, `training.py`, `face_centroids.py`

### P2 - Fix Production Code Issues

1. **Create migration for new config keys**: 4 keys missing from migrations
2. **Fix `regex` deprecation**: Update to `pattern` in `api/routes/faces.py`
3. **Update prototype quota message**: Align code with test expectations or vice versa
4. **Fix InsightFace mock leakage**: Ensure model loading is always mocked in batch tests

### P3 - Architecture Improvements

1. **Standardize test client usage**: Remove all direct `TestClient(app)` usage
2. **Add CI enforcement**: Fail build on test failures (currently 35 failures indicate this isn't enforced)
3. **Implement the TODO test stubs**: 5 unified progress tests are empty
4. **Add coverage thresholds**: Set minimum coverage gate (e.g., 60%) to prevent regression
