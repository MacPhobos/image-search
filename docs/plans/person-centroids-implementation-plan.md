# Person Centroids Implementation Plan

**Date**: 2026-01-15
**Status**: DRAFT - Awaiting Review
**Author**: PM Agent

---

## Executive Summary

This plan implements **person centroids** for improved face→person suggestion generation. The system will compute robust centroid embeddings for each person, store them in Qdrant with versioning metadata, and use them to find additional face candidates.

### Key Findings from Research

**Critical Discovery**: Centroid infrastructure partially exists but is **unused**:
- `PersonPrototype.role=CENTROID` exists in DB model
- `FaceAssigner.compute_person_centroids()` method exists but never called
- Current "Find More" uses individual exemplar faces, not centroids

**Recommendation**: Build a **dedicated centroid system** with proper versioning, rather than extending the existing prototype system. This provides:
- Clean separation of concerns (prototypes vs. centroids)
- Proper versioning infrastructure (`model_version`, `centroid_version`)
- Multi-centroid support (global + cluster modes)
- Staleness detection

---

## 1. Data Model

### 1.1 Qdrant Collection: `person_centroids`

**Separate from `faces` collection** for clean versioning and lifecycle management.

```python
COLLECTION_NAME = "person_centroids"
VECTOR_DIM = 512  # ArcFace dimension
DISTANCE = Distance.COSINE

# Payload schema
{
    "person_id": str,           # UUID of person (KEYWORD indexed)
    "centroid_id": str,         # UUID of this centroid (KEYWORD indexed)
    "model_version": str,       # Embedding model hash/version (KEYWORD indexed)
    "centroid_version": int,    # Algorithm version (INTEGER indexed)
    "centroid_type": str,       # "global" | "cluster" (KEYWORD indexed)
    "cluster_label": str,       # "global", "k2_0", "k2_1", etc. (KEYWORD)
    "n_faces": int,             # Number of faces used (INTEGER)
    "created_at": str,          # ISO8601 timestamp
    "source_hash": str,         # Hash of source face IDs for staleness
    "build_params": str,        # JSON string of algorithm parameters
}

# Indexes (for filtering)
INDEXED_FIELDS = [
    ("person_id", PayloadSchemaType.KEYWORD),
    ("centroid_id", PayloadSchemaType.KEYWORD),
    ("model_version", PayloadSchemaType.KEYWORD),
    ("centroid_version", PayloadSchemaType.INTEGER),
    ("centroid_type", PayloadSchemaType.KEYWORD),
]
```

**Point ID Strategy**: Use `centroid_id` (UUID) as the Qdrant point ID for 1:1 mapping.

### 1.2 Database Table: `person_centroid`

```sql
CREATE TABLE person_centroid (
    -- Primary key
    centroid_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign keys
    person_id UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,

    -- Qdrant reference (1:1 with point_id)
    qdrant_point_id UUID NOT NULL UNIQUE,

    -- Versioning (core requirement)
    model_version VARCHAR(64) NOT NULL,      -- e.g., "arcface_v1_r100"
    centroid_version INTEGER NOT NULL,       -- Algorithm version, starts at 1

    -- Centroid type
    centroid_type VARCHAR(16) NOT NULL DEFAULT 'global',  -- global, cluster
    cluster_label VARCHAR(32) DEFAULT 'global',           -- global, k2_0, k2_1, etc.

    -- Build metadata
    n_faces INTEGER NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'active',  -- active, deprecated, building, failed

    -- Staleness detection
    source_face_ids_hash VARCHAR(64),  -- SHA256 of sorted face_instance_ids

    -- Algorithm parameters (JSON)
    build_params JSONB,

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Constraints
    CONSTRAINT check_centroid_type CHECK (centroid_type IN ('global', 'cluster')),
    CONSTRAINT check_status CHECK (status IN ('active', 'deprecated', 'building', 'failed'))
);

-- Indexes for efficient queries
CREATE INDEX ix_person_centroid_person_id ON person_centroid(person_id);
CREATE INDEX ix_person_centroid_version ON person_centroid(person_id, model_version, centroid_version);
CREATE INDEX ix_person_centroid_status ON person_centroid(status) WHERE status = 'active';
CREATE UNIQUE INDEX ix_person_centroid_unique_active
    ON person_centroid(person_id, model_version, centroid_version, centroid_type, cluster_label)
    WHERE status = 'active';
```

**Key Design Decisions**:
1. `centroid_id` is PK and also Qdrant point ID (clean 1:1 mapping)
2. `source_face_ids_hash` enables staleness detection without storing full ID list
3. Unique index prevents duplicate active centroids for same (person, version, type, label)
4. Separate from `PersonPrototype` table for clean separation

---

## 2. Versioning Strategy

### 2.1 Model Version (`model_version`)

Identifies the embedding model used to generate face vectors.

```python
# Current model identifier
MODEL_VERSION = "arcface_r100_glint360k_v1"

# Computation: hash of model config
def compute_model_version(model_name: str, model_config: dict) -> str:
    """Generate deterministic model version string."""
    config_str = json.dumps(model_config, sort_keys=True)
    hash_input = f"{model_name}:{config_str}"
    return hashlib.sha256(hash_input.encode()).hexdigest()[:16]

# Example output: "arcface_r100_a7f3b2c1"
```

**When Model Version Changes**:
- All existing centroids for that model become stale
- Centroids with different `model_version` are incompatible
- Query filter: `model_version == current_model_version`

### 2.2 Centroid Version (`centroid_version`)

Tracks algorithm evolution:

| Version | Description |
|---------|-------------|
| 1 | Basic mean centroid (no trimming) |
| 2 | Mean with outlier trimming (5-10%) |
| 3 | Robust centroid with cluster modes |

**When Centroid Version Changes**:
- Old centroids deprecated, not deleted
- Rebuild triggered on next request
- Stored in config: `CENTROID_ALGORITHM_VERSION = 2`

### 2.3 Staleness Detection

```python
def compute_source_hash(face_instance_ids: list[UUID]) -> str:
    """Compute hash of face IDs for staleness detection."""
    sorted_ids = sorted(str(fid) for fid in face_instance_ids)
    return hashlib.sha256(":".join(sorted_ids).encode()).hexdigest()[:16]

def is_centroid_stale(centroid: PersonCentroid, current_face_ids: list[UUID]) -> bool:
    """Check if centroid needs recomputation."""
    if centroid.model_version != CURRENT_MODEL_VERSION:
        return True
    if centroid.centroid_version != CURRENT_CENTROID_VERSION:
        return True
    if centroid.source_face_ids_hash != compute_source_hash(current_face_ids):
        return True
    return False
```

**Staleness Triggers**:
1. Model version mismatch
2. Centroid algorithm version mismatch
3. Face assignment changes (hash mismatch)
4. Manual invalidation (user request)

---

## 3. Centroid Computation Algorithms

### 3.1 Global Centroid (v1 - Always Computed)

```python
def compute_global_centroid(
    embeddings: np.ndarray,  # Shape: (n_faces, 512)
    trim_outliers: bool = True
) -> np.ndarray:
    """
    Compute robust global centroid for a person.

    Algorithm:
    1. Compute initial mean
    2. If trim_outliers: remove bottom X% by cosine similarity to mean
    3. Recompute mean on remaining vectors
    4. L2 normalize final centroid

    Args:
        embeddings: Face embeddings (n_faces, 512), already L2 normalized
        trim_outliers: Whether to trim low-similarity outliers

    Returns:
        Normalized centroid vector (512,)
    """
    n_faces = len(embeddings)

    # Step 1: Initial mean (normalized)
    initial_mean = np.mean(embeddings, axis=0)
    initial_mean = initial_mean / np.linalg.norm(initial_mean)

    if not trim_outliers or n_faces < 50:
        # No trimming for small sets (curated labels are high quality)
        return initial_mean

    # Step 2: Compute cosine similarity to initial mean
    similarities = embeddings @ initial_mean  # (n_faces,)

    # Step 3: Determine trim percentage
    if n_faces <= 300:
        trim_pct = 0.05  # Drop bottom 5%
    else:
        trim_pct = 0.10  # Drop bottom 10%

    # Step 4: Remove outliers
    threshold = np.percentile(similarities, trim_pct * 100)
    mask = similarities >= threshold
    trimmed_embeddings = embeddings[mask]

    # Step 5: Recompute mean
    centroid = np.mean(trimmed_embeddings, axis=0)
    centroid = centroid / np.linalg.norm(centroid)

    return centroid
```

**Trimming Rules** (from user specification):
- `n < 50`: No trimming (curated labels are high quality)
- `50 <= n <= 300`: Trim bottom 5%
- `n > 300`: Trim bottom 10%

### 3.2 Multi-Centroid / Cluster Centroids (v2 - High Volume Persons)

```python
from sklearn.cluster import KMeans

def compute_cluster_centroids(
    embeddings: np.ndarray,
    min_faces_for_clustering: int = 200,
    min_cluster_fraction: float = 0.08,
    min_cluster_size_abs: int = 20,
) -> list[tuple[str, np.ndarray]]:
    """
    Compute cluster-based centroids for high-volume persons.

    Captures appearance modes (glasses/no-glasses, age periods, lighting).

    Args:
        embeddings: Face embeddings (n_faces, 512)
        min_faces_for_clustering: Minimum faces to attempt clustering
        min_cluster_fraction: Minimum cluster size as fraction of total
        min_cluster_size_abs: Minimum absolute cluster size

    Returns:
        List of (cluster_label, centroid_vector) tuples
        e.g., [("k2_0", vec0), ("k2_1", vec1)]
    """
    n_faces = len(embeddings)

    if n_faces < min_faces_for_clustering:
        return []  # Not enough faces for clustering

    min_cluster_size = max(min_cluster_size_abs, int(min_cluster_fraction * n_faces))

    # Try K=2 first
    k = 2
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = kmeans.fit_predict(embeddings)

    # Validate cluster sizes
    cluster_sizes = [np.sum(labels == i) for i in range(k)]

    if min(cluster_sizes) < min_cluster_size:
        # Clusters too unbalanced, return empty (use global only)
        return []

    # Compute centroid per cluster with optional trimming
    centroids = []
    for i in range(k):
        cluster_mask = labels == i
        cluster_embeddings = embeddings[cluster_mask]

        # Apply same trimming logic within cluster
        if len(cluster_embeddings) >= 50:
            cluster_centroid = compute_global_centroid(cluster_embeddings, trim_outliers=True)
        else:
            # Simple mean for small clusters
            cluster_centroid = np.mean(cluster_embeddings, axis=0)
            cluster_centroid = cluster_centroid / np.linalg.norm(cluster_centroid)

        centroids.append((f"k{k}_{i}", cluster_centroid))

    return centroids

def try_higher_k(embeddings: np.ndarray, current_k: int = 2) -> list[tuple[str, np.ndarray]]:
    """
    Optional: Try K=3 if K=2 produces weak separation (future enhancement).
    """
    # Measure cluster separation quality
    # If weak, try K=3
    # Implementation deferred to v2
    pass
```

### 3.3 Full Centroid Computation Pipeline

```python
async def compute_centroids_for_person(
    db: AsyncSession,
    qdrant: FaceQdrant,
    person_id: UUID,
    model_version: str,
    centroid_version: int,
    enable_clustering: bool = True,
) -> list[PersonCentroid]:
    """
    Full pipeline to compute and store centroids for a person.

    Returns:
        List of created PersonCentroid records
    """
    # Step 1: Get all face embeddings for this person
    embeddings, face_ids = await get_person_face_embeddings(db, qdrant, person_id)

    n_faces = len(embeddings)
    if n_faces < 2:
        raise InsufficientFacesError(f"Person {person_id} has only {n_faces} faces")

    source_hash = compute_source_hash(face_ids)
    results = []

    # Step 2: Always compute global centroid
    global_centroid = compute_global_centroid(embeddings, trim_outliers=True)

    global_record = await store_centroid(
        db=db,
        qdrant=qdrant,
        person_id=person_id,
        centroid_vector=global_centroid,
        centroid_type="global",
        cluster_label="global",
        n_faces=n_faces,
        source_hash=source_hash,
        model_version=model_version,
        centroid_version=centroid_version,
    )
    results.append(global_record)

    # Step 3: Optionally compute cluster centroids (v2)
    if enable_clustering and n_faces >= 200:
        cluster_centroids = compute_cluster_centroids(embeddings)

        for cluster_label, cluster_vector in cluster_centroids:
            cluster_record = await store_centroid(
                db=db,
                qdrant=qdrant,
                person_id=person_id,
                centroid_vector=cluster_vector,
                centroid_type="cluster",
                cluster_label=cluster_label,
                n_faces=n_faces,  # Total faces (not cluster size)
                source_hash=source_hash,
                model_version=model_version,
                centroid_version=centroid_version,
            )
            results.append(cluster_record)

    await db.commit()
    return results
```

---

## 4. On-Demand Workflow

### 4.1 Workflow Diagram

```
User clicks "Find More" or "Compute Centroids"
    ↓
API: POST /api/v1/faces/centroids/persons/{person_id}/suggestions
    ↓
Backend: Check for active centroids
    ↓
    ├── IF centroids exist AND not stale:
    │       Use existing centroids
    ↓
    └── IF missing OR stale:
            Build centroids now:
            1. Scroll Qdrant for person's face embeddings
            2. Compute global centroid (with trimming)
            3. Optionally compute cluster centroids
            4. Upsert to person_centroids collection
            5. Write DB records
    ↓
Search faces collection using centroid(s)
    ↓
Apply filters:
    - must_not: is_prototype = true
    - must_not: person_id = {person_id}
    - optional: person_id IS NULL (unassigned only)
    ↓
Dedupe and rank results
    ↓
Return suggestions (face_instance_id, asset_id, score, centroid_label)
```

### 4.2 Pseudocode: On-Demand Suggestion Generation

```python
async def get_centroid_suggestions(
    db: AsyncSession,
    qdrant_faces: FaceQdrant,
    qdrant_centroids: CentroidQdrant,
    person_id: UUID,
    options: CentroidSuggestionOptions,
) -> CentroidSuggestionResponse:
    """
    Main workflow for centroid-based suggestions.
    """
    model_version = get_current_model_version()
    centroid_version = get_current_centroid_version()

    # Step 1: Check for existing active centroids
    existing_centroids = await db.execute(
        select(PersonCentroid).where(
            PersonCentroid.person_id == person_id,
            PersonCentroid.model_version == model_version,
            PersonCentroid.centroid_version == centroid_version,
            PersonCentroid.status == "active",
        )
    )
    existing_centroids = list(existing_centroids.scalars().all())

    # Step 2: Check staleness
    current_face_ids = await get_person_face_ids(db, person_id)

    need_rebuild = False
    if not existing_centroids:
        need_rebuild = True
    else:
        # Check if any centroid is stale
        for centroid in existing_centroids:
            if is_centroid_stale(centroid, current_face_ids):
                need_rebuild = True
                break

    # Step 3: Build centroids if needed
    if need_rebuild:
        # Deprecate old centroids
        await deprecate_centroids(db, person_id)

        # Build new centroids
        new_centroids = await compute_centroids_for_person(
            db=db,
            qdrant=qdrant_faces,
            person_id=person_id,
            model_version=model_version,
            centroid_version=centroid_version,
            enable_clustering=options.enable_clustering,
        )
        existing_centroids = new_centroids

    # Step 4: Search for candidates using centroid(s)
    all_candidates = []

    for centroid in existing_centroids:
        # Get centroid vector from Qdrant
        centroid_vector = await qdrant_centroids.get_vector(centroid.qdrant_point_id)

        # Search faces collection
        candidates = await qdrant_faces.search_similar_faces(
            query_embedding=centroid_vector,
            limit=options.top_k_per_centroid,  # Default: 200
            score_threshold=options.min_similarity,  # Default: 0.65
            filters={
                "must_not": [
                    {"key": "is_prototype", "match": {"value": True}},
                    {"key": "person_id", "match": {"value": str(person_id)}},
                ],
                # Optional: only unassigned
                **({"must_not": [{"key": "person_id", "match": {"any": True}}]}
                   if options.unassigned_only else {}),
            },
            with_payload=True,
            with_vectors=False,
        )

        for candidate in candidates:
            all_candidates.append({
                "face_instance_id": candidate.payload["face_instance_id"],
                "asset_id": candidate.payload["asset_id"],
                "score": candidate.score,
                "centroid_id": str(centroid.centroid_id),
                "centroid_label": centroid.cluster_label,
            })

    # Step 5: Dedupe and rank
    deduped = dedupe_candidates(all_candidates)
    ranked = rank_candidates(deduped, options.ranking_strategy)

    # Step 6: Apply final limit
    final = ranked[:options.max_results]  # Default: 200

    return CentroidSuggestionResponse(
        person_id=person_id,
        centroids_used=[c.centroid_id for c in existing_centroids],
        candidates=final,
        total_candidates=len(ranked),
        rebuilt_centroids=need_rebuild,
    )
```

---

## 5. API Design

### 5.1 Centroid Management Endpoints

```python
# POST /api/v1/faces/centroids/persons/{person_id}/compute
# Compute/recompute centroids for a person
@router.post("/centroids/persons/{person_id}/compute")
async def compute_centroids(
    person_id: UUID,
    request: ComputeCentroidsRequest,
    db: AsyncSession = Depends(get_db),
) -> ComputeCentroidsResponse:
    """
    Compute centroids for a person. Can be called on-demand or triggered by UI.

    Request:
    {
        "force_rebuild": false,      # If true, rebuild even if not stale
        "enable_clustering": true,   # If true, compute cluster centroids for high-volume
        "min_faces": 2               # Minimum faces required
    }

    Response:
    {
        "person_id": "...",
        "centroids": [
            {
                "centroid_id": "...",
                "centroid_type": "global",
                "cluster_label": "global",
                "n_faces": 150,
                "model_version": "arcface_r100_a7f3b2c1",
                "centroid_version": 2,
                "created_at": "2026-01-15T10:30:00Z"
            },
            ...
        ],
        "rebuilt": true,
        "stale_reason": "face_assignments_changed"
    }
    """

# GET /api/v1/faces/centroids/persons/{person_id}
# Get existing centroids for a person
@router.get("/centroids/persons/{person_id}")
async def get_centroids(
    person_id: UUID,
    include_stale: bool = False,
    db: AsyncSession = Depends(get_db),
) -> GetCentroidsResponse:
    """
    Get centroids for a person.

    Response:
    {
        "person_id": "...",
        "centroids": [...],
        "is_stale": false,
        "stale_reason": null
    }
    """

# DELETE /api/v1/faces/centroids/persons/{person_id}
# Invalidate/delete centroids for a person
@router.delete("/centroids/persons/{person_id}")
async def delete_centroids(
    person_id: UUID,
    db: AsyncSession = Depends(get_db),
) -> DeleteCentroidsResponse:
    """Delete all centroids for a person (triggers rebuild on next request)."""
```

### 5.2 Centroid-Based Suggestion Endpoint

```python
# POST /api/v1/faces/centroids/persons/{person_id}/suggestions
# Get face suggestions using centroids
@router.post("/centroids/persons/{person_id}/suggestions")
async def get_centroid_suggestions(
    person_id: UUID,
    request: CentroidSuggestionRequest,
    db: AsyncSession = Depends(get_db),
) -> CentroidSuggestionResponse:
    """
    Get face suggestions using person centroids.

    Request:
    {
        "min_similarity": 0.65,        # Minimum cosine similarity
        "max_results": 200,            # Maximum suggestions to return
        "unassigned_only": true,       # Only suggest unassigned faces
        "exclude_prototypes": true,    # Exclude prototype faces
        "auto_rebuild": true           # Auto-rebuild if stale
    }

    Response:
    {
        "person_id": "...",
        "centroids_used": ["centroid_id_1", "centroid_id_2"],
        "suggestions": [
            {
                "face_instance_id": "...",
                "asset_id": "...",
                "score": 0.78,
                "matched_centroid": "global",
                "thumbnail_url": "/api/v1/faces/..."
            },
            ...
        ],
        "total_found": 150,
        "rebuilt_centroids": false
    }
    """
```

### 5.3 Pydantic Schemas

```python
# Request schemas
class ComputeCentroidsRequest(BaseModel):
    force_rebuild: bool = False
    enable_clustering: bool = True
    min_faces: int = Field(default=2, ge=2)

class CentroidSuggestionRequest(BaseModel):
    min_similarity: float = Field(default=0.65, ge=0.5, le=0.95)
    max_results: int = Field(default=200, ge=1, le=500)
    unassigned_only: bool = True
    exclude_prototypes: bool = True
    auto_rebuild: bool = True

# Response schemas
class CentroidInfo(BaseModel):
    centroid_id: UUID
    centroid_type: Literal["global", "cluster"]
    cluster_label: str
    n_faces: int
    model_version: str
    centroid_version: int
    created_at: datetime
    is_stale: bool = False

class CentroidSuggestion(BaseModel):
    face_instance_id: UUID
    asset_id: UUID
    score: float
    matched_centroid: str
    thumbnail_url: str | None = None

class CentroidSuggestionResponse(BaseModel):
    person_id: UUID
    centroids_used: list[UUID]
    suggestions: list[CentroidSuggestion]
    total_found: int
    rebuilt_centroids: bool
```

---

## 6. Qdrant Query Patterns

### 6.1 Scroll Faces for Person (Vector Retrieval)

```python
async def get_person_face_embeddings(
    qdrant: FaceQdrant,
    person_id: UUID,
    batch_size: int = 100,
) -> tuple[np.ndarray, list[UUID]]:
    """
    Efficiently retrieve all face embeddings for a person.

    Uses scroll API for large collections (up to 1000+ vectors).
    """
    embeddings = []
    face_ids = []
    offset = None

    while True:
        result = await qdrant.scroll(
            collection_name="faces",
            scroll_filter=Filter(
                must=[
                    FieldCondition(
                        key="person_id",
                        match=MatchValue(value=str(person_id)),
                    ),
                ],
            ),
            limit=batch_size,
            offset=offset,
            with_payload=True,
            with_vectors=True,
        )

        points, next_offset = result

        for point in points:
            embeddings.append(point.vector)
            face_ids.append(UUID(point.payload["face_instance_id"]))

        if next_offset is None:
            break
        offset = next_offset

    return np.array(embeddings), face_ids
```

### 6.2 Upsert Centroid to Qdrant

```python
async def upsert_centroid(
    qdrant: QdrantClient,
    centroid_id: UUID,
    centroid_vector: np.ndarray,
    payload: dict,
) -> None:
    """
    Upsert centroid to person_centroids collection.
    """
    await qdrant.upsert(
        collection_name="person_centroids",
        points=[
            PointStruct(
                id=str(centroid_id),
                vector=centroid_vector.tolist(),
                payload=payload,
            )
        ],
        wait=True,
    )
```

### 6.3 Search Faces Using Centroid

```python
async def search_faces_with_centroid(
    qdrant: FaceQdrant,
    centroid_vector: np.ndarray,
    person_id: UUID,
    top_k: int = 200,
    score_threshold: float = 0.65,
    unassigned_only: bool = True,
) -> list[ScoredPoint]:
    """
    Search faces collection using centroid vector.

    Filters:
    - Exclude prototypes (is_prototype=False)
    - Exclude already-assigned to this person
    - Optionally: only unassigned faces
    """
    must_not_conditions = [
        FieldCondition(
            key="is_prototype",
            match=MatchValue(value=True),
        ),
        FieldCondition(
            key="person_id",
            match=MatchValue(value=str(person_id)),
        ),
    ]

    if unassigned_only:
        # Only faces with no person_id assigned
        must_conditions = [
            IsNullCondition(is_null=IsNull(key="person_id")),
        ]
    else:
        must_conditions = []

    return await qdrant.search(
        collection_name="faces",
        query_vector=centroid_vector.tolist(),
        query_filter=Filter(
            must=must_conditions,
            must_not=must_not_conditions,
        ),
        limit=top_k,
        score_threshold=score_threshold,
        with_payload=True,
        with_vectors=False,
    )
```

---

## 7. UI Design: Compute Centroids Dialog

### 7.1 Dialog Placement

**Location**: Add button beside "Re-scan for Suggestions" in `routes/people/[personId]/+page.svelte`

```svelte
<div class="flex gap-2">
    <Button onclick={() => showFindMoreDialog = true}>
        Re-scan for Suggestions
    </Button>
    <Button variant="outline" onclick={() => showComputeCentroidsDialog = true}>
        Compute Centroids
    </Button>
</div>
```

### 7.2 ComputeCentroidsDialog Component

**Location**: `src/lib/components/faces/ComputeCentroidsDialog.svelte`

```svelte
<script lang="ts">
    import { Dialog } from 'bits-ui';
    import { computeCentroids, getCentroids, getCentroidSuggestions } from '$lib/api/faces';

    interface Props {
        open: boolean;
        personId: string;
        personName: string;
        labeledFaceCount: number;
        onClose: () => void;
        onSuggestionsReady: (count: number) => void;
    }

    let { open, personId, personName, labeledFaceCount, onClose, onSuggestionsReady }: Props = $props();

    // Options state
    let enableClustering = $state(true);
    let minSimilarity = $state(0.65);
    let maxResults = $state(200);
    let unassignedOnly = $state(true);

    // Progress state
    let step = $state<'options' | 'computing' | 'searching' | 'results'>('options');
    let centroids = $state<CentroidInfo[]>([]);
    let suggestions = $state<CentroidSuggestion[]>([]);
    let error = $state<string | null>(null);

    async function handleCompute() {
        step = 'computing';
        error = null;

        try {
            // Step 1: Compute/fetch centroids
            const result = await computeCentroids(personId, {
                forceRebuild: false,
                enableClustering,
                minFaces: 2,
            });
            centroids = result.centroids;

            // Step 2: Get suggestions
            step = 'searching';
            const suggestionsResult = await getCentroidSuggestions(personId, {
                minSimilarity,
                maxResults,
                unassignedOnly,
                excludePrototypes: true,
                autoRebuild: false, // Already computed
            });

            suggestions = suggestionsResult.suggestions;
            step = 'results';

            if (suggestions.length > 0) {
                onSuggestionsReady(suggestions.length);
            }
        } catch (e) {
            error = e instanceof Error ? e.message : 'Unknown error';
            step = 'options';
        }
    }
</script>

<Dialog.Root bind:open onOpenChange={(o) => !o && onClose()}>
    <Dialog.Portal>
        <Dialog.Overlay class="fixed inset-0 bg-black/50" />
        <Dialog.Content class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2
                               bg-white rounded-lg shadow-xl p-6 w-[500px] max-h-[80vh] overflow-y-auto">
            <Dialog.Title class="text-lg font-semibold mb-4">
                Compute Centroids for {personName}
            </Dialog.Title>

            {#if step === 'options'}
                <div class="space-y-4">
                    <p class="text-sm text-gray-600">
                        This person has <strong>{labeledFaceCount}</strong> labeled faces.
                        Computing centroids will create optimized embeddings for finding additional matches.
                    </p>

                    <!-- Options -->
                    <div class="space-y-3">
                        <label class="flex items-center gap-2">
                            <input type="checkbox" bind:checked={enableClustering} />
                            <span>Enable cluster centroids (recommended for 200+ faces)</span>
                        </label>

                        <div>
                            <label class="block text-sm font-medium mb-1">
                                Minimum Similarity: {minSimilarity.toFixed(2)}
                            </label>
                            <input type="range" min="0.5" max="0.85" step="0.05"
                                   bind:value={minSimilarity} class="w-full" />
                            <div class="flex justify-between text-xs text-gray-500">
                                <span>Broad (0.50)</span>
                                <span>Strict (0.85)</span>
                            </div>
                        </div>

                        <div>
                            <label class="block text-sm font-medium mb-1">Max Results</label>
                            <select bind:value={maxResults} class="w-full border rounded p-2">
                                <option value={50}>50</option>
                                <option value={100}>100</option>
                                <option value={200}>200 (recommended)</option>
                                <option value={500}>500</option>
                            </select>
                        </div>

                        <label class="flex items-center gap-2">
                            <input type="checkbox" bind:checked={unassignedOnly} />
                            <span>Only unassigned faces</span>
                        </label>
                    </div>

                    <div class="flex justify-end gap-2 mt-6">
                        <Button variant="outline" onclick={onClose}>Cancel</Button>
                        <Button onclick={handleCompute}>Compute & Search</Button>
                    </div>
                </div>

            {:else if step === 'computing'}
                <div class="text-center py-8">
                    <Spinner class="mx-auto mb-4" />
                    <p>Computing centroids from {labeledFaceCount} faces...</p>
                </div>

            {:else if step === 'searching'}
                <div class="text-center py-8">
                    <Spinner class="mx-auto mb-4" />
                    <p>Searching for similar faces...</p>
                </div>

            {:else if step === 'results'}
                <div class="space-y-4">
                    <!-- Centroid Summary -->
                    <div class="bg-gray-50 rounded p-3">
                        <h4 class="font-medium mb-2">Centroids Computed</h4>
                        <ul class="text-sm space-y-1">
                            {#each centroids as c}
                                <li class="flex justify-between">
                                    <span>{c.cluster_label} ({c.centroid_type})</span>
                                    <span class="text-gray-500">from {c.n_faces} faces</span>
                                </li>
                            {/each}
                        </ul>
                    </div>

                    <!-- Results Summary -->
                    <div class="bg-green-50 rounded p-3">
                        <h4 class="font-medium text-green-800 mb-1">
                            {suggestions.length} Suggestions Found
                        </h4>
                        <p class="text-sm text-green-700">
                            {#if suggestions.length > 0}
                                Ready to review potential matches.
                            {:else}
                                No new matches found at similarity threshold {minSimilarity.toFixed(2)}.
                            {/if}
                        </p>
                    </div>

                    <div class="flex justify-end gap-2 mt-6">
                        <Button variant="outline" onclick={onClose}>Close</Button>
                        {#if suggestions.length > 0}
                            <Button onclick={() => onSuggestionsReady(suggestions.length)}>
                                View Suggestions
                            </Button>
                        {/if}
                    </div>
                </div>
            {/if}

            {#if error}
                <div class="mt-4 p-3 bg-red-50 text-red-700 rounded">
                    Error: {error}
                </div>
            {/if}
        </Dialog.Content>
    </Dialog.Portal>
</Dialog.Root>
```

---

## 8. Guardrails & Best Practices

### 8.1 Thresholds and Limits

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Min faces for centroid | 2 | Minimum for meaningful average |
| Min faces for clustering | 200 | Need sufficient data for modes |
| Trim % (50-300 faces) | 5% | Light outlier removal |
| Trim % (300+ faces) | 10% | More aggressive for large sets |
| Min cluster size (abs) | 20 | Prevent tiny clusters |
| Min cluster size (%) | 8% | Scale with dataset |
| Default similarity threshold | 0.65 | Balanced recall/precision |
| Top-K per centroid | 200 | Reasonable result set |
| Max results | 500 | UI/memory limit |

### 8.2 Feedback Loop Prevention

**Critical Rule**: Never include suggested faces in centroid computation until user confirms.

```python
async def get_person_face_embeddings_for_centroid(
    db: AsyncSession,
    qdrant: FaceQdrant,
    person_id: UUID,
) -> tuple[np.ndarray, list[UUID]]:
    """
    Get face embeddings for centroid computation.

    EXCLUDES:
    - Pending suggestions (not yet confirmed)
    - Recently auto-assigned faces (optional: require user review period)
    """
    # Only include faces that are explicitly labeled by user
    # NOT faces from auto-assignment or pending suggestions
    confirmed_faces = await db.execute(
        select(FaceInstance).where(
            FaceInstance.person_id == person_id,
            FaceInstance.id.notin_(
                select(FaceSuggestion.face_instance_id)
                .where(FaceSuggestion.status == FaceSuggestionStatus.PENDING)
            ),
        )
    )
    # ... proceed with embedding retrieval
```

### 8.3 Staleness Handling

```python
# When to mark centroid stale:
STALENESS_TRIGGERS = [
    "model_version_changed",      # New embedding model deployed
    "centroid_version_changed",   # Algorithm updated
    "faces_added",                # New faces assigned to person
    "faces_removed",              # Faces unassigned from person
    "manual_invalidation",        # User requested rebuild
]

# Auto-rebuild triggers:
AUTO_REBUILD_THRESHOLDS = {
    "new_faces_count": 10,        # Rebuild if 10+ new faces since last build
    "new_faces_percent": 0.20,    # Rebuild if 20%+ new faces
}
```

### 8.4 Error Handling

```python
class CentroidError(Exception):
    """Base class for centroid errors."""

class InsufficientFacesError(CentroidError):
    """Raised when person has too few faces for centroid computation."""

class StaleModelError(CentroidError):
    """Raised when model version mismatch detected."""

class ClusteringFailedError(CentroidError):
    """Raised when clustering produces invalid results."""

# API error responses
HTTP_422_INSUFFICIENT_FACES = {
    "detail": "Person has fewer than 2 labeled faces"
}

HTTP_409_REBUILD_IN_PROGRESS = {
    "detail": "Centroid rebuild already in progress"
}
```

---

## 9. Implementation Plan

### Phase 1: Core Infrastructure (v1) - 3 days

**Goal**: Basic centroid computation and storage

| Task | Effort | Files |
|------|--------|-------|
| Create Alembic migration for `person_centroid` table | 2h | `db/migrations/versions/xxx_add_person_centroid.py` |
| Create `CentroidQdrant` service class | 4h | `src/.../vector/centroid_qdrant.py` |
| Create `CentroidService` with basic computation | 4h | `src/.../services/centroid_service.py` |
| Add config values (model version, centroid version) | 1h | `src/.../core/config.py` |
| Unit tests for centroid computation | 3h | `tests/unit/test_centroid_service.py` |

**Deliverables**:
- DB table created
- Qdrant collection bootstrapped
- Global centroid computation working
- Basic unit tests

### Phase 2: API Endpoints (v1) - 2 days

**Goal**: REST API for centroid management

| Task | Effort | Files |
|------|--------|-------|
| Create centroid router with endpoints | 4h | `src/.../api/routes/face_centroids.py` |
| Create Pydantic schemas | 2h | `src/.../api/centroid_schemas.py` |
| Register router in main.py | 0.5h | `src/.../main.py` |
| Integration tests | 3h | `tests/api/test_face_centroids.py` |
| Update API contract | 1h | `docs/api-contract.md` |

**Deliverables**:
- `POST /centroids/persons/{id}/compute`
- `GET /centroids/persons/{id}`
- `POST /centroids/persons/{id}/suggestions`
- API contract updated

### Phase 3: Frontend Integration (v1) - 2 days

**Goal**: UI dialog for centroid computation

| Task | Effort | Files |
|------|--------|-------|
| Add API client functions | 2h | `src/lib/api/faces.ts` |
| Create `ComputeCentroidsDialog.svelte` | 4h | `src/lib/components/faces/ComputeCentroidsDialog.svelte` |
| Add button to person detail page | 1h | `src/routes/people/[personId]/+page.svelte` |
| Integration with FindMoreResultsDialog | 2h | (reuse existing) |
| Frontend tests | 2h | `src/tests/components/ComputeCentroidsDialog.test.ts` |

**Deliverables**:
- "Compute Centroids" button visible
- Dialog with options
- Results display
- Integration with suggestion viewer

### Phase 4: Multi-Centroid & Clustering (v2) - 3 days

**Goal**: Advanced centroid modes for high-volume persons

| Task | Effort | Files |
|------|--------|-------|
| Implement KMeans clustering | 4h | `src/.../services/centroid_service.py` |
| Add cluster validation logic | 2h | (same file) |
| Update API to return multiple centroids | 2h | `src/.../api/routes/face_centroids.py` |
| Update UI to show centroid types | 2h | `ComputeCentroidsDialog.svelte` |
| Performance testing with 500+ faces | 3h | `tests/performance/` |

**Deliverables**:
- Cluster centroids for 200+ face persons
- Cluster validation
- Multi-centroid search merging

### Phase 5: Polish & Optimization (v2) - 2 days

**Goal**: Production readiness

| Task | Effort | Files |
|------|--------|-------|
| Auto-staleness detection | 2h | `src/.../services/centroid_service.py` |
| Background rebuild job | 3h | `src/.../jobs/centroid_jobs.py` |
| Metrics/logging | 2h | Various |
| Documentation | 2h | `docs/` |
| Load testing | 3h | Performance tests |

**Deliverables**:
- Automatic staleness detection
- Background rebuild capability
- Production documentation

---

## 10. Summary

### What We're Building

A **person centroid system** that:
1. Computes robust centroid embeddings from labeled faces
2. Stores centroids in dedicated Qdrant collection with versioning
3. Uses centroids to find additional face→person suggestions
4. Provides UI for on-demand centroid computation

### Key Design Decisions

1. **Separate collection** (`person_centroids`) for clean lifecycle management
2. **Dedicated DB table** (`person_centroid`) rather than extending `PersonPrototype`
3. **On-demand computation** with staleness detection (not continuous updates)
4. **Global centroid always** + optional cluster centroids for high-volume persons
5. **No feedback loops**: Pending suggestions excluded from centroid computation

### Effort Summary

| Phase | Days | Status |
|-------|------|--------|
| Phase 1: Core Infrastructure | 3 | v1 |
| Phase 2: API Endpoints | 2 | v1 |
| Phase 3: Frontend Integration | 2 | v1 |
| Phase 4: Multi-Centroid | 3 | v2 |
| Phase 5: Polish | 2 | v2 |
| **Total** | **12 days** | |

### v1 Deliverables (~7 days)
- Global centroid computation
- DB + Qdrant storage with versioning
- REST API endpoints
- Basic UI dialog

### v2 Deliverables (~5 days)
- Cluster centroids (K=2)
- Auto-staleness detection
- Background rebuild
- Performance optimization

---

**Ready for Review**

Please review this plan and let me know if you'd like any adjustments before implementation begins.
