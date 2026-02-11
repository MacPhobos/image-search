# Database & Qdrant Research: Unknown Person Face Detection

**Research Agent**: Database & Qdrant Research (Agent 3 of 5)
**Date**: 2026-02-07
**Status**: Complete
**Scope**: PostgreSQL data models, Qdrant architecture, query capabilities, and strategy options for discovering and grouping unassigned faces by visual similarity

---

## Executive Summary

This research document analyzes the existing database and Qdrant vector store architecture to inform the implementation of the "Unknown Person Face Detection" feature -- a capability to discover UNASSIGNED faces (those not yet linked to any person) and group them by visual similarity with an adjustable threshold.

**Key Findings:**

1. **PostgreSQL schema is well-suited** for the feature. The `FaceInstance` table already has nullable `person_id` and `cluster_id` fields with proper indexes. Unassigned faces are identified by `person_id IS NULL`.

2. **Qdrant lacks native "IS NULL" filtering**, which is the primary technical challenge. The existing codebase already works around this by scrolling all records and filtering in Python (see `get_unlabeled_faces_with_embeddings()`).

3. **Four viable query strategies** exist, ranging from pure Python-side clustering (using existing HDBSCAN/DBSCAN infrastructure) to Qdrant-native grouping (limited to payload field grouping, not embedding-based clustering).

4. **The recommended approach is Option A (Qdrant Scroll + Python Clustering)** -- extending the existing dual-mode clustering pattern with on-demand, threshold-adjustable grouping and an optional pre-computation layer for performance at scale.

5. **Existing infrastructure provides 80%+ of needed capabilities.** The dual-mode clusterer, face embedding retrieval, centroid computation, and cluster confidence calculation are all implemented and tested.

---

## 1. Current Data Model Analysis

### 1.1 Face-Related Tables

#### FaceInstance (`face_instances`)

The core table for face data. Located at `/export/workspace/image-search/image-search-service/src/image_search_service/db/models.py`, lines 459-528.

```
Table: face_instances
----------------------------------------------
Column                  Type            Nullable    Notes
----------------------------------------------
id                      UUID (PK)       No          Primary key, auto-generated
asset_id                Integer (FK)    No          -> image_assets.id, CASCADE delete
bbox_x                  Integer         No          Bounding box coordinates
bbox_y                  Integer         No
bbox_w                  Integer         No
bbox_h                  Integer         No
landmarks               JSONB           Yes         5-point facial landmarks
detection_confidence    Float           No          Detection quality (0-1)
quality_score           Float           Yes         Overall quality metric (0-1)
qdrant_point_id         UUID            No          Unique, maps to Qdrant face point
cluster_id              String(100)     Yes         Cluster assignment (e.g., "unknown_cluster_0")
person_id               UUID (FK)       Yes         -> persons.id, SET NULL on delete
created_at              DateTime(tz)    No          Server-generated
updated_at              DateTime(tz)    No          Auto-updated
```

**Critical fields for Unknown Person Detection:**
- `person_id IS NULL` = face is UNASSIGNED (the target population)
- `cluster_id` = existing cluster assignment from batch clustering jobs
- `qdrant_point_id` = link to 512-dim ArcFace embedding in Qdrant
- `quality_score` = useful for filtering low-quality faces from grouping

**Indexes:**
- `ix_face_instances_person_id` -- efficient filtering by person assignment
- `ix_face_instances_cluster_id` -- efficient filtering by cluster
- `ix_face_instances_quality` -- efficient quality-based filtering
- `ix_face_instances_asset_id` -- efficient asset lookups
- `uq_face_instance_location` -- unique constraint on (asset_id, bbox_x, bbox_y, bbox_w, bbox_h)

**Unique constraints:**
- `(asset_id, bbox_x, bbox_y, bbox_w, bbox_h)` prevents duplicate detections at the same location

#### Person (`persons`)

Located at lines 411-456 of models.py.

```
Table: persons
----------------------------------------------
Column          Type            Nullable    Notes
----------------------------------------------
id              UUID (PK)       No          Primary key
name            String(255)     No          Person name (unique, case-insensitive)
status          PersonStatus    No          active / merged / hidden
merged_into_id  UUID (FK)       Yes         Self-referencing for merge operations
birth_date      Date            Yes         For age-era prototype selection
created_at      DateTime(tz)    No
updated_at      DateTime(tz)    No
```

**Relationships to FaceInstance:**
- `Person.face_instances` (one-to-many) -- all faces assigned to this person
- `FaceInstance.person` (many-to-one) -- the assigned person (nullable)

#### PersonCentroid (`person_centroid`)

Located at lines 856-953 of models.py.

```
Table: person_centroid
----------------------------------------------
Column              Type            Nullable    Notes
----------------------------------------------
centroid_id         UUID (PK)       No          Primary key
person_id           UUID (FK)       No          -> persons.id, CASCADE
qdrant_point_id     UUID            No          Unique, maps to Qdrant centroid point
model_version       String(64)      No          e.g., "arcface_r100_glint360k_v1"
centroid_version    Integer         No          Algorithm version (default: 2)
centroid_type       CentroidType    No          global / cluster
cluster_label       String(32)      Yes         "global", "k2_0", etc.
n_faces             Integer         No          Number of source faces
status              CentroidStatus  No          active / deprecated / building / failed
source_face_ids_hash String(64)     Yes         For staleness detection
build_params        JSONB           Yes         Algorithm parameters
created_at          DateTime(tz)    No
updated_at          DateTime(tz)    No
```

#### FaceSuggestion (`face_suggestions`)

Located at lines 734-802 of models.py. Stores face-to-person match suggestions.

```
Table: face_suggestions
----------------------------------------------
Column                  Type        Nullable    Notes
----------------------------------------------
id                      Integer     No          PK, auto-increment
face_instance_id        UUID (FK)   No          -> face_instances.id
suggested_person_id     UUID (FK)   No          -> persons.id
confidence              Float       No          Cosine similarity score
source_face_id          UUID (FK)   Yes         Reference face that triggered suggestion
status                  String(20)  No          pending / accepted / rejected / expired
matching_prototype_ids  JSONB       Yes         Multi-prototype match tracking
prototype_scores        JSONB       Yes         Per-prototype similarity scores
aggregate_confidence    Float       Yes         Aggregated score across prototypes
prototype_match_count   Integer     Yes         Number of matching prototypes
created_at              DateTime    No
reviewed_at             DateTime    Yes
```

### 1.2 Entity-Relationship Summary

```
ImageAsset (1) ---> (N) FaceInstance
                          |
                          |-- person_id (nullable) --> Person (0..1)
                          |-- cluster_id (nullable)    (string label)
                          |-- qdrant_point_id -------> Qdrant "faces" collection

Person (1) ---> (N) FaceInstance
Person (1) ---> (N) PersonPrototype ---> Qdrant "faces" collection (is_prototype=true)
Person (1) ---> (N) PersonCentroid  ---> Qdrant "person_centroids" collection
Person (1) ---> (N) FaceSuggestion
```

### 1.3 SQL Query Patterns for Unassigned Faces

**Count unassigned faces:**
```sql
SELECT COUNT(*) FROM face_instances WHERE person_id IS NULL;
```

**Get unassigned faces with quality filter:**
```sql
SELECT id, asset_id, qdrant_point_id, quality_score, detection_confidence
FROM face_instances
WHERE person_id IS NULL
  AND quality_score >= 0.5
ORDER BY quality_score DESC
LIMIT 1000;
```

**Get unassigned faces excluding already-clustered:**
```sql
SELECT id, qdrant_point_id
FROM face_instances
WHERE person_id IS NULL
  AND cluster_id IS NULL
ORDER BY created_at DESC;
```

**Existing code pattern** (from `faces/assigner.py`, line 97):
```python
query = select(FaceInstance).where(
    FaceInstance.person_id.is_(None),
    FaceInstance.cluster_id.is_(None),
)
```

---

## 2. Current Qdrant Architecture

### 2.1 Collections Overview

The system manages 4 Qdrant collections, bootstrapped via `/export/workspace/image-search/image-search-service/src/image_search_service/scripts/bootstrap_qdrant.py`:

| Collection | Vector Dim | Distance | Purpose | Client Module |
|---|---|---|---|---|
| `image_assets` | 512 (CLIP) | Cosine | Semantic image search | `vector/qdrant.py` |
| `faces` | 512 (ArcFace) | Cosine | Face embeddings + person assignment | `vector/face_qdrant.py` |
| `person_centroids` | 512 (ArcFace) | Cosine | Person average embeddings | `vector/centroid_qdrant.py` |
| `image_assets_siglip` | 768 (SigLIP) | Cosine | Enhanced image search (optional) | `vector/qdrant.py` |

### 2.2 Faces Collection -- Detailed Schema

**Collection name**: Configurable via `QDRANT_FACE_COLLECTION` env var (default: `"faces"`)
**Vector dimension**: 512 (ArcFace/InsightFace)
**Distance metric**: Cosine
**Client library**: `qdrant-client>=1.12.0`

**Payload schema (per point):**
```json
{
  "asset_id": "string (UUID)",
  "face_instance_id": "string (UUID)",
  "person_id": "string (UUID) | ABSENT",
  "cluster_id": "string | ABSENT",
  "detection_confidence": 0.95,
  "quality_score": 0.87,
  "taken_at": "2024-01-15T12:00:00Z | ABSENT",
  "bbox": {"x": 100, "y": 200, "w": 50, "h": 50},
  "is_prototype": false
}
```

**CRITICAL DESIGN DECISION**: `person_id` is ABSENT (key deleted from payload) for unassigned faces, NOT set to null or empty string. This means:
- Qdrant has no `IS NULL` filter concept
- Checking `person_id` absence requires scrolling and Python-side filtering
- The existing `get_unlabeled_faces_with_embeddings()` method at `face_qdrant.py:579` implements this workaround

**Payload Indexes (5 total):**
| Field | Index Type | Purpose |
|---|---|---|
| `person_id` | KEYWORD | Filter faces by assigned person |
| `cluster_id` | KEYWORD | Filter by cluster assignment |
| `is_prototype` | BOOL | Filter prototype faces for matching |
| `asset_id` | KEYWORD | Filter faces by source image |
| `face_instance_id` | KEYWORD | Direct face lookup |

### 2.3 Person Centroids Collection

**Collection name**: `person_centroids` (configurable via `QDRANT_CENTROID_COLLECTION`)
**Vector dimension**: 512
**Distance metric**: Cosine

**Payload schema:**
```json
{
  "person_id": "string (UUID)",
  "centroid_id": "string (UUID)",
  "model_version": "arcface_r100_glint360k_v1",
  "centroid_version": 2,
  "centroid_type": "global",
  "cluster_label": "global",
  "n_faces": 150,
  "created_at": "2026-01-15T10:00:00Z",
  "source_hash": "abc123...",
  "build_params": {}
}
```

**Payload Indexes (5 total):** person_id (KEYWORD), centroid_id (KEYWORD), model_version (KEYWORD), centroid_version (INTEGER), centroid_type (KEYWORD)

### 2.4 Docker Infrastructure

From `/export/workspace/image-search/image-search-service/docker-compose.dev.yml`:
- **Qdrant image**: `qdrant/qdrant:latest`
- **HTTP port**: 6333
- **gRPC port**: 6334
- **Persistent volume**: `qdrant_data`
- **No replication** (single-node development setup)

---

## 3. Qdrant Query Capabilities for Face Grouping

### 3.1 Relevant Qdrant APIs

#### query_points (Primary Search)
- Searches by vector similarity with optional payload filtering
- Supports `score_threshold` for minimum similarity
- Supports `must` / `must_not` filter conditions
- Used extensively in existing codebase (search_similar_faces, search_against_prototypes, search_faces_with_centroid)

#### query_points_groups (Native Grouping)
- Groups search results by a payload field value
- Parameters: `group_by` (field path), `limit` (max groups), `group_size` (max points per group)
- **LIMITATION**: Only supports `keyword` and `integer` payload types for grouping
- **LIMITATION**: Does NOT support grouping by vector similarity / clustering
- **LIMITATION**: No `offset` pagination support
- **USE CASE**: Could group by existing `cluster_id` but CANNOT discover NEW similarity groups

#### scroll (Paginated Retrieval)
- Scrolls through all points with optional filters
- Supports `with_vectors=True` to retrieve embeddings
- Used in existing code for `get_unlabeled_faces_with_embeddings()`
- Returns (records, next_offset) for pagination

#### Recommend API
- Finds similar points using positive/negative examples (point IDs)
- Could find faces similar to a "seed face" without needing the actual vector
- Useful for interactive "find more like this" workflows

### 3.2 Critical Limitation: No Native "IS NULL" Filter

Qdrant does not support filtering for ABSENT payload keys. The existing codebase works around this:

**Current workaround** (`face_qdrant.py`, lines 579-652):
```python
def get_unlabeled_faces_with_embeddings(self, quality_threshold=0.5, limit=10000):
    """Scroll ALL faces, filter in Python for missing person_id."""
    face_embeddings = []
    offset = None
    while len(face_embeddings) < limit:
        records, next_offset = self.client.scroll(
            collection_name=_get_face_collection_name(),
            limit=100, offset=offset,
            with_vectors=True, with_payload=True,
        )
        for record in records:
            if "person_id" in record.payload:
                continue  # Skip assigned faces
            # ... quality check, extract embedding
            face_embeddings.append((face_instance_id, embedding))
        if next_offset is None:
            break
        offset = next_offset
    return face_embeddings[:limit]
```

**Performance implication**: Must scroll through ALL points (assigned + unassigned) to find those without `person_id`. For 10K+ faces, this creates O(N) scan overhead.

**Alternative approaches to consider:**
1. **Sentinel value**: Store `person_id: "UNASSIGNED"` instead of deleting the key. Enables Qdrant filter `person_id == "UNASSIGNED"` with O(1) index lookup.
2. **Boolean flag**: Add `is_assigned: false` payload field with BOOL index. Enables direct filtering.
3. **Separate collection**: Maintain a separate `unassigned_faces` collection. More complex but fastest reads.

### 3.3 Existing Qdrant Query Patterns in Codebase

| Pattern | Location | Description |
|---|---|---|
| Centroid-based face search | `centroid_qdrant.py:281-338` | Uses centroid vector to query faces collection with `must_not` person_id filter |
| Prototype-based matching | `face_qdrant.py:458-479` | Searches faces with `is_prototype=True` filter |
| Similar face search | `face_qdrant.py:398-456` | General similarity search with optional person_id, cluster_id, is_prototype filters |
| Unlabeled face retrieval | `face_qdrant.py:579-652` | Full scan with Python-side null filtering |
| Person_id update | `face_qdrant.py:365-396` | Sets or deletes person_id payload on face points |
| Cluster_id update | inferred from `dual_clusterer.py:336` | `qdrant.update_cluster_ids(point_ids, cluster_id)` |

---

## 4. Query Strategy Options

### Option A: Qdrant Scroll + Python-Side Clustering (Recommended)

**Approach**: Extend the existing `get_unlabeled_faces_with_embeddings()` + `DualModeClusterer` pattern to support on-demand, threshold-adjustable grouping.

**Data flow:**
```
1. PostgreSQL: SELECT face IDs WHERE person_id IS NULL [+ quality filter]
2. Qdrant scroll: Retrieve embeddings for those face IDs (batch by 100)
3. Python: Run HDBSCAN/DBSCAN/Agglomerative with user-adjustable threshold
4. Python: Calculate cluster confidence (avg pairwise similarity)
5. Return grouped results (optionally persist to cluster_id)
```

**Implementation:**
```python
# Step 1: Get unassigned face IDs from PostgreSQL
faces = select(FaceInstance).where(
    FaceInstance.person_id.is_(None),
    FaceInstance.quality_score >= min_quality,
).limit(max_faces)

# Step 2: Batch-retrieve embeddings from Qdrant
embeddings = []
for batch in chunk(faces, 100):
    points = qdrant.client.retrieve(
        collection_name="faces",
        ids=[str(f.qdrant_point_id) for f in batch],
        with_vectors=True,
    )
    embeddings.extend(...)

# Step 3: Cluster with adjustable threshold
clusterer = HDBSCAN(
    min_cluster_size=user_min_group_size,
    metric="precomputed",
)
# OR
clusterer = AgglomerativeClustering(
    distance_threshold=1.0 - user_similarity_threshold,  # Convert cosine sim to distance
    metric="cosine",
)
labels = clusterer.fit_predict(embeddings)

# Step 4: Calculate confidence per group
for cluster_id in unique_labels:
    cluster_faces = embeddings[labels == cluster_id]
    confidence = avg_pairwise_cosine_similarity(cluster_faces)
```

**Pros:**
- Leverages existing infrastructure (DualModeClusterer, HDBSCAN, confidence calculation)
- Full control over clustering algorithm and parameters
- Adjustable threshold (eps, min_cluster_size) per request
- No Qdrant API limitations to work around
- Already tested in production with the dual-mode clustering pipeline

**Cons:**
- Must load all unassigned embeddings into Python memory
- O(N^2) pairwise distance computation for clustering
- Network overhead: scroll/retrieve all embeddings from Qdrant
- Cold-start latency for first request (seconds for 1K+ faces)

**Performance estimates:**
| Face Count | Embedding Retrieval | Clustering | Total |
|---|---|---|---|
| 500 | ~2s | ~0.5s | ~2.5s |
| 2,000 | ~8s | ~3s | ~11s |
| 5,000 | ~20s | ~15s | ~35s |
| 10,000 | ~40s | ~60s | ~100s |

### Option B: Qdrant query_points_groups (Native Grouping by Pre-computed Cluster)

**Approach**: Pre-compute clusters (via batch job), store `cluster_id` in Qdrant payload, then use `query_points_groups` to retrieve grouped results.

**Data flow:**
```
1. Background job: Run clustering on unassigned faces (same as Option A step 1-3)
2. Persist: Store cluster_id in both PostgreSQL and Qdrant payload
3. Query time: Use Qdrant query_points_groups(group_by="cluster_id")
4. Return pre-computed groups
```

**Implementation at query time:**
```python
# Use Qdrant native grouping (fast at query time)
results = qdrant.client.query_points_groups(
    collection_name="faces",
    group_by="cluster_id",
    limit=20,           # Max groups to return
    group_size=10,       # Max faces per group
    query_filter=Filter(
        must_not=[
            FieldCondition(key="person_id", match=MatchValue(value="*"))  # Exclude assigned
        ]
    ),
    with_payload=True,
)
```

**Pros:**
- Very fast at query time (uses Qdrant indexes)
- Low memory usage at query time (no Python-side embedding processing)
- Pagination-friendly for UI consumption

**Cons:**
- **CRITICAL**: `query_points_groups` groups by EXISTING payload field values, NOT by vector similarity. Requires pre-computed clusters.
- Threshold NOT adjustable at query time (locked to whatever clustering was run in the batch job)
- Requires background job infrastructure for pre-computation
- Stale results if faces are added/removed between batch runs
- Cannot filter by "person_id IS NULL" natively (same IS NULL limitation)
- No offset pagination in Qdrant grouping API

### Option C: Hybrid -- Pre-compute + On-Demand Refinement

**Approach**: Combine Options A and B. Pre-compute coarse clusters via background job for fast initial display, then refine on-demand when user adjusts threshold.

**Data flow:**
```
1. Background job: Compute clusters at default threshold, store in DB + Qdrant
2. Initial load: Query PostgreSQL for pre-computed cluster groups (fast)
3. User adjusts threshold: Re-cluster on-demand using Option A logic
4. Optionally persist updated clusters
```

**Pros:**
- Fast initial load from pre-computed data
- Threshold adjustable via on-demand re-clustering
- Progressive refinement UX (show stale data immediately, refresh on demand)

**Cons:**
- More complex implementation (two code paths)
- Cache invalidation challenges (when are pre-computed clusters stale?)
- Must manage "freshness" indicator in UI

### Option D: Qdrant Pairwise Similarity + Client-Side Grouping

**Approach**: For each unassigned face, find its K nearest unassigned neighbors using Qdrant similarity search. Build a similarity graph, then apply connected components or community detection for grouping.

**Data flow:**
```
1. PostgreSQL: Get unassigned face IDs
2. For each face: Qdrant query_points(face_embedding, exclude assigned)
3. Build similarity graph: face_i -> [(face_j, score), ...]
4. Connected components at user threshold
5. Return groups
```

**Pros:**
- Uses Qdrant's optimized ANN search (HNSW index)
- Does not need to load all embeddings into memory at once
- Naturally handles "neighborhood" grouping

**Cons:**
- **O(N) Qdrant queries** (one per unassigned face) -- extremely slow for large N
- Network overhead dominates
- Connected components may produce different results than direct clustering
- Threshold adjustment requires re-running all queries

---

## 5. Performance Considerations

### 5.1 Data Volume Estimates

Based on typical usage patterns for a personal/family photo library:

| Metric | Small | Medium | Large | Very Large |
|---|---|---|---|---|
| Total images | 5,000 | 20,000 | 50,000 | 100,000+ |
| Total face instances | 3,000 | 15,000 | 40,000 | 100,000+ |
| Assigned faces | 2,000 | 10,000 | 30,000 | 70,000+ |
| **Unassigned faces** | **1,000** | **5,000** | **10,000** | **30,000+** |
| Persons (known) | 20 | 50 | 100 | 200+ |
| Clusters (unknown) | 50 | 200 | 500 | 1,000+ |

### 5.2 Memory Requirements

For Python-side clustering (Option A):
- Each 512-dim float32 embedding: 2,048 bytes (2 KB)
- 1,000 faces: ~2 MB embeddings
- 5,000 faces: ~10 MB embeddings
- 10,000 faces: ~20 MB embeddings
- Pairwise distance matrix (N x N float64): N^2 * 8 bytes
  - 1,000 faces: ~8 MB
  - 5,000 faces: ~200 MB
  - 10,000 faces: ~800 MB (approaching memory limits)

**Recommendation**: For >5,000 unassigned faces, use mini-batch clustering or approximate methods (e.g., HDBSCAN with approximate_predict).

### 5.3 Qdrant Performance Characteristics

- **Point retrieval** (by ID): ~1ms per point, ~10ms per batch of 100
- **Scroll** (100 points per page): ~5-15ms per page
- **query_points** (similarity search): ~2-10ms per query (HNSW index)
- **query_points_groups**: Similar to query_points + grouping overhead (~5-20ms)
- **Full scroll of faces collection**: O(N/100) pages at ~10ms each
  - 5,000 faces: ~500ms
  - 10,000 faces: ~1s
  - 50,000 faces: ~5s

### 5.4 Network Transfer

Embedding retrieval from Qdrant to Python:
- Each embedding with payload: ~2.5 KB
- 1,000 faces: ~2.5 MB transfer
- 5,000 faces: ~12.5 MB transfer
- 10,000 faces: ~25 MB transfer

### 5.5 Existing Performance Patterns

The existing dual-mode clusterer (`dual_clusterer.py`) fetches embeddings **one at a time** from Qdrant:
```python
for face in unknown_faces:
    embedding = self._get_face_embedding(face["qdrant_point_id"])  # 1 Qdrant call per face
```

**Performance improvement opportunity**: Batch-retrieve embeddings using `qdrant.client.retrieve(ids=[...list...], with_vectors=True)` instead of individual calls. This could reduce N Qdrant roundtrips to N/100 batch calls.

---

## 6. Recommended Approach

### Primary Recommendation: Option A (Qdrant Scroll + Python Clustering) with Batch Retrieval Optimization

**Rationale:**
1. **Minimal new code** -- extends existing proven patterns (DualModeClusterer, HDBSCAN)
2. **Threshold adjustability** -- core UX requirement, only achievable with on-demand clustering
3. **Algorithm flexibility** -- can switch between HDBSCAN/DBSCAN/Agglomerative per user preference
4. **Confidence calculation** -- existing `calculate_cluster_confidence()` function directly applicable
5. **No Qdrant limitations** -- avoids IS NULL filter issue and query_points_groups limitations

### Enhancement: Sentinel Value for Fast Unassigned Filtering

To eliminate the O(N) full-scan problem, add an `is_assigned` boolean payload field to the faces collection:

```python
# On face creation:
payload["is_assigned"] = False

# On person assignment:
qdrant.set_payload(points=[face_id], payload={"is_assigned": True, "person_id": str(person_id)})

# On person un-assignment:
qdrant.set_payload(points=[face_id], payload={"is_assigned": False})
qdrant.delete_payload(points=[face_id], keys=["person_id"])
```

This enables efficient Qdrant filtering:
```python
results = qdrant.client.scroll(
    collection_name="faces",
    scroll_filter=Filter(must=[
        FieldCondition(key="is_assigned", match=MatchValue(value=False))
    ]),
    with_vectors=True,
    limit=100,
)
```

**Requires**: Adding payload index `is_assigned` (BOOL) to faces collection bootstrap.

### Enhancement: Batch Embedding Retrieval

Replace the current one-at-a-time embedding retrieval with batch calls:

```python
# Current (slow):
for face in faces:
    embedding = qdrant.client.retrieve(ids=[str(face.qdrant_point_id)], with_vectors=True)

# Proposed (fast):
for batch in chunk(faces, 100):
    points = qdrant.client.retrieve(
        collection_name="faces",
        ids=[str(f.qdrant_point_id) for f in batch],
        with_vectors=True,
    )
```

### Enhancement: Optional Pre-computation (Option C Hybrid)

For collections with >5,000 unassigned faces, add background pre-computation:

```python
# Background job (runs after face detection completes):
def precompute_unknown_groups():
    embeddings = get_unlabeled_faces_with_embeddings()
    labels = hdbscan_cluster(embeddings, min_cluster_size=3)
    update_cluster_ids(labels)  # Persist to DB + Qdrant

# API endpoint:
# If pre-computed clusters exist AND user uses default threshold -> return pre-computed
# If user adjusts threshold -> re-cluster on-demand
```

---

## 7. SQL/Qdrant Query Examples for the Feature

### 7.1 Get Unassigned Face Count (PostgreSQL)

```sql
SELECT COUNT(*) as unassigned_count,
       AVG(quality_score) as avg_quality,
       MIN(created_at) as oldest_face,
       MAX(created_at) as newest_face
FROM face_instances
WHERE person_id IS NULL;
```

### 7.2 Get Unassigned Face IDs with Quality Filter (PostgreSQL)

```python
query = (
    select(FaceInstance.id, FaceInstance.qdrant_point_id, FaceInstance.quality_score)
    .where(FaceInstance.person_id.is_(None))
    .where(FaceInstance.quality_score >= min_quality)
    .order_by(FaceInstance.quality_score.desc())
    .limit(max_faces)
)
```

### 7.3 Batch Retrieve Embeddings (Qdrant)

```python
# Retrieve embeddings in batches of 100
point_ids = [str(f.qdrant_point_id) for f in faces]
all_embeddings = []

for i in range(0, len(point_ids), 100):
    batch_ids = point_ids[i:i+100]
    points = qdrant_client.retrieve(
        collection_name="faces",
        ids=batch_ids,
        with_vectors=True,
        with_payload=False,  # Don't need payload, saves bandwidth
    )
    for point in points:
        if point.vector:
            all_embeddings.append(np.array(point.vector))
```

### 7.4 On-Demand Clustering with Adjustable Threshold

```python
from sklearn.cluster import AgglomerativeClustering
from sklearn.metrics.pairwise import cosine_distances

# Convert user's similarity threshold to distance threshold
distance_threshold = 1.0 - user_similarity_threshold  # e.g., 0.7 sim -> 0.3 dist

embeddings_matrix = np.array(all_embeddings)

# Agglomerative clustering respects exact distance threshold
clusterer = AgglomerativeClustering(
    n_clusters=None,
    distance_threshold=distance_threshold,
    linkage="average",
    metric="cosine",
)
labels = clusterer.fit_predict(embeddings_matrix)
```

### 7.5 Filter Unassigned Faces in Qdrant (with Sentinel Value)

```python
# Scroll ONLY unassigned faces (no full scan needed)
records, next_offset = qdrant_client.scroll(
    collection_name="faces",
    scroll_filter=Filter(must=[
        FieldCondition(key="is_assigned", match=MatchValue(value=False)),
    ]),
    limit=100,
    with_vectors=True,
    with_payload=True,
)
```

### 7.6 Get Pre-computed Cluster Groups (PostgreSQL)

```sql
SELECT
    cluster_id,
    COUNT(*) as face_count,
    AVG(quality_score) as avg_quality,
    MIN(created_at) as oldest,
    MAX(created_at) as newest
FROM face_instances
WHERE person_id IS NULL
  AND cluster_id IS NOT NULL
  AND cluster_id NOT LIKE 'unknown_noise_%'
GROUP BY cluster_id
HAVING COUNT(*) >= 2
ORDER BY COUNT(*) DESC;
```

---

## 8. Open Questions

### 8.1 For Architecture Decision

1. **Should we add the `is_assigned` sentinel value to Qdrant?**
   - Pro: Eliminates O(N) full-scan for finding unassigned faces
   - Con: Requires data migration for existing faces, adds payload maintenance overhead
   - Recommendation: YES, this is a significant performance win for the feature

2. **Should clustering run synchronously (API request) or asynchronously (background job)?**
   - For <2,000 faces: Synchronous is acceptable (~10s)
   - For >2,000 faces: Background job with progress tracking (similar to existing find-more pattern)
   - Recommendation: Support both with automatic fallback to async above threshold

3. **Should we persist discovered groups to `cluster_id`?**
   - Pro: Enables fast re-display, Qdrant native grouping
   - Con: Stale data if new faces arrive, user may want different thresholds
   - Recommendation: Persist as `transient_cluster_id` (separate from batch clustering results)

### 8.2 For Implementation

4. **What clustering algorithm should be default?**
   - HDBSCAN: Best for variable-density clusters, automatic cluster count
   - Agglomerative: Exact threshold control, deterministic
   - Recommendation: Agglomerative as default (exact threshold mapping), HDBSCAN as advanced option

5. **Maximum unassigned faces to process in a single request?**
   - Memory limit: ~5,000 faces per request (200 MB distance matrix)
   - Recommendation: 5,000 limit with pagination for larger sets

6. **How to handle the "noise" faces (singletons that don't cluster)?**
   - Option: Show as "unclustered" section
   - Option: Filter out below minimum group size
   - Recommendation: Both -- show noise count, but only display groups meeting min_size threshold

### 8.3 For UI/UX Integration

7. **Real-time threshold adjustment -- should it re-cluster or just filter?**
   - Re-cluster: More accurate but slower (seconds)
   - Filter by pre-computed confidence: Instant but less accurate
   - Recommendation: Pre-compute at multiple thresholds, interpolate for instant feedback

8. **Should groups show cluster confidence?**
   - Yes -- reuse existing `calculate_cluster_confidence()` from `face_clustering_service.py`
   - Display as "group cohesion" or "similarity score" in UI

---

## 9. Cross-References to Other Research Agents

This research should be consumed alongside:
- **Agent 1**: Existing Codebase Analysis -- understand full face pipeline
- **Agent 2**: UI/UX Research -- how threshold adjustment is presented
- **Agent 4**: Algorithm Research -- clustering algorithm selection details
- **Agent 5**: Integration Research -- how this fits into the overall architecture

---

## 10. Appendix: File Reference Index

| File | Path (relative to image-search-service/src/image_search_service/) | Relevance |
|---|---|---|
| Database models | `db/models.py` | FaceInstance, Person, PersonCentroid, FaceSuggestion schemas |
| Face Qdrant client | `vector/face_qdrant.py` | Face collection CRUD, unlabeled face retrieval, similarity search |
| Centroid Qdrant client | `vector/centroid_qdrant.py` | Centroid upsert/delete, centroid-based face search |
| Main Qdrant client | `vector/qdrant.py` | Image asset vectors, general Qdrant patterns |
| Bootstrap script | `scripts/bootstrap_qdrant.py` | Collection creation, payload indexes |
| Dual-mode clusterer | `faces/dual_clusterer.py` | HDBSCAN/DBSCAN/Agglomerative clustering |
| Face assigner | `faces/assigner.py` | Prototype-based face assignment, two-tier thresholds |
| Centroid service | `services/centroid_service.py` | Centroid computation with outlier trimming |
| Clustering service | `services/face_clustering_service.py` | Cluster confidence calculation, representative face selection |
| Face API routes | `api/routes/faces.py` | Cluster endpoints, person management |
| Suggestion routes | `api/routes/face_suggestions.py` | Suggestion CRUD, find-more, centroid find-more |
| Config | `core/config.py` | All configurable thresholds and settings |
