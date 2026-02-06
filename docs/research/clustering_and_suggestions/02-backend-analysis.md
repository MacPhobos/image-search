# Backend Architecture Analysis: Clustering & Suggestions System

**Research Date**: 2026-02-06
**Researcher**: Backend Researcher (Claude Opus 4.6)
**Scope**: Python backend service (`image-search-service/`) -- API routes, data models, services, queue jobs, vector clients, and configuration for face clustering, suggestions, and centroid systems.

---

## Executive Summary

The image-search backend is a **FastAPI + PostgreSQL + Qdrant + Redis/RQ** service implementing a sophisticated face clustering and suggestion pipeline. The architecture follows a dual-storage pattern: PostgreSQL for relational metadata and Qdrant for vector embeddings (512-dim ArcFace). Background processing is handled by Redis/RQ workers executing synchronous jobs due to macOS fork-safety requirements.

**Key Strengths:**
- Well-structured dual-mode clustering (supervised + unsupervised)
- Robust centroid computation with outlier trimming and staleness detection
- Comprehensive resume/pause support for long-running face detection sessions
- Good use of perceptual hash deduplication in centroid suggestions
- Production safety guards on destructive operations (Qdrant collection reset)

**Critical Concerns:**
- **Pervasive N+1 query pattern** in Qdrant embedding retrieval (7 call sites retrieve embeddings one-by-one in loops)
- **Massive API route file** (`faces.py` at 2,450 lines with 28+ endpoints) violates single-responsibility principle
- **Sync/async session mismatch** between API routes (async) and worker jobs (sync), with duplicated logic
- **Duplicated centroid computation logic** across `centroid_service.py` (full featured) and `find_more_centroid_suggestions_job` (simplified inline)
- **Missing race condition protection** on face assignment -- no SELECT FOR UPDATE or optimistic locking
- **Hardcoded collection name** `"faces"` in `clusterer.py` line 88 bypasses configurable collection name

---

## 1. API Architecture for Clustering & Faces

### 1.1 Endpoint Map

The face/clustering API is split across four route files:

**`api/routes/faces.py`** (2,450 lines, 28+ endpoints):
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/clusters` | List face clusters (line 84) |
| GET | `/clusters/{cluster_id}` | Get cluster detail (line 310) |
| POST | `/clusters/{cluster_id}/label` | Label a cluster as a person (line 341) |
| POST | `/clusters/{cluster_id}/split` | Split a cluster (line 440) |
| GET | `/people` | Unified people list (line 474) |
| GET | `/persons` | Person list (line 523) |
| GET | `/persons/{person_id}` | Person detail (line 575) |
| POST | `/persons` | Create person (line 632) |
| PATCH | `/persons/{person_id}` | Update person (line 665) |
| GET | `/persons/{person_id}/photos` | Person photos (line 729) |
| POST | `/persons/{person_id}/merge` | Merge persons (line 1050) |
| POST | `/persons/{person_id}/photos/bulk-remove` | Bulk remove photos (line 1134) |
| POST | `/persons/{person_id}/photos/bulk-move` | Bulk move photos (line 1276) |
| POST | `/detect/{asset_id}` | Detect faces in asset (line 1532) |
| POST | `/cluster` | Cluster all faces (line 1567) |
| GET | `/assets/{asset_id}` | Get face instances for asset (line 1593) |
| POST | `/faces/{face_id}/assign` | Assign face to person (line 1616) |
| DELETE | `/faces/{face_id}/person` | Unassign face (line 1754) |
| GET | `/faces/{face_id}/suggestions` | Get suggestions for face (line 1872) |
| POST | `/cluster/dual` | Dual-mode cluster (line 1989) |
| POST | `/train` | Train matching model (line 2035) |
| POST | `/prototypes/...` | Prototype operations (line 2086) |
| DELETE | `/prototypes/...` | Delete prototypes (line 2128) |
| GET | `/stats` | Face statistics (line 2182) |

**`api/routes/face_suggestions.py`** (928 lines, 8 endpoints):
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/faces/suggestions` | List suggestions (grouped or flat) (line 314) |
| GET | `/faces/suggestions/stats` | Suggestion statistics (line 385) |
| GET | `/faces/suggestions/{id}` | Get suggestion (line 441) |
| POST | `/faces/suggestions/{id}/accept` | Accept suggestion (line 502) |
| POST | `/faces/suggestions/{id}/reject` | Reject suggestion (line 574) |
| POST | `/faces/suggestions/bulk-action` | Bulk accept/reject (line 651) |
| POST | `/faces/suggestions/persons/{id}/find-more` | Find more via prototypes (line 754) |
| POST | `/faces/suggestions/persons/{id}/find-more-centroid` | Find more via centroid (line 841) |

**`api/routes/face_centroids.py`** (484 lines, 4 endpoints):
| Method | Path | Purpose |
|--------|------|---------|
| POST | `/faces/centroids/persons/{id}/compute` | Compute centroids (line 80) |
| GET | `/faces/centroids/persons/{id}` | Get centroids (line 181) |
| POST | `/faces/centroids/persons/{id}/suggestions` | Centroid suggestions (line 261) |
| DELETE | `/faces/centroids/persons/{id}` | Delete centroids (line 429) |

### 1.2 Issues Found

**CRITICAL: God Object Route File**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/api/routes/faces.py`
- **Lines**: 2,450
- **Endpoints**: 28+
- **Impact**: Extremely difficult to maintain, test, or review. Violates SRP. Contains cluster operations, person CRUD, face assignment, detection, training, and prototype management all in one file.

**CRITICAL: Missing Optimistic Locking on Face Assignment**
- **Files**: `api/routes/faces.py` (line 1616, `assign_face`), `api/routes/face_suggestions.py` (line 527, `accept_suggestion`)
- **Pattern**: Both endpoints read a `FaceInstance`, set `person_id`, then commit -- without `SELECT FOR UPDATE` or version checking. Two concurrent suggestion acceptances for the same face could cause inconsistent state.

**MODERATE: Inconsistent Pagination**
- `face_suggestions.py` uses group-based pagination with window functions (sophisticated, line 203) alongside legacy flat pagination (line 105).
- `faces.py` person photos endpoint (line 729) uses offset-based pagination.
- No cursor-based pagination anywhere -- problematic for large datasets with concurrent insertions.

**MODERATE: Redundant Suggestion Response Building**
- `face_suggestions.py` has `_build_suggestion_response()` helper (line 48) but `get_suggestion()` (line 441-499) duplicates the same response construction inline instead of using the helper.

**LOW: Query Parameter Aliasing Inconsistency**
- Some endpoints use `alias="pageSize"` (camelCase) while others use snake_case, creating inconsistent API surface.

### 1.3 Recommendations

1. **Split `faces.py`** into separate route files: `face_clusters.py`, `face_persons.py`, `face_detection.py`, `face_assignment.py`, `face_prototypes.py`, `face_training.py`.
2. **Add optimistic locking** to face assignment endpoints using a version column or `SELECT FOR UPDATE`.
3. **Standardize pagination** across all endpoints -- consider cursor-based for large result sets.
4. **Remove duplicated response building** in `get_suggestion()` -- use `_build_suggestion_response()`.

---

## 2. Data Model & Storage

### 2.1 Database Schema (PostgreSQL)

**File**: `/export/workspace/image-search/image-search-service/src/image_search_service/db/models.py` (971 lines)

**Core Face Models:**

| Model | PK | Key Fields | Relationships |
|-------|-----|-----------|---------------|
| `Person` | UUID `id` | name, status (active/merged/hidden), birth_date, merged_into_id | faces, prototypes, centroids |
| `FaceInstance` | UUID `id` | asset_id FK, bbox (x/y/w/h), detection_confidence, quality_score, qdrant_point_id, cluster_id, person_id FK, landmarks JSONB | person, asset, suggestions |
| `PersonPrototype` | UUID `id` | person_id FK, face_instance_id FK, qdrant_point_id, role (enum), age_era_bucket, is_pinned | person, face_instance |
| `FaceSuggestion` | int `id` | face_instance_id FK, suggested_person_id FK, confidence, source_face_id FK, status, multi-prototype scoring fields (matching_prototype_ids, prototype_scores, aggregate_confidence) | face_instance, person |
| `PersonCentroid` | UUID `centroid_id` | person_id FK, qdrant_point_id, model_version, centroid_version int, centroid_type enum, cluster_label, n_faces, status, source_face_ids_hash, build_params JSONB | person |
| `FaceDetectionSession` | UUID `id` | status, training_session_id FK, batch_size, min_confidence, total_images, processed_images, faces_detected, asset_ids_json TEXT, current_asset_index | training_session |
| `FaceAssignmentEvent` | UUID `id` | face_instance_id FK, person_id FK, event_type, notes | face_instance, person |

**Key Enums:**
- `PersonStatus`: active, merged, hidden
- `CentroidType`: GLOBAL, CLUSTER
- `CentroidStatus`: ACTIVE, DEPRECATED, FAILED
- `PrototypeRole`: centroid, exemplar, primary, temporal, fallback
- `FaceSuggestionStatus`: PENDING, ACCEPTED, REJECTED, EXPIRED

**Notable Indexes:**
- Composite unique partial index on `PersonCentroid`: `(person_id, model_version, centroid_version, centroid_type, cluster_label)` WHERE `status = 'active'` -- prevents duplicate active centroids.
- Unique partial index on `FaceSuggestion`: `(face_instance_id, suggested_person_id)` WHERE `status = 'pending'` -- prevents duplicate pending suggestions.

### 2.2 Vector Storage (Qdrant)

Two Qdrant collections:

**`faces` collection** (managed by `FaceQdrantClient`):
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/vector/face_qdrant.py` (868 lines)
- **Dimension**: 512 (ArcFace/InsightFace)
- **Distance**: Cosine
- **Payload indexes**: person_id (keyword), cluster_id (keyword), is_prototype (bool), asset_id (keyword), face_instance_id (keyword)
- **Collection name**: Configurable via `QDRANT_FACE_COLLECTION` env var (default: "faces")

**`person_centroids` collection** (managed by `CentroidQdrantClient`):
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/vector/centroid_qdrant.py` (531 lines)
- **Dimension**: 512 (same ArcFace space)
- **Distance**: Cosine
- **Payload indexes**: person_id (keyword), centroid_id (keyword), model_version (keyword), centroid_version (integer), centroid_type (keyword)
- **Collection name**: Hardcoded as `CENTROID_COLLECTION_NAME = "person_centroids"` (not configurable)

### 2.3 Dual-Storage Consistency

The system uses a "Qdrant-first, DB-second" commit pattern:
1. Face embeddings are upserted to Qdrant first (`face_qdrant.upsert_face()`)
2. Then the DB record is committed
3. On Qdrant failure, the DB transaction is rolled back

**File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/service.py` -- `FaceProcessingService` follows this pattern in `_process_single_asset()`.

**File**: `/export/workspace/image-search/image-search-service/src/image_search_service/services/centroid_service.py` (lines 380-394) -- `compute_centroids_for_person()` adds centroid to DB, flushes, then upserts to Qdrant. On Qdrant failure, marks centroid as FAILED but does NOT roll back the DB flush.

### 2.4 Issues Found

**CRITICAL: Hardcoded Collection Name in Clusterer**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/clusterer.py`, line 88
- **Code**: `collection_name="faces"` -- hardcoded instead of using `_get_face_collection_name()`
- **Impact**: Tests that set `QDRANT_FACE_COLLECTION=test_faces` will accidentally query the production collection during clustering.

**MODERATE: Inconsistent Collection Name Configuration**
- `FaceQdrantClient`: Collection name from env var via `get_settings().qdrant_face_collection` (configurable)
- `CentroidQdrantClient`: Collection name hardcoded as constant `CENTROID_COLLECTION_NAME = "person_centroids"` (not configurable)
- **Impact**: Cannot use test collection for centroids; no safety guard like the face collection has.

**MODERATE: Missing Test Safety Guard on Centroid Collection**
- `FaceQdrantClient.reset_collection()` (line 798-851) has `PYTEST_CURRENT_TEST` safety guard that prevents deleting production collection during tests.
- `CentroidQdrantClient.reset_collection()` (line 477-514) has NO such safety guard.

**MODERATE: Qdrant/DB Consistency Gap in Centroid Creation**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/services/centroid_service.py`, lines 363-394
- After `db.flush()` (line 364), if Qdrant upsert fails (line 381), the centroid status is set to FAILED, but the DB record persists. The caller must commit to persist the FAILED status, but this is fragile.

**LOW: Deprecated Field Still in Model**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/db/models.py`
- `FaceDetectionSession.faces_assigned` is marked `(DEPRECATED: sum of faces_assigned_to_persons + clusters_created)` in a comment, but the column and field remain. Should be removed via migration.

**LOW: VectorDeletionLog Model**
- Present in models but unclear if actively used -- appears to be an audit mechanism for vector deletions.

### 2.5 Recommendations

1. **Fix hardcoded collection name** in `clusterer.py` line 88 -- replace `"faces"` with `_get_face_collection_name()`.
2. **Make centroid collection name configurable** via env var, matching the pattern used for faces.
3. **Add test safety guard** to `CentroidQdrantClient.reset_collection()`.
4. **Improve centroid creation atomicity** -- consider wrapping Qdrant upsert + DB commit in a try/except that fully rolls back on either failure.
5. **Remove deprecated `faces_assigned`** field via a new Alembic migration.

---

## 3. Background Processing & Queues

### 3.1 Queue Architecture

**File**: `/export/workspace/image-search/image-search-service/src/image_search_service/queue/face_jobs.py` (2,126 lines)

All background jobs use **Redis/RQ** with **synchronous database operations** due to macOS fork-safety requirements. The module header (lines 1-27) documents this constraint thoroughly.

**Job Inventory (15 jobs):**

| Job Function | Line | Purpose |
|-------------|------|---------|
| `detect_faces_job` | 48 | Batch face detection for assets |
| `cluster_faces_job` | 96 | HDBSCAN clustering of unlabeled faces |
| `assign_faces_job` | 147 | Assign faces to persons via prototypes |
| `compute_centroids_job` | 199 | Compute/update person centroids |
| `backfill_faces_job` | 226 | Backfill face detection for existing assets |
| `cluster_dual_job` | 286 | Dual-mode clustering (supervised + unsupervised) |
| `train_person_matching_job` | 338 | Train face matching with triplet loss |
| `recluster_after_training_job` | 395 | Re-cluster after training |
| `detect_faces_for_session_job` | 421 | Session-based face detection with resume |
| `propagate_person_label_job` | 964 | Propagate labels from person prototypes |
| `expire_old_suggestions_job` | 1129 | Expire stale suggestions |
| `cleanup_orphaned_suggestions_job` | 1192 | Clean up orphaned suggestions |
| `find_more_suggestions_job` | 1259 | Find more suggestions via random prototype sampling |
| `propagate_person_label_multiproto_job` | 1563 | Multi-prototype label propagation |
| `find_more_centroid_suggestions_job` | 1810 | Find more suggestions via centroid |

### 3.2 Session-Based Detection with Resume

**File**: `/export/workspace/image-search/image-search-service/src/image_search_service/queue/face_jobs.py`, `detect_faces_for_session_job()` (lines 421-680+)

The session job supports:
- **First run**: Queries assets, stores IDs as JSON in `asset_ids_json`, begins processing
- **Resume**: Loads stored IDs, continues from `current_asset_index`
- **Pause/Cancel**: Checks session status between batches via `db_session.refresh(session)`
- **Progress tracking**: Dual progress -- DB session fields + Redis cache for real-time SSE

### 3.3 Find-More Suggestion Jobs

Two parallel approaches exist:

**Prototype-Based** (`find_more_suggestions_job`, line 1259):
- Samples labeled faces using Quality + Diversity weighted selection
- For each sampled face, queries Qdrant for similar unassigned faces
- Aggregates results using MAX score
- Creates FaceSuggestion records
- **N+1 pattern**: Calls `qdrant.get_embedding_by_point_id()` per face in loop (line 1423)

**Centroid-Based** (`find_more_centroid_suggestions_job`, line 1810):
- Gets or computes centroid for person
- Searches faces collection with single centroid vector
- Creates FaceSuggestion records
- **N+1 pattern** when computing centroid on-demand (line 1929)
- **Duplicated logic**: Inline centroid computation (lines 1944-2003) instead of calling `centroid_service.compute_centroids_for_person()`

### 3.4 Progress Tracking

Both find-more jobs use Redis for progress tracking:
- Progress keys: `find_more:progress:{person_id}:{job_uuid}` or `find_more_centroid:progress:{person_id}:{job_uuid}`
- TTL: 1 hour (`ex=3600`)
- Phases: queued -> selecting/starting -> searching -> creating -> completed/failed
- **Issue**: Each progress update creates a new Redis connection (lines 1321-1337), rather than reusing one.

### 3.5 Issues Found

**CRITICAL: Duplicated Centroid Computation Logic**
- **File 1**: `/export/workspace/image-search/image-search-service/src/image_search_service/services/centroid_service.py`, `compute_centroids_for_person()` (line 258) -- Full-featured with outlier trimming, versioning, deprecation of old centroids
- **File 2**: `/export/workspace/image-search/image-search-service/src/image_search_service/queue/face_jobs.py`, `find_more_centroid_suggestions_job()` (lines 1944-2003) -- Simplified inline version without outlier trimming (`trim_outliers=False`), no old centroid deprecation, no stale check
- **Impact**: Bug fixes or algorithm changes in one location won't propagate to the other. The inline version computes a lower-quality centroid.

**CRITICAL: N+1 Qdrant Embedding Retrieval (7 call sites)**
- Every call to `get_embedding_by_point_id()` makes an individual Qdrant `retrieve()` request. This is used in loops across 7 locations:
  1. `centroid_service.py:206` -- `get_person_face_embeddings()`
  2. `face_jobs.py:1423` -- `find_more_suggestions_job`
  3. `face_jobs.py:1629` -- `propagate_person_label_multiproto_job`
  4. `face_jobs.py:1929` -- `find_more_centroid_suggestions_job`
  5. `faces.py:1902` -- face suggestion endpoint
  6. `face_clustering_service.py:90` -- clustering service
  7. `dual_clusterer.py` -- (via `_get_face_embedding()` method)
- **Impact**: For a person with 100 faces, centroid computation makes 100 separate Qdrant HTTP requests. The `retrieve()` API supports batch IDs but is never used with batches.

**MODERATE: Redis Connection Per Progress Update**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/queue/face_jobs.py`, `update_progress()` function in both find-more jobs (lines 1317-1337, 1854-1876)
- Creates a new `Redis.from_url()` connection for every progress update instead of reusing.

**MODERATE: face_jobs.py is 2,126 Lines**
- All 15 job functions are in a single file. Jobs range from simple (20 lines) to complex (300+ lines). Should be split by domain.

**LOW: `recluster_after_training_job` is a pass-through**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/queue/face_jobs.py`, line 395
- Simply calls `cluster_dual_job()` with no additional logic. Exists only as a named alias for queue scheduling clarity, but could be replaced by direct queue call.

### 3.6 Recommendations

1. **Extract centroid computation** from `find_more_centroid_suggestions_job` -- call existing `centroid_service` functions instead of duplicating logic.
2. **Add batch embedding retrieval** to `FaceQdrantClient` -- implement `get_embeddings_by_point_ids(point_ids: list[UUID]) -> dict[UUID, list[float]]` using Qdrant's batch retrieve API.
3. **Reuse Redis connections** in job progress updates -- pass the connection as parameter or use a module-level factory.
4. **Split `face_jobs.py`** into `detection_jobs.py`, `clustering_jobs.py`, `suggestion_jobs.py`, `training_jobs.py`.

---

## 4. Critical Bugs & Technical Debt

### 4.1 Performance Bottlenecks

**N+1 Qdrant Queries (CRITICAL)**

The most impactful performance issue is the one-by-one embedding retrieval pattern. The Qdrant Python client's `retrieve()` method accepts a list of IDs but is always called with a single ID.

Example from `centroid_service.py` lines 205-214:
```python
for face in faces:
    embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
    if embedding is not None:
        face_ids.append(face.id)
        embeddings.append(embedding)
```

**Fix**: Add a `retrieve_batch()` method to `FaceQdrantClient`:
```python
def get_embeddings_batch(self, point_ids: list[uuid.UUID]) -> dict[uuid.UUID, list[float]]:
    points = self.client.retrieve(
        collection_name=_get_face_collection_name(),
        ids=[str(pid) for pid in point_ids],
        with_vectors=True,
        with_payload=False,
    )
    return {uuid.UUID(str(p.id)): p.vector for p in points if p.vector}
```

**Full-Table Scan for Unlabeled Faces (MODERATE)**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/vector/face_qdrant.py`, `get_unlabeled_faces_with_embeddings()` (lines 573-646)
- Comment on line 589: "Qdrant doesn't have a native 'is null' filter, so we scroll all records and filter in Python"
- For large collections, this scrolls through ALL faces to find those without `person_id` -- O(N) full scan.

**HDBSCAN Memory (MODERATE)**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/dual_clusterer.py`
- HDBSCAN with `metric="precomputed"` requires computing full N*N distance matrix in memory. For 50,000 faces this is ~10GB of float32 data.
- The default `max_faces=50000` in `cluster_faces_job` (line 98) could trigger OOM.

### 4.2 Error Handling Gaps

**Silent Failures in Face Processing**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/service.py`
- `process_assets_batch()` accumulates errors in a list and returns them as `error_details` in the result dict -- errors don't propagate, they're silently collected.

**Qdrant Point Missing Treated as Warning**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/services/centroid_service.py`, lines 210-214
- When a face's Qdrant embedding is not found, a warning is logged but the face is silently skipped. This can produce centroids based on incomplete data with no indication to the caller.

**Payload Field Missing in Find-More Job**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/queue/face_jobs.py`, line 1438 and 2062
- Both find-more jobs extract `face_id` from Qdrant result payload: `result.payload.get("face_id")`. However, the `FaceQdrantClient.upsert_face()` stores the field as `"face_instance_id"` (line 196), NOT `"face_id"`. This mismatch means the find-more jobs will never find matching face IDs, silently producing zero suggestions.
- **Severity**: CRITICAL -- This is a likely functional bug.

### 4.3 Technical Debt

**Sync/Async Duality**
- API routes use `AsyncSession` via `get_db()` dependency
- Queue jobs use `SyncSession` via `get_sync_session()` due to fork-safety
- Result: Business logic is duplicated in two forms (async for routes, sync for jobs)
- Example: Centroid computation exists in async form (`centroid_service.py`) and sync form (inline in `face_jobs.py`)

**Missing `exclude_prototypes` Filter**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/api/routes/face_centroids.py`, line 381
- Comment: "Note: is_prototype field doesn't exist in FaceInstance model / Skipping exclude_prototypes filter for now"
- The `is_prototype` field exists in Qdrant payload but not in the FaceInstance SQLAlchemy model, creating a data model inconsistency.

**Dead Code Paths in Asset Path Resolution**
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/service.py`
- `_resolve_asset_path()` checks for `file_path` and `source_path` attributes that don't exist on the `ImageAsset` model.

**TODOs and Incomplete Implementations**
- `clusterer.py:80`: `# TODO: Add taken_at range filter based on time_bucket when needed` -- time-bounded clustering is not implemented.

### 4.4 Configuration Issues

**File**: `/export/workspace/image-search/image-search-service/src/image_search_service/core/config.py` (242 lines)

The configuration is well-structured using pydantic-settings, but several thresholds create hidden coupling:

| Setting | Default | Used By | Concern |
|---------|---------|---------|---------|
| `face_person_match_threshold` | 0.7 | `dual_clusterer.py` | Also hardcoded as default in `cluster_dual_job()` (line 287) |
| `face_suggestion_min_confidence` | 0.7 | suggestion routes | But `find_more_suggestions_job` reads from `SystemConfig` table instead (line 1314) |
| `centroid_min_faces` | 2 | `centroid_service.py` | But `find_more_centroid_suggestions_job` hardcodes 5 (line 1912) |
| `face_suggestion_max_results` | 5 (max 10) | suggestion endpoints | But `find_more` defaults to 100 (line 1263) |

**Multiple Configuration Sources**: Settings come from env vars (pydantic-settings), `SystemConfig` DB table (via `ConfigService`), and hardcoded defaults in job function signatures.

### 4.5 Migration History

32 migrations tracked in `/export/workspace/image-search/image-search-service/src/image_search_service/db/migrations/versions/`. Key migrations:
- `009_add_face_detection_tables.py` -- Initial face tables
- `012_post_train_suggestions.py` -- Post-training suggestion support
- `d1e2f3g4h5i6_add_person_centroid_table.py` -- Centroid table
- `56f6544da217_add_face_suggestions.py` -- Suggestion table
- `c1d2e3f4g5h6_add_pending_suggestion_unique_index.py` -- Race condition protection via unique partial index
- `f6a668d072bb_add_multi_prototype_scoring_fields_to_.py` -- Multi-prototype scoring
- `add_temporal_prototype_fields.py` -- Temporal prototype support
- `0d2febc7f1d5_add_resume_fields_to_face_detection_.py` -- Session resume

Some migrations use naming convention (numbered: `001_` through `012_`) while later ones use auto-generated hex names, suggesting the project transitioned from manual to auto-naming.

---

## 5. Clustering Algorithm Integration

### 5.1 Architecture Overview

The system implements three clustering approaches:

**A. Basic HDBSCAN Clustering** (`FaceClusterer`)
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/clusterer.py` (333 lines)
- Pure unsupervised HDBSCAN on unlabeled faces
- Scrolls Qdrant for unlabeled faces using `IsEmptyCondition` on `person_id`
- Updates both DB (`FaceInstance.cluster_id`) and Qdrant (`cluster_id` payload)
- Supports sub-cluster splitting via `recluster_within_cluster()` (line 217)
- Uses euclidean metric on normalized vectors (equivalent to cosine for unit vectors)

**B. Dual-Mode Clustering** (`DualModeClusterer`)
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/faces/dual_clusterer.py` (427 lines)
- **Phase 1 (Supervised)**: Computes centroids from each person's labeled faces, matches unlabeled faces via cosine similarity against `person_match_threshold` (configurable, default 0.7)
- **Phase 2 (Unsupervised)**: Clusters remaining unknowns using HDBSCAN (default), DBSCAN, or Agglomerative
- **Phase 3**: Saves results to DB and Qdrant
- Uses precomputed cosine distance matrix for HDBSCAN

**C. Centroid-Based Suggestion** (`CentroidService` + `CentroidQdrantClient`)
- **File**: `/export/workspace/image-search/image-search-service/src/image_search_service/services/centroid_service.py` (426 lines)
- Computes robust centroid with outlier trimming:
  - n < 50 faces: No trimming (simple mean)
  - 50-300 faces: Trim bottom 5% by similarity to initial mean
  - 300+ faces: Trim bottom 10%
- Staleness detection via model version + algorithm version + source face IDs hash
- Centroids stored in separate Qdrant collection for efficient querying

### 5.2 Clustering Pipeline Flow

```
Face Detection Session
    |
    v
[detect_faces_for_session_job]
    |
    +-- Per batch: FaceProcessingService.process_assets_batch()
    |       |
    |       +-- Load images (I/O thread via ThreadPoolExecutor)
    |       +-- Detect faces (GPU, main thread)
    |       +-- Store in Qdrant (batch upsert)
    |       +-- Store in PostgreSQL (FaceInstance records)
    |
    v
[cluster_dual_job] (optional, triggered manually)
    |
    +-- Phase 1: Supervised Matching
    |       |
    |       +-- Get all persons with labeled faces
    |       +-- Compute centroids per person (N+1 embedding retrieval)
    |       +-- Match unlabeled faces against centroids
    |       +-- Assign matches above threshold
    |
    +-- Phase 2: Unsupervised Clustering
    |       |
    |       +-- Get remaining unlabeled faces
    |       +-- Compute distance matrix (memory-intensive)
    |       +-- Run HDBSCAN/DBSCAN/Agglomerative
    |       +-- Assign cluster IDs
    |
    +-- Phase 3: Persist
            |
            +-- Update FaceInstance.cluster_id in DB
            +-- Update cluster_id payload in Qdrant
            +-- Generate cluster statistics
    |
    v
[find_more_suggestions] OR [find_more_centroid_suggestions]
    |
    +-- Per person: Search Qdrant for similar unassigned faces
    +-- Create FaceSuggestion records (status=PENDING)
    |
    v
User Review (accept/reject suggestions via API)
    |
    +-- Accept: face.person_id = suggested_person_id
    +-- Optional: auto_find_more triggers new find-more jobs
```

### 5.3 Algorithm Parameters

**Supervised Phase (Dual Mode):**
- `person_match_threshold`: 0.7 (from `FACE_PERSON_MATCH_THRESHOLD` env var)
- Centroid computation: Simple mean of labeled face embeddings (no outlier trimming in dual_clusterer, unlike centroid_service)

**Unsupervised Phase:**
- HDBSCAN: `min_cluster_size=3` (from `FACE_UNKNOWN_MIN_CLUSTER_SIZE`), `min_samples=3`
- DBSCAN: `eps=0.5` (from `FACE_UNKNOWN_EPS`), `min_samples=3`
- Agglomerative: `distance_threshold=0.5`
- All use cosine distance via precomputed matrix or euclidean on normalized vectors

**Centroid Service:**
- `centroid_model_version`: "arcface_r100_glint360k_v1"
- `centroid_algorithm_version`: 2
- `centroid_min_faces`: 2 (env configurable)
- Trim thresholds: 5% for 50-300 faces, 10% for 300+

### 5.4 Issues Found

**CRITICAL: Two Different Centroid Computation Approaches**
- `dual_clusterer.py` computes simple mean centroids per person for matching (no trimming, no versioning)
- `centroid_service.py` computes robust centroids with outlier trimming, versioning, and persistence
- These can produce different results for the same person, leading to inconsistent matching behavior depending on which code path is triggered.

**MODERATE: Distance Matrix Memory**
- `dual_clusterer.py` builds full N*N distance matrix for HDBSCAN. With `max_faces=50000`, this is 50000^2 * 4 bytes = ~10GB RAM.
- The basic `FaceClusterer` avoids this by not using precomputed distances (uses euclidean on raw embeddings).

**MODERATE: Cluster ID Format Inconsistency**
- `FaceClusterer` generates: `clu_{uuid_hex[:12]}` (line 146)
- `DualModeClusterer` generates: `unknown_clu_{uuid_hex[:12]}` for unsupervised clusters
- Sub-clusters get: `{original_cluster_id}_sub_{uuid_hex[:8]}`
- No single format standard for cluster IDs

**LOW: Time-Bucket Clustering Not Implemented**
- Both `FaceClusterer.cluster_unlabeled_faces()` (line 44) and `cluster_faces_job()` (line 101) accept `time_bucket` parameter
- The filter is commented out (line 80-82): `# TODO: Add taken_at range filter based on time_bucket when needed`

### 5.5 Recommendations

1. **Unify centroid computation** -- use `centroid_service.compute_global_centroid()` in `dual_clusterer.py` instead of computing simple means.
2. **Add batch embedding retrieval** -- critical path for both clustering and centroid computation.
3. **Add memory-bounded clustering** -- for large face counts, use incremental HDBSCAN or switch to approximate methods to avoid O(N^2) memory.
4. **Standardize cluster ID format** across all clustering paths.
5. **Implement time-bucket clustering** or remove the parameter if not needed.

---

## 6. Cross-Cutting Concerns

### 6.1 Singleton Pattern Usage

Both Qdrant clients use class-level singletons:
- `FaceQdrantClient._instance` (line 65)
- `CentroidQdrantClient._instance` (line 53)

These are NOT thread-safe (no locking on `get_instance()`). In RQ workers (single-threaded) this is fine, but in the async FastAPI server with multiple concurrent requests, two requests could race to create the singleton.

### 6.2 Logging

Structured logging is used consistently with `logger = logging.getLogger(__name__)`. Job functions include `job_id` context in all log messages. The `core/logging.py` module provides utilities, though most files use standard `logging` directly.

### 6.3 Thread Safety

Thread safety is properly managed for:
- PIL image loading: `_image_load_lock` in `training_jobs.py` (line 86)
- EXIF parsing: `_exif_parse_lock` in `exif_service.py` (line 28)
- Model loading: `_model_lock` in `embedding.py` (line 34) and `siglip_embedding.py` (line 36)

### 6.4 Security Considerations

- Destructive admin operations protected by `ALLOW_DESTRUCTIVE_ADMIN_OPS` env var check (admin.py line 69)
- Face collection reset protected by `ALLOW_PRODUCTION_RESET` env var (face_qdrant.py line 822)
- Test collection safety guard via `PYTEST_CURRENT_TEST` check (face_qdrant.py line 814)
- No authentication/authorization layer visible on API routes -- assumes upstream proxy handles auth

---

## Summary of Issues by Severity

### CRITICAL (5)
1. **Payload field mismatch** in find-more jobs: `"face_id"` vs `"face_instance_id"` -- silently produces zero suggestions
2. **N+1 Qdrant queries** across 7 call sites -- significant performance degradation
3. **Duplicated centroid computation** logic between service and job -- divergent behavior
4. **Missing optimistic locking** on face assignment -- concurrent modification risk
5. **Hardcoded collection name** `"faces"` in `clusterer.py` -- test safety bypass

### MODERATE (8)
1. Inconsistent centroid collection name configuration (not configurable)
2. Missing test safety guard on centroid collection reset
3. Qdrant/DB consistency gap in centroid creation failure path
4. Full-table scan for unlabeled faces in Qdrant
5. HDBSCAN O(N^2) memory for large face sets
6. Redis connection created per progress update
7. Two different centroid approaches (simple mean vs. trimmed) produce different results
8. Multiple configuration sources (env vars + SystemConfig DB + hardcoded defaults)

### LOW (5)
1. Deprecated `faces_assigned` field still in model
2. Cluster ID format inconsistency across clustering approaches
3. Time-bucket clustering accepted but not implemented
4. Dead code paths in `_resolve_asset_path()`
5. `recluster_after_training_job` is a trivial pass-through

### STRUCTURAL (3)
1. `faces.py` route file: 2,450 lines, 28+ endpoints -- needs decomposition
2. `face_jobs.py`: 2,126 lines, 15 jobs -- needs decomposition
3. Sync/async session duality forces business logic duplication

---

## Files Analyzed

| File | Lines | Purpose |
|------|-------|---------|
| `src/image_search_service/api/routes/faces.py` | 2,450 | Main face API routes (28+ endpoints) |
| `src/image_search_service/api/routes/face_suggestions.py` | 928 | Suggestion CRUD and find-more endpoints |
| `src/image_search_service/api/routes/face_centroids.py` | 484 | Centroid compute/get/suggest/delete |
| `src/image_search_service/db/models.py` | 971 | All SQLAlchemy models |
| `src/image_search_service/core/config.py` | 242 | Pydantic settings |
| `src/image_search_service/queue/face_jobs.py` | 2,126 | All 15 background job functions |
| `src/image_search_service/services/centroid_service.py` | 426 | Centroid computation and staleness |
| `src/image_search_service/faces/service.py` | 748 | Face detection pipeline |
| `src/image_search_service/faces/clusterer.py` | 333 | Basic HDBSCAN clustering |
| `src/image_search_service/faces/dual_clusterer.py` | 427 | Dual-mode clustering |
| `src/image_search_service/vector/face_qdrant.py` | 868 | Face Qdrant client |
| `src/image_search_service/vector/centroid_qdrant.py` | 531 | Centroid Qdrant client |
