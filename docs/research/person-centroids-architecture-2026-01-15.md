# Person Centroids Architecture Analysis

**Date**: 2026-01-15
**Purpose**: Understand existing architecture for implementing person centroids for face→person suggestions
**Context**: Research for person centroid feature development

---

## Executive Summary

The image-search system already has comprehensive infrastructure for:
- Face embeddings storage in Qdrant (512-dim ArcFace vectors)
- Person prototype management (CENTROID, EXEMPLAR, TEMPORAL, PRIMARY, FALLBACK roles)
- Face suggestion generation with multi-prototype matching
- Existing centroid computation in `FaceAssigner.compute_person_centroids()`

**Key Finding**: Centroid infrastructure exists but is **not actively used** in suggestion generation. Current system uses individual exemplar faces as prototypes.

---

## 1. Qdrant Collections

### Face Collection Schema

**Location**: `image-search-service/src/image_search_service/vector/face_qdrant.py`

**Collection Name**: `faces` (configurable via `QDRANT_FACE_COLLECTION`)

**Vector Configuration**:
```python
FACE_VECTOR_DIM = 512  # ArcFace/InsightFace embedding dimension
Distance = Distance.COSINE
```

**Payload Schema**:
```python
{
    "asset_id": str,              # UUID of image asset (string format)
    "face_instance_id": str,      # UUID of face instance record
    "person_id": str | None,      # UUID of assigned person (optional)
    "cluster_id": str | None,     # Cluster ID from clustering algorithm
    "detection_confidence": float, # Face detection confidence (0.0-1.0)
    "quality_score": float | None, # Face quality score (0.0-1.0)
    "taken_at": str | None,       # ISO datetime string
    "bbox": dict | None,          # {x, y, w, h} pixel coordinates
    "is_prototype": bool          # Whether this face is a prototype
}
```

**Indexed Fields** (for efficient filtering):
- `person_id` (KEYWORD)
- `cluster_id` (KEYWORD)
- `is_prototype` (BOOL)
- `asset_id` (KEYWORD)
- `face_instance_id` (KEYWORD)

**Key Methods**:
```python
# Collection management
ensure_collection() -> None

# Vector operations
upsert_face(point_id, embedding, ...) -> None
upsert_faces_batch(faces: list[dict]) -> None

# Payload updates
update_payload(point_id, payload_updates) -> None
update_person_ids(point_ids, person_id) -> None

# Search operations
search_similar_faces(query_embedding, limit, score_threshold,
                     filter_person_id, filter_cluster_id, filter_is_prototype)
search_against_prototypes(query_embedding, limit, score_threshold)

# Retrieval
get_embedding_by_point_id(point_id) -> list[float] | None
scroll_faces(limit, offset, filter_person_id, filter_cluster_id, include_vectors)
```

**Important**: Vectors are stored at `face_instance_id` granularity. Centroids would be stored as **synthetic points** with special metadata.

---

## 2. Database Models

### Person Table

**Location**: `image-search-service/src/image_search_service/db/models.py:411-457`

```python
class Person(Base):
    __tablename__ = "persons"

    id: UUID (primary key)
    name: str
    status: PersonStatus (ACTIVE, MERGED, HIDDEN)
    merged_into_id: UUID | None
    birth_date: date | None
    created_at: datetime
    updated_at: datetime

    # Relationships
    face_instances: list[FaceInstance]
    prototypes: list[PersonPrototype]
```

### FaceInstance Table

**Location**: `image-search-service/src/image_search_service/db/models.py:459-529`

```python
class FaceInstance(Base):
    __tablename__ = "face_instances"

    id: UUID (primary key)
    asset_id: int (FK to image_assets)

    # Bounding box
    bbox_x: int
    bbox_y: int
    bbox_w: int
    bbox_h: int

    # Detection metadata
    landmarks: dict | None (JSONB - 5-point facial landmarks)
    detection_confidence: float
    quality_score: float | None

    # Vector storage reference
    qdrant_point_id: UUID (unique - references Qdrant point)

    # Assignment
    cluster_id: str | None
    person_id: UUID | None (FK to persons)

    created_at: datetime
    updated_at: datetime

    # Unique constraint on (asset_id, bbox_x, bbox_y, bbox_w, bbox_h)
```

**Key Insight**: `qdrant_point_id` links DB record to Qdrant vector. For centroids, we'd either:
1. Create synthetic `FaceInstance` records (not recommended)
2. Store centroid point ID in `PersonPrototype` with `face_instance_id=NULL`

### PersonPrototype Table

**Location**: `image-search-service/src/image_search_service/db/models.py:531-584`

```python
class PrototypeRole(str, Enum):
    CENTROID = "centroid"   # Computed average/centroid
    EXEMPLAR = "exemplar"   # High-quality representative face
    PRIMARY = "primary"     # User-pinned definitive photo
    TEMPORAL = "temporal"   # Age-era based exemplar
    FALLBACK = "fallback"   # Lower quality, fills era gaps

class PersonPrototype(Base):
    __tablename__ = "person_prototypes"

    id: UUID (primary key)
    person_id: UUID (FK to persons, cascade delete)
    face_instance_id: UUID | None (FK to face_instances, set null on delete)
    qdrant_point_id: UUID (NOT NULL - can exist without face_instance)
    role: PrototypeRole

    # Temporal metadata
    age_era_bucket: str | None (infant, child, teen, young_adult, adult, senior)
    decade_bucket: str | None

    # Pinning metadata
    is_pinned: bool (default=False)
    pinned_by: str | None
    pinned_at: datetime | None

    created_at: datetime
```

**Critical Observation**: The model **already supports** centroids:
- `role=PrototypeRole.CENTROID`
- `face_instance_id=None` for synthetic centroids
- `qdrant_point_id` can reference centroid vector in Qdrant

**Indexes**:
- `ix_person_prototypes_person_id`
- `ix_person_prototypes_role` (person_id, role)
- `ix_person_prototypes_era` (person_id, age_era_bucket)
- `ix_person_prototypes_pinned` (person_id, is_pinned)

---

## 3. Face Suggestion System

### Current Flow

**Endpoint**: `POST /api/v1/faces/suggestions/persons/{person_id}/find-more`

**Location**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py:741-815`

**Process**:
1. User triggers "Find More" from UI
2. Backend enqueues `find_more_suggestions_job`
3. Job samples N labeled faces as temporary prototypes
4. Searches Qdrant for similar unlabeled faces
5. Creates `FaceSuggestion` records for matches above threshold

**Key Configuration** (from `config.py`):
```python
face_suggestion_min_confidence: float = 0.7  # Similarity threshold
face_suggestion_max_results: int = 5         # Max suggestions per query
face_prototype_max_exemplars: int = 5        # Max prototypes per person
```

### Face Suggestion Endpoints

**Location**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

```python
GET  /faces/suggestions                    # List suggestions (paginated)
GET  /faces/suggestions/{id}               # Get single suggestion
POST /faces/suggestions/{id}/accept        # Accept suggestion
POST /faces/suggestions/{id}/reject        # Reject suggestion
POST /faces/suggestions/bulk-action        # Batch accept/reject
GET  /faces/suggestions/stats              # Acceptance rate stats

POST /faces/suggestions/persons/{person_id}/find-more  # Generate suggestions
```

### FaceSuggestion Model

**Location**: `image-search-service/src/image_search_service/db/models.py:718-787`

```python
class FaceSuggestion(Base):
    __tablename__ = "face_suggestions"

    id: int (primary key)
    face_instance_id: UUID (FK to face_instances)
    suggested_person_id: UUID (FK to persons)
    confidence: float  # Cosine similarity score
    source_face_id: UUID (FK to face_instances - which prototype matched)
    status: FaceSuggestionStatus (PENDING, ACCEPTED, REJECTED, EXPIRED)

    created_at: datetime
    reviewed_at: datetime | None

    # Multi-prototype scoring (PHASE 5+)
    matching_prototype_ids: list[str] | None (JSONB)
    prototype_scores: dict[str, float] | None (JSONB)
    aggregate_confidence: float | None
    prototype_match_count: int | None
```

**Indexes**:
- `ix_face_suggestions_face_instance_id`
- `ix_face_suggestions_suggested_person_id`
- `ix_face_suggestions_status`
- `ix_face_suggestions_aggregate_confidence`

### Current Suggestion Generation

**Location**: `image-search-service/src/image_search_service/faces/assigner.py:34-256`

**Two-Tier Threshold System**:
```python
auto_assign_threshold = 0.85  # Auto-assign without review
suggestion_threshold = 0.70   # Create suggestion for manual review
```

**Process** (`FaceAssigner.assign_new_faces()`):
1. Get unlabeled faces from DB
2. For each face:
   - Retrieve embedding from Qdrant
   - Search against prototypes (filtered by `is_prototype=True`)
   - If score ≥ auto_assign_threshold → auto-assign to person
   - Else if score ≥ suggestion_threshold → create FaceSuggestion
   - Else → leave unassigned
3. Batch update DB and Qdrant

**Key Method**:
```python
qdrant.search_against_prototypes(
    query_embedding=embedding,
    limit=3,  # Top 3 matches
    score_threshold=suggestion_threshold
)
```

**Current Limitation**: Uses individual exemplar faces as prototypes, not centroids.

---

## 4. UI Patterns

### FindMoreDialog Component

**Location**: `image-search-ui/src/lib/components/faces/FindMoreDialog.svelte`

**Features**:
- Select prototype count (10, 50, 100, 1000, or All faces)
- Adjustable similarity threshold (0.55-0.70)
- Progress tracking via `jobProgressStore`
- Results panel showing:
  - Suggestions created
  - Candidates found
  - Prototypes used
  - Duplicates skipped

**Key Props**:
```typescript
interface Props {
    open: boolean;
    personId: string;
    personName: string;
    labeledFaceCount: number;
    onClose: () => void;
    onComplete: (suggestionsFound: number) => void;
}
```

**API Integration**:
```typescript
const response = await startFindMoreSuggestions(personId, {
    prototypeCount: actualCount,
    minConfidence: parseFloat(selectedThresholdStr)
});

// Tracks job progress using Redis keys
jobProgressStore.trackJob(
    response.jobId,
    response.progressKey,  // "find_more:progress:{person_id}:{job_uuid}"
    personId,
    personName,
    handleJobComplete,
    handleJobError
);
```

### Person Detail Page

**Location**: `image-search-ui/src/routes/people/[personId]/+page.svelte`

**Features**:
- Tabs: Photos, Prototypes
- "Recompute Prototypes" button (triggers backend recomputation)
- "Re-scan for Suggestions" button (opens FindMoreDialog)
- Temporal coverage indicator
- Prototype pinning UI

**State Management**:
```typescript
let person = $state<Person | null>(null);
let prototypes = $state<Prototype[]>([]);
let coverage = $state<TemporalCoverage | null>(null);
let showFindMoreDialog = $state(false);
```

**Dialog Patterns**:
- Modals use Svelte 5 `$state` for open/close
- Pass state via props, receive updates via callbacks
- No global modal state

### TemporalTimeline Component

**Location**: `image-search-ui/src/lib/components/faces/TemporalTimeline.svelte`

**Purpose**: Visual timeline of prototypes across age eras

**Features**:
- Displays prototypes grouped by age era bucket
- Shows coverage gaps
- Visual indicators for pinned prototypes
- Clickable timeline for era navigation

---

## 5. Existing Centroid/Prototype Patterns

### Centroid Computation (Existing but Unused)

**Location**: `image-search-service/src/image_search_service/faces/assigner.py:286-391`

```python
def compute_person_centroids(self) -> dict:
    """Compute centroid embeddings for all persons and update prototypes.

    This creates/updates CENTROID-type prototypes for each person,
    which can improve matching accuracy.
    """
    # For each active person:
    # 1. Get all face embeddings
    embeddings = []
    for face in faces:
        embedding = self._get_face_embedding(face.qdrant_point_id)
        if embedding:
            embeddings.append(embedding)

    # 2. Compute centroid (mean of normalized vectors)
    centroid = np.mean(embeddings, axis=0)
    centroid = centroid / np.linalg.norm(centroid)  # Re-normalize

    # 3. Check if centroid prototype exists
    existing_centroid = self.db.execute(
        select(PersonPrototype).where(
            PersonPrototype.person_id == person.id,
            PersonPrototype.role == PrototypeRole.CENTROID,
        )
    ).scalar_one_or_none()

    # 4. Update or create centroid in Qdrant
    if existing_centroid:
        qdrant.upsert_face(
            point_id=existing_centroid.qdrant_point_id,
            embedding=centroid.tolist(),
            asset_id=faces[0].asset_id,  # Use first face's asset
            face_instance_id=faces[0].id,
            detection_confidence=1.0,
            person_id=person.id,
            is_prototype=True,
        )
    else:
        centroid_point_id = uuid.uuid4()
        prototype = PersonPrototype(
            person_id=person.id,
            face_instance_id=None,  # Centroid has no specific face
            qdrant_point_id=centroid_point_id,
            role=PrototypeRole.CENTROID,
        )
        self.db.add(prototype)

        qdrant.upsert_face(
            point_id=centroid_point_id,
            embedding=centroid.tolist(),
            asset_id=faces[0].asset_id,
            face_instance_id=faces[0].id,
            detection_confidence=1.0,
            person_id=person.id,
            is_prototype=True,
        )
```

**Critical Notes**:
- Method exists but is **not called** in the suggestion generation pipeline
- Stores centroid as synthetic Qdrant point (not linked to real face)
- Uses first face's `asset_id` and `face_instance_id` for metadata (hacky)
- Marks centroid with `is_prototype=True` for prototype filtering

### Prototype Service

**Location**: `image-search-service/src/image_search_service/services/prototype_service.py`

**Key Functions**:

```python
# Prototype creation (called on face assignment)
async def create_or_update_prototypes(
    db, qdrant, person_id, newly_labeled_face_id,
    max_exemplars=5, min_quality_threshold=0.5
) -> PersonPrototype | None

# Smart temporal selection (Phase 4)
async def select_temporal_prototypes(
    db, qdrant, person_id, preserve_pins=True
) -> list[PersonPrototype]

# Prototype pruning
async def prune_prototypes(
    db, qdrant, person_id, exemplars, max_exemplars
) -> list[UUID]

# Full recomputation (triggered by "Recompute Prototypes" button)
async def recompute_prototypes_for_person(
    db, qdrant, person_id, preserve_pins=True
) -> dict
```

**Temporal Mode** (current approach):
- Selects prototypes to cover age eras (infant, child, teen, young_adult, adult, senior)
- Prioritizes temporal diversity over centroid accuracy
- Roles: TEMPORAL (era coverage), EXEMPLAR (high quality), FALLBACK (low quality)

**Centroid Integration Opportunity**:
- Add centroid computation to `recompute_prototypes_for_person()`
- Store centroid as `role=CENTROID` prototype
- Use centroid in suggestion generation alongside exemplars

---

## 6. Implementation Recommendations

### Option A: Centroid-Only Suggestions

**Approach**: Use person centroid as the single prototype for similarity search

**Pros**:
- Simplest implementation
- Represents "average" appearance across all labeled faces
- Single vector per person = fast search

**Cons**:
- May not capture appearance variation (young vs. old photos)
- Less accurate than multi-exemplar approach

**Changes Required**:
1. Call `compute_person_centroids()` after face assignments
2. Modify `search_against_prototypes()` to filter by `role=CENTROID`
3. Update `find_more_suggestions_job` to use centroid

### Option B: Centroid + Exemplar Hybrid

**Approach**: Use centroid for initial broad search, refine with exemplars

**Pros**:
- Centroid captures general appearance
- Exemplars validate with real face images
- Balances speed and accuracy

**Cons**:
- More complex logic
- Two-stage search

**Changes Required**:
1. Stage 1: Search with centroid (threshold=0.65)
2. Stage 2: Re-score candidates against exemplars (threshold=0.70)
3. Create suggestions for matches passing both stages

### Option C: Multi-Prototype Ensemble (Existing Approach + Centroid)

**Approach**: Include centroid as one of many prototypes

**Pros**:
- Leverages existing multi-prototype infrastructure
- Centroid adds robustness to exemplar set
- Graceful degradation if exemplars insufficient

**Cons**:
- Most complex
- Requires tuning weights

**Changes Required**:
1. Add centroid to prototype set (role=CENTROID)
2. Modify `search_against_prototypes()` to include centroids
3. Update `FaceSuggestion.matching_prototype_ids` to track centroid matches

---

## 7. Architecture Diagrams

### Current Suggestion Flow

```
User clicks "Find More"
    ↓
UI: FindMoreDialog opens
    ↓
API: POST /faces/suggestions/persons/{person_id}/find-more
    ↓
Backend: Enqueue find_more_suggestions_job
    ↓
Job: Sample N labeled faces as prototypes
    ↓
    For each unlabeled face:
        ↓
        Retrieve embedding from Qdrant
        ↓
        Search against prototypes (is_prototype=True)
        ↓
        If score ≥ threshold:
            ↓
            Create FaceSuggestion record
    ↓
Job: Update progress in Redis
    ↓
UI: Display results in FindMoreDialog
```

### Proposed Centroid Flow

```
Face assigned to person
    ↓
Backend: create_or_update_prototypes()
    ↓
Backend: compute_person_centroids()
    ↓
    Get all face embeddings for person
    ↓
    Compute centroid = mean(embeddings)
    ↓
    Normalize centroid vector
    ↓
    Upsert to Qdrant with role=CENTROID
    ↓
    Create/update PersonPrototype record
        - person_id
        - qdrant_point_id (synthetic UUID)
        - role=CENTROID
        - face_instance_id=NULL
        - is_prototype=True (in Qdrant payload)
```

### Suggestion Generation with Centroid

```
find_more_suggestions_job
    ↓
Get person's prototypes (CENTROID + EXEMPLAR + TEMPORAL)
    ↓
    For each unlabeled face:
        ↓
        Retrieve embedding from Qdrant
        ↓
        Search against all prototypes (including centroid)
        ↓
        Aggregate scores across prototypes
        ↓
        If aggregate_score ≥ threshold:
            ↓
            Create FaceSuggestion with:
                - matching_prototype_ids (may include centroid)
                - prototype_scores (centroid score + exemplar scores)
                - aggregate_confidence (max or weighted avg)
```

---

## 8. Key Files Reference

### Backend Files

| File | Purpose | Lines |
|------|---------|-------|
| `src/image_search_service/vector/face_qdrant.py` | Qdrant face collection client | 868 |
| `src/image_search_service/db/models.py` | Database models (Person, FaceInstance, PersonPrototype) | 838 |
| `src/image_search_service/api/routes/face_suggestions.py` | Face suggestion endpoints | 815 |
| `src/image_search_service/faces/assigner.py` | Face assignment and centroid computation | 410 |
| `src/image_search_service/services/prototype_service.py` | Prototype management service | 1209 |
| `src/image_search_service/core/config.py` | Configuration settings | 189 |

### Frontend Files

| File | Purpose | Lines |
|------|---------|-------|
| `src/lib/components/faces/FindMoreDialog.svelte` | Find More suggestions dialog | 362 |
| `src/routes/people/[personId]/+page.svelte` | Person detail page | ~500 |
| `src/lib/components/faces/TemporalTimeline.svelte` | Prototype timeline visualization | ~300 |

---

## 9. Next Steps

### Immediate Actions

1. **Decision**: Choose centroid approach (A, B, or C)
2. **Test Existing**: Call `compute_person_centroids()` manually to verify it works
3. **Integration Point**: Identify where to trigger centroid computation
4. **Search Strategy**: Decide if centroid replaces or augments exemplars

### Implementation Phases

**Phase 1: Basic Centroid Support**
- [ ] Enable `compute_person_centroids()` in face assignment pipeline
- [ ] Verify centroid storage in Qdrant and DB
- [ ] Test centroid retrieval and search

**Phase 2: Suggestion Integration**
- [ ] Modify `search_against_prototypes()` to include centroids
- [ ] Update `find_more_suggestions_job` to use centroids
- [ ] Test suggestion quality with centroids

**Phase 3: UI Enhancements**
- [ ] Add centroid indicator in prototype list
- [ ] Show "computed from N faces" metadata
- [ ] Add "Recompute Centroid" button

**Phase 4: Optimization**
- [ ] Tune centroid weight in ensemble scoring
- [ ] Add centroid staleness detection (recompute when N new faces added)
- [ ] Performance monitoring and benchmarking

---

## 10. Open Questions

1. **Centroid Storage**: Should centroids use synthetic `asset_id` or new column in PersonPrototype?
2. **Update Frequency**: When to recompute centroids? (After every assignment, batch updates, manual trigger)
3. **Minimum Faces**: Require N faces before computing centroid? (Current code uses N=2)
4. **Versioning**: Track centroid version/hash to detect staleness?
5. **Multi-Prototype Scoring**: How to weight centroid vs. exemplar scores?
6. **UI Visibility**: Should users see/manage centroids explicitly?

---

## Conclusion

The system has **robust infrastructure** for person centroids:
- Qdrant supports synthetic prototype points
- Database model includes `PrototypeRole.CENTROID`
- Centroid computation function exists (`compute_person_centroids()`)

**Critical Gap**: Centroid computation exists but is **not integrated** into the suggestion pipeline. The "Find More" feature currently samples individual faces as temporary prototypes, not using centroids.

**Recommended Path**: Implement **Option C (Multi-Prototype Ensemble)** to add centroids alongside existing exemplars, providing robustness without discarding the temporal diversity approach already implemented.

**Effort Estimate**:
- Backend integration: 2-3 days
- Suggestion pipeline updates: 1-2 days
- UI enhancements: 1 day
- Testing and optimization: 2-3 days
**Total**: ~1 week for full centroid integration

---

**Research Completed**: 2026-01-15
**Analyst**: Claude (Sonnet 4.5)
