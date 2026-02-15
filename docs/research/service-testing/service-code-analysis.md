# Service Code Analysis: Testable Surface Inventory

**Date**: 2026-02-06
**Scope**: `image-search-service/src/image_search_service/`
**Purpose**: Comprehensive mapping of all testable surfaces for test coverage analysis

---

## 1. Architecture Overview

### Tech Stack
- **Framework**: FastAPI (async) with uvicorn
- **Database**: PostgreSQL via SQLAlchemy 2.0 async (asyncpg driver)
- **Vector DB**: Qdrant (3 separate collection clients)
- **Queue**: Redis + RQ (background job processing)
- **ML Models**: OpenCLIP (image/text embeddings), SigLIP (alternative embeddings), InsightFace/RetinaFace (face detection), HDBSCAN (clustering)
- **Package Manager**: uv

### Application Entry Point
- `main.py`: `create_app()` factory with lifespan manager
  - Startup: preloads embedding model, starts file watcher
  - Shutdown: stops watcher, closes DB, closes Qdrant
  - CORS middleware (conditionally applied based on `enable_cors` setting)
  - Registers 2 top-level routers: `router` (health) and `api_v1_router` (all `/api/v1/` routes)

### Router Registration (17 routers)
All under `/api/v1/` prefix:
1. `admin_router` — `/api/v1/admin/`
2. `assets_router` — `/api/v1/assets/`
3. `categories_router` — `/api/v1/categories/`
4. `config_router` — `/api/v1/config/`
5. `images_router` — `/api/v1/images/`
6. `search_router` — `/api/v1/search/`
7. `training_router` — `/api/v1/training/`
8. `evidence_router` — `/api/v1/training/` (evidence sub-routes)
9. `system_router` — `/api/v1/system/`
10. `vectors_router` — `/api/v1/vectors/`
11. `faces_router` — `/api/v1/faces/`
12. `face_sessions_router` — `/api/v1/face-sessions/`
13. `face_suggestions_router` — `/api/v1/face-suggestions/`
14. `face_centroids_router` — `/api/v1/centroids/`
15. `queues_router` — `/api/v1/queues/`
16. `jobs_router` — `/api/v1/jobs/`
17. `workers_router` — `/api/v1/workers/`
Plus: `job_progress_router` (SSE streaming)

---

## 2. API Endpoints (Complete Inventory)

### 2.1 Health (`/health`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Basic health check (no external deps) |

### 2.2 Search (`/api/v1/search/`)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/search` | Text-based semantic search |
| POST | `/search/by-image/{asset_id}` | Image similarity search |
| GET | `/search/{asset_id}/similar` | Find similar images |
| POST | `/search/hybrid` | Hybrid text+image search (RRF) |
| POST | `/search/compose/{asset_id}` | Composed image retrieval (image + text modifier) |

### 2.3 Assets (`/api/v1/assets/`)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/assets/ingest` | Ingest images from filesystem |
| GET | `/assets` | List assets (paginated) |
| GET | `/assets/{asset_id}` | Get single asset |

### 2.4 Images (`/api/v1/images/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/images/{asset_id}/thumbnail` | Serve thumbnail |
| GET | `/images/{asset_id}/full` | Serve full image |
| POST | `/images/thumbnails/batch` | Batch thumbnail retrieval (data URIs) |
| POST | `/images/{asset_id}/thumbnail/generate` | Generate thumbnail on demand |

### 2.5 Training (`/api/v1/training/`)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/training/sessions` | Create training session |
| GET | `/training/sessions` | List sessions (paginated) |
| GET | `/training/sessions/{id}` | Get session detail |
| PATCH | `/training/sessions/{id}` | Update session |
| DELETE | `/training/sessions/{id}` | Delete session |
| POST | `/training/directories/scan` | Scan directory for images |
| GET | `/training/directories` | List directories |
| GET | `/training/directories/preview` | Preview directory contents |
| GET | `/training/directories/preview/thumbnail` | Preview thumbnail for directory |
| POST | `/training/directories/ignore` | Ignore a directory |
| DELETE | `/training/directories/ignore` | Unignore a directory |
| GET | `/training/directories/ignored` | List ignored directories |
| GET | `/training/sessions/{id}/subdirectories` | List subdirectories |
| PATCH | `/training/sessions/{id}/subdirectories` | Update subdirectory selections |
| GET | `/training/sessions/{id}/progress` | Get session progress |
| GET | `/training/sessions/{id}/jobs` | List session jobs |
| POST | `/training/sessions/{id}/start` | Start training |
| POST | `/training/sessions/{id}/pause` | Pause training |
| POST | `/training/sessions/{id}/cancel` | Cancel training |
| POST | `/training/sessions/{id}/restart` | Restart training |
| POST | `/training/sessions/{id}/restart/face-detection` | Restart face detection |
| POST | `/training/sessions/{id}/restart/face-clustering` | Restart face clustering |
| POST | `/training/sessions/{id}/restart/training` | Restart only training phase |
| GET | `/training/sessions/{id}/restart/status` | Get restart status |
| GET | `/training/jobs/{job_id}` | Get job detail |
| POST | `/training/sessions/{id}/thumbnails` | Generate thumbnails for session |
| POST | `/training/scan/incremental` | Incremental directory scan |

### 2.6 Evidence (`/api/v1/training/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/training/evidence` | List evidence (paginated, filterable) |
| GET | `/training/evidence/{id}` | Get evidence detail |
| GET | `/training/evidence/{id}/detail` | Get evidence with asset info |
| GET | `/training/assets/{asset_id}/training-history` | Get asset training history |
| GET | `/training/sessions/{id}/evidence/stats` | Get evidence stats |
| DELETE | `/training/evidence/{id}` | Delete evidence |

### 2.7 Categories (`/api/v1/categories/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/categories` | List categories (paginated) |
| POST | `/categories` | Create category (409 on duplicate) |
| GET | `/categories/{id}` | Get category with session count |
| PATCH | `/categories/{id}` | Partial update |
| DELETE | `/categories/{id}` | Delete (guards: default, has-sessions) |

### 2.8 Config (`/api/v1/config/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/config/face-matching` | Get face matching config |
| PUT | `/config/face-matching` | Update face matching config |
| GET | `/config/face-suggestions` | Get face suggestion settings |
| PUT | `/config/face-suggestions` | Update face suggestion settings |
| GET | `/config/face-clustering-unknown` | Get unknown face clustering config |
| PUT | `/config/face-clustering-unknown` | Update unknown face clustering config |
| GET | `/config/{category}` | Get config by category |
| PUT | `/config/{key}` | Update single config value |

### 2.9 Faces (`/api/v1/faces/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/faces/clusters` | List face clusters (with sorting/filtering) |
| GET | `/faces/clusters/{id}` | Get cluster detail |
| POST | `/faces/clusters/{id}/label` | Label cluster → person |
| POST | `/faces/clusters/{id}/split` | Split cluster (re-cluster) |
| GET | `/faces/people` | Unified people list (persons + clusters) |
| GET | `/faces/persons` | List persons |
| GET | `/faces/persons/{id}` | Get person detail |
| POST | `/faces/persons` | Create person |
| PATCH | `/faces/persons/{id}` | Update person |
| GET | `/faces/persons/{id}/photos` | Get person's photos (paginated) |
| POST | `/faces/persons/{id}/photos/assign` | Assign faces to person |
| GET | `/faces/persons/{id}/prototypes` | Get person prototypes |
| POST | `/faces/persons/{id}/merge` | Merge persons |
| POST | `/faces/persons/{id}/photos/bulk-remove` | Bulk remove photos from person |
| POST | `/faces/persons/{id}/photos/bulk-move` | Bulk move photos between persons |
| POST | `/faces/detect/{asset_id}` | Detect faces in image |
| POST | `/faces/cluster` | Run clustering |
| GET | `/faces/assets/{asset_id}` | Get faces for asset |
| POST | `/faces/faces/{face_id}/assign` | Assign face to person |
| DELETE | `/faces/faces/{face_id}/person` | Unassign face |
| GET | `/faces/faces/{face_id}/suggestions` | Get face suggestions |
| POST | `/faces/cluster/dual` | Dual-mode clustering |
| POST | `/faces/train` | Train face matching |
| POST | `/faces/persons/{id}/prototypes/recompute` | Recompute prototypes |
| DELETE | `/faces/persons/{id}/prototypes/{proto_id}` | Delete prototype |
| DELETE | `/faces/persons/{id}/prototypes/unpinned` | Delete unpinned prototypes |
| GET | `/faces/persons/{id}/prototypes/coverage` | Get temporal coverage |
| GET | `/faces/stats` | Get face stats |
| POST | `/faces/persons/{id}/prototypes/{proto_id}/pin` | Pin prototype |

### 2.10 Face Sessions (`/api/v1/face-sessions/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/face-sessions` | List sessions (with status filter) |
| GET | `/face-sessions/{id}` | Get session |
| POST | `/face-sessions` | Create session (enqueues RQ job) |
| POST | `/face-sessions/{id}/pause` | Pause session |
| POST | `/face-sessions/{id}/resume` | Resume session |
| DELETE | `/face-sessions/{id}` | Cancel session |
| GET | `/face-sessions/{id}/events` | SSE stream for progress |

### 2.11 Face Suggestions (`/api/v1/face-suggestions/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/face-suggestions` | List suggestions (grouped or flat mode) |
| GET | `/face-suggestions/stats` | Get suggestion stats |
| GET | `/face-suggestions/{id}` | Get suggestion detail |
| POST | `/face-suggestions/{id}/accept` | Accept suggestion |
| POST | `/face-suggestions/{id}/reject` | Reject suggestion |
| POST | `/face-suggestions/bulk-action` | Bulk accept/reject (with auto-find-more) |
| POST | `/face-suggestions/find-more/{person_id}` | Find more suggestions (prototype-based) |
| POST | `/face-suggestions/find-more-centroid/{person_id}` | Find more centroid suggestions |

### 2.12 Face Centroids (`/api/v1/centroids/`)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/centroids/persons/{id}/compute` | Compute centroids |
| GET | `/centroids/persons/{id}` | Get centroids (with staleness) |
| POST | `/centroids/persons/{id}/suggestions` | Get centroid suggestions |
| DELETE | `/centroids/persons/{id}` | Delete centroids |

### 2.13 Vectors (`/api/v1/vectors/`)
| Method | Path | Description |
|--------|------|-------------|
| DELETE | `/vectors/by-directory` | Delete by directory (confirm required) |
| POST | `/vectors/retrain` | Retrain directory |
| GET | `/vectors/directories/stats` | Get directory stats |
| DELETE | `/vectors/by-asset/{id}` | Delete by asset |
| DELETE | `/vectors/by-session/{id}` | Delete by session |
| DELETE | `/vectors/by-category/{id}` | Delete by category |
| POST | `/vectors/cleanup-orphans` | Cleanup orphans (confirm required) |
| POST | `/vectors/reset` | Reset collection (double confirm) |
| GET | `/vectors/deletion-logs` | Get deletion logs |

### 2.14 Admin (`/api/v1/admin/`)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/admin/data/delete-all` | Delete all data (destructive) |
| POST | `/admin/persons/export` | Export person metadata |
| POST | `/admin/persons/import` | Import person metadata |

### 2.15 System (`/api/v1/system/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/system/watcher/status` | Get file watcher status |
| POST | `/system/watcher/start` | Start file watcher |
| POST | `/system/watcher/stop` | Stop file watcher |

### 2.16 Queues (`/api/v1/queues/`, `/api/v1/jobs/`, `/api/v1/workers/`)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/queues` | Queues overview |
| GET | `/queues/{name}` | Queue detail |
| GET | `/jobs/{id}` | Job detail |
| GET | `/workers` | Workers list |

### 2.17 Job Progress (SSE)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/jobs/{id}/progress` | SSE stream for job events |
| GET | `/jobs/{id}/status` | Non-streaming status check |

**Total API Endpoints: ~105+**

---

## 3. Database Models (14 models + 10 enums)

### 3.1 Models

| Model | Table | PK Type | Key Relationships |
|-------|-------|---------|-------------------|
| `Category` | `categories` | int (auto) | → TrainingSession[] |
| `ImageAsset` | `image_assets` | int (auto) | → TrainingJob[], TrainingEvidence[], FaceInstance[] |
| `TrainingSession` | `training_sessions` | int (auto) | → Category, TrainingSubdirectory[], TrainingJob[], TrainingEvidence[] |
| `TrainingSubdirectory` | `training_subdirectories` | int (auto) | → TrainingSession |
| `TrainingJob` | `training_jobs` | int (auto) | → TrainingSession, ImageAsset |
| `TrainingEvidence` | `training_evidence` | int (auto) | → ImageAsset, TrainingSession |
| `VectorDeletionLog` | `vector_deletion_logs` | int (auto) | — |
| `Person` | `persons` | UUID | → FaceInstance[], PersonPrototype[] |
| `FaceInstance` | `face_instances` | UUID | → ImageAsset, Person |
| `PersonPrototype` | `person_prototypes` | UUID | → Person, FaceInstance |
| `FaceAssignmentEvent` | `face_assignment_events` | UUID | — (audit log) |
| `PersonCentroid` | `person_centroid` | UUID | → Person |
| `FaceDetectionSession` | `face_detection_sessions` | UUID | — |
| `FaceSuggestion` | `face_suggestions` | int (auto) | → FaceInstance (2 FKs), Person |
| `SystemConfig` | `system_configs` | int (auto) | — |
| `IgnoredDirectory` | `ignored_directories` | int (auto) | — |

### 3.2 Enums
- `TrainingStatus`: PENDING, QUEUED, TRAINING, TRAINED, FAILED
- `SessionStatus`: PENDING, RUNNING, PAUSED, COMPLETED, CANCELLED, FAILED
- `JobStatus`: PENDING, RUNNING, COMPLETED, FAILED, CANCELLED, SKIPPED
- `SubdirectoryStatus`: PENDING, TRAINING, TRAINED, FAILED
- `PersonStatus`: ACTIVE, MERGED, HIDDEN
- `PrototypeRole`: CENTROID, EXEMPLAR, PRIMARY, TEMPORAL, FALLBACK
- `AgeEraBucket`: INFANT, CHILD, TEEN, YOUNG_ADULT, ADULT, SENIOR
- `CentroidType`: GLOBAL, CLUSTER
- `CentroidStatus`: ACTIVE, DEPRECATED, BUILDING, FAILED
- `FaceDetectionSessionStatus`: PENDING, PROCESSING, COMPLETED, FAILED, PAUSED, CANCELLED
- `FaceSuggestionStatus`: PENDING, ACCEPTED, REJECTED, EXPIRED
- `ConfigDataType`: FLOAT, INT, STRING, BOOLEAN

### 3.3 Key Constraints & Indexes
- `ImageAsset.path`: UNIQUE
- `FaceInstance`: UNIQUE on (asset_id, bbox_x, bbox_y, bbox_w, bbox_h) — idempotency
- `Person.name`: UNIQUE (case-insensitive via `func.lower()`)
- `PersonCentroid`: UNIQUE composite on (person_id, model_version, centroid_version, centroid_type, cluster_label)
- `SystemConfig.key`: UNIQUE
- `IgnoredDirectory.path`: UNIQUE
- 30+ indexes across all tables

---

## 4. Service Layer (24 service modules)

### 4.1 Core Services

| Service | Key Methods | Integration Points |
|---------|-------------|-------------------|
| `EmbeddingService` | `embed_text()`, `embed_image()`, `embed_batch()` | OpenCLIP model, GPU/CPU device |
| `SigLIPEmbeddingService` | `embed_text()`, `embed_image()`, `embed_batch()` | SigLIP model, GPU/CPU device |
| `embedding_router` | `get_search_embedding_service()` | Routes between CLIP/SigLIP based on config |
| `ConfigService` | `get_value()`, `set_value()`, `get_typed()` | PostgreSQL (async) |
| `SyncConfigService` | Same as above but synchronous | PostgreSQL (sync, for workers) |
| `PersonService` | CRUD + merge, person listing | PostgreSQL |
| `TrainingService` | Session lifecycle, job management | PostgreSQL, Redis/RQ |
| `ExifService` | `extract_exif()`, `parse_gps()` | PIL/Pillow |

### 4.2 Face Services

| Service | Key Methods | Integration Points |
|---------|-------------|-------------------|
| `FaceClusterer` | `cluster_unlabeled_faces()`, `recluster_within_cluster()` | Qdrant, HDBSCAN, PostgreSQL |
| `DualModeClusterer` | Combines known+unknown clustering | FaceAssigner, FaceClusterer |
| `FaceAssigner` | `assign_faces_to_known_persons()` | Qdrant (face+centroid), PostgreSQL |
| `FaceProcessingService` | Batched face detection with threading | InsightFace, Qdrant, PostgreSQL |
| `FaceTrainer` | `TripletFaceDataset`, training pipeline | Qdrant, PostgreSQL |
| `FaceClusteringService` | `calculate_cluster_confidence()`, representative selection | Qdrant, numpy |
| `CentroidService` | `compute_centroids_for_person()`, staleness detection | Qdrant (centroid), PostgreSQL |

### 4.3 Utility Services

| Service | Key Methods | Integration Points |
|---------|-------------|-------------------|
| `fusion` | `reciprocal_rank_fusion()`, `weighted_rrf()` | Pure computation (no external deps) |
| `perceptual_hash` | `compute_perceptual_hash()`, `compute_hash_hamming_distance()` | PIL/Pillow |
| `temporal_service` | `classify_age_era()`, `compute_temporal_quality_score()` | Pure computation |
| `PrototypeService` | Legacy + temporal strategies, pin/unpin | Qdrant, PostgreSQL |
| `AdminService` | `delete_all_data()`, export/import metadata | PostgreSQL, Qdrant, Redis |
| `ThumbnailService` | `generate_thumbnail()` | PIL/Pillow, filesystem |
| `DirectoryService` | `scan_directory()` | Filesystem |
| `AssetDiscovery` | `discover_assets()` | Filesystem |
| `WatcherManager` | Singleton file watcher | Filesystem (watchdog) |
| `PeriodicScanner` | Periodic fallback scanning | Filesystem, Redis |

### 4.4 Restart Services (Abstract Base Pattern)

| Service | Base | Description |
|---------|------|-------------|
| `RestartServiceBase` | ABC | Abstract base with `_execute_restart()` |
| `FaceDetectionRestartService` | RestartServiceBase | Restart face detection pipeline |
| `FaceClusteringRestartService` | RestartServiceBase | Restart clustering pipeline |
| `TrainingRestartService` | RestartServiceBase | Restart training pipeline |

---

## 5. Background Jobs (6 job modules, 25+ job functions)

### 5.1 Face Jobs (`queue/face_jobs.py`) — Largest module

| Job Function | Triggers | Dependencies |
|--------------|----------|-------------|
| `detect_faces_job` | Manual API call | InsightFace, Qdrant, PostgreSQL |
| `detect_faces_for_session_job` | Face session create/resume | InsightFace, Qdrant, PostgreSQL, Redis (progress) |
| `cluster_faces_job` | Manual API call | HDBSCAN, Qdrant, PostgreSQL |
| `assign_faces_job` | Manual API call | Qdrant (face+centroid), PostgreSQL |
| `compute_centroids_job` | Manual or post-training | Qdrant (centroid), PostgreSQL |
| `dual_cluster_job` | Manual API call | DualModeClusterer |
| `train_face_matching_job` | Manual API call | FaceTrainer |
| `propagate_person_label_job` | After face assignment | Qdrant, PostgreSQL |
| `expire_old_suggestions_job` | Scheduled/manual | PostgreSQL |
| `find_more_suggestions_job` | After bulk accept | Qdrant, PostgreSQL |
| `find_more_centroid_suggestions_job` | Manual API call | CentroidQdrant, PostgreSQL |
| `generate_post_training_suggestions_job` | Post-training | Qdrant, PostgreSQL, ConfigService |
| `generate_centroid_post_training_suggestions_job` | Post-training (centroid) | CentroidQdrant, PostgreSQL |

### 5.2 Training Jobs (`queue/training_jobs.py`)

| Job Function | Triggers | Dependencies |
|--------------|----------|-------------|
| `train_session` | Session start | EmbeddingService, Qdrant, PostgreSQL, Redis |
| `train_batch` | Batch processing | EmbeddingService, Qdrant, PostgreSQL |
| `train_single_asset` | Per-image processing | EmbeddingService, Qdrant, PostgreSQL, ExifService |
| `_build_evidence_metadata` | Internal | EmbeddingService metadata |

### 5.3 Other Jobs

| Job Function | Module | Dependencies |
|--------------|--------|-------------|
| `index_asset` | `jobs.py` | EmbeddingService, Qdrant, PostgreSQL |
| `update_asset_person_ids_job` | `jobs.py` | Qdrant, PostgreSQL |
| `process_image` | `jobs.py` | EmbeddingService, Qdrant, PostgreSQL |
| `process_new_image` | `auto_detection_jobs.py` | EmbeddingService, face detection |
| `scan_directory_incremental` | `auto_detection_jobs.py` | Filesystem, PostgreSQL |
| `backfill_perceptual_hashes` | `hash_backfill_jobs.py` | PIL, PostgreSQL |
| `generate_thumbnails_batch` | `thumbnail_jobs.py` | PIL, filesystem |
| `generate_single_thumbnail` | `thumbnail_jobs.py` | PIL, filesystem |

### 5.4 Worker Infrastructure

| Component | Description |
|-----------|-------------|
| `worker.py` | Worker entry point with CLI |
| `listener_worker.py` | Listener-based worker for GPU-safe execution |
| `progress.py` | ProgressTracker (Redis-backed) |
| `macos_fork_safety.py` | macOS-specific fork safety utilities |

---

## 6. Vector/Qdrant Layer (3 clients)

### 6.1 Main Qdrant Client (`vector/qdrant.py`)

| Function | Description |
|----------|-------------|
| `get_qdrant_client()` | Singleton factory |
| `ensure_collection()` | Create collection if missing |
| `upsert_vector()` | Upsert with payload |
| `search_vectors()` | Similarity search with filters |
| `delete_vectors_by_directory()` | Delete by path prefix |
| `delete_vectors_by_asset_id()` | Delete by asset |
| `delete_vectors_by_session_id()` | Delete by session |
| `get_directory_stats()` | Count vectors per directory |
| `cleanup_orphaned_vectors()` | Remove orphans |
| `reset_collection()` | Full collection reset |
| `update_vector_payload()` | Update payload fields |
| `close_qdrant()` | Cleanup |

### 6.2 Face Qdrant Client (`vector/face_qdrant.py`)
- Singleton `FaceQdrantClient` class
- Collection: `face_embeddings` (512-dim, cosine distance)
- Operations: upsert, search, scroll, update cluster IDs, update person IDs, delete

### 6.3 Centroid Qdrant Client (`vector/centroid_qdrant.py`)
- Singleton `CentroidQdrantClient` class
- Collection: `person_centroids` (512-dim, cosine distance)
- Operations: upsert, search by similarity, get by person, delete by person

---

## 7. Pydantic Schemas (7 schema files)

### 7.1 Core Schemas (`api/schemas.py`)
- `Asset`, `LocationMetadata`, `CameraMetadata` — with model validators
- `IngestRequest/Response`, `SearchRequest/Response`, `SearchResult`
- `PaginatedResponse[T]` — generic paginated wrapper
- `BatchThumbnailRequest/Response`
- `ImageSearchRequest`, `SimilarSearchRequest`
- `HybridSearchRequest/Result/Response`
- `ComposeSearchRequest/Response`
- `ErrorResponse`

### 7.2 Other Schema Files
- `face_schemas.py` — Face, person, cluster, prototype schemas
- `training_schemas.py` — Session, job, progress, directory schemas
- `category_schemas.py` — Category CRUD schemas
- `centroid_schemas.py` — Centroid compute/get/delete schemas
- `face_session_schemas.py` — Face detection session schemas
- `vector_schemas.py` — Vector deletion/stats schemas
- `queue_schemas.py` — Queue monitoring schemas

---

## 8. Critical Data Flows

### 8.1 Image Ingestion Flow
```
POST /assets/ingest → asset_discovery → DB insert → RQ enqueue
  → train_single_asset job:
    → ExifService.extract_exif() → DB update
    → EmbeddingService.embed_image() → Qdrant upsert
    → ThumbnailService.generate_thumbnail()
    → TrainingEvidence creation
```

### 8.2 Search Flow
```
POST /search → EmbeddingService.embed_text()
  → Qdrant search_vectors() (with optional filters)
  → PostgreSQL enrichment (asset metadata)
  → SearchResponse
```

### 8.3 Face Detection Flow
```
POST /face-sessions → Create DB session → RQ enqueue
  → detect_faces_for_session_job:
    → FaceProcessingService (batched, threaded)
      → InsightFace detect → FaceInstance DB insert → Qdrant upsert
    → Progress tracking (Redis)
    → Post-detection: clustering → suggestions
```

### 8.4 Face Clustering Flow
```
POST /faces/cluster/dual → DualModeClusterer:
  1. FaceAssigner.assign_faces_to_known_persons()
     → Search prototypes/centroids in Qdrant
     → Assign above threshold → DB update + Qdrant sync
  2. FaceClusterer.cluster_unlabeled_faces()
     → Scroll Qdrant for unlabeled → HDBSCAN → DB cluster_id update
  → Post-clustering: suggestions generation
```

### 8.5 Suggestion Lifecycle
```
Generate: find_more_suggestions_job → search prototypes → create FaceSuggestion
Accept: POST /face-suggestions/{id}/accept
  → DB: face.person_id = suggestion.person_id
  → Qdrant: update person_id payload
  → Optionally: auto-find-more
Reject: POST /face-suggestions/{id}/reject → status = REJECTED
Bulk: POST /face-suggestions/bulk-action → batch accept/reject + auto-find-more
```

### 8.6 Centroid Computation Flow
```
POST /centroids/persons/{id}/compute
  → centroid_service.compute_centroids_for_person()
    → Get all face embeddings from Qdrant
    → Compute mean embedding (numpy)
    → Upsert to centroid Qdrant collection
    → Create PersonCentroid DB record
    → Staleness detection via source_face_ids_hash
```

---

## 9. Integration Points (External Dependencies)

| System | Usage | Mock Strategy |
|--------|-------|---------------|
| **PostgreSQL** | All CRUD, relationships, transactions | SQLite in-memory via SQLAlchemy |
| **Qdrant** (3 collections) | Vector storage, similarity search, payload updates | Mock QdrantClient |
| **Redis** | RQ job queue, progress tracking, caching | Mock Redis/RQ |
| **OpenCLIP** | Text/image embedding (768-dim CLIP) | Mock EmbeddingService |
| **SigLIP** | Alternative embedding (768-dim SigLIP) | Mock SigLIPEmbeddingService |
| **InsightFace** | Face detection, face embedding (512-dim) | Mock detect_faces |
| **HDBSCAN** | Face clustering algorithm | Mock or test with synthetic data |
| **PIL/Pillow** | Thumbnail generation, EXIF extraction, perceptual hashing | Mock or use test images |
| **Filesystem** | Image file serving, directory scanning | Temp directories in tests |
| **watchdog** | File system watcher for auto-detection | Mock WatcherManager |

---

## 10. Error Handling Patterns

### 10.1 HTTP Error Responses
- **404**: Asset/session/person/suggestion not found
- **409**: Duplicate category name, duplicate constraint violations
- **400**: Invalid parameters, bad state transitions
- **422**: Pydantic validation failures
- **500**: Unexpected errors (caught at route level)
- **503**: Service unavailable (Qdrant/Redis down)

### 10.2 State Machine Transitions
- **TrainingSession**: PENDING → RUNNING → COMPLETED/FAILED; RUNNING → PAUSED → RUNNING
- **FaceDetectionSession**: PENDING → PROCESSING → COMPLETED/FAILED; PROCESSING → PAUSED → PROCESSING
- **FaceSuggestion**: PENDING → ACCEPTED/REJECTED/EXPIRED

### 10.3 Graceful Degradation Patterns
- Health endpoint works without any external service
- Embedding model preload failure is non-fatal (logs warning)
- Qdrant sync failures during suggestion accept are non-fatal (continue processing)
- Missing thumbnails return placeholder response

---

## 11. Existing Test Coverage Summary

### 11.1 Test Files (65 files)
```
tests/
├── api/                  (24 test files)
│   ├── test_health.py
│   ├── test_search.py
│   ├── test_ingest.py
│   ├── test_images.py
│   ├── test_categories.py
│   ├── test_training_routes.py
│   ├── test_training_deduplication.py
│   ├── test_training_unified_progress.py
│   ├── test_faces_routes.py
│   ├── test_face_suggestions.py
│   ├── test_face_suggestions_list.py
│   ├── test_face_suggestion_cleanup.py
│   ├── test_face_session_suggestions.py
│   ├── test_suggestion_regeneration.py
│   ├── test_clusters_filtering.py
│   ├── test_get_faces_for_asset.py
│   ├── test_get_faces_orphaned_person.py
│   ├── test_person_name_regression.py
│   ├── test_prototype_endpoints.py
│   ├── test_recompute_with_rescan.py
│   ├── test_unified_people_endpoint.py
│   ├── test_queues.py
│   └── test_vectors.py
├── unit/                 (22 test files)
│   ├── api/
│   │   ├── test_schemas.py
│   │   ├── test_face_schemas.py
│   │   ├── test_config_post_training_suggestions.py
│   │   └── test_config_unknown_clustering.py
│   ├── queue/
│   │   ├── test_centroid_post_training.py
│   │   ├── test_multiproto_propagation.py
│   │   └── test_post_training_suggestions_job.py
│   ├── services/
│   │   ├── test_exif_service.py
│   │   ├── test_face_clustering_service.py
│   │   └── test_person_service.py
│   ├── test_admin_export_import.py
│   ├── test_bootstrap_qdrant.py
│   ├── test_centroid_qdrant.py
│   ├── test_config_keys_sync.py
│   ├── test_embedding_router.py
│   ├── test_embedding_service.py
│   ├── test_face_jobs_payload.py
│   ├── test_faces_cli.py
│   ├── test_fusion.py
│   ├── test_perceptual_hash.py
│   ├── test_person_birth_date.py
│   ├── test_prototype_selection.py
│   ├── test_qdrant_safety.py
│   ├── test_qdrant_wrapper.py
│   ├── test_siglip_embedding_service.py
│   ├── test_suggestion_qdrant_sync.py
│   └── test_temporal_service.py
├── faces/                (4 test files)
│   ├── test_assigner.py
│   ├── test_clusterer.py
│   ├── test_detector.py
│   └── test_service.py
├── integration/          (3 test files)
│   ├── test_listener_worker.py
│   ├── test_restart_workflows.py
│   └── test_temporal_migration.py
└── core/
    └── test_device.py
```

---

## 12. Identified Test Gaps (Untested Surfaces)

### 12.1 API Routes Without Dedicated Tests
- `/api/v1/config/*` — Config CRUD endpoints (only schema/unit tests, no route tests)
- `/api/v1/evidence/*` — Evidence endpoints (no route tests)
- `/api/v1/face-centroids/*` — Centroid endpoints (no route tests)
- `/api/v1/face-sessions/*` — Session lifecycle endpoints (no route tests)
- `/api/v1/system/*` — Watcher endpoints (no route tests)
- `/api/v1/admin/*` — Admin destructive endpoints (no route tests, only unit tests for export/import logic)
- `/api/v1/jobs/*` — Job progress SSE (no tests)
- `/api/v1/training/directories/preview*` — Directory preview (no tests)
- `/api/v1/training/directories/ignore*` — Directory ignore (no tests)
- `/api/v1/training/scan/incremental` — Incremental scan (no tests)
- Several face route endpoints in `faces.py` (bulk-move, bulk-remove, merge, prototypes)

### 12.2 Services Without Tests
- `admin_service.py` — `delete_all_data()` function (destructive operation)
- `centroid_service.py` — All centroid computation logic
- `directory_service.py` — Directory scanning
- `asset_discovery.py` — Asset discovery
- `file_watcher.py` — File system watcher
- `periodic_scanner.py` — Periodic scanning
- `thumbnail_service.py` — Thumbnail generation
- `training_service.py` — TrainingService class methods
- `watcher_manager.py` — WatcherManager singleton
- All restart services (`face_detection_restart_service.py`, etc.)

### 12.3 Background Jobs Without Tests
- `detect_faces_job`, `detect_faces_for_session_job`
- `cluster_faces_job`, `assign_faces_job`
- `dual_cluster_job`
- `train_face_matching_job`
- `train_session`, `train_batch`, `train_single_asset`
- `index_asset`, `process_image`
- `process_new_image`, `scan_directory_incremental`
- `generate_thumbnails_batch`, `generate_single_thumbnail`
- `backfill_perceptual_hashes`
- `expire_old_suggestions_job`

### 12.4 Database Model Coverage Gaps
- State machine transitions (session status, job status)
- Cascade delete behavior
- Unique constraint violation handling
- Index effectiveness

### 12.5 Integration Gaps
- End-to-end flows (ingest → search, detect → cluster → suggest → accept)
- SSE streaming endpoints
- Worker/queue integration
- Multi-collection Qdrant operations (main + face + centroid consistency)

---

## 13. Testability Assessment by Priority

### HIGH PRIORITY (Critical Paths)

| Surface | Complexity | External Deps | Risk |
|---------|-----------|---------------|------|
| Search flow (text → embed → Qdrant → enrich) | Medium | Qdrant, CLIP | Core feature |
| Face detection session lifecycle | High | InsightFace, Qdrant, Redis | Complex state machine |
| Suggestion accept/reject with Qdrant sync | Medium | Qdrant, PostgreSQL | Data consistency |
| Centroid computation & staleness | Medium | Qdrant, numpy | Correctness |
| Training session lifecycle | High | RQ, PostgreSQL, CLIP | Complex state machine |
| Bulk operations (move/remove/accept) | High | Qdrant, PostgreSQL | Atomicity |

### MEDIUM PRIORITY (Important Features)

| Surface | Complexity | External Deps | Risk |
|---------|-----------|---------------|------|
| HDBSCAN clustering | Medium | HDBSCAN, Qdrant | Algorithm correctness |
| Prototype management (temporal) | Medium | Qdrant, PostgreSQL | Strategy correctness |
| Hybrid/compose search | Medium | 2x embedding models, Qdrant | Fusion logic |
| Config service (DB-backed) | Low | PostgreSQL | State management |
| Vector deletion operations | Medium | Qdrant, PostgreSQL | Destructive ops safety |

### LOW PRIORITY (Supporting Features)

| Surface | Complexity | External Deps | Risk |
|---------|-----------|---------------|------|
| Directory scanning | Low | Filesystem | Edge cases |
| Thumbnail generation | Low | PIL | Image format handling |
| Queue monitoring | Low | Redis | Read-only |
| File watcher | Low | watchdog | OS-specific |
| Incremental scanning | Medium | Filesystem, PostgreSQL | Idempotency |

---

## 14. Key Architectural Observations

1. **Lazy initialization pattern**: No connections at import time; all external clients created on first use. This is excellent for testability.

2. **Dependency injection**: FastAPI `Depends()` used extensively, making mock injection straightforward.

3. **Sync/async split**: Workers use synchronous DB sessions (`SyncSession`), API uses async (`AsyncSession`). Tests must handle both patterns.

4. **3 Qdrant collections**: Main (image embeddings), face (face embeddings), centroid (person centroids). Each has its own client singleton. Tests need to mock all three independently.

5. **Complex state machines**: Training sessions and face detection sessions have multi-state lifecycles with pause/resume/restart. These need thorough transition testing.

6. **Post-action triggers**: Many operations trigger follow-up jobs (e.g., after bulk accept → auto-find-more; after training → generate suggestions). These chain effects need integration testing.

7. **Perceptual hash deduplication**: Used in centroid suggestions to avoid showing duplicate images. Important for quality.

8. **Temporal prototype strategy**: Age-era based prototype selection with coverage gap analysis. Complex logic requiring thorough unit testing.

9. **CORS middleware logic appears inverted**: In `main.py`, CORS is added when `enable_cors` is False (line 77), which seems like a potential bug or confusing naming.

10. **GPU/CPU device handling**: `core/device.py` handles Apple Silicon, CUDA, and CPU fallback. Workers need macOS fork-safety for Metal.
