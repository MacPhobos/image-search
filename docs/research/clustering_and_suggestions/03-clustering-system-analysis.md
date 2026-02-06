# Clustering System End-to-End Analysis

**Researcher**: Clustering Systems Researcher
**Date**: 2026-02-06
**Scope**: End-to-end investigation of face clustering, centroid computation, training pipelines, suggestion generation, and Qdrant vector database integration

---

## Executive Summary

The image-search system implements a sophisticated face recognition pipeline spanning detection, clustering, labeling, centroid computation, and suggestion generation. The architecture uses InsightFace (`buffalo_l`) for 512-dimensional ArcFace embeddings, HDBSCAN for unsupervised clustering, a dual-mode clusterer for combined supervised/unsupervised operation, and Qdrant as the vector database with two dedicated collections (`faces` and `person_centroids`).

The system operates in three distinct modes:
1. **Batch clustering** -- Post-training HDBSCAN clustering of unlabeled faces into identity groups
2. **Incremental assignment** -- Real-time matching of new faces to known persons via prototype/centroid similarity
3. **Suggestion generation** -- Multi-strategy approach (single-prototype, multi-prototype, centroid-based, and "find more" random sampling) for surfacing likely matches for human review

Key strengths include the robust centroid system with outlier trimming and staleness detection, the two-tier threshold system separating auto-assignment from suggestion creation, and the comprehensive post-training pipeline that automatically queues suggestion jobs. Key concerns include the O(n^2) pairwise confidence calculation for large clusters, potential embedding retrieval bottlenecks from per-face Qdrant lookups in the dual clusterer, and the dual-collection architecture requiring careful synchronization between `faces` and `person_centroids`.

---

## 1. Clustering Algorithm Analysis

### 1.1 Primary Algorithm: HDBSCAN

The system uses HDBSCAN (Hierarchical Density-Based Spatial Clustering of Applications with Noise) as its primary clustering algorithm.

**Implementation**: `image-search-service/src/image_search_service/faces/clusterer.py`, class `FaceClusterer`, lines 14-332

**Configuration parameters** (from `config.py`, lines 101-107):

| Parameter | Config Key | Default | Description |
|---|---|---|---|
| `min_cluster_size` | `FACE_UNKNOWN_MIN_CLUSTER_SIZE` | 3 | Minimum faces to form a cluster |
| `min_samples` | Factory default | 3 | Minimum samples for HDBSCAN core point |
| `cluster_selection_epsilon` | Constructor default | 0.0 | Epsilon for cluster selection (0 = default EOM behavior) |
| `metric` | Constructor default | `"euclidean"` | Distance metric for `FaceClusterer` |
| `unknown_method` | `FACE_UNKNOWN_CLUSTERING_METHOD` | `"hdbscan"` | Clustering algorithm selection |
| `unknown_eps` | `FACE_UNKNOWN_EPS` | 0.5 | Distance threshold for DBSCAN/Agglomerative |

**Important metric observation**: The `FaceClusterer` (single-mode) uses `metric="euclidean"` directly on the embedding vectors (line 23, `clusterer.py`). The code comments at line 200-201 note: "For normalized face embeddings, euclidean distance works well (it's proportional to cosine distance for unit vectors)." Since InsightFace produces normalized embeddings, this is mathematically equivalent to cosine distance. HDBSCAN also uses `core_dist_n_jobs=-1` for multi-core parallelism (line 207).

In contrast, the `DualModeClusterer` (`dual_clusterer.py`, line 368-386) uses a **precomputed cosine distance matrix** via `sklearn.metrics.pairwise.cosine_distances` with `metric="precomputed"` and `cluster_selection_method="eom"` (Excess of Mass). This is a more explicit and arguably safer approach to cosine-based clustering.

### 1.2 Dual-Mode Clusterer

**Implementation**: `image-search-service/src/image_search_service/faces/dual_clusterer.py`, class `DualModeClusterer`, lines 18-427

The dual-mode clusterer executes a two-phase pipeline:

**Phase 1: Supervised Assignment** (lines 128-209)
1. Collects all labeled faces (faces with `person_id` set)
2. Groups embeddings by `person_id` and computes per-person centroids via `np.mean()` with re-normalization
3. For each unlabeled face, computes cosine similarity against all person centroids
4. Assigns face to best-matching person if similarity >= `person_match_threshold` (default 0.7, from `config.py` line 98: `FACE_PERSON_MATCH_THRESHOLD`)
5. Faces below threshold remain in the "still unknown" pool

**Phase 2: Unsupervised Clustering** (lines 211-270)
1. Takes remaining unknown faces from Phase 1
2. Supports three clustering algorithms (configurable via `unknown_method`):
   - **HDBSCAN** (default): Precomputed cosine distance matrix, EOM selection method (`_cluster_hdbscan`, lines 368-386)
   - **DBSCAN**: Direct cosine metric, `eps` parameter controls maximum distance (`_cluster_dbscan`, lines 388-397)
   - **Agglomerative**: Average linkage, cosine metric, `distance_threshold` for flat clustering (`_cluster_agglomerative`, lines 399-409)
3. Noise points (label -1) are assigned individual cluster IDs: `unknown_noise_{face_id}`
4. Clustered faces get IDs: `unknown_cluster_{label}`

**Phase 3: Result Persistence** (`save_dual_mode_results`, lines 272-342)
- Batch-updates `FaceInstance.person_id` in PostgreSQL for supervised assignments
- Batch-updates `FaceInstance.cluster_id` for unsupervised clusters
- Updates Qdrant payloads via `update_person_ids()` and `update_cluster_ids()`
- Single transaction commit at end

### 1.3 Cluster ID Generation

The `FaceClusterer` generates cluster IDs with the format `clu_{uuid4_hex[:12]}` (line 146, `clusterer.py`). Sub-clusters from re-clustering use `{parent_cluster_id}_sub_{uuid4_hex[:8]}` (line 288). The `DualModeClusterer` uses `unknown_cluster_{label_int}` and `unknown_noise_{face_id}` formats.

### 1.4 Input Feature Space

- **Embedding model**: InsightFace `buffalo_l` (ArcFace ResNet-100 trained on Glint360K)
- **Embedding dimension**: 512 (constant `FACE_VECTOR_DIM = 512` in `face_qdrant.py` line 34)
- **Normalization**: Embeddings are pre-normalized by InsightFace (detector.py line 122-123: `face.embedding` is "already normalized by InsightFace")
- **Quality filtering**: Faces with `quality_score < quality_threshold` (default 0.5) are excluded from clustering in `FaceClusterer.cluster_unlabeled_faces()` (line 42-43)
- **Max faces**: Configurable limit (default 50,000) to prevent memory issues (line 44)

### 1.5 Re-clustering (Cluster Splitting)

`FaceClusterer.recluster_within_cluster()` (lines 217-310) enables splitting of existing clusters:
1. Retrieves all faces in a specific cluster from Qdrant via `scroll_faces()`
2. Requires at least `min_cluster_size * 2` faces (safety check, line 269)
3. Runs HDBSCAN with a tighter `min_cluster_size` parameter
4. Assigns sub-cluster IDs and updates both PostgreSQL and Qdrant
5. Returns summary with original size, number of sub-clusters, and new IDs

---

## 2. Centroid System

### 2.1 Centroid Computation Algorithm

**Implementation**: `image-search-service/src/image_search_service/services/centroid_service.py`, function `compute_global_centroid()`, lines 36-109

The centroid computation uses a robust mean with outlier trimming:

**Algorithm steps**:
1. Compute initial mean of all embeddings (`np.mean(embeddings, axis=0)`)
2. Normalize initial mean to unit vector
3. Compute cosine similarities between each embedding and the initial mean
4. Apply size-dependent outlier trimming:
   - **n < 50 faces**: No trimming (insufficient data for statistical outlier detection)
   - **50 <= n <= 300 faces**: Trim bottom `trim_threshold_small` (default 5%, `config.py` line 222-226)
   - **n > 300 faces**: Trim bottom `trim_threshold_large` (default 10%, `config.py` line 229-235)
5. Sort embeddings by similarity in descending order, keep top `(1 - trim_pct)` faces
6. Recompute mean from trimmed set
7. Normalize final centroid to unit vector
8. Cast to float32

**Configuration** (from `config.py`, lines 199-235):

| Parameter | Default | Description |
|---|---|---|
| `centroid_model_version` | `"arcface_r100_glint360k_v1"` | Model version tag for compatibility checking |
| `centroid_algorithm_version` | 2 | Algorithm version for staleness detection |
| `centroid_min_faces` | 2 | Minimum faces required to compute centroid |
| `centroid_clustering_min_faces` | 200 (min=50) | Minimum faces for cluster-based centroids (future feature) |
| `centroid_trim_threshold_small` | 0.05 (max=0.2) | Outlier trim for 50-300 faces |
| `centroid_trim_threshold_large` | 0.10 (max=0.3) | Outlier trim for 300+ faces |

### 2.2 Centroid Storage Architecture

**Database model**: `PersonCentroid` in `db/models.py`, lines 856-953

Key fields:
- `centroid_id` (UUID primary key, also used as Qdrant point ID for 1:1 mapping)
- `person_id` (FK to `persons`)
- `qdrant_point_id` (unique, same as centroid_id)
- `model_version` (string, e.g., `"arcface_r100_glint360k_v1"`)
- `centroid_version` (integer, e.g., 2)
- `centroid_type` (enum: GLOBAL or CLUSTER)
- `cluster_label` (string, default `"global"`)
- `n_faces` (count of source faces)
- `status` (enum: ACTIVE, DEPRECATED, FAILED)
- `source_face_ids_hash` (SHA-256 hash of sorted face IDs for staleness detection)
- `build_params` (JSONB, stores trim parameters and face count)

**Unique constraint** (lines 938-946): Partial unique index on `(person_id, model_version, centroid_version, centroid_type, cluster_label)` ensuring only one active centroid per person per configuration.

**Qdrant collection**: `person_centroids` (separate from `faces`)
- **Implementation**: `vector/centroid_qdrant.py`, class `CentroidQdrantClient`
- **Dimension**: 512 (constant `CENTROID_VECTOR_DIM`, line 33)
- **Distance**: Cosine (`Distance.COSINE`, line 103)
- **Payload indexes**: `person_id` (KEYWORD), `centroid_id` (KEYWORD), `model_version` (KEYWORD), `centroid_version` (INTEGER), `centroid_type` (KEYWORD)

### 2.3 Staleness Detection

**Implementation**: `centroid_service.py`, function `is_centroid_stale()`, lines 127-174

Three staleness triggers (checked in order):
1. **Model version mismatch**: `centroid.model_version != current_model_version`
2. **Algorithm version mismatch**: `centroid.centroid_version != current_algorithm_version`
3. **Source hash mismatch**: `centroid.source_face_ids_hash != compute_source_hash(current_face_ids)`

The source hash is computed as `hashlib.sha256(":".join(sorted(face_id_strings))).hexdigest()[:16]` (lines 112-124). This means any change in face assignments (adding, removing, or reassigning a face to/from a person) will trigger centroid recomputation.

### 2.4 Centroid Lifecycle

**Main entry point**: `compute_centroids_for_person()` (lines 258-396)

1. Retrieve face embeddings from Qdrant via `get_person_face_embeddings()`
2. Check minimum face count (`centroid_min_faces`, default 2)
3. If not force-rebuilding, check if existing active centroid is still fresh via `is_centroid_stale()`
4. **Deprecate old centroids BEFORE creating new one** (critical for unique constraint, line 322-325)
5. Compute robust centroid with outlier trimming
6. Create `PersonCentroid` DB record
7. Upsert centroid vector to `person_centroids` Qdrant collection
8. On Qdrant failure, mark centroid as FAILED in DB

### 2.5 Centroid-Based Face Search

**Implementation**: `centroid_qdrant.py`, method `search_faces_with_centroid()`, lines 275-332

This method searches the **`faces` collection** (not the `person_centroids` collection) using a centroid vector as the query:
- Uses `query_points()` API against `faces` collection
- Default `score_threshold` of 0.7
- Can exclude faces already assigned to the centroid's person via `must_not` filter on `person_id`
- Returns `ScoredPoint` objects with payload metadata

This cross-collection search is the basis for centroid-based suggestion generation.

---

## 3. Training Sessions and Workflow

### 3.1 Full Training Lifecycle

**Implementation**: `services/training_service.py`, class `TrainingService`, 1264 lines

**Session State Machine**:
```
PENDING --> RUNNING --> COMPLETED
  |           |
  |           +--> FAILED
  |           |
  |           +--> PAUSED --> RUNNING (resume)
  |           |
  |           +--> CANCELLED
  |
  +--> RUNNING (start)
```

**States defined** in `db/models.py`, lines 48-56: `PENDING`, `RUNNING`, `PAUSED`, `COMPLETED`, `CANCELLED`, `FAILED`

### 3.2 Training Pipeline (3 Phases)

The training pipeline is orchestrated by `detect_faces_for_session_job()` in `queue/face_jobs.py` (lines 421-961):

**Phase 1: Training / Face Detection (weight: 30% + 65% = 95%)**
1. Create/resume `FaceDetectionSession` linked to `TrainingSession`
2. Query assets to process (either from linked training session via `TrainingEvidence`, or all unprocessed)
3. Store asset IDs as JSON for resume support
4. Process in batches using `FaceProcessingService.process_assets_batch()`:
   - Producer-consumer pattern: `ThreadPoolExecutor` pre-loads images from disk (I/O)
   - Main thread runs InsightFace inference (GPU)
   - Each detected face: create `FaceInstance` in DB + upsert embedding to Qdrant
   - Quality score computed via `DetectedFace.compute_quality_score()` (area * 0.5 + confidence * 0.5)
5. Update progress in Redis for real-time tracking
6. Support pause/cancel via session status polling per batch

**Phase 2: Auto-Assignment (within detection job)**
After face detection completes:
1. `FaceAssigner.assign_new_faces()` runs with config-based thresholds
2. Two-tier threshold system (from `SyncConfigService`):
   - **Auto-assign threshold** (`face_auto_assign_threshold`): High confidence --> immediate person assignment
   - **Suggestion threshold** (`face_suggestion_threshold`): Medium confidence --> create `FaceSuggestion` for review
3. Searches against all `PersonPrototype` records via `search_against_prototypes()` (score_threshold from suggestion_threshold)

**Phase 3: Clustering (weight: 5%)**
After assignment:
1. `FaceClusterer.cluster_unlabeled_faces()` with:
   - `quality_threshold=0.3` (lower than normal to include more faces)
   - `max_faces=10000`
   - `min_cluster_size=3`
2. Groups remaining unlabeled faces into identity clusters

**Post-Training Suggestion Generation** (lines 786-912):
After clustering, the system queues suggestion jobs for all persons:
1. Reads configuration: `post_training_suggestions_mode` (all/top_n), `post_training_use_centroids`, `centroid_min_faces_for_suggestions`, `post_training_suggestions_top_n_count`
2. For each person with labeled faces:
   - If centroids enabled AND `face_count >= min_faces_for_centroid`: queue `find_more_centroid_suggestions_job` (min_similarity=0.70, max_results=50)
   - Else: queue `propagate_person_label_multiproto_job` (min_confidence=0.7, max_suggestions=50)
3. Jobs are queued to RQ default queue with 10-minute timeout

### 3.3 Job Deduplication

**Implementation**: `training_service.py`, method `create_training_jobs()` (referenced from summary)

Uses perceptual hash (`dHash` algorithm, 16-character hex string stored in `ImageAsset.perceptual_hash`) to detect duplicate images:
- Groups assets by perceptual hash
- Creates `PENDING` job for the representative (oldest) image
- Creates `SKIPPED` job for duplicates
- Prevents redundant GPU inference on visually identical images

### 3.4 Progress Tracking

Three-phase weighted progress (from `get_session_progress_unified()`):
- **Training phase**: 30% weight (job completion ratio)
- **Face detection phase**: 65% weight (processed/total images)
- **Clustering phase**: 5% weight (binary: done or not)

Real-time progress uses Redis cache with key pattern `face_detection:{session_id}:progress` (1 hour TTL).

---

## 4. Cluster Quality and Edge Cases

### 4.1 Cluster Confidence Calculation

**Implementation**: `services/face_clustering_service.py`, class `FaceClusteringService`, lines 16-203

`calculate_cluster_confidence()` (lines 29-122):
1. Retrieve Qdrant point IDs for all faces in cluster (or use pre-fetched IDs)
2. **Single face = 1.0 confidence** (line 73)
3. **Performance optimization**: Sample max 20 faces for large clusters (`max_faces_for_calculation`, line 33)
4. Retrieve embeddings from Qdrant (per-face retrieval via `get_embedding_by_point_id`)
5. If fewer than 2 valid embeddings, return 0.0
6. Compute **all pairwise cosine similarities** (O(n^2) computation, lines 108-113)
7. Return mean similarity as confidence score

**Cosine similarity implementation** (`_cosine_similarity()`, lines 124-145):
- Manual normalization: `norm1 = np.linalg.norm(vec1)`, `norm2 = np.linalg.norm(vec2)`
- Zero-vector guard: returns 0.0 if either norm is 0
- Clamping to `[-1.0, 1.0]` via `np.clip` (protects against floating-point overflow)

### 4.2 Display Thresholds for Unknown Clusters

**Configuration** (`config.py`, lines 109-126):
- `UNKNOWN_FACE_CLUSTER_MIN_CONFIDENCE`: 0.70 (minimum intra-cluster confidence for display)
- `UNKNOWN_FACE_CLUSTER_MIN_SIZE`: 2 (minimum faces per cluster for display)

These settings filter which unknown clusters are shown to users -- clusters below these thresholds are hidden from the UI but remain in the database.

### 4.3 Representative Face Selection

`select_representative_face()` (lines 147-202) uses a composite quality score:
```
quality = (quality_score or 0.5)
        + min(bbox_area / 10000, 0.2)      -- up to 0.2 bonus for larger faces
        + (detection_confidence or 0.0) * 0.1  -- up to 0.1 bonus for high confidence
```
The face with the highest composite score becomes the representative for the cluster.

### 4.4 Edge Cases and Handling

| Edge Case | Handling | Location |
|---|---|---|
| Single face in cluster | Returns confidence 1.0 | `face_clustering_service.py:73` |
| No embeddings found | Returns confidence 0.0 with warning log | `face_clustering_service.py:99-105` |
| Large cluster (>20 faces) | Random sampling of 20 faces | `face_clustering_service.py:77-84` |
| Zero-norm vectors | Cosine similarity returns 0.0 | `face_clustering_service.py:138-139` |
| Fewer faces than min_cluster_size | Returns `clusters_found=0`, all faces as noise | `clusterer.py:115-124` |
| Re-cluster too-small cluster | Returns `{"status": "too_small"}` without modification | `clusterer.py:269-270` |
| Missing Qdrant embeddings | Skips face with warning, continues processing | `clusterer.py:100-103` |
| HDBSCAN not installed | Raises ImportError with install instructions | `clusterer.py:196-198` |
| Duplicate face suggestions | Checked via `(face_instance_id, suggested_person_id, status=pending)` query | `assigner.py:184-190` |

### 4.5 Noise Handling

HDBSCAN labels noise points as -1. These faces are:
- In `FaceClusterer`: Left without a cluster_id (not assigned to any cluster)
- In `DualModeClusterer`: Assigned individual pseudo-cluster IDs (`unknown_noise_{face_id}`) to maintain tracking

---

## 5. Suggestion Generation

### 5.1 Suggestion Data Model

**Implementation**: `db/models.py`, class `FaceSuggestion`, lines 734-802

Key fields:
- `face_instance_id` (FK, the candidate face)
- `suggested_person_id` (FK, the suggested person)
- `confidence` (cosine similarity score from single best prototype)
- `source_face_id` (FK, the prototype/face that generated this suggestion)
- `status` (pending/accepted/rejected/expired)
- `aggregate_confidence` (MAX score across all matching prototypes)
- `matching_prototype_ids` (JSONB list of all prototypes that matched)
- `prototype_scores` (JSONB map of prototype_id -> score)
- `prototype_match_count` (number of prototypes that matched above threshold)

### 5.2 Suggestion Generation Strategies

The system supports four distinct suggestion generation strategies:

#### Strategy 1: Single-Prototype Propagation
**Job**: `propagate_person_label_job()` (`face_jobs.py`, lines 964-1127)

Triggered when a user labels a face. Uses the single newly-labeled face as a query vector:
1. Get source face embedding from Qdrant
2. Query `faces` collection for similar faces (`query_points` with score_threshold from config)
3. Filter: skip self, skip already-assigned faces, skip existing pending suggestions
4. Create `FaceSuggestion` records for qualifying matches
5. Maximum suggestions from `face_suggestion_max_results` config

#### Strategy 2: Multi-Prototype Propagation
**Job**: `propagate_person_label_multiproto_job()` (`face_jobs.py`, lines 1563-1807)

Uses ALL prototypes for a person to generate higher-quality suggestions:
1. Get all `PersonPrototype` records for the person
2. For each prototype, retrieve embedding and search Qdrant for similar unassigned faces
3. **Aggregate results**: For each candidate face, collect scores from all matching prototypes
4. Use MAX score across prototypes as `aggregate_confidence`
5. Sort candidates by aggregate confidence, take top `max_suggestions` (default 50)
6. Select `source_face_id` as the highest-quality matching prototype
7. Store multi-prototype metadata (matching_prototype_ids, prototype_scores, prototype_match_count)
8. Supports `preserve_existing` mode (default True) to avoid expiring existing pending suggestions

#### Strategy 3: Centroid-Based Suggestions
**Job**: `find_more_centroid_suggestions_job()` (`face_jobs.py`, lines 1810-2127)

The fastest strategy, using pre-computed person centroids:
1. Look up active `PersonCentroid` (GLOBAL type) from database
2. If no centroid exists, compute one on-demand:
   - Retrieve all face embeddings for person from Qdrant
   - Requires minimum 5 faces
   - Simple mean + normalization (no outlier trimming in on-demand computation, line 1979: `"trim_outliers": False`)
   - Store in both PostgreSQL and `person_centroids` Qdrant collection
3. Retrieve centroid vector from `person_centroids` Qdrant collection
4. Search `faces` collection using centroid via `search_faces_with_centroid()`
5. Default parameters: `min_similarity=0.65`, `max_results=200`
6. Filter: skip already-assigned, skip existing pending suggestions
7. Create `FaceSuggestion` records with `source_face_id = centroid.centroid_id`

**Architectural note**: The centroid job's on-demand computation at line 1951-2008 differs from the formal `compute_centroids_for_person()` in `centroid_service.py` -- it uses simpler mean without outlier trimming and hardcodes `trim_outliers=False` in build_params. This could cause inconsistency when a formal centroid is later computed with trimming.

#### Strategy 4: Find More (Random Sampling)
**Job**: `find_more_suggestions_job()` (`face_jobs.py`, lines 1259-1560)

Discovery-oriented strategy using quality-weighted random sampling:
1. Get all labeled faces for person (excluding existing prototypes)
2. Requires minimum 10 labeled faces
3. Score each face: `quality * 0.7 + diversity_bonus * 0.3`
   - `diversity_bonus` penalizes repeated use of same asset: `0.3 - min(0.3, usage_count * 0.1)`
4. Weighted random selection of top `prototype_count * 2` candidates
5. For each selected face, search Qdrant for similar unassigned faces
6. Aggregate results using MAX score across all selected prototype faces
7. Deduplicate against existing pending suggestions
8. Sort by max_score, create suggestions for top `max_suggestions`
9. Real-time progress tracking via Redis (`progress_key`)

### 5.3 Two-Tier Threshold System

**Implementation**: `faces/assigner.py`, `assign_new_faces()`, lines 34-256

The `FaceAssigner` reads thresholds from `SyncConfigService`:
- `face_auto_assign_threshold`: Score >= this threshold causes **automatic assignment** (face.person_id is set immediately)
- `face_suggestion_threshold`: Score >= this threshold (but < auto_assign) creates a **FaceSuggestion** for human review
- Scores below `face_suggestion_threshold` result in the face being left unassigned

This separation prevents low-confidence auto-assignments while still surfacing potential matches to users.

### 5.4 Suggestion Lifecycle

```
PENDING --> ACCEPTED (face assigned to person)
  |
  +--> REJECTED (no action taken)
  |
  +--> EXPIRED (auto-expired by cleanup job or prototype invalidation)
```

**Expiration**: `expire_old_suggestions_job()` (lines 1129-1189) expires pending suggestions older than `face_suggestion_expiry_days` config.

**Orphan cleanup**: `cleanup_orphaned_suggestions_job()` (lines 1192-1256) expires suggestions where the source face's person assignment changed (source face unassigned or moved to different person).

### 5.5 Bulk Operations

**API endpoint**: `POST /api/v1/faces/suggestions/bulk-action` (`face_suggestions.py`, lines 651-751)

Supports bulk accept/reject with optional auto-find-more:
- Accepts `suggestion_ids`, `action` (accept/reject), `auto_find_more` flag
- On accept: sets `face.person_id`, updates suggestion status
- If `auto_find_more=True` on accept: queues `find_more_suggestions_job` for each affected person
- Tracks affected `person_ids` for find-more targeting
- Progress tracking via Redis for queued find-more jobs

### 5.6 Suggestion Pagination

Two modes supported (`face_suggestions.py`, lines 105-311):
1. **Grouped mode** (default, `grouped=true`): Groups suggestions by person, ordered by max confidence, with configurable `groups_per_page` and `suggestions_per_group` from SystemConfig
2. **Flat mode** (`grouped=false`): Simple pagination ordered by confidence descending

---

## 6. Vector Database (Qdrant) Integration

### 6.1 Collection Architecture

The system uses **two Qdrant collections**:

#### Collection 1: `faces`
**Client**: `vector/face_qdrant.py`, class `FaceQdrantClient`, 868 lines

| Property | Value |
|---|---|
| Name | `faces` (configurable via `QDRANT_FACE_COLLECTION`, `config.py` line 29) |
| Vector Dimension | 512 (`FACE_VECTOR_DIM`, line 34) |
| Distance Metric | Cosine (`Distance.COSINE`, line 116) |
| Singleton | Yes (class-level `_instance`, `_client`) |

**Payload schema** (from `upsert_face()` method):
- `asset_id` (int)
- `face_instance_id` (UUID string)
- `detection_confidence` (float)
- `quality_score` (float)
- `bbox_x`, `bbox_y`, `bbox_w`, `bbox_h` (int)
- `person_id` (UUID string or None)
- `cluster_id` (string or None)
- `is_prototype` (bool)
- `taken_at` (ISO datetime string or None)

**Payload indexes** (created in `_ensure_collection()`, lines 124-162):
1. `person_id` -- KEYWORD (for filtering faces by person)
2. `cluster_id` -- KEYWORD (for filtering faces by cluster)
3. `is_prototype` -- BOOL (for prototype-only searches)
4. `asset_id` -- KEYWORD (for per-asset face queries)
5. `face_instance_id` -- KEYWORD (for point-to-face lookups)

#### Collection 2: `person_centroids`
**Client**: `vector/centroid_qdrant.py`, class `CentroidQdrantClient`, 531 lines

| Property | Value |
|---|---|
| Name | `person_centroids` (constant, line 34) |
| Vector Dimension | 512 (`CENTROID_VECTOR_DIM`, line 33) |
| Distance Metric | Cosine (`Distance.COSINE`, line 103) |
| Singleton | Yes (class-level `_instance`, `_client`) |

**Payload indexes** (lines 111-144):
1. `person_id` -- KEYWORD
2. `centroid_id` -- KEYWORD
3. `model_version` -- KEYWORD
4. `centroid_version` -- INTEGER
5. `centroid_type` -- KEYWORD

### 6.2 Key Search Operations

#### `search_similar_faces()` (`face_qdrant.py`)
- Searches `faces` collection for similar face embeddings
- Default `score_threshold=0.5`
- Default `limit=10`
- Returns `ScoredPoint` objects

#### `search_against_prototypes()` (`face_qdrant.py`)
- Searches `faces` collection with filter `is_prototype=true`
- Default `score_threshold=0.6` (higher than general search)
- Used by `FaceAssigner` for incremental face assignment

#### `search_faces_with_centroid()` (`centroid_qdrant.py`)
- **Cross-collection search**: Uses centroid vector to query `faces` collection
- Default `score_threshold=0.7`
- Can exclude faces assigned to the centroid's person
- Default `limit=100`

#### `get_unlabeled_faces_with_embeddings()` (`face_qdrant.py`)
- Scrolls through `faces` collection
- Filters for faces without `person_id` and with quality >= `quality_threshold` (default 0.5)
- **Important limitation**: Qdrant lacks a native "is null" filter, so this method scrolls ALL records and filters in Python

### 6.3 Batch Operations

- `upsert_faces_batch()`: Batch upsert of multiple face embeddings
- `update_cluster_ids()`: Batch update of cluster_id payloads
- `update_person_ids()`: Batch update of person_id payloads
- Uses Qdrant's `set_payload()` for efficient payload-only updates

### 6.4 Safety Mechanisms

- **Collection deletion guard** (`face_qdrant.py`): Production collection cannot be deleted during tests (checks collection name pattern)
- **Lazy initialization**: Both clients use lazy loading for the Qdrant connection
- **Error handling**: All Qdrant operations are wrapped in try/except with structured logging
- **Centroid deprecation before creation**: Avoids unique constraint violations by deprecating old centroids before creating new ones

### 6.5 Consistency Concerns

The dual-collection architecture introduces several consistency concerns:

1. **Face-Centroid sync**: When face assignments change, centroids may become stale. The `source_face_ids_hash` mechanism detects this but requires an explicit check/recompute.

2. **Prototype-to-centroid migration**: The `FaceAssigner.compute_person_centroids()` method (lines 286-390 in `assigner.py`) stores centroids in the **`faces` collection** (not `person_centroids`), using `is_prototype=True` and `is_centroid=True` payloads. This is a separate mechanism from the formal `CentroidQdrantClient` and `PersonCentroid` model. This dual storage path could lead to confusion.

3. **On-demand vs formal centroid computation**: The `find_more_centroid_suggestions_job()` can create centroids on-demand with `trim_outliers=False`, while the formal `compute_centroids_for_person()` uses outlier trimming. These can produce different centroid vectors for the same person.

---

## 7. Configuration and Thresholds Reference

### 7.1 Complete Configuration Map

**File**: `image-search-service/src/image_search_service/core/config.py`

| Config Key | Default | Type | Lines | Purpose |
|---|---|---|---|---|
| `FACE_MODEL_NAME` | `buffalo_l` | str | 87 | InsightFace model name |
| `FACE_PERSON_MATCH_THRESHOLD` | 0.7 | float | 98 | Supervised assignment threshold |
| `FACE_UNKNOWN_CLUSTERING_METHOD` | `hdbscan` | str | 101-103 | Clustering algorithm |
| `FACE_UNKNOWN_MIN_CLUSTER_SIZE` | 3 | int | 104-106 | Min faces per cluster |
| `FACE_UNKNOWN_EPS` | 0.5 | float | 107 | DBSCAN/Agglomerative distance |
| `UNKNOWN_FACE_CLUSTER_MIN_CONFIDENCE` | 0.70 | float | 110-119 | Display threshold for unknown clusters |
| `UNKNOWN_FACE_CLUSTER_MIN_SIZE` | 2 | int | 120-126 | Min size for display |
| `FACE_SUGGESTION_MIN_CONFIDENCE` | 0.7 | float | 129-135 | Suggestion confidence floor |
| `FACE_SUGGESTION_MAX_RESULTS` | 5 | int | 136-141 | Max suggestions per query |
| `FACE_SUGGESTIONS_AUTO_RESCAN_ON_RECOMPUTE` | False | bool | 143-147 | Auto-regenerate on prototype change |
| `FACE_PROTOTYPE_MIN_QUALITY` | 0.5 | float | 150-155 | Min quality for prototypes |
| `FACE_PROTOTYPE_MAX_EXEMPLARS` | 5 | int | 156-163 | Max exemplar prototypes/person |
| `FACE_PROTOTYPE_TEMPORAL_MODE` | True | bool | 166-169 | Age-era based selection |
| `CENTROID_MODEL_VERSION` | `arcface_r100_glint360k_v1` | str | 200-204 | Centroid model tag |
| `CENTROID_ALGORITHM_VERSION` | 2 | int | 205-209 | Centroid algorithm version |
| `CENTROID_MIN_FACES` | 2 | int | 210-215 | Min faces for centroid |
| `CENTROID_CLUSTERING_MIN_FACES` | 200 | int | 216-221 | Min faces for cluster centroids |
| `CENTROID_TRIM_THRESHOLD_SMALL` | 0.05 | float | 222-228 | Trim for 50-300 faces |
| `CENTROID_TRIM_THRESHOLD_LARGE` | 0.10 | float | 229-235 | Trim for 300+ faces |

### 7.2 Database-Backed Dynamic Configuration

Additional thresholds are stored in the `system_configs` table (`SystemConfig` model, lines 814-854) and accessed via `ConfigService`/`SyncConfigService`:

| Key | Purpose | Used By |
|---|---|---|
| `face_auto_assign_threshold` | Auto-assignment threshold | `FaceAssigner.assign_new_faces()` |
| `face_suggestion_threshold` | Suggestion creation threshold | `FaceAssigner`, propagation jobs |
| `face_suggestion_expiry_days` | Days before auto-expiry | `expire_old_suggestions_job()` |
| `face_suggestion_groups_per_page` | Pagination config | `face_suggestions.py` |
| `face_suggestion_items_per_group` | Pagination config | `face_suggestions.py` |
| `face_suggestion_max_results` | Max suggestions | propagation jobs |
| `post_training_suggestions_mode` | all / top_n | `detect_faces_for_session_job()` |
| `post_training_suggestions_top_n_count` | Top N persons | `detect_faces_for_session_job()` |
| `post_training_use_centroids` | Enable centroid suggestions | `detect_faces_for_session_job()` |
| `centroid_min_faces_for_suggestions` | Min faces for centroid path | `detect_faces_for_session_job()` |

---

## 8. Data Flow Diagrams

### 8.1 Training Pipeline Data Flow

```
User triggers training session
         |
         v
[TrainingService.create_session()]
         |
         v
[TrainingService.enqueue_training()]
  - Discover assets from root directory
  - Create TrainingJob records (with perceptual hash dedup)
  - Enqueue RQ job
         |
         v
[detect_faces_for_session_job()] -- RQ Worker
         |
         +--- Phase 1: Face Detection
         |    |
         |    +--- For each batch of assets:
         |    |    - ThreadPoolExecutor: Load images from disk (I/O parallel)
         |    |    - Main thread: InsightFace inference (GPU)
         |    |    - Create FaceInstance records in PostgreSQL
         |    |    - Upsert 512-dim embeddings to Qdrant "faces" collection
         |    |    - Update Redis progress cache
         |    |
         +--- Phase 2: Auto-Assignment
         |    |
         |    +--- FaceAssigner.assign_new_faces()
         |    |    - Get unassigned faces created during session
         |    |    - Search against PersonPrototype embeddings in Qdrant
         |    |    - Score >= auto_assign_threshold: Set face.person_id
         |    |    - Score >= suggestion_threshold: Create FaceSuggestion
         |    |    - Update Qdrant person_id payloads
         |    |
         +--- Phase 3: Clustering
         |    |
         |    +--- FaceClusterer.cluster_unlabeled_faces()
         |    |    - Scroll unlabeled faces from Qdrant
         |    |    - Run HDBSCAN (euclidean metric, min_cluster_size=3)
         |    |    - Assign cluster IDs (clu_{uuid})
         |    |    - Update PostgreSQL + Qdrant cluster_id
         |    |
         +--- Phase 4: Post-Training Suggestions
              |
              +--- For each person with labeled faces:
                   - If centroids enabled + enough faces:
                     Queue find_more_centroid_suggestions_job()
                   - Else:
                     Queue propagate_person_label_multiproto_job()
```

### 8.2 Suggestion Generation Data Flow

```
[Trigger: User labels face / Post-training / Manual "Find More"]
         |
         v
[Select Strategy] -- based on trigger and configuration
  |     |     |     |
  |     |     |     +--- Strategy 1: Single Prototype
  |     |     |          - Source face embedding -> query Qdrant faces
  |     |     |          - Filter: unassigned, no existing suggestion
  |     |     |          - Create FaceSuggestion (single score)
  |     |     |
  |     |     +--- Strategy 2: Multi-Prototype
  |     |          - ALL person prototypes -> each queries Qdrant faces
  |     |          - Aggregate: MAX score across prototypes
  |     |          - Store multi-prototype metadata in FaceSuggestion
  |     |
  |     +--- Strategy 3: Centroid-Based
  |          - Get/compute person centroid
  |          - Single centroid -> query Qdrant faces (cross-collection)
  |          - Create FaceSuggestion (centroid as source)
  |
  +--- Strategy 4: Find More (Random Sampling)
       - Quality+Diversity weighted sampling of labeled faces
       - Each sampled face -> query Qdrant faces
       - Aggregate: MAX score across all sampled faces
       - Create FaceSuggestion with multi-prototype metadata
         |
         v
[FaceSuggestion records created (status=PENDING)]
         |
         v
[User reviews in UI]
  |     |     |
  |     |     +--- Accept: face.person_id = suggested_person_id
  |     |     |    (optionally triggers auto-find-more)
  |     |
  |     +--- Reject: suggestion.status = REJECTED
  |
  +--- No action: eventually auto-expired by cleanup job
```

---

## 9. Recommendations

### Priority 1: Critical Improvements

1. **Unify centroid computation paths**: The on-demand centroid computation in `find_more_centroid_suggestions_job()` (line 1951-2008 of `face_jobs.py`) uses simple mean without outlier trimming, while the formal `compute_centroids_for_person()` in `centroid_service.py` applies robust outlier trimming. These should use the same algorithm for consistency. **Recommendation**: Extract the centroid computation from `centroid_service.compute_global_centroid()` into a shared utility function and call it from both paths.

2. **Eliminate dual centroid storage**: The `FaceAssigner.compute_person_centroids()` method (lines 286-390 in `assigner.py`) stores centroids as regular face embeddings in the `faces` collection with `is_prototype=True, is_centroid=True`, while the formal `CentroidQdrantClient` stores them in the separate `person_centroids` collection. This dual-path storage can lead to stale or conflicting centroids. **Recommendation**: Deprecate the `FaceAssigner.compute_person_centroids()` method in favor of the formal `CentroidService` pipeline.

3. **Address O(n^2) confidence calculation scalability**: `calculate_cluster_confidence()` computes all pairwise cosine similarities. With the sampling cap at 20 faces, this means up to 190 pairs -- manageable but represents a design ceiling. For future cluster sizes, consider a more scalable metric (e.g., average distance to centroid, which is O(n)). **Recommendation**: Add an alternative `calculate_cluster_confidence_centroid()` method that computes mean cosine similarity to the cluster centroid instead of all pairs.

### Priority 2: Performance Optimizations

4. **Batch embedding retrieval in DualModeClusterer**: The `_get_face_embedding()` method (lines 344-366 in `dual_clusterer.py`) makes individual Qdrant `retrieve()` calls per face. For large face sets, this creates N sequential network round-trips. **Recommendation**: Use Qdrant's batch `retrieve()` with multiple IDs in a single call, or use `scroll()` with a filter to get all embeddings at once.

5. **Pre-filter unlabeled faces in Qdrant**: The `get_unlabeled_faces_with_embeddings()` method in `face_qdrant.py` scrolls all records and filters in Python due to Qdrant's lack of "is null" filter. **Recommendation**: Add a boolean payload field `has_person_id` (default False) and index it, allowing efficient Qdrant-side filtering. Update this field whenever `person_id` changes.

6. **Post-training suggestion batching**: The current approach queues one RQ job per person, which could overwhelm the queue for systems with many persons. **Recommendation**: Batch multiple persons per job (e.g., process 10 persons per job) and add configurable concurrency limits.

### Priority 3: Robustness Improvements

7. **Add centroid consistency validation**: Implement a periodic job that validates centroid vectors in Qdrant match the expected values based on current face assignments. This would catch desync between PostgreSQL and Qdrant. **Recommendation**: Create a `validate_centroids_job()` that recomputes source hashes and flags inconsistencies.

8. **Improve HDBSCAN metric consistency**: The `FaceClusterer` uses `metric="euclidean"` while `DualModeClusterer` uses `metric="precomputed"` with cosine distances. While mathematically equivalent for normalized vectors, this inconsistency could cause confusion. **Recommendation**: Standardize on precomputed cosine distance across both clusterers for clarity.

9. **Add suggestion feedback loop**: Currently rejected suggestions do not influence future suggestion generation. **Recommendation**: Track rejection patterns per person and per face, and use rejection history to suppress similar suggestions in future runs (e.g., increase the effective threshold for faces that have been rejected before).

---

## Appendix A: File Reference

| File | Purpose | Key Classes/Functions |
|---|---|---|
| `faces/detector.py` | Face detection with InsightFace | `DetectedFace`, `detect_faces()`, `detect_faces_from_path()` |
| `faces/clusterer.py` | Single-mode HDBSCAN clustering | `FaceClusterer`, `cluster_unlabeled_faces()`, `recluster_within_cluster()` |
| `faces/dual_clusterer.py` | Dual-mode supervised+unsupervised | `DualModeClusterer`, `cluster_all_faces()`, `assign_to_known_people()` |
| `faces/trainer.py` | Triplet loss training | `FaceTrainer`, `TripletFaceDataset`, `fine_tune_for_person_clustering()` |
| `faces/assigner.py` | Incremental face assignment | `FaceAssigner`, `assign_new_faces()`, `compute_person_centroids()` |
| `faces/service.py` | Face processing orchestration | `FaceProcessingService`, `process_asset()`, `process_assets_batch()` |
| `services/centroid_service.py` | Centroid computation and management | `compute_global_centroid()`, `compute_centroids_for_person()`, `is_centroid_stale()` |
| `services/face_clustering_service.py` | Cluster quality metrics | `FaceClusteringService`, `calculate_cluster_confidence()`, `select_representative_face()` |
| `services/training_service.py` | Training session lifecycle | `TrainingService`, `create_session()`, `enqueue_training()` |
| `vector/face_qdrant.py` | Qdrant client for faces | `FaceQdrantClient`, `search_similar_faces()`, `search_against_prototypes()` |
| `vector/centroid_qdrant.py` | Qdrant client for centroids | `CentroidQdrantClient`, `search_faces_with_centroid()`, `upsert_centroid()` |
| `queue/face_jobs.py` | RQ background jobs | All `*_job()` functions for detection, clustering, assignment, suggestions |
| `api/routes/face_suggestions.py` | Suggestion API endpoints | `list_suggestions()`, `accept_suggestion()`, `bulk_suggestion_action()` |
| `core/config.py` | Application configuration | `Settings` class with all threshold defaults |
| `db/models.py` | Database models | `FaceInstance`, `Person`, `PersonPrototype`, `PersonCentroid`, `FaceSuggestion` |

## Appendix B: Threshold Decision Tree

```
New face detected
     |
     v
Has quality_score >= 0.5? --NO--> Skip (not included in clustering)
     |
    YES
     |
     v
Search against prototypes (score_threshold = suggestion_threshold)
     |
     +--- Score >= auto_assign_threshold
     |    --> AUTO-ASSIGN: face.person_id = best_match.person_id
     |
     +--- Score >= suggestion_threshold (but < auto_assign)
     |    --> SUGGEST: Create FaceSuggestion(status=PENDING)
     |
     +--- Score < suggestion_threshold
          --> UNASSIGNED: face remains without person_id
               |
               v
          Run clustering (HDBSCAN)
               |
               +--- Assigned to cluster
               |    --> face.cluster_id = clu_{uuid}
               |
               +--- Noise point (label = -1)
                    --> face remains unclustered
```
