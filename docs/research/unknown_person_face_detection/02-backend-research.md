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

---

## 1.5 Resolved Requirements (User Decisions)

**Context**: The following requirements have been finalized based on product direction and affect backend implementation:

### Scale and Performance

| Requirement | Decision | Backend Impact |
|---|---|---|
| **Expected scale** | **50,000 unlabeled faces** | O(N²) HDBSCAN is infeasible. MUST use chunked/batched approach or alternative algorithm. Original analysis assumed 5K-10K faces - this is a critical scale change. |
| **Processing model** | **Pre-computed clustering via background job** | Clustering runs as background job (not on-demand). Default threshold: 0.70. Mandatory async processing. |
| **Re-computation trigger** | **Triggered on person creation** | After accepting a group and creating a person, auto-trigger re-clustering in background. |
| **Auto find-more** | **Always trigger after person creation** | Accept/create-person endpoint must enqueue find-more job automatically. |

### Filtering and Pagination

| Requirement | Decision | Backend Impact |
|---|---|---|
| **Minimum group size** | **5 faces per group (default)** | Filter groups server-side with `face_count >= 5`. Must be configurable via Admin UI settings (requires admin settings endpoint integration). |
| **Pagination** | **50 groups per page, sort by face count descending** | Endpoint must support offset/limit pagination and sort by group size (most faces first). |
| **Default threshold** | **0.70 confidence minimum** | Configurable, but 0.70 is the baseline for production use. |

### Group Management

| Requirement | Decision | Backend Impact |
|---|---|---|
| **Dismissed groups** | **Store in database** | Needs new database table or field to track dismissals. Must be keyed by membership hash for stability across re-clustering. |
| **Group identity** | **Membership hash determines identity** | The set of face IDs in a cluster determines its identity. If the same faces cluster together across runs, it's the "same" group. Requires computing stable hash of sorted face_instance_ids. |
| **Partial acceptance** | **Accept subset of faces from group** | Accept endpoint must handle `face_ids_to_exclude` parameter. Not all-or-nothing. |

### User Experience

| Requirement | Decision | Backend Impact |
|---|---|---|
| **MVP scope** | **"Suggested New Persons" (simpler version first)** | Ship minimal viable feature before advanced enhancements. Focus on core workflow: discover → review → accept/dismiss. |
| **Auto-refresh** | **UI auto-refreshes when background job completes** | May need SSE/WebSocket or polling endpoint for job status. Reuse existing Redis progress key pattern. |

### Key Architectural Constraints from Decisions

1. **50K scale mandate** requires either:
   - Chunked/batched HDBSCAN (process N faces at a time, e.g., 10K chunks)
   - Hierarchical clustering (cluster sub-groups first, then merge)
   - Approximate clustering (sampling or k-means initialization)
   - OR accepting longer job execution times (10-30 minutes acceptable for background job)

2. **Membership hash requirement** adds complexity:
   - Must compute stable hash: `hash(sorted(face_instance_ids))`
   - Must persist hash with cluster metadata
   - Must check dismissed table by hash before showing group

3. **Admin settings integration** requires:
   - New admin settings endpoint or extending existing `SyncConfigService`
   - Settings: `min_group_size`, `min_confidence`, `auto_trigger_discovery`

4. **Partial acceptance** changes accept flow:
   - Original design: label entire cluster
   - New requirement: accept subset, leave remainder unlabeled or in cluster

---

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

**Design Notes (Post-Decision Updates)**:
- All endpoints updated to reflect **50K scale requirement** (chunked processing)
- **Pagination defaults**: 50 groups per page, sort by face count descending
- **Admin settings integration**: min_group_size configurable, default 5
- **Membership hash tracking**: All group responses include stable hash
- **Partial acceptance**: Accept endpoint supports face exclusion list

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
        default=5, ge=2, le=50,
        description="Minimum faces per candidate person group (default: 5)",
    )
    min_quality: float = Field(
        default=0.3, ge=0.0, le=1.0,
        description="Minimum face quality score to include",
    )
    max_faces: int = Field(
        default=50000, ge=100, le=100000,
        description="Maximum unassigned faces to process (default: 50K, supports up to 100K with chunking)",
    )
    min_cluster_confidence: float = Field(
        default=0.70, ge=0.0, le=1.0,
        description="Minimum intra-cluster confidence for a candidate group (default: 0.70)",
    )
    eps: float = Field(
        default=0.5, ge=0.0, le=2.0,
        description="Distance threshold for DBSCAN/Agglomerative",
    )
    chunk_size: int = Field(
        default=10000, ge=1000, le=20000,
        description="Chunk size for batched clustering (handles 50K+ faces)",
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
groups_per_page: int = Query(50, ge=1, le=100)  # Default: 50 per user decision
faces_per_group: int = Query(6, ge=1, le=20)
min_confidence: float | None = Query(None, ge=0.0, le=1.0)  # Default from admin settings: 0.70
min_group_size: int | None = Query(None, ge=2)  # Default from admin settings: 5
sort_by: str = Query("face_count", regex="^(confidence|face_count|quality)$")  # Default: face_count (most faces first)
sort_order: str = Query("desc", regex="^(asc|desc)$")
include_dismissed: bool = Query(False)  # Show dismissed groups (default: hide)
```

**Response Schema**: `UnknownPersonCandidatesResponse`
```python
class UnknownPersonCandidateGroup(CamelCaseModel):
    """A candidate unknown person group."""

    group_id: str  # cluster_id or generated group identifier
    membership_hash: str  # Stable hash of sorted face_instance_ids (for dismissal tracking)
    face_count: int  # Total faces in this group
    cluster_confidence: float  # Intra-cluster similarity score
    avg_quality: float  # Average face quality score
    representative_face: FaceInstanceResponse  # Best face for thumbnail
    sample_faces: list[FaceInstanceResponse]  # Limited by faces_per_group
    suggested_name: str | None = None  # Auto-generated placeholder
    is_dismissed: bool = False  # True if user dismissed this group
    dismissed_at: datetime | None = None

class UnknownPersonCandidatesResponse(CamelCaseModel):
    """Grouped response for unknown person candidates."""

    groups: list[UnknownPersonCandidateGroup]
    total_groups: int  # Total candidate groups (excluding dismissed unless include_dismissed=True)
    total_unassigned_faces: int  # Total unassigned faces in system
    total_noise_faces: int  # Faces not in any cluster (noise)
    total_dismissed_groups: int  # Number of dismissed groups
    page: int
    groups_per_page: int
    faces_per_group: int
    last_discovery_at: datetime | None  # When discovery was last run
    min_group_size_setting: int  # Current admin setting for min_group_size
    min_confidence_setting: float  # Current admin setting for min_confidence
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
        description="Automatically trigger find-more after creating person (ALWAYS True per user decision)",
    )
    face_ids_to_exclude: list[UUID] | None = Field(
        default=None,
        description="Optional list of face IDs to exclude from the new person (partial acceptance)",
    )
    trigger_reclustering: bool = Field(
        default=True,
        description="Trigger re-clustering in background after person creation",
    )
```

**Response Schema**: `AcceptUnknownPersonResponse`
```python
class AcceptUnknownPersonResponse(CamelCaseModel):
    """Response from accepting an unknown person candidate."""

    person_id: UUID
    person_name: str
    faces_assigned: int
    faces_excluded: int
    prototypes_created: int
    find_more_job_id: str  # ALWAYS triggered per user decision
    reclustering_job_id: str | None = None  # If trigger_reclustering=True
```

**Behavior** (Updated per user decisions):
1. Validates the group exists and is not already labeled
2. Creates a new `Person` record
3. **Assigns selected faces** (all faces MINUS face_ids_to_exclude) to the new person
4. **Leaves excluded faces unlabeled** (remain in pool for future clustering)
5. Creates prototypes (representative + exemplars from assigned faces only)
6. Syncs to Qdrant (update person_id for assigned faces)
7. **ALWAYS triggers find-more job** (auto_find_more parameter kept for backward compat but defaults to True)
8. **Triggers re-clustering job** if trigger_reclustering=True (default) to refresh candidate groups
9. This is essentially the same as `POST /faces/clusters/{cluster_id}/label` but with:
   - Partial acceptance support (face exclusion)
   - Mandatory find-more trigger
   - Optional re-clustering trigger

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
    membership_hash: str  # Hash of the dismissed group (stable across re-clustering)
    faces_affected: int
    marked_as_noise: bool
```

**Behavior** (Updated per user decisions):
1. **Compute membership hash**: `hash(sorted(face_instance_ids))` to create stable group identity
2. **Store dismissal record** in database (new table or field) keyed by membership_hash
3. If `mark_as_noise=True`:
   - Set `cluster_id = '-1'` on all faces in the group (prevents re-clustering)
   - Record dismissal with noise flag
4. If `mark_as_noise=False`:
   - Keep `cluster_id` unchanged (allows faces to appear in different groups after re-clustering)
   - Record dismissal with membership hash (if same faces cluster together again, group remains dismissed)
5. Dismissed groups are hidden from candidate listing by default (unless `include_dismissed=True`)
6. **Key requirement**: Dismissal persists across re-clustering runs. If the same set of faces clusters together again, the group remains dismissed.

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

### 5.1 Phase 1: MVP Implementation (Updated per User Decisions)

**Goal**: Ship "Suggested New Persons" feature with 50K scale support.

**Key Changes from Original Analysis**:
- ✅ Pre-computed clustering (mandatory, not optional)
- ✅ Dismissed groups database table (new requirement)
- ✅ Membership hash tracking (new requirement)
- ✅ Partial acceptance support (face exclusion list)
- ✅ Auto find-more trigger (always enabled)
- ✅ Re-clustering trigger (optional but default true)
- ✅ Admin settings integration (min_group_size=5, min_confidence=0.70)
- ✅ 50 groups per page, sort by face count descending

#### Implementation Steps:

1. **Database migration** (NEW - required for MVP):
   - Create `dismissed_unknown_person_groups` table
   - Add indexes: membership_hash (unique), cluster_id, dismissed_at
   - Migration: `alembic revision -m "Add dismissed unknown person groups table"`

2. **New route file**: `api/routes/unknown_persons.py`
   - Register router in `main.py` under `/api/v1/faces/unknown-persons`
   - All endpoints from Section 4.2 (updated with user decisions)

3. **New schema file**: `api/schemas/unknown_person_schemas.py`
   - All schemas from Section 4.2 above
   - Follow CamelCaseModel pattern
   - Include membership_hash in responses

4. **New service file**: `services/unknown_person_service.py` (NEW)
   - `compute_membership_hash(face_ids: list[UUID]) -> str`
   - `is_group_dismissed(membership_hash: str) -> bool`
   - `dismiss_group(membership_hash: str, cluster_id: str, face_count: int, reason: str | None, mark_as_noise: bool)`
   - `get_dismissed_groups() -> list[DismissedUnknownPersonGroup]`

5. **Stats endpoint** (`GET /stats`):
   - Query: `SELECT COUNT(*) FROM face_instances WHERE person_id IS NULL`
   - Query: `SELECT COUNT(*) FROM face_instances WHERE person_id IS NULL AND cluster_id IS NOT NULL AND cluster_id != '-1'`
   - Query: `SELECT COUNT(DISTINCT cluster_id) FROM face_instances WHERE person_id IS NULL AND cluster_id IS NOT NULL AND cluster_id != '-1'`
   - Query: `SELECT COUNT(*) FROM dismissed_unknown_person_groups` (NEW)

6. **Candidates listing** (`GET /candidates`):
   - Reuse SQL pattern from `list_clusters()` with filter `include_labeled=False`
   - **Compute membership hash for each group** (for dismissal check)
   - **Filter out dismissed groups** (unless `include_dismissed=True`)
   - Add confidence computation (pre-computed or sampled)
   - **Default sort**: face_count DESC (most faces first)
   - **Default pagination**: 50 groups per page
   - Return grouped response with admin settings (min_group_size, min_confidence)

7. **Accept endpoint** (`POST /candidates/{group_id}/accept`):
   - Validate group exists and not already labeled
   - **NEW**: Handle `face_ids_to_exclude` parameter (partial acceptance)
   - Create new Person
   - Assign faces (minus excluded) to person
   - **Leave excluded faces unlabeled** (remain in pool)
   - Create prototypes (from assigned faces only)
   - Sync to Qdrant
   - **ALWAYS trigger find-more job** (auto_find_more=True by default)
   - **Optionally trigger re-clustering job** (trigger_reclustering=True by default)
   - Return response with both job IDs

8. **Dismiss endpoint** (`POST /candidates/{group_id}/dismiss`):
   - Query all faces in group
   - **Compute membership hash** from face IDs
   - **Store dismissal record** in `dismissed_unknown_person_groups` table
   - If `mark_as_noise=True`: Set `cluster_id = '-1'` on all faces
   - If `mark_as_noise=False`: Keep cluster_id unchanged
   - Return response with membership hash

9. **Discover endpoint** (`POST /discover`):
   - Validate parameters (default: min_cluster_size=5, min_confidence=0.70, max_faces=50K, chunk_size=10K)
   - Enqueue new `discover_unknown_persons_job()` (background job)
   - Return job_id and progress_key for SSE tracking

10. **New background job**: `discover_unknown_persons_job()` in `queue/face_jobs.py`
    - Step 1: Get unassigned face embeddings from Qdrant
    - Step 2: Run HDBSCAN clustering (with chunking support for 50K+ faces)
    - Step 3: Compute cluster quality metrics (confidence, avg quality)
    - Step 4: **Compute membership hash for each cluster**
    - Step 5: Filter by min_cluster_size and min_confidence
    - Step 6: Update `FaceInstance.cluster_id` for all faces
    - Step 7: Report progress via Redis SSE
    - Note: Job may take 10-30 minutes for 50K faces (acceptable for MVP)

11. **Admin settings integration**:
    - Extend `SyncConfigService` or create new settings table
    - Settings: `unknown_person_min_group_size`, `unknown_person_min_confidence`, `unknown_person_max_faces`, `unknown_person_chunk_size`
    - Default values: 5, 0.70, 50000, 10000
    - Expose via admin API endpoints (or reuse existing config endpoints)

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

#### Phase 1: Minimal schema changes (MVP)

**Required for user decisions**:
- New table: `dismissed_unknown_person_groups` (track dismissed groups by membership hash)
- Reuse existing `FaceInstance.cluster_id` and `FaceInstance.person_id` (no changes)

```python
class DismissedUnknownPersonGroup(Base):
    """Tracks dismissed candidate person groups by membership hash.

    Key requirement: Dismissed groups persist across re-clustering runs.
    If the same set of faces clusters together again, the group remains dismissed.
    """
    __tablename__ = "dismissed_unknown_person_groups"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    membership_hash: Mapped[str] = mapped_column(String(64), unique=True, index=True)  # SHA256 of sorted face_instance_ids
    cluster_id: Mapped[str | None] = mapped_column(String(50), nullable=True, index=True)  # Current cluster_id (may change across runs)
    face_count: Mapped[int] = mapped_column(Integer)  # Number of faces when dismissed
    reason: Mapped[str | None] = mapped_column(Text, nullable=True)  # Optional dismissal reason
    marked_as_noise: Mapped[bool] = mapped_column(Boolean, default=False)  # True if faces marked as noise
    dismissed_at: Mapped[datetime] = mapped_column(server_default=func.now())
    dismissed_by: Mapped[str | None] = mapped_column(String(255), nullable=True)  # Future: user identification

    # Optional: Store face IDs for debugging (JSON array)
    face_instance_ids: Mapped[list[str] | None] = mapped_column(JSON, nullable=True)
```

**Membership Hash Computation**:
```python
import hashlib
import json

def compute_membership_hash(face_instance_ids: list[uuid.UUID]) -> str:
    """Compute stable hash of face instance IDs.

    Same set of faces always produces same hash, regardless of order or cluster_id.
    """
    sorted_ids = sorted(str(fid) for fid in face_instance_ids)
    hash_input = json.dumps(sorted_ids, sort_keys=True)
    return hashlib.sha256(hash_input.encode()).hexdigest()
```

#### Phase 2: Enhanced metadata tables (optional, post-MVP)

```python
class UnknownPersonDiscovery(Base):
    """Tracks discovery job results and metadata."""
    __tablename__ = "unknown_person_discoveries"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    job_id: Mapped[str] = mapped_column(String(36), index=True)
    status: Mapped[str] = mapped_column(String(20))  # completed/failed
    params: Mapped[dict] = mapped_column(JSON)  # Clustering parameters used
    total_faces_processed: Mapped[int] = mapped_column(Integer)
    clusters_found: Mapped[int] = mapped_column(Integer)
    noise_count: Mapped[int] = mapped_column(Integer)
    execution_time_seconds: Mapped[float | None] = mapped_column(Float, nullable=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

class ClusterMetadata(Base):
    """Pre-computed metadata for face clusters."""
    __tablename__ = "cluster_metadata"

    cluster_id: Mapped[str] = mapped_column(String(50), primary_key=True)
    membership_hash: Mapped[str] = mapped_column(String(64), unique=True, index=True)  # Stable identity
    face_count: Mapped[int] = mapped_column(Integer)
    cluster_confidence: Mapped[float] = mapped_column(Float)
    avg_quality: Mapped[float] = mapped_column(Float)
    representative_face_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("face_instances.id"))
    centroid_vector_id: Mapped[uuid.UUID | None] = mapped_column(nullable=True)
    discovery_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("unknown_person_discoveries.id"))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
```

**Admin Settings Integration** (extends existing `SyncConfigService` or new table):
```python
# Add to existing config service or new settings table
{
    "unknown_person_min_group_size": 5,  # Default: 5 faces
    "unknown_person_min_confidence": 0.70,  # Default: 0.70
    "unknown_person_auto_trigger_discovery": False,  # Default: manual trigger
    "unknown_person_max_faces": 50000,  # Default: 50K
    "unknown_person_chunk_size": 10000,  # Default: 10K
}
```

---

## 6. Performance Analysis

**⚠️ CRITICAL SCALE UPDATE**: User decision mandates **50,000 unlabeled faces** as the expected production scale, not 5K-10K as originally analyzed. This changes all performance requirements.

### 6.1 Scale Considerations (Updated for 50K Target)

| Metric | Original Analysis | **User Decision** | Impact |
|---|---|---|---|---|
| Total faces in Qdrant | 5,000-20,000 | 50,000-100,000 | Scroll + filter time significantly longer |
| **Unassigned faces** | 500-5,000 | **50,000** | **HDBSCAN O(N²) = 10-30 minutes with standard approach** |
| Existing clusters | 50-200 | 500-1,000 | Listing query time manageable with indexes |
| Faces per cluster | 3-50 | 5-100 | Confidence computation time (sampling already handles this) |
| **Min group size** | 2-3 | **5 (default)** | Fewer groups to display, better quality |
| **Groups per page** | 10 | **50** | More data per response, requires efficient pagination |

### 6.2 Computational Cost Analysis (Revised for 50K Scale)

#### Retrieving Unassigned Faces from Qdrant

**Current approach** (`get_unlabeled_faces_with_embeddings()`):
- Scrolls ALL faces (100 per page) and filters in Python for missing `person_id`
- **For 50,000 unassigned faces**: ~500 scroll requests, ~25-50 seconds
- **For 100,000 total faces with 50,000 unassigned**: 1,000 scroll requests, ~50-100 seconds

**Optimization opportunity** (CRITICAL for 50K scale):
- Add a Qdrant payload index on `person_id` and use `must_not` filter with a sentinel value (e.g., store `"unassigned"` instead of omitting the field)
- Or maintain a separate "unassigned faces" collection (write complexity vs read performance tradeoff)
- **Recommendation**: Implement Qdrant filter optimization BEFORE 50K deployment

#### HDBSCAN Clustering (CRITICAL BOTTLENECK)

- **Time complexity**: O(N²) for distance matrix computation where N = number of faces
- **For 5,000 faces**: ~2-5 seconds (original analysis - ACCEPTABLE)
- **⚠️ For 50,000 faces**: ~200-500 seconds (3-8 minutes) BEST CASE, potentially 10-30 minutes
- **Memory**: O(N²) for distance matrix -- 5,000 faces = ~100MB, **50,000 faces = ~10GB** (may exceed worker memory limits)

**MANDATORY MITIGATION STRATEGIES for 50K scale**:

| Strategy | Pros | Cons | Feasibility |
|---|---|---|---|
| **Chunked/Batched HDBSCAN** | Process N faces at a time (e.g., 10K chunks), merge clusters | Reduced memory, parallelizable | Medium complexity, best ROI |
| **Hierarchical Clustering** | Cluster sub-groups first, merge hierarchically | Handles large N efficiently | High complexity |
| **Approximate Clustering** | Sample subset (e.g., 20K faces) for clustering | Fast, low memory | May miss small clusters |
| **Accept Long Job Times** | Simple implementation, no algorithm changes | 10-30 minute background jobs | Low complexity, acceptable for MVP |
| **Two-tier Discovery** | Fast clustering on 10K sample, full clustering on-demand | Best UX + performance balance | Medium complexity |

**Recommended Approach for MVP**:
1. **Accept 10-30 minute job execution times** for initial MVP (background job is async, users can continue working)
2. **Implement chunked HDBSCAN** in Phase 2 if job times become problematic
3. **Monitor job execution times** and adjust strategy based on real-world data

**Implementation Note**: The `chunk_size` parameter in `DiscoverUnknownPersonsRequest` (default 10K) enables chunked processing from the start.

#### Cluster Confidence Computation

- `calculate_cluster_confidence()` uses pairwise cosine similarity
- Already samples max 20 faces for large clusters (existing optimization)
- For 200 clusters of average size 25: ~200 * 190 comparisons = negligible

### 6.3 Pre-compute vs On-demand (RESOLVED)

**User Decision**: **Pre-computed clustering via background job** (MANDATORY, not optional)

| Approach | User Decision | Rationale |
|---|---|---|
| **Pre-compute (background job)** | ✅ **SELECTED** | 50K faces = 10-30 minute clustering time. MUST be background job. |
| **On-demand (API-triggered)** | ❌ **REJECTED** | Too slow for 50K scale. Users cannot wait 10-30 minutes for results. |
| **Hybrid** | Partial (refresh button only) | Pre-compute is primary mode, "refresh" triggers new background job |

**Implementation**:
- Clustering ALWAYS runs as background job
- "Discover" button enqueues job, UI shows progress via SSE
- "Refresh" button re-triggers discovery job
- UI auto-refreshes when job completes (SSE notification or polling)

### 6.4 Caching Strategy

**Updated for 50K scale and membership hash requirement**:

1. **Redis cache for cluster metadata**: Cache cluster stats (face_count, confidence, representative_face, **membership_hash**) with TTL of 1 hour
2. **Membership hash computation**: Compute once during discovery job, store in cluster metadata table
3. **Dismissed groups cache**: Store dismissed membership hashes in Redis for fast filtering (avoid DB query on every listing)
4. **Invalidation triggers**:
   - New face detection (trigger re-clustering job)
   - Face assignment changes (person created, faces assigned)
   - Cluster dismissal (update dismissed cache)
5. **Progress tracking**: Reuse existing Redis progress key pattern from find-more jobs

### 6.5 Response Time Targets (Updated for 50K Scale)

| Endpoint | Target | Strategy | 50K Scale Impact |
|---|---|---|---|
| `GET /candidates` | <500ms | SQL query + pre-computed confidence + membership hash | Group count increased (50/page), dismissed filter added |
| `GET /candidates/{id}` | <1s | Direct cluster query (may return 50-100 faces) | Larger groups = more data |
| `POST /discover` | Immediate (async) | Returns job_id, work happens in background | Job execution time: 10-30 minutes for 50K faces |
| `POST /candidates/{id}/accept` | <3s | Transaction: create person + assign faces + trigger find-more + trigger re-clustering | Additional jobs enqueued (find-more + re-cluster) |
| `POST /candidates/{id}/dismiss` | <500ms | Store dismissal record with membership hash | Simple DB write |
| `GET /stats` | <200ms | Cached aggregates | No change |

**Key Performance Note**: Job execution time (10-30 minutes for 50K faces) is acceptable because:
1. Job is fully asynchronous (users can browse other features)
2. Progress visible via SSE (users know job is working)
3. Auto-refresh on completion (users notified when results ready)
4. Pre-computed results remain cached until next discovery run

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

### 8.1 Implementation Priority (Updated per User Decisions)

**⚠️ Priority changes based on user decisions**: Dismissal storage and membership hash tracking moved to P0 (required for MVP).

| Priority | Item | Effort | Impact | User Decision Impact |
|---|---|---|---|---|
| **P0 (MVP Blockers)** | | | | |
| P0 | Database migration: `dismissed_unknown_person_groups` table | Small | High | **NEW REQUIREMENT** - dismissal persistence |
| P0 | Membership hash computation and storage | Small | High | **NEW REQUIREMENT** - stable group identity |
| P0 | `POST /discover` (trigger discovery job) | Medium | High | **MANDATORY** - pre-computed clustering only |
| P0 | `discover_unknown_persons_job()` with 50K scale support | Large | High | **CRITICAL** - chunked processing for 50K faces |
| P0 | `GET /candidates` (listing with dismissal filter) | Medium | High | **UPDATED** - filter dismissed groups, 50/page, sort by face_count |
| P0 | `GET /candidates/{id}` (detail view) | Small | High | Essential for UX |
| P0 | `POST /candidates/{id}/accept` with partial acceptance | Medium | High | **UPDATED** - face exclusion list, auto find-more, re-clustering trigger |
| P0 | `POST /candidates/{id}/dismiss` with hash storage | Medium | High | **UPDATED** - store to DB, membership hash tracking |
| P0 | Admin settings integration (min_group_size, min_confidence) | Small | Medium | **NEW REQUIREMENT** - configurable defaults |
| **P1 (MVP Nice-to-Have)** | | | | |
| P1 | `GET /stats` with dismissed groups count | Small | Medium | Dashboard insight + dismissed group visibility |
| P1 | SSE progress tracking for discovery job | Small | High | Reuse existing pattern, 10-30 min jobs need progress |
| P1 | Redis cache for dismissed groups (fast filtering) | Small | Medium | Performance optimization for listing |
| **P2 (Post-MVP Enhancements)** | | | | |
| P2 | Chunked HDBSCAN (if MVP job times problematic) | Large | High | Performance optimization for 50K+ scale |
| P2 | Qdrant filter optimization (sentinel value for person_id) | Medium | High | Performance optimization for retrieval |
| P2 | Pre-computed cluster metadata table | Medium | Medium | Performance optimization for listing |
| P2 | Similar person matching (centroid comparison) | Medium | Medium | Helps identify duplicates |
| **P3 (Future)** | | | | |
| P3 | Incremental clustering | Large | High | Scalability for 100K+ faces |
| P3 | Active learning from user feedback | Large | Medium | Improves accuracy over time |
| P3 | Two-tier discovery (sample + full clustering) | Large | High | UX + performance balance |

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

New settings for `core/config.py` (updated for user decisions):

```python
# Unknown person discovery settings
unknown_person_discovery_auto_trigger: bool = Field(
    default=False,
    description="Auto-trigger discovery after face detection sessions",
)
unknown_person_min_cluster_size: int = Field(
    default=5, ge=2, le=50,
    description="Minimum faces per candidate person group (default: 5 per user decision)",
)
unknown_person_min_confidence: float = Field(
    default=0.70, ge=0.0, le=1.0,
    description="Minimum cluster confidence for candidate groups (default: 0.70 per user decision)",
)
unknown_person_max_faces: int = Field(
    default=50000, ge=100, le=100000,
    description="Maximum unassigned faces to process during discovery (default: 50K per user decision)",
)
unknown_person_chunk_size: int = Field(
    default=10000, ge=1000, le=20000,
    description="Chunk size for batched clustering to handle 50K+ faces",
)
unknown_person_groups_per_page: int = Field(
    default=50, ge=1, le=100,
    description="Default groups per page in candidate listing (default: 50 per user decision)",
)
```

### 8.7 Scale Concerns and Mitigation Plan

**⚠️ CRITICAL CONCERNS** arising from 50K scale requirement:

| Concern | Original Analysis | User Requirement | Gap | Mitigation |
|---|---|---|---|---|
| **HDBSCAN Performance** | Analyzed for 5K-10K faces (2-5s) | **50K faces expected** | 10-30 minute execution time | Accept long job time for MVP, implement chunked clustering in Phase 2 |
| **Memory Usage** | 5K faces = ~100MB | 50K faces = ~10GB | May exceed worker memory limits | Monitor memory usage, implement chunked processing if needed |
| **Qdrant Scroll Performance** | Acceptable for 20K total faces | 50K-100K total faces expected | 50-100 second retrieval time | Implement Qdrant filter optimization (sentinel value for person_id) |
| **Group Count** | 50-200 groups expected | Potentially 500-1,000+ groups | Pagination performance, UI scrolling | Implemented: 50 groups/page, sort by face count (most faces first) |
| **Dismissed Groups Storage** | Not originally analyzed | Must persist across re-clustering | Membership hash computation cost | Compute once during discovery, cache in Redis for fast filtering |
| **Re-clustering Trigger** | Not originally analyzed | Trigger after person creation | Multiple long-running jobs in sequence | Queue jobs with appropriate priority, show job status in UI |
| **Partial Acceptance** | Full cluster labeling only | Accept subset of faces from group | Remaining faces stay in pool | Modified accept flow: exclude selected faces, leave unlabeled |

**Recommended Pre-Deployment Testing** for 50K scale:
1. **Load test HDBSCAN** with 50K synthetic embeddings to measure actual execution time and memory usage
2. **Benchmark Qdrant scroll** with 100K faces to validate retrieval time estimates
3. **Test chunked clustering** implementation with various chunk sizes (5K, 10K, 20K)
4. **Validate membership hash** stability across re-clustering runs (same faces = same hash)
5. **Monitor Redis memory** for dismissed groups cache with 1,000+ dismissed groups
6. **Test re-clustering trigger** after person creation (ensure job doesn't block accept flow)

**Phase 2 Optimization Priorities** (if MVP performance insufficient):
1. **P0**: Chunked HDBSCAN processing (10K chunks with hierarchical merging)
2. **P0**: Qdrant filter optimization (sentinel value for unassigned faces)
3. **P1**: Redis cache for cluster metadata (reduce DB queries on listing)
4. **P1**: Background job priority tuning (prevent re-clustering from blocking find-more)
5. **P2**: Incremental clustering (only re-cluster newly detected faces)
6. **P2**: Approximate clustering (sample-based for initial discovery, full clustering on-demand)

---

## 9. Summary of Changes from Original Analysis

**Document Update**: 2026-02-11
**Reason**: User decisions finalized, major scale requirement change

### 9.1 Critical Changes

| Area | Original Analysis | User Decision | Impact |
|---|---|---|---|
| **Scale** | 5K-10K unlabeled faces | **50K unlabeled faces** | HDBSCAN execution time: 2-5s → 10-30 min. Mandatory chunked processing. |
| **Processing Mode** | Optional pre-compute vs on-demand | **Pre-computed only (mandatory)** | Background job is the only mode. No on-demand clustering. |
| **Dismissal Storage** | Optional analytics | **Database storage (required)** | New table: `dismissed_unknown_person_groups` with membership hash tracking. |
| **Group Identity** | Cluster ID (unstable) | **Membership hash (stable)** | Dismissals persist across re-clustering runs. Same faces = same group. |
| **Acceptance Mode** | Full cluster labeling | **Partial acceptance (subset)** | Accept endpoint handles face exclusion list. Remaining faces stay unlabeled. |
| **Find-More** | Optional trigger | **Always trigger (mandatory)** | Auto find-more after person creation is always enabled. |
| **Re-clustering** | Not analyzed | **Trigger after person creation** | New requirement: re-cluster background job after accept. |
| **Min Group Size** | 2-3 faces (default) | **5 faces (default)** | Fewer groups, higher quality. Configurable via admin settings. |
| **Pagination** | 10 groups/page | **50 groups/page** | More data per response, sort by face count descending. |
| **Default Confidence** | 0.65 | **0.70** | Higher quality threshold. |

### 9.2 New Requirements

1. **Database migration**: Create `dismissed_unknown_person_groups` table
2. **Membership hash service**: Compute and store stable group identities
3. **Chunked clustering**: Support 50K+ faces with configurable chunk size (default 10K)
4. **Admin settings integration**: Expose min_group_size, min_confidence, max_faces, chunk_size
5. **Partial acceptance logic**: Accept subset of faces from group, leave remainder unlabeled
6. **Re-clustering trigger**: Auto-trigger re-clustering after person creation
7. **Dismissed groups filter**: Hide dismissed groups from listing by default
8. **Performance monitoring**: Track job execution time for 50K scale optimization

### 9.3 MVP Scope ("Suggested New Persons")

**In Scope**:
- Pre-computed clustering via background job (10-30 min execution time acceptable)
- List candidate groups with pagination (50/page, sort by face count)
- Accept group as new person (with partial acceptance support)
- Dismiss group (with membership hash tracking and DB storage)
- Stats endpoint (with dismissed groups count)
- Admin settings for min_group_size (default 5) and min_confidence (default 0.70)
- Auto find-more after person creation (always enabled)
- Auto re-clustering after person creation (optional, default enabled)

**Out of Scope** (Post-MVP):
- Chunked HDBSCAN optimization (accept long job times for MVP)
- Qdrant filter optimization (accept scroll performance for MVP)
- Similar person matching (centroid comparison)
- Incremental clustering
- Active learning from user feedback
- Two-tier discovery (sample + full clustering)

### 9.4 Risk Mitigation Plan

| Risk | Likelihood | Mitigation |
|---|---|---|
| HDBSCAN too slow for 50K faces | Medium | Accept 10-30 min job time for MVP; implement chunked HDBSCAN in Phase 2 if needed |
| Worker memory exceeded (10GB for 50K faces) | Medium | Monitor memory usage; implement chunked processing if exceeded |
| Qdrant scroll too slow (50-100s for 50K faces) | Medium | Accept scroll time for MVP; implement filter optimization in Phase 2 |
| User confusion with dismissed groups | Low | Clear UX messaging; show dismissed count in stats |
| Re-clustering job blocks find-more | Low | Use job priority queues; show job status in UI |
| Membership hash collisions | Very Low | Use SHA256 (64 hex chars); collision probability negligible |

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
