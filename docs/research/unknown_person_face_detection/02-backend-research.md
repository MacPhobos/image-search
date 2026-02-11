# Unknown Person Face Detection - Backend Research

> **Research Agent**: Backend Research
> **Date**: 2026-02-07
> **Scope**: Deep analysis of existing face-related backend code and design proposal for unknown person discovery
> **Status**: Complete

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Backend Architecture (Face-Related)](#2-current-backend-architecture-face-related)
3. [Existing Endpoints Analysis](#3-existing-endpoints-analysis)
4. [Proposed API Design](#4-proposed-api-design)
5. [Implementation Strategy](#5-implementation-strategy)
6. [Performance Analysis](#6-performance-analysis)
7. [Code Patterns to Follow](#7-code-patterns-to-follow)
8. [Recommendations](#8-recommendations)

---

## 1. Executive Summary

### Problem Statement

The current image-search application can suggest faces for **existing known persons** (via prototype matching, centroid search, and find-more flows) but cannot **discover NEW unknown persons** from the pool of unassigned faces. Users must manually browse individual unassigned faces or existing HDBSCAN clusters to identify new people -- a tedious process when thousands of unassigned faces exist.

### Key Finding

The backend already has most of the building blocks needed:

- **512-dim ArcFace embeddings** stored in Qdrant (`faces` collection) with cosine distance
- **HDBSCAN clustering** (dual-mode: supervised + unsupervised) via `DualModeClusterer`
- **Cluster listing with confidence scoring** via `list_clusters()` endpoint
- **Grouped pagination pattern** used by the face suggestions system (`SuggestionGroup` / `FaceSuggestionsGroupedResponse`)
- **Background job infrastructure** (Redis + RQ with progress tracking via Redis keys and SSE)
- **`get_unlabeled_faces_with_embeddings()`** in FaceQdrantClient -- retrieves all unassigned faces from Qdrant

### Gap Analysis

| Capability | Exists? | Location | Gap |
|---|---|---|---|
| Retrieve unassigned face embeddings | Yes | `FaceQdrantClient.get_unlabeled_faces_with_embeddings()` | Scrolls ALL faces, filters in Python (no Qdrant "is null" filter) -- works but O(N) for all faces |
| HDBSCAN clustering of unlabeled faces | Yes | `DualModeClusterer.cluster_all_faces()` | Runs as batch job, persists cluster_id on FaceInstance but does not present results as "candidate persons" |
| Cluster listing with quality metrics | Yes | `GET /faces/clusters` | Shows existing clusters, but not optimized for "unknown person discovery" UX |
| Grouped pagination by person | Yes | `GET /faces/suggestions?grouped=true` | Pattern for grouping by person exists but only works for existing persons |
| On-demand re-clustering with custom params | Partial | `POST /faces/cluster/dual` | Runs full clustering, not targeted "discover unknowns from unassigned" |
| Unknown person creation from cluster | Yes | `POST /faces/clusters/{cluster_id}/label` | Labels a cluster as a new person |
| **Dedicated "Discover Unknown Persons" endpoint** | **NO** | -- | **Primary gap: No endpoint to dynamically cluster unassigned faces and present candidate person groups** |
| **Quality-ranked unknown person candidates** | **NO** | -- | No ranking of clusters by "likelihood of being a real person" |
| **Pre-computed unknown person candidates** | **NO** | -- | No background job that pre-computes unknown person candidates for quick browsing |

### Recommendation

Build a new **Unknown Person Discovery** subsystem consisting of:
1. A background job that clusters unassigned faces and stores results as "candidate person groups"
2. API endpoints to list, browse, accept (create person), and dismiss candidate groups
3. A lightweight on-demand re-clustering endpoint for user-triggered discovery
4. Integration with existing `PersonService` unified people view

---

## 2. Current Backend Architecture (Face-Related)

### 2.1 Technology Stack

| Component | Technology | Details |
|---|---|---|
| Web Framework | FastAPI (async) | Python 3.12, all routes async |
| Database | PostgreSQL + SQLAlchemy 2.0 | asyncpg driver, async sessions |
| Vector Database | Qdrant | Cosine distance, 512-dim ArcFace embeddings |
| Background Jobs | Redis + RQ | 4-tier priority queues, sync operations in workers |
| Face Model | InsightFace `buffalo_l` | ArcFace r100 architecture, 512-dim output |
| Centroid Model | ArcFace r100 glint360k v1 | Same embedding space as face model |

### 2.2 Data Model (Key Entities)

```
ImageAsset (1) ----< (N) FaceInstance (1) ----< (N) FaceSuggestion
                           |                           |
                           |-- person_id (FK, nullable) -> Person
                           |-- cluster_id (nullable string)
                           |-- qdrant_point_id (UUID)
                           |-- detection_confidence
                           |-- quality_score
                           |
Person (1) ----< (N) PersonPrototype
   |                    |-- face_instance_id
   |                    |-- role (primary/temporal/exemplar)
   |                    |-- is_pinned
   |
   |----< (N) PersonCentroid
                |-- centroid_id (UUID PK)
                |-- qdrant_point_id
                |-- centroid_type (global/cluster)
                |-- model_version
                |-- centroid_version
                |-- n_faces
                |-- source_face_ids_hash
                |-- status (active/deprecated/building/failed)
```

**Key observations about FaceInstance**:
- `person_id IS NULL` indicates an unassigned face (the key filter for unknown person detection)
- `cluster_id` is a nullable string set by HDBSCAN clustering (e.g., "0", "1", "-1" for noise)
- `qdrant_point_id` links to the 512-dim embedding in Qdrant's `faces` collection
- `quality_score` (0.0-1.0) is computed during detection and can be used for filtering

**FaceSuggestion fields relevant to the new feature**:
- `confidence`: Cosine similarity score (0.0-1.0)
- `status`: pending / accepted / rejected / expired
- Multi-prototype scoring: `matching_prototype_ids`, `prototype_scores`, `aggregate_confidence`, `prototype_match_count`

### 2.3 Qdrant Collections

| Collection | Dim | Distance | Payload Fields | Purpose |
|---|---|---|---|---|
| `faces` | 512 | Cosine | asset_id, face_instance_id, person_id, cluster_id, detection_confidence, quality_score, is_prototype | Face embeddings for similarity search |
| `person_centroids` | 512 | Cosine | person_id, centroid_id, model_version, centroid_version, centroid_type, cluster_label, n_faces | Person centroid embeddings |
| `image_assets` | 512 | Cosine | (CLIP embeddings) | Semantic image search (unrelated to faces) |

**Critical Qdrant limitation**: No native "is null" filter for payload fields. The `get_unlabeled_faces_with_embeddings()` method scrolls all records and filters in Python. This is O(N) where N = total faces in the collection.

### 2.4 Background Job Architecture

Jobs are defined in `queue/face_jobs.py` and executed by RQ workers:

```
face_jobs.py (sync operations, fork-safe for macOS)
  |
  |-- detect_faces_job()                    # Face detection batch
  |-- detect_faces_for_session_job()        # Full session pipeline
  |-- cluster_faces_job()                   # HDBSCAN clustering
  |-- cluster_dual_job()                    # Supervised + unsupervised
  |-- assign_faces_job()                    # Auto-assign to known persons
  |-- compute_centroids_job()               # Person centroid computation
  |-- propagate_person_label_job()          # Single-source face suggestion
  |-- propagate_person_label_multiproto_job()  # Multi-prototype suggestions
  |-- find_more_suggestions_job()           # Random sampling find-more
  |-- find_more_centroid_suggestions_job()  # Centroid-based find-more
  |-- expire_old_suggestions_job()          # Cleanup stale suggestions
  |-- cleanup_orphaned_suggestions_job()    # Cleanup orphaned suggestions
```

**All jobs use**:
- `get_sync_session()` for database access (sync, not async -- required for RQ fork model)
- `get_current_job()` for job ID tracking
- Redis for progress updates (JSON payloads with phase/current/total/message)
- Standard return dict with `status`, counts, and optional `error`

### 2.5 Configuration System

Settings are managed via `core/config.py` (pydantic-settings) with environment variables:

| Setting | Default | Description |
|---|---|---|
| `face_person_match_threshold` | 0.7 | Minimum similarity for supervised clustering |
| `face_unknown_clustering_method` | "hdbscan" | Unsupervised clustering algorithm |
| `face_unknown_min_cluster_size` | 3 | Min faces per HDBSCAN cluster |
| `face_unknown_eps` | 0.5 | DBSCAN/agglomerative distance threshold |
| `unknown_face_cluster_min_confidence` | 0.70 | Min intra-cluster confidence for display |
| `unknown_face_cluster_min_size` | 2 | Min faces per cluster for display |
| `face_suggestion_min_confidence` | 0.7 | Min confidence for face suggestions |
| `face_suggestion_max_results` | 5 | Max suggestions to return |
| `centroid_min_faces` | 2 | Min faces for centroid computation |
| `centroid_trim_threshold_small` | 0.05 | Outlier trim for 50-300 faces |
| `centroid_trim_threshold_large` | 0.10 | Outlier trim for 300+ faces |

Additionally, a `SyncConfigService` provides database-backed configuration for dynamic settings like `face_suggestion_threshold`, `face_suggestion_expiry_days`, `post_training_suggestions_mode`, and `post_training_use_centroids`.

---

## 3. Existing Endpoints Analysis

### 3.1 Cluster Endpoints (in `faces.py` routes)

| Endpoint | Method | Description | Relevance to New Feature |
|---|---|---|---|
| `/faces/clusters` | GET | List clusters with pagination, confidence filtering, min size filtering | HIGH -- existing pattern for browsing unknown face groups |
| `/faces/clusters/{cluster_id}` | GET | Get cluster detail with all faces | HIGH -- pattern for viewing a candidate person group |
| `/faces/clusters/{cluster_id}/label` | POST | Label cluster as a new person (creates Person, assigns faces, creates prototypes) | HIGH -- this IS the "accept unknown person" flow |
| `/faces/clusters/{cluster_id}/split` | POST | Split cluster into sub-clusters | MEDIUM -- useful for refining candidate groups |
| `/faces/cluster` | POST | Trigger HDBSCAN clustering | HIGH -- existing clustering trigger |
| `/faces/cluster/dual` | POST | Trigger dual-mode clustering (supervised + unsupervised) | HIGH -- the closest existing feature |

**Key implementation details from `list_clusters()`**:
- Uses `GROUP BY cluster_id` with aggregate functions (count, avg quality)
- Supports `include_labeled` filter (bool_or for PostgreSQL, MAX with CAST for SQLite)
- Cluster confidence is calculated on-the-fly using FaceClusteringService when `min_confidence` filter is specified
- Standard pagination: page/page_size/total
- Returns `ClusterSummary` with: cluster_id, face_count, sample_face_ids, avg_quality, cluster_confidence, representative_face_id, person_id, person_name

### 3.2 Face Suggestion Endpoints (in `face_suggestions.py` routes)

| Endpoint | Method | Description | Relevance to New Feature |
|---|---|---|---|
| `/faces/suggestions` | GET | List suggestions (grouped or flat) | HIGH -- grouped pagination pattern |
| `/faces/suggestions/stats` | GET | Suggestion statistics | MEDIUM -- pattern for stats endpoint |
| `/faces/suggestions/{id}/accept` | POST | Accept suggestion | HIGH -- pattern for accept/create-person flow |
| `/faces/suggestions/{id}/reject` | POST | Reject suggestion | HIGH -- pattern for dismiss flow |
| `/faces/suggestions/bulk-action` | POST | Bulk accept/reject | HIGH -- pattern for bulk operations |
| `/faces/suggestions/persons/{id}/find-more` | POST | Find more via random prototypes | MEDIUM -- background job with progress |
| `/faces/suggestions/persons/{id}/find-more-centroid` | POST | Find more via centroid | MEDIUM -- centroid search pattern |

**Key implementation details from `_list_suggestions_grouped()`**:
- Uses SQL window functions: `ROW_NUMBER() OVER (PARTITION BY suggested_person_id ORDER BY confidence DESC)`
- Groups suggestions by person, limits suggestions per group
- Returns `FaceSuggestionsGroupedResponse` with: groups[], total_groups, total_suggestions, page, groups_per_page, suggestions_per_group
- Each `SuggestionGroup` contains: person_id, person_name, suggestion_count, max_confidence, suggestions[]

### 3.3 Centroid Endpoints (in `face_centroids.py` routes)

| Endpoint | Method | Description | Relevance |
|---|---|---|---|
| `/faces/centroids/persons/{id}/compute` | POST | Compute centroid for person | LOW -- requires existing person |
| `/faces/centroids/persons/{id}` | GET | Get centroids for person | LOW |
| `/faces/centroids/persons/{id}/suggestions` | POST | Get suggestions via centroid | MEDIUM -- centroid search flow |

### 3.4 Unified People View (in `faces.py` routes)

| Endpoint | Method | Description | Relevance |
|---|---|---|---|
| `/faces/people` | GET | Unified list of identified persons + unidentified clusters + noise | HIGH -- this endpoint already combines persons and clusters |

**Implementation**: Delegates to `PersonService.get_all_people()` which returns `UnifiedPeopleListResponse` with `PersonType` enum: IDENTIFIED, UNIDENTIFIED, NOISE.

### 3.5 Face Assignment Flow

The accept-suggestion flow (`POST /faces/suggestions/{id}/accept`) performs:
1. Updates `FaceInstance.person_id` to the suggested person
2. Updates `FaceSuggestion.status` to "accepted"
3. Syncs person_id to Qdrant via `qdrant.update_person_ids()`
4. Optionally triggers find-more jobs for the person

The label-cluster flow (`POST /faces/clusters/{cluster_id}/label`) performs:
1. Creates a new `Person` record with the given name
2. Assigns all faces in the cluster to the new person
3. Creates `PersonPrototype` records (representative + exemplars)
4. Updates Qdrant with person_id for all faces
5. Returns `LabelClusterResponse` with person_id, person_name, faces_labeled, prototypes_created

---

## 4. Proposed API Design

### 4.1 Design Philosophy

The unknown person discovery feature should:
- **Reuse existing patterns** (grouped pagination, background jobs with progress, CamelCase schemas)
- **Leverage existing infrastructure** (HDBSCAN clustering, Qdrant similarity search, cluster labeling)
- **Support two modes**: pre-computed (background job) and on-demand (API-triggered)
- **Provide a review workflow** similar to suggestions: browse candidates, accept (create person), or dismiss

### 4.2 New Endpoints

#### 4.2.1 Trigger Unknown Person Discovery (Background Job)

```
POST /api/v1/faces/unknown-persons/discover
```

**Request Schema**: `DiscoverUnknownPersonsRequest`
```python
class DiscoverUnknownPersonsRequest(CamelCaseModel):
    """Request to discover unknown persons from unassigned faces."""

    clustering_method: str = Field(
        default="hdbscan",
        pattern="^(hdbscan|dbscan|agglomerative)$",
        description="Clustering algorithm to use",
    )
    min_cluster_size: int = Field(
        default=3, ge=2, le=50,
        description="Minimum faces per candidate person group",
    )
    min_quality: float = Field(
        default=0.3, ge=0.0, le=1.0,
        description="Minimum face quality score to include",
    )
    max_faces: int = Field(
        default=10000, ge=100, le=50000,
        description="Maximum unassigned faces to process",
    )
    min_cluster_confidence: float = Field(
        default=0.65, ge=0.0, le=1.0,
        description="Minimum intra-cluster confidence for a candidate group",
    )
    eps: float = Field(
        default=0.5, ge=0.0, le=2.0,
        description="Distance threshold for DBSCAN/Agglomerative",
    )
```

**Response Schema**: `DiscoverUnknownPersonsJobResponse`
```python
class DiscoverUnknownPersonsJobResponse(CamelCaseModel):
    """Response when starting an unknown persons discovery job."""

    job_id: str
    status: str  # "queued"
    progress_key: str  # Redis key for SSE progress
    params: dict  # Echo back the clustering parameters
```

**Behavior**:
1. Enqueues a background job (`discover_unknown_persons_job`)
2. Job retrieves unassigned faces from Qdrant (via `get_unlabeled_faces_with_embeddings()`)
3. Runs HDBSCAN clustering on the embeddings
4. Computes intra-cluster confidence for each cluster
5. Filters by min_cluster_confidence and min_cluster_size
6. Stores results in a new `UnknownPersonCandidate` table (or reuses `FaceInstance.cluster_id`)
7. Reports progress via Redis SSE

#### 4.2.2 List Unknown Person Candidates (Grouped)

```
GET /api/v1/faces/unknown-persons/candidates
```

**Query Parameters**:
```python
page: int = Query(1, ge=1)
groups_per_page: int = Query(10, ge=1, le=50)
faces_per_group: int = Query(6, ge=1, le=20)
min_confidence: float | None = Query(None, ge=0.0, le=1.0)
min_group_size: int | None = Query(None, ge=2)
sort_by: str = Query("confidence", regex="^(confidence|face_count|quality)$")
sort_order: str = Query("desc", regex="^(asc|desc)$")
```

**Response Schema**: `UnknownPersonCandidatesResponse`
```python
class UnknownPersonCandidateGroup(CamelCaseModel):
    """A candidate unknown person group."""

    group_id: str  # cluster_id or generated group identifier
    face_count: int  # Total faces in this group
    cluster_confidence: float  # Intra-cluster similarity score
    avg_quality: float  # Average face quality score
    representative_face: FaceInstanceResponse  # Best face for thumbnail
    sample_faces: list[FaceInstanceResponse]  # Limited by faces_per_group
    suggested_name: str | None = None  # Auto-generated placeholder

class UnknownPersonCandidatesResponse(CamelCaseModel):
    """Grouped response for unknown person candidates."""

    groups: list[UnknownPersonCandidateGroup]
    total_groups: int  # Total candidate groups
    total_unassigned_faces: int  # Total unassigned faces in system
    total_noise_faces: int  # Faces not in any cluster (noise)
    page: int
    groups_per_page: int
    faces_per_group: int
    last_discovery_at: datetime | None  # When discovery was last run
```

**Behavior**:
- Queries clusters from `FaceInstance` where `person_id IS NULL AND cluster_id IS NOT NULL AND cluster_id != '-1'`
- Groups by cluster_id with aggregates (count, avg quality)
- Computes cluster confidence (either pre-computed or on-the-fly via sampling)
- Applies filters and sorting
- Returns paginated grouped response

#### 4.2.3 Get Unknown Person Candidate Detail

```
GET /api/v1/faces/unknown-persons/candidates/{group_id}
```

**Response Schema**: `UnknownPersonCandidateDetailResponse`
```python
class UnknownPersonCandidateDetailResponse(CamelCaseModel):
    """Detailed view of a candidate person group with all faces."""

    group_id: str
    face_count: int
    cluster_confidence: float
    avg_quality: float
    faces: list[FaceInstanceResponse]  # All faces in the group
    similar_persons: list[SimilarPersonMatch] | None = None  # Persons with similar faces

class SimilarPersonMatch(CamelCaseModel):
    """A known person that is similar to this unknown group."""

    person_id: UUID
    person_name: str
    similarity: float  # Cosine similarity between group centroid and person centroid
    face_count: int
```

**Behavior**:
- Returns all faces in the cluster/group
- Optionally computes similarity to existing persons (if centroids exist) to help users decide if this is a known person or truly new

#### 4.2.4 Accept Unknown Person (Create Person from Group)

```
POST /api/v1/faces/unknown-persons/candidates/{group_id}/accept
```

**Request Schema**: `AcceptUnknownPersonRequest`
```python
class AcceptUnknownPersonRequest(CamelCaseModel):
    """Request to accept a candidate group as a new person."""

    name: str = Field(min_length=1, max_length=255)
    auto_find_more: bool = Field(
        default=True,
        description="Automatically trigger find-more after creating person",
    )
    face_ids_to_exclude: list[UUID] | None = Field(
        default=None,
        description="Optional list of face IDs to exclude from the new person",
    )
```

**Response Schema**: `AcceptUnknownPersonResponse`
```python
class AcceptUnknownPersonResponse(CamelCaseModel):
    """Response from accepting an unknown person candidate."""

    person_id: UUID
    person_name: str
    faces_assigned: int
    prototypes_created: int
    find_more_job_id: str | None = None  # If auto_find_more was triggered
```

**Behavior**:
1. Validates the group exists and is not already labeled
2. Creates a new `Person` record
3. Assigns all faces (minus excluded) to the new person
4. Creates prototypes (representative + exemplars)
5. Syncs to Qdrant
6. Optionally triggers find-more job
7. This is essentially the same as `POST /faces/clusters/{cluster_id}/label` but with the unknown persons UX context

#### 4.2.5 Dismiss Unknown Person Candidate

```
POST /api/v1/faces/unknown-persons/candidates/{group_id}/dismiss
```

**Request Schema**: `DismissUnknownPersonRequest`
```python
class DismissUnknownPersonRequest(CamelCaseModel):
    """Request to dismiss a candidate group."""

    reason: str | None = Field(
        default=None,
        description="Optional reason for dismissal",
    )
    mark_as_noise: bool = Field(
        default=False,
        description="Mark faces as noise (prevents re-clustering)",
    )
```

**Response Schema**: `DismissUnknownPersonResponse`
```python
class DismissUnknownPersonResponse(CamelCaseModel):
    """Response from dismissing a candidate."""

    group_id: str
    faces_affected: int
    marked_as_noise: bool
```

**Behavior**:
1. If `mark_as_noise=True`: Set `cluster_id = '-1'` on all faces in the group (prevents re-clustering)
2. If `mark_as_noise=False`: Clear `cluster_id` on all faces (allows re-clustering with different params)
3. Optionally store the dismissal for analytics

#### 4.2.6 Unknown Persons Discovery Stats

```
GET /api/v1/faces/unknown-persons/stats
```

**Response Schema**: `UnknownPersonsStatsResponse`
```python
class UnknownPersonsStatsResponse(CamelCaseModel):
    """Statistics for unknown person discovery."""

    total_unassigned_faces: int  # Faces with person_id IS NULL
    total_clustered_faces: int  # Unassigned faces with cluster_id
    total_noise_faces: int  # cluster_id = '-1' or noise markers
    total_unclustered_faces: int  # No cluster_id, no person_id
    candidate_groups: int  # Number of qualifying candidate groups
    avg_group_size: float
    avg_group_confidence: float
    last_discovery_at: datetime | None
    last_discovery_params: dict | None
```

### 4.3 Full Endpoint Summary

| Endpoint | Method | Purpose |
|---|---|---|
| `/faces/unknown-persons/discover` | POST | Trigger discovery background job |
| `/faces/unknown-persons/candidates` | GET | List candidate groups (paginated) |
| `/faces/unknown-persons/candidates/{group_id}` | GET | View candidate group detail |
| `/faces/unknown-persons/candidates/{group_id}/accept` | POST | Create person from candidate |
| `/faces/unknown-persons/candidates/{group_id}/dismiss` | POST | Dismiss candidate |
| `/faces/unknown-persons/stats` | GET | Discovery statistics |

---

## 5. Implementation Strategy

### 5.1 Phase 1: Leverage Existing Clusters (Minimal New Code)

The simplest approach reuses the existing `cluster_id` on `FaceInstance` populated by `cluster_dual_job()`:

**No new database tables needed** for Phase 1. The existing clusters (`FaceInstance.cluster_id`) already represent candidate person groups. The new endpoints simply query and present these clusters differently.

#### Implementation Steps:

1. **New route file**: `api/routes/unknown_persons.py`
   - Register router in `main.py` under `/api/v1/faces/unknown-persons`

2. **New schema file**: `api/unknown_person_schemas.py`
   - All schemas from Section 4.2 above
   - Follow CamelCaseModel pattern

3. **Stats endpoint** (`GET /stats`):
   - Query: `SELECT COUNT(*) FROM face_instances WHERE person_id IS NULL`
   - Query: `SELECT COUNT(*) FROM face_instances WHERE person_id IS NULL AND cluster_id IS NOT NULL AND cluster_id != '-1'`
   - Query: `SELECT COUNT(DISTINCT cluster_id) FROM face_instances WHERE person_id IS NULL AND cluster_id IS NOT NULL AND cluster_id != '-1'`

4. **Candidates listing** (`GET /candidates`):
   - Reuse the same SQL pattern from `list_clusters()` with filter `include_labeled=False`
   - Add confidence computation (either pre-computed or sampled)
   - Return grouped response

5. **Accept endpoint** (`POST /candidates/{group_id}/accept`):
   - Delegate to existing `label_cluster()` logic from `faces.py`
   - Extract to shared service layer: `ClusterLabelingService`

6. **Dismiss endpoint** (`POST /candidates/{group_id}/dismiss`):
   - Simple SQL UPDATE on `FaceInstance.cluster_id`

7. **Discover endpoint** (`POST /discover`):
   - Enqueue `cluster_dual_job()` or new `discover_unknown_persons_job()`

### 5.2 Phase 2: Enhanced Discovery (New Background Job)

Adds a dedicated discovery job with optimized workflow:

1. **New job**: `discover_unknown_persons_job()` in `queue/face_jobs.py`
   - Step 1: Get unassigned face embeddings from Qdrant
   - Step 2: Run HDBSCAN clustering
   - Step 3: Compute cluster quality metrics
   - Step 4: Compare cluster centroids against existing person centroids
   - Step 5: Store results and update progress

2. **Pre-computed cluster confidence**: Store `cluster_confidence` as a column on `FaceInstance` or in a new `ClusterMetadata` table to avoid on-the-fly computation.

3. **Similar person matching**: For each candidate group, compute centroid and search against `person_centroids` collection to find potentially matching existing persons.

### 5.3 Phase 3: Smart Discovery (ML-Enhanced)

Future enhancements:
- **Incremental clustering**: Only re-cluster newly detected faces (not all unassigned)
- **Active learning**: Track user accept/dismiss patterns to improve clustering parameters
- **Confidence calibration**: Adjust clustering thresholds based on user feedback
- **Cross-cluster merging**: Detect when two clusters likely belong to the same person

### 5.4 Database Schema Changes

#### Phase 1: No schema changes
- Reuse existing `FaceInstance.cluster_id` and `FaceInstance.person_id`

#### Phase 2: Optional metadata table

```python
class UnknownPersonDiscovery(Base):
    """Tracks discovery job results and metadata."""
    __tablename__ = "unknown_person_discoveries"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    job_id: Mapped[str] = mapped_column(String(36))
    status: Mapped[str] = mapped_column(String(20))  # completed/failed
    params: Mapped[dict] = mapped_column(JSON)
    total_faces_processed: Mapped[int] = mapped_column(Integer)
    clusters_found: Mapped[int] = mapped_column(Integer)
    noise_count: Mapped[int] = mapped_column(Integer)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

class ClusterMetadata(Base):
    """Pre-computed metadata for face clusters."""
    __tablename__ = "cluster_metadata"

    cluster_id: Mapped[str] = mapped_column(String(50), primary_key=True)
    face_count: Mapped[int] = mapped_column(Integer)
    cluster_confidence: Mapped[float] = mapped_column(Float)
    avg_quality: Mapped[float] = mapped_column(Float)
    representative_face_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("face_instances.id"))
    centroid_vector_id: Mapped[uuid.UUID | None] = mapped_column(nullable=True)
    discovery_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("unknown_person_discoveries.id"))
    dismissed: Mapped[bool] = mapped_column(Boolean, default=False)
    dismissed_at: Mapped[datetime | None] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

---

## 6. Performance Analysis

### 6.1 Scale Considerations

| Metric | Typical Value | Worst Case | Impact |
|---|---|---|---|
| Total faces in Qdrant | 5,000-20,000 | 100,000+ | Scroll + filter time for unlabeled faces |
| Unassigned faces | 500-5,000 | 50,000+ | HDBSCAN computation time |
| Existing clusters | 50-200 | 1,000+ | Listing query time |
| Faces per cluster | 3-50 | 500+ | Confidence computation time |

### 6.2 Computational Cost Analysis

#### Retrieving Unassigned Faces from Qdrant

**Current approach** (`get_unlabeled_faces_with_embeddings()`):
- Scrolls ALL faces (100 per page) and filters in Python for missing `person_id`
- For 20,000 total faces with 5,000 unassigned: 200 scroll requests, ~10-20 seconds
- For 100,000 total faces with 50,000 unassigned: 1,000 scroll requests, ~50-100 seconds

**Optimization opportunity**:
- Add a Qdrant payload index on `person_id` and use `must_not` filter with a sentinel value (e.g., store `"unassigned"` instead of omitting the field)
- Or maintain a separate "unassigned faces" collection (write complexity vs read performance tradeoff)

#### HDBSCAN Clustering

- **Time complexity**: O(N^2) for distance matrix computation where N = number of faces
- **For 5,000 faces**: ~2-5 seconds (well-suited for background job)
- **For 50,000 faces**: ~200-500 seconds (requires batching or sampling)
- **Memory**: O(N^2) for distance matrix -- 5,000 faces = ~100MB, 50,000 faces = ~10GB

**Recommendation**: Cap `max_faces` at 10,000-20,000 for interactive use. For larger datasets, use sampling or incremental approaches.

#### Cluster Confidence Computation

- `calculate_cluster_confidence()` uses pairwise cosine similarity
- Already samples max 20 faces for large clusters (existing optimization)
- For 200 clusters of average size 25: ~200 * 190 comparisons = negligible

### 6.3 Pre-compute vs On-demand

| Approach | Pros | Cons |
|---|---|---|
| **Pre-compute (background job)** | Fast browsing, no wait for user | Stale if new faces added; storage overhead |
| **On-demand (API-triggered)** | Always fresh results | Slow for large datasets; blocks user flow |
| **Hybrid (recommended)** | Fast browsing with refresh option | Slightly more complex; best UX |

**Recommended approach**: Pre-compute clusters via background job (triggered after face detection sessions or manually), with a "Refresh" button that re-runs discovery.

### 6.4 Caching Strategy

1. **Redis cache for cluster metadata**: Cache cluster stats (face_count, confidence, representative_face) with TTL of 1 hour
2. **Invalidation triggers**: New face detection, face assignment changes, cluster relabeling
3. **Progress tracking**: Reuse existing Redis progress key pattern from find-more jobs

### 6.5 Response Time Targets

| Endpoint | Target | Strategy |
|---|---|---|
| `GET /candidates` (pre-computed) | <200ms | SQL query + cached confidence |
| `GET /candidates` (on-demand confidence) | <2s | Sample-based confidence computation |
| `GET /candidates/{id}` | <500ms | Direct cluster query |
| `POST /discover` | Immediate (async) | Returns job_id, work happens in background |
| `POST /candidates/{id}/accept` | <1s | Transaction: create person + assign faces |

---

## 7. Code Patterns to Follow

### 7.1 Schema Pattern (CamelCaseModel)

All schemas must extend `CamelCaseModel` from `api/face_session_schemas.py` or `api/face_schemas.py`:

```python
class CamelCaseModel(BaseModel):
    model_config = ConfigDict(
        populate_by_name=True,
        alias_generator=to_camel,
    )
```

### 7.2 Pagination Pattern

**Standard flat pagination** (used by clusters, persons):
```python
class XxxListResponse(CamelCaseModel):
    items: list[XxxResponse]
    total: int
    page: int
    page_size: int
```

**Grouped pagination** (used by suggestions):
```python
class XxxGroupedResponse(CamelCaseModel):
    groups: list[XxxGroup]
    total_groups: int
    total_items: int
    page: int
    groups_per_page: int
    items_per_group: int
```

### 7.3 Background Job Pattern

```python
def my_job(param1: str, param2: int = 10) -> dict[str, Any]:
    """Docstring with fork-safety note."""
    job = get_current_job()
    job_id = job.id if job else "no-job"

    logger.info(f"[{job_id}] Starting job...")

    db_session = get_sync_session()
    try:
        # ... job logic (sync operations only) ...

        # Progress updates via Redis
        update_progress("phase_name", current, total, "message")

        return {"status": "completed", "count": N}
    except Exception as e:
        logger.exception(f"[{job_id}] Error: {e}")
        return {"status": "error", "message": str(e)}
    finally:
        db_session.close()
```

### 7.4 Route Handler Pattern

```python
@router.get("/endpoint", response_model=ResponseSchema)
async def endpoint_handler(
    param: int = Query(1, ge=1),
    db: AsyncSession = Depends(get_db),
) -> ResponseSchema:
    """Docstring."""
    # Query logic using async SQLAlchemy
    result = await db.execute(query)
    # Transform to response schema
    return ResponseSchema(...)
```

### 7.5 Qdrant Client Pattern (Singleton)

```python
class MyQdrantClient:
    _instance: "MyQdrantClient | None" = None
    _client: QdrantClient | None = None

    @classmethod
    def get_instance(cls) -> "MyQdrantClient":
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance

    @property
    def client(self) -> QdrantClient:
        if self._client is None:
            settings = get_settings()
            self._client = QdrantClient(url=settings.qdrant_url, ...)
        return self._client
```

### 7.6 Cluster Confidence Computation Pattern

From `FaceClusteringService.calculate_cluster_confidence()`:
```python
# Sample max 20 faces for large clusters
sample_size = min(len(face_ids), 20)
if len(face_ids) > sample_size:
    face_ids = random.sample(face_ids, sample_size)

# Compute pairwise cosine similarity
for i, j in combinations(range(len(embeddings)), 2):
    sim = cosine_similarity(embeddings[i], embeddings[j])
    similarities.append(sim)

confidence = sum(similarities) / len(similarities)  # Average pairwise similarity
```

### 7.7 Cluster Labeling Pattern

From `POST /faces/clusters/{cluster_id}/label` (existing):
1. Query all faces in cluster
2. Create `Person` with given name
3. Update `FaceInstance.person_id` for all faces
4. Select representative face (highest composite quality score)
5. Create `PersonPrototype` records
6. Sync person_id to Qdrant
7. Return `LabelClusterResponse`

This exact pattern should be reused for the accept-unknown-person flow.

### 7.8 Testing Pattern

- Tests use in-memory SQLite (not Postgres)
- Qdrant mocked via dependency injection
- Test naming: `test_{behavior}_when_{condition}_then_{result}()`
- Test structure: `tests/api/test_unknown_persons.py` and `tests/unit/test_unknown_persons_service.py`

---

## 8. Recommendations

### 8.1 Implementation Priority

| Priority | Item | Effort | Impact |
|---|---|---|---|
| P0 | `GET /candidates` (listing from existing clusters) | Small | High -- immediate value |
| P0 | `GET /candidates/{id}` (detail view) | Small | High -- essential for UX |
| P0 | `POST /candidates/{id}/accept` (reuse label_cluster logic) | Small | High -- completes the flow |
| P1 | `POST /candidates/{id}/dismiss` | Small | Medium -- cleanup flow |
| P1 | `GET /stats` | Small | Medium -- dashboard insight |
| P1 | `POST /discover` (trigger re-clustering) | Medium | High -- enables fresh discovery |
| P2 | Similar person matching (centroid comparison) | Medium | Medium -- helps identify duplicates |
| P2 | Pre-computed cluster metadata table | Medium | Medium -- improves listing performance |
| P3 | Incremental clustering | Large | High -- scalability for 100k+ faces |
| P3 | Active learning from user feedback | Large | Medium -- improves accuracy over time |

### 8.2 Architecture Decisions

1. **Reuse `cluster_id` on FaceInstance** for Phase 1 rather than creating a new entity. The existing dual-mode clustering already populates this field.

2. **New route file** (`unknown_persons.py`) rather than adding to the already 2,300+ line `faces.py` file.

3. **Shared service layer**: Extract cluster labeling logic into a `ClusterService` that both `faces.py` and `unknown_persons.py` can use, avoiding code duplication.

4. **Background job for discovery**: Always run clustering as a background job since HDBSCAN is O(N^2). Never block the API request.

5. **Hybrid pagination**: Use grouped pagination for the candidates listing (similar to suggestions) with configurable `faces_per_group` to show sample faces.

### 8.3 Technical Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| HDBSCAN too slow for 50K+ faces | Medium | Cap max_faces, implement sampling, background job |
| Qdrant scroll too slow for unlabeled face retrieval | Medium | Add sentinel value for person_id, create payload index |
| Cluster quality inconsistent across runs | Low | Store and version discovery results, allow manual param tuning |
| User confusion between clusters view and unknown persons view | Medium | Clear UX distinction, potentially deprecate raw clusters view |

### 8.4 Integration Points

1. **Post-detection trigger**: After `detect_faces_for_session_job()` completes, optionally queue `discover_unknown_persons_job()` to refresh candidate groups.

2. **Unified people view**: Add unknown person candidates as a new `PersonType.CANDIDATE` in `UnifiedPeopleListResponse`.

3. **Frontend notification**: When discovery job completes and finds new candidate groups, notify the UI via SSE (reuse existing progress key pattern).

4. **Suggestion system integration**: After accepting an unknown person, auto-trigger find-more to discover additional faces that belong to the new person.

### 8.5 API Contract Impact

New endpoints should be added under the existing `/api/v1/faces/` prefix. The API contract document (`docs/api-contract.md`) must be updated with:
- New endpoint definitions
- New request/response schemas
- New error codes (404 for unknown group_id, 409 for already-labeled group)

Frontend types must be regenerated after backend changes: `npm run gen:api`.

### 8.6 Configuration Additions

New settings for `core/config.py`:

```python
# Unknown person discovery settings
unknown_person_discovery_auto_trigger: bool = Field(
    default=False,
    description="Auto-trigger discovery after face detection sessions",
)
unknown_person_min_cluster_size: int = Field(
    default=3, ge=2,
    description="Minimum faces per candidate person group",
)
unknown_person_min_confidence: float = Field(
    default=0.65, ge=0.0, le=1.0,
    description="Minimum cluster confidence for candidate groups",
)
unknown_person_max_faces: int = Field(
    default=10000, ge=100,
    description="Maximum unassigned faces to process during discovery",
)
```

---

## Appendix A: File Reference

| File | Purpose | Lines |
|---|---|---|
| `api/routes/faces.py` | Main face routes (clusters, persons, assignments) | ~2,300 |
| `api/routes/face_suggestions.py` | Face suggestion routes | ~800 |
| `api/routes/face_centroids.py` | Centroid routes | ~300 |
| `api/face_schemas.py` | Face-related Pydantic schemas | 564 |
| `api/face_session_schemas.py` | Session and suggestion schemas | 284 |
| `db/models.py` | SQLAlchemy models | ~800 |
| `core/config.py` | Application settings | 253 |
| `vector/face_qdrant.py` | Qdrant face collection client | ~700 |
| `vector/centroid_qdrant.py` | Qdrant centroid collection client | 540 |
| `services/face_clustering_service.py` | Clustering utilities | ~300 |
| `services/centroid_service.py` | Centroid computation | ~400 |
| `services/person_service.py` | Unified person view service | ~200 |
| `faces/dual_clusterer.py` | Dual-mode clustering (supervised + unsupervised) | ~300 |
| `queue/face_jobs.py` | Background job definitions | 2,136 |

## Appendix B: Existing Cluster Flow Diagram

```
[Face Detection] ──> [Qdrant: faces collection]
                              │
                              ▼
[cluster_dual_job()] ──> [HDBSCAN on unlabeled faces]
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
              [Cluster 0] [Cluster 1] [Noise (-1)]
                    │         │
                    ▼         ▼
        [FaceInstance.cluster_id = "0"] [FaceInstance.cluster_id = "1"]
                    │
                    ▼
        [GET /faces/clusters] ──> [User views clusters]
                    │
                    ▼
        [POST /faces/clusters/{id}/label] ──> [Creates Person + assigns faces]
```

## Appendix C: Proposed Unknown Person Discovery Flow

```
[User triggers "Discover Unknown Persons"]
        │
        ▼
[POST /faces/unknown-persons/discover]
        │
        ▼
[RQ Job: discover_unknown_persons_job()]
        │
        ├── 1. get_unlabeled_faces_with_embeddings() from Qdrant
        ├── 2. HDBSCAN clustering
        ├── 3. Compute cluster quality metrics
        ├── 4. Compare against existing person centroids
        └── 5. Store results (cluster_id on FaceInstance)
        │
        ▼ (Progress via Redis SSE)
        │
[GET /faces/unknown-persons/candidates] ──> [User browses candidate groups]
        │
        ├── [Accept] ──> POST /candidates/{id}/accept
        │                   ├── Create Person
        │                   ├── Assign faces
        │                   ├── Create prototypes
        │                   └── Trigger find-more (optional)
        │
        └── [Dismiss] ──> POST /candidates/{id}/dismiss
                            ├── Mark as noise (optional)
                            └── Clear cluster_id (allow re-clustering)
```
