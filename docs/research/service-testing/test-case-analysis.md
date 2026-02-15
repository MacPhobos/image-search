# Image Search Service - Comprehensive Test Case Analysis

**Date**: 2026-02-06
**Researcher**: test-researcher
**Scope**: All test files in `image-search-service/tests/`

---

## Executive Summary

The image-search-service test suite contains **60+ test files** organized across 7 directories with approximately **450+ individual test functions**. The suite uses SQLite in-memory databases and in-memory Qdrant instances to run without external dependencies. Overall test quality is high, with consistent patterns, good assertion coverage, and thorough edge-case testing. Key gaps exist in integration-level coverage, error-path testing for some services, and several commented-out/skipped tests that were never implemented.

---

## 1. Test Infrastructure

### 1.1 Root `conftest.py` (`tests/conftest.py`)

**Fixtures provided:**

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `clear_settings_cache` | autouse | Prevents production settings leaking into tests |
| `use_test_settings` | autouse | Overrides Qdrant collection names to `test_*` variants |
| `db_engine` | function | In-memory SQLite async engine with table creation/teardown |
| `db_session` | function | Async session with rollback after each test |
| `sync_db_engine` | function | Synchronous SQLite for background job tests |
| `sync_db_session` | function | Synchronous session with rollback |
| `qdrant_client` | function | In-memory Qdrant with `test_image_assets` and `test_faces` collections |
| `mock_embedding_service` | function | `MockEmbeddingService` using MD5 hash for deterministic 512-dim vectors |
| `face_qdrant_client` | function | `FaceQdrantClient` wrapper injected with mock Qdrant |
| `temp_image_factory` | function | Factory for creating test images with gradient patterns |
| `test_client` | function | Full `AsyncClient` with all FastAPI dependency overrides |
| `mock_queue` | function | Mock RQ queue tracking enqueued jobs |

**Key design decisions:**
- `MockEmbeddingService` generates deterministic vectors via MD5 hashing - eliminates OpenCLIP dependency
- Two autouse fixtures (`clear_settings_cache`, `use_test_settings`) provide critical safety: collection names always prefixed with `test_` preventing accidental production data deletion
- `test_client` overrides 5 dependencies: `get_db`, `get_sync_db`, `get_qdrant_client`, `get_face_qdrant_client`, `get_embedding_service`

### 1.2 Faces `conftest.py` (`tests/faces/conftest.py`)

Additional fixtures for face-specific tests:
- `mock_face_embedding`, `mock_detected_face`, `mock_image_asset`
- `mock_face_instance`, `mock_person`, `mock_qdrant_client`, `mock_insightface`

---

## 2. Test Directory Structure & File Inventory

```
tests/
├── conftest.py                              # Root fixtures (11 fixtures)
├── api/                                     # API route tests (25 files)
│   ├── test_health.py                       # 2 tests
│   ├── test_ingest.py                       # 5 tests
│   ├── test_categories.py                   # 14 tests
│   ├── test_vectors.py                      # 17 tests
│   ├── test_search.py                       # 19 tests
│   ├── test_images.py                       # 7 tests
│   ├── test_queues.py                       # 14 tests
│   ├── test_faces_routes.py                 # ~35 tests
│   ├── test_get_faces_for_asset.py          # 2 tests
│   ├── test_get_faces_orphaned_person.py    # 1 test
│   ├── test_person_name_regression.py       # 5 tests
│   ├── test_face_suggestions_list.py        # 14 tests
│   ├── test_face_suggestions.py             # 12 tests
│   ├── test_face_suggestion_cleanup.py      # 8 tests
│   ├── test_face_session_suggestions.py     # 7 tests
│   ├── test_suggestion_regeneration.py      # 11 tests
│   ├── test_prototype_endpoints.py          # 11 tests
│   ├── test_training_routes.py              # 12 tests
│   ├── test_clusters_filtering.py           # ~8 tests
│   ├── test_training_deduplication.py       # ~6 tests
│   ├── test_training_unified_progress.py    # ~8 tests
│   ├── test_unified_people_endpoint.py      # ~10 tests
│   └── test_recompute_with_rescan.py        # ~5 tests
├── core/                                    # Core utility tests (1 file)
│   └── test_device.py                       # ~4 tests (device detection)
├── faces/                                   # Face pipeline tests (5 files)
│   ├── conftest.py                          # Face-specific fixtures
│   ├── test_detector.py                     # ~10 tests
│   ├── test_assigner.py                     # ~8 tests
│   ├── test_clusterer.py                    # ~10 tests
│   └── test_service.py                      # ~8 tests
├── integration/                             # Integration tests (3 files)
│   ├── test_listener_worker.py              # ~6 tests
│   ├── test_restart_workflows.py            # ~8 tests
│   └── test_temporal_migration.py           # ~5 tests
└── unit/                                    # Unit tests (16+ files)
    ├── test_embedding_service.py            # ~8 tests
    ├── test_bootstrap_qdrant.py             # ~6 tests
    ├── test_qdrant_safety.py                # ~4 tests
    ├── test_qdrant_wrapper.py               # 16 tests
    ├── test_centroid_qdrant.py              # 14 tests
    ├── test_config_keys_sync.py             # 10 tests
    ├── test_temporal_service.py             # ~57 tests (8 classes)
    ├── test_person_birth_date.py            # 12 tests (+4 commented stubs)
    ├── test_admin_export_import.py          # ~22 tests (+4 skipped)
    ├── test_embedding_router.py             # 7 tests
    ├── test_fusion.py                       # 13 tests
    ├── test_faces_cli.py                    # ~8 tests
    ├── test_siglip_embedding_service.py     # ~6 tests
    ├── test_perceptual_hash.py              # ~8 tests
    ├── test_face_jobs_payload.py            # ~6 tests
    ├── test_prototype_selection.py          # ~10 tests
    ├── test_suggestion_qdrant_sync.py       # ~6 tests
    ├── api/                                 # Schema/config unit tests (4 files)
    │   ├── test_schemas.py                  # 15 tests (3 classes)
    │   ├── test_face_schemas.py             # ~10 tests
    │   ├── test_config_unknown_clustering.py # ~8 tests
    │   └── test_config_post_training_suggestions.py # 13 tests
    ├── services/                            # Service unit tests (3 files)
    │   ├── test_face_clustering_service.py  # 15 tests (3 classes)
    │   ├── test_person_service.py           # 14 tests
    │   └── test_exif_service.py             # 26 tests
    └── queue/                               # Queue job tests (3 files)
        ├── test_centroid_post_training.py   # 13 tests (4 classes)
        ├── test_post_training_suggestions_job.py # 11 tests (4 classes)
        └── test_multiproto_propagation.py   # 12 tests

```

---

## 3. Detailed Test Analysis by Category

### 3.1 API Route Tests (`tests/api/`)

**Coverage summary:**

| Endpoint Area | File | Test Count | Rating |
|---------------|------|------------|--------|
| Health | test_health.py | 2 | Good |
| Ingest | test_ingest.py | 5 | Good |
| Categories | test_categories.py | 14 | Excellent - full CRUD |
| Vectors/Qdrant | test_vectors.py | 17 | Excellent |
| Search | test_search.py | 19 | Excellent - text/image/hybrid/compose |
| Images | test_images.py | 7 | Good |
| Queues | test_queues.py | 14 | Excellent |
| Face Routes | test_faces_routes.py | ~35 | Excellent - comprehensive |
| Face/Asset | test_get_faces_for_asset.py | 2 | Regression tests |
| Face/Orphan | test_get_faces_orphaned_person.py | 1 | Edge case |
| Person Name | test_person_name_regression.py | 5 | Regression tests |
| Suggestions List | test_face_suggestions_list.py | 14 | Excellent |
| Suggestions Gen | test_face_suggestions.py | 12 | Excellent |
| Suggestion Cleanup | test_face_suggestion_cleanup.py | 8 | Good |
| Session Suggestions | test_face_session_suggestions.py | 7 | Good |
| Suggestion Regen | test_suggestion_regeneration.py | 11 | Good |
| Prototypes | test_prototype_endpoints.py | 11 | Good |
| Training Routes | test_training_routes.py | 12 | Good |
| Cluster Filtering | test_clusters_filtering.py | ~8 | Good |
| Training Dedup | test_training_deduplication.py | ~6 | Good |
| Unified Progress | test_training_unified_progress.py | ~8 | Good |
| Unified People | test_unified_people_endpoint.py | ~10 | Good |
| Recompute/Rescan | test_recompute_with_rescan.py | ~5 | Good |

**Patterns:**
- All API tests use `test_client` fixture (httpx AsyncClient with ASGITransport)
- Tests are `async def` with `@pytest.mark.asyncio`
- Request/response assertion pattern: send request, check status code, validate JSON body
- Good use of database setup within tests (create entities, then test endpoint behavior)

**Strengths:**
- Face suggestions have 5 separate test files covering generation, listing, cleanup, sessions, and regeneration
- Regression tests exist for specific bugs (orphaned persons, person name population)
- Good boundary testing (empty results, pagination, invalid inputs)

### 3.2 Unit Tests - Schemas (`tests/unit/api/`)

#### `test_schemas.py` (15 tests, 3 classes)
- **TestLocationMetadata** (2 tests): Alias serialization (`lat`/`lng`), required field validation
- **TestCameraMetadata** (4 tests): Both fields, make-only, model-only, both None
- **TestAsset** (9 tests): Basic fields, computed fields (url, thumbnail_url, filename), filename extraction from paths, EXIF metadata (taken_at, camera, location), partial/missing metadata, JSON serialization with camelCase aliases, dict construction

#### `test_face_schemas.py` (~10 tests)
- Face schema validation tests (persisted output - details from earlier reads)

#### `test_config_unknown_clustering.py` (~8 tests)
- Unknown clustering configuration validation (persisted output)

#### `test_config_post_training_suggestions.py` (13 tests, 1 class)
- **TestPostTrainingSuggestionsConfig**: GET returns post-training fields, PUT mode switching (all/top_n), invalid mode rejection (422), boundary values (1, 100), too high/low/negative count rejection, partial updates (mode-only, count-only), preserves other config values
- Uses `TestClient(app)` directly (synchronous) rather than async test_client
- Pattern concern: This uses real FastAPI app without dependency overrides, potentially hitting real database

### 3.3 Unit Tests - Services (`tests/unit/services/`)

#### `test_face_clustering_service.py` (15 tests, 3 classes)
- **TestCalculateClusterConfidence** (8 tests): Single face (perfect), identical embeddings (~1.0), orthogonal (near 0), high/low similarity clusters, large cluster sampling (max_faces_for_calculation), missing embeddings (0.0), empty cluster (ValueError)
- **TestCosineSimilarity** (4 tests): Identical (1.0), orthogonal (0.0), opposite (-1.0), zero-norm (0.0)
- **TestSelectRepresentativeFace** (3 tests): Highest quality selection, empty cluster returns None, larger bbox preference when quality equal
- Uses `np.random.seed(42)` for reproducible random vector tests

#### `test_person_service.py` (14 tests, 1 class)
- **TestPersonService**: Display name generation (regular/noise/sequential), filtering behavior (only identified, only unidentified, include noise, all types, no noise when zero), sorting (face_count desc/asc, name asc/desc, mixed types), count accuracy, empty result
- Uses `AsyncMock` for internal service methods (`_get_identified_people`, `_get_unidentified_clusters`, `_get_noise_faces`)

#### `test_exif_service.py` (26 tests)
- **EXIF extraction**: DateTimeOriginal, DateTimeDigitized, precedence, missing datetime, camera make/model, field truncation (100 chars), GPS coordinates (N/W and S/E hemispheres), missing GPS, metadata dict, no EXIF, nonexistent file, corrupt image
- **Internal methods**: `_parse_exif_datetime` (valid, invalid, bytes), `_convert_to_degrees`, `_parse_gps` (valid, missing, invalid coords)
- **Sanitization (`_sanitize_for_json`)**: Null byte removal (strings, bytes, nested structures), type preservation (int, float, bool, None, datetime), IFDRational conversion (exposure time, f-number, focal length, zero denominator fallback), null bytes in camera fields (regression for asyncpg CharacterNotInRepertoireError)
- **Other**: Singleton pattern (`get_exif_service`), thread safety (10 concurrent threads), whitespace trimming, complete EXIF example
- Excellent test - uses real PIL image creation with EXIF injection via sub-IFDs

### 3.4 Unit Tests - Queue Jobs (`tests/unit/queue/`)

#### `test_centroid_post_training.py` (13 tests, 4 classes)
- **TestCentroidPostTrainingEnabled** (3 tests): Centroid job for sufficient faces (>= min_faces), prototype job for insufficient faces, mixed scenario (both job types)
- **TestCentroidPostTrainingDisabled** (2 tests): All persons get prototype jobs when disabled, prototype used even with high face count
- **TestCentroidThresholdBoundary** (2 tests): Exactly at threshold (uses centroid), one below (uses prototype)
- **TestConfigurationDefaults** (2 tests): Default centroids enabled, default min_faces is 5
- **TestJobTypeLogging** (3 tests): Correct job_type/reason logging for centroid, insufficient_faces, centroids_disabled
- Uses real sync DB session with Person, FaceInstance, ImageAsset models

#### `test_post_training_suggestions_job.py` (11 tests, 4 classes)
- **TestPostTrainingSuggestionsAllMode** (2 tests): Returns all persons with faces, empty result
- **TestPostTrainingSuggestionsTopNMode** (3 tests): LIMIT 3, LIMIT 1, LIMIT > count
- **TestPostTrainingSuggestionsEdgeCases** (2 tests): Zero-face persons excluded (HAVING clause), descending order verification
- **TestPostTrainingSuggestionsQueryComponents** (4 tests): HAVING filters zero count, JOIN matching, LIMIT after ORDER BY
- Tests raw SQLAlchemy queries directly (not through service layer)

#### `test_multiproto_propagation.py` (12 tests, 1 class)
- **TestMultiPrototypePropagation**: Person not found error, no prototypes error, single prototype suggestions, score aggregation (MAX across prototypes), max score as aggregate_confidence, best quality prototype as source, expiring old pending suggestions, skips assigned faces, max_suggestions limit, min_confidence threshold, invalid face_id handling (not-a-uuid, None payload, missing field), detailed metadata verification
- Comprehensive factory fixtures: `create_person`, `create_image_asset`, `create_face_instance`, `create_prototype`, `create_suggestion`, `mock_qdrant`
- Tests actual `propagate_person_label_multiproto_job` function with patched dependencies

### 3.5 Unit Tests - Core Components

#### `test_config_keys_sync.py` (10 tests)
- Critical drift detection between ConfigService.DEFAULTS, migration files, and API endpoints
- Regex parsing of migration SQL files to verify INSERT statements match DEFAULTS
- Business logic: suggestion_threshold < auto_assign_threshold constraint

#### `test_temporal_service.py` (~57 tests, 8 classes)
- Most comprehensive single test file in the suite
- Covers: age era classification (14 boundary tests), era age ranges, decade extraction, temporal quality scoring, coverage gaps, birth year estimation (with outlier handling), temporal metadata extraction, face enrichment pipeline

#### `test_qdrant_wrapper.py` (16 tests)
- Wrapper function tests: upsert, search (limit, offset), ensure_collection (create/idempotent), person_id filters (camelCase/snake_case compatibility), combined filters, update_payload operations

#### `test_centroid_qdrant.py` (14 tests)
- Singleton pattern, collection lifecycle (create/skip/reset), CRUD operations, UUID-to-string conversion, dict vector format handling, scroll with filters, search_faces_with_centroid

#### `test_embedding_router.py` (7 tests)
- CLIP/SigLIP selection logic: default CLIP, SigLIP when enabled, gradual rollout (deterministic bucketing by user_id), 0%/100% rollout, use_siglip override, random bucketing without user_id

#### `test_fusion.py` (13 tests)
- RRF algorithm: empty/single/dual lists, same/reversed order, non-overlapping results, k parameter sensitivity, weighted variants, metadata capture, monotonic score ordering

#### `test_person_birth_date.py` (12 tests + 4 commented stubs)
- `calculate_age_at_date`: basic calculation, before/on/after birthday, None inputs, infant/zero/leap year/elderly, negative age returns 0
- 4 commented-out integration test stubs (never implemented)

#### `test_admin_export_import.py` (~22 active + 4 skipped)
- Export: empty DB, persons without faces, with faces, max limit, ordering, excludes inactive/hidden, alphabetical, path normalization, verify_paths
- Bbox matching: exact, tolerance, outside, closest, empty
- Import: dry run, creates persons, case-insensitive find, face matching, missing images, tolerance, Qdrant updates, error continuation
- 4 skipped tests: batch embedding check (not implemented)

### 3.6 Face Pipeline Tests (`tests/faces/`)

| File | Test Count | Coverage |
|------|------------|----------|
| test_detector.py | ~10 | Face detection with InsightFace mock |
| test_assigner.py | ~8 | Face-to-person assignment |
| test_clusterer.py | ~10 | HDBSCAN clustering |
| test_service.py | ~8 | Orchestration service |

### 3.7 Integration Tests (`tests/integration/`)

| File | Test Count | Coverage |
|------|------------|----------|
| test_listener_worker.py | ~6 | Event listener + worker coordination |
| test_restart_workflows.py | ~8 | Training session restart flows |
| test_temporal_migration.py | ~5 | Temporal data migration |

---

## 4. Mocking Patterns

### 4.1 Dependency Injection Overrides (Primary Pattern)
```python
# FastAPI dependency override via test_client fixture
app.dependency_overrides[get_db] = override_get_db
app.dependency_overrides[get_qdrant_client] = override_get_qdrant
app.dependency_overrides[get_embedding_service] = override_get_embedding
```
Used in: All API tests via `test_client` fixture

### 4.2 MagicMock + Fixture Injection
```python
# Inject mock directly into singleton instance
client = CentroidQdrantClient.get_instance()
client._client = mock_qdrant_client
```
Used in: `test_centroid_qdrant.py`

### 4.3 Monkeypatch + Environment Variables
```python
monkeypatch.setenv("QDRANT_COLLECTION", "test_image_assets")
get_settings.cache_clear()
```
Used in: `conftest.py` autouse fixtures

### 4.4 `unittest.mock.patch` Context Managers
```python
with patch("module.get_sync_session", return_value=sync_db_session):
    result = job_function(args)
```
Used in: Queue job tests, embedding router tests

### 4.5 AsyncMock for Service Internals
```python
service._get_identified_people = AsyncMock(return_value=mock_data)
```
Used in: `test_person_service.py`

### 4.6 Factory Fixtures
```python
@pytest.fixture
def create_face_instance(sync_db_session, create_image_asset):
    def _create(person_id=None, quality_score=0.75, ...):
        ...
    return _create
```
Used in: `test_multiproto_propagation.py`, `test_centroid_post_training.py`

---

## 5. Assertion Patterns

### Common Assertion Categories

1. **Status code checks**: `assert response.status_code == 200`
2. **JSON body validation**: `assert data["field"] == expected`
3. **List length**: `assert len(results) == N`
4. **Ordering verification**: `assert values == sorted(values, reverse=True)`
5. **Float comparison**: `assert abs(value - expected) < 0.0001`
6. **Exception testing**: `with pytest.raises(ValueError, match="pattern")`
7. **Mock call verification**: `mock.assert_called_once()`, `mock.assert_not_called()`
8. **Type checking**: `assert isinstance(result, ExpectedType)`
9. **Membership testing**: `assert "key" in data`, `assert item in collection`
10. **DB state verification**: `session.get(Model, id)` after operation

---

## 6. Coverage Gaps and Anti-Patterns

### 6.1 Coverage Gaps

#### Missing/Thin Coverage Areas

1. **Admin routes**: No dedicated API test file for admin endpoints (export/import tested at unit level only, not through HTTP)
2. **Error response format consistency**: Tests check status codes but rarely validate error response body structure (e.g., `{"detail": "message"}` format)
3. **Concurrent request handling**: No tests for race conditions (e.g., two users accepting the same suggestion simultaneously)
4. **Pagination edge cases**: Some endpoints lack offset+limit boundary tests (e.g., offset > total items)
5. **Config endpoint isolation**: `test_config_post_training_suggestions.py` uses `TestClient(app)` directly without dependency overrides, potentially hitting real config storage
6. **WebSocket/streaming**: No test coverage visible (may not be applicable)
7. **Background job error recovery**: Limited testing of job failure + retry scenarios
8. **Rate limiting/throttling**: No tests (feature may not be implemented yet)
9. **CORS configuration**: Not tested
10. **Request validation edge cases**: Limited testing of malformed JSON, missing Content-Type headers

#### Skipped/Commented Tests

1. **`test_admin_export_import.py`**: 4 skipped tests for batch embedding check functions (not implemented)
2. **`test_person_birth_date.py`**: 4 commented-out integration test stubs (PATCH endpoint, name-only update, photos-with-age)

### 6.2 Anti-Patterns Identified

1. **Direct singleton mutation**: `CentroidQdrantClient._instance = None` in tests - fragile, couples tests to implementation detail
2. **TestClient without overrides**: `test_config_post_training_suggestions.py` creates `TestClient(app)` at module level without dependency injection, potentially using real database/services
3. **Inline business logic in test**: `test_centroid_post_training.py` reimplements the centroid/prototype decision tree in test code rather than testing the actual function - tests may pass even if production logic diverges
4. **Non-deterministic random in tests**: `test_face_clustering_service.py` uses `np.random.seed(42)` in one test but not another (`test_low_similarity_cluster_returns_low_confidence` uses unseeded random)
5. **Large fixture chains**: `test_multiproto_propagation.py` has 5-level fixture chains (sync_db_session -> create_image_asset -> create_face_instance -> create_prototype -> test) - complex dependency graphs make failures harder to diagnose
6. **Overly broad assertions**: Some tests use `0.0 <= confidence <= 1.0` which will pass for any valid float - doesn't verify correctness

### 6.3 Strengths

1. **Zero external dependencies**: All tests run without Postgres, Redis, or Qdrant - excellent for CI/CD
2. **Regression test discipline**: Multiple dedicated regression test files (person name, orphaned persons, face-for-asset)
3. **Boundary testing**: Good boundary value analysis in temporal service, config validation, threshold tests
4. **Safety fixtures**: Autouse fixtures prevent test-to-production collection name leaking (critical safety feature)
5. **Factory fixtures**: Well-designed factory patterns in queue tests for creating complex test data
6. **Comprehensive EXIF testing**: `test_exif_service.py` creates real images with EXIF metadata via PIL - high fidelity
7. **Config drift detection**: `test_config_keys_sync.py` prevents DEFAULTS/migration/endpoint divergence
8. **Thread safety testing**: EXIF service tested with 10 concurrent threads
9. **Sanitization regression tests**: PostgreSQL JSONB null-byte issue has dedicated regression tests

---

## 7. Test Count Summary

| Category | Files | Approx. Test Count |
|----------|-------|--------------------|
| API Routes | 23 | ~225 |
| Core | 1 | ~4 |
| Faces Pipeline | 4 | ~36 |
| Integration | 3 | ~19 |
| Unit - General | 12 | ~115 |
| Unit - API/Schemas | 4 | ~46 |
| Unit - Services | 3 | ~55 |
| Unit - Queue | 3 | ~36 |
| **Total** | **53+** | **~450+** |

---

## 8. Recommendations

### High Priority

1. **Fix `test_config_post_training_suggestions.py` isolation**: Replace module-level `TestClient(app)` with proper dependency-overridden client to prevent test pollution
2. **Implement skipped tests**: The 4 skipped batch embedding tests in `test_admin_export_import.py` should be implemented or removed
3. **Remove commented stubs**: The 4 commented integration test stubs in `test_person_birth_date.py` should either be implemented or deleted

### Medium Priority

4. **Test production decision logic directly**: `test_centroid_post_training.py` should test the actual job dispatching function rather than reimplementing the decision tree
5. **Add concurrent access tests**: Face suggestion acceptance, person assignment, and training operations should have concurrent access tests
6. **Standardize random seeding**: All tests using random vectors should use fixed seeds for reproducibility
7. **Add admin endpoint API tests**: Export/import functionality only has unit tests - add HTTP-level integration tests

### Low Priority

8. **Error response format tests**: Add assertions on error response body structure, not just status codes
9. **Pagination boundary tests**: Add tests for offset > total_items, negative offset, zero limit
10. **Add performance regression tests**: Key operations (search, clustering) should have timing assertions or benchmarks

---

## 9. Technology & Framework Summary

| Component | Technology | Version Evidence |
|-----------|-----------|-----------------|
| Test Runner | pytest | pytest-asyncio for async tests |
| HTTP Client | httpx (AsyncClient + ASGITransport) | For FastAPI testing |
| Mocking | unittest.mock (MagicMock, AsyncMock, patch) | Standard library |
| DB (Test) | SQLite in-memory (aiosqlite + sqlite3) | Both async and sync |
| Vector DB (Test) | Qdrant in-memory (`:memory:` mode) | qdrant-client |
| Image Creation | PIL/Pillow | For EXIF and temp image tests |
| Array Operations | NumPy | For vector operations in clustering tests |
| Validation | Pydantic | Schema validation tests |

---

*Analysis complete. All 53+ test files across 7 directories have been cataloged.*
