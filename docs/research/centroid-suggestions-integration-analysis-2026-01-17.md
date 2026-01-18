# Centroid-Based Suggestions Integration Analysis

**Date**: 2026-01-17
**Purpose**: Analyze integration of centroid-based suggestions into `/faces/suggestions` flow
**Status**: Research Complete

---

## Executive Summary

The face suggestions system currently uses **prototype-based matching** via `FaceAssigner`, which creates persistent `FaceSuggestion` database records. A new **centroid-based approach** has been implemented that provides faster, more consistent results. This analysis evaluates how to integrate both approaches into the existing `/faces/suggestions` workflow.

**Key Findings**:
- **Current System**: Prototype-based matching creates FaceSuggestion records → UI displays from database
- **New System**: Centroid-based search returns suggestions directly → NO database records
- **Integration Gap**: UI components expect database-backed suggestions with `id` field for selection tracking
- **Recommended Path**: Create FaceSuggestion records from centroid results (backend hybrid approach)

---

## 1. Current State: Prototype-Based Suggestions

### 1.1 Architecture Overview

**Flow**:
```
Unassigned Face → FaceAssigner.assign_new_faces()
                         ↓
        Vector Search Against Prototypes (Qdrant)
                         ↓
          Apply Thresholds (0.70-0.85 range)
                         ↓
        CREATE FaceSuggestion (status='pending')
                         ↓
        UI: GET /api/v1/faces/suggestions?grouped=true
                         ↓
    Display via SuggestionGroupCard component
```

### 1.2 Key Components

**Backend**:
- **FaceAssigner** (`faces/assigner.py`) - Core matching logic
- **Database Model**: `FaceSuggestion` table with 15+ fields
- **API Endpoint**: `/api/v1/faces/suggestions?grouped=true`
- **Background Jobs**: `assign_new_faces()`, `find_more_suggestions_job()`

**Frontend**:
- **Page**: `/faces/suggestions/+page.svelte`
- **Components**: `SuggestionGroupCard`, `SuggestionDetailModal`, `FindMoreResultsDialog`
- **Data Flow**: Fetch from DB → Display → Accept/Reject → Update DB

### 1.3 FaceSuggestion Schema

**Database Fields** (Critical for UI):
```typescript
interface FaceSuggestion {
  id: number;                          // PRIMARY KEY (used for selection tracking)
  faceInstanceId: string;              // UUID
  suggestedPersonId: string;           // UUID
  confidence: number;                  // 0.0-1.0
  status: 'pending' | 'accepted' | 'rejected' | 'expired';

  // UI Display
  faceThumbnailUrl: string | null;
  fullImageUrl: string | null;
  personName: string | null;
  path: string;                        // Photo filesystem path

  // Face Metadata
  bboxX/Y/W/H: number | null;         // Bounding box
  detectionConfidence: number | null;
  qualityScore: number | null;

  // Multi-Prototype Support
  matchingPrototypeIds?: string[] | null;
  prototypeScores?: Record<string, number> | null;
  aggregateConfidence?: number | null;
  prototypeMatchCount?: number | null;

  // Audit
  sourceFaceId: string;                // Prototype that triggered match
  createdAt: string;
  reviewedAt: string | null;
}
```

### 1.4 Current "Find More" Implementation

**Trigger Points**:
1. **Auto-triggered**: After bulk accept via `bulkSuggestionAction({ autoFindMore: true })`
2. **Manual**: "Find More" button in `SuggestionGroupCard` → `FindMoreDialog`
3. **Person Page**: "Find More Suggestions" button → `FindMoreDialog`

**FindMoreDialog Workflow**:
```
User configures options (prototype count, threshold)
            ↓
POST /api/v1/faces/suggestions/persons/{personId}/find-more
            ↓
Background Job: find_more_suggestions_job()
  - Sample random labeled faces (weighted by quality)
  - Search Qdrant for similar faces
  - CREATE FaceSuggestion records
            ↓
Job Progress Tracking (jobProgressStore)
            ↓
FindMoreResultsDialog loads from DB
            ↓
SuggestionGroupCard displays suggestions
```

**Characteristics**:
- ✅ Job-based (async, non-blocking)
- ✅ Creates persistent FaceSuggestion records
- ✅ Integrates seamlessly with existing UI
- ❌ Slow for persons with many faces (samples random prototypes)
- ❌ Inconsistent results (different prototypes each run)

---

## 2. New Centroid Approach

### 2.1 Architecture Overview

**Flow**:
```
Person → Compute/Retrieve Centroids (cached)
              ↓
    Single Vector Search (Qdrant)
              ↓
    Return CentroidSuggestion[] (direct)
              ↓
    NO FaceSuggestion records created
```

### 2.2 Key Components

**Backend**:
- **Service**: `compute_centroids_for_person()` (centroids/service.py)
- **Qdrant Client**: `CentroidQdrantClient.search_faces_with_centroid()`
- **API Endpoint**: `POST /api/v1/faces/centroids/persons/{personId}/suggestions`
- **Caching**: Centroids stored in DB + Qdrant (staleness detection)

**Frontend**:
- **Dialog**: `ComputeCentroidsDialog.svelte`
- **Results**: `CentroidResultsDialog.svelte` (displays CentroidSuggestion[])
- **Data Flow**: Direct API response → Transform → Display → Accept → Direct assignment

### 2.3 CentroidSuggestion Schema

**API Response**:
```typescript
interface CentroidSuggestion {
  faceInstanceId: string;    // UUID (NO numeric id!)
  assetId: string;           // Asset ID as string
  score: number;             // 0.0-1.0 similarity
  matchedCentroid: string;   // Centroid ID that matched
  thumbnailUrl: string | null;
}

interface CentroidSuggestionResponse {
  personId: string;
  centroidsUsed: string[];        // List of centroid IDs
  suggestions: CentroidSuggestion[];
  totalFound: number;
  rebuiltCentroids: boolean;      // True if centroid was stale
}
```

### 2.4 Current Centroid Integration Points

**Where It's Used**:
1. **Person Detail Page** (`/people/[personId]/+page.svelte`)
   - "Compute Centroids" button → `ComputeCentroidsDialog`
   - Results displayed in `CentroidResultsDialog`
   - Direct accept/reject (no FaceSuggestion records)

**Characteristics**:
- ✅ Synchronous (fast, <1s for 100 faces)
- ✅ Consistent results (centroid is stable)
- ✅ Cached (auto-rebuild on staleness)
- ✅ Single vector search (vs N for prototypes)
- ❌ No FaceSuggestion records (can't use existing UI)
- ❌ No job progress tracking
- ❌ Not integrated with `/faces/suggestions` page

---

## 3. Integration Analysis

### 3.1 Schema Alignment Requirements

To display centroid suggestions in existing UI, we need to map:

| Field | FaceSuggestion | CentroidSuggestion | Solution |
|-------|---------------|-------------------|----------|
| `id` | `number` (PK) | ❌ Missing | Generate temp ID OR create DB record |
| `faceInstanceId` | `string` | `string` ✅ | Direct mapping |
| `suggestedPersonId` | `string` | ❌ Missing | Pass from context |
| `confidence` | `number` | `score` ✅ | Rename field |
| `status` | `'pending'` | ❌ Missing | Always 'pending' |
| `faceThumbnailUrl` | `string \| null` | `thumbnailUrl` ✅ | Direct mapping |
| `fullImageUrl` | `string \| null` | ❌ Missing | Construct from `assetId` |
| `personName` | `string \| null` | ❌ Missing | Pass from context |
| `path` | `string` | ❌ Missing | Fetch from FaceInstance OR skip |
| `bboxX/Y/W/H` | `number \| null` | ❌ Missing | Fetch from FaceInstance OR skip |
| `sourceFaceId` | `string` | `matchedCentroid` ✅ | Use centroid ID |
| `createdAt` | `string` | ❌ Missing | Use `new Date()` client-side |

**Critical Gap**: `id` field required for selection tracking (`Set<number>` in UI)

### 3.2 UI Component Dependencies

**SuggestionGroupCard** expects:
```typescript
interface SuggestionGroup {
  personId: string;
  personName: string | null;
  suggestions: FaceSuggestion[];     // REQUIRES id field!
  pendingCount: number;
  labeledFaceCount?: number;
}
```

**Selection State**:
```typescript
let selectedIds = $state<Set<number>>(new Set());  // Uses FaceSuggestion.id

function handleSelect(id: number, selected: boolean) {
  if (selected) selectedIds.add(id);
  else selectedIds.delete(id);
}
```

**Bulk Actions**:
```typescript
async function handleBulkAction(action: 'accept' | 'reject') {
  await bulkSuggestionAction([...selectedIds], action, { ... });
  // Expects FaceSuggestion IDs
}
```

**Problem**: Centroid suggestions lack numeric `id`, breaking selection logic.

---

## 4. Integration Options

### Option A: Backend Hybrid - Create FaceSuggestion Records from Centroids ✅ RECOMMENDED

**Approach**: New backend endpoint that bridges both worlds.

**Implementation**:
```
POST /api/v1/faces/suggestions/persons/{personId}/find-more-centroid

Request:
{
  "minSimilarity": 0.65,
  "maxResults": 200,
  "unassignedOnly": true
}

Backend Job:
1. Compute/retrieve centroids (cached)
2. Search Qdrant with centroid
3. CREATE FaceSuggestion records for each match:
   - faceInstanceId: from centroid result
   - suggestedPersonId: person_id
   - confidence: centroid score
   - sourceFaceId: centroid_id
   - status: 'pending'
4. Return: FindMoreJobResponse (same as current)
   {
     "jobId": "...",
     "progressKey": "...",
     "status": "queued"
   }

Response Type: FindMoreJobResponse (reuse existing)
```

**Frontend Changes** (Minimal):
```typescript
// FindMoreDialog.svelte - Add mode toggle
let searchMode = $state<'dynamic' | 'centroid'>('centroid');

async function handleSubmit() {
  if (searchMode === 'centroid') {
    response = await startFindMoreCentroidSuggestions(personId, options);
  } else {
    response = await startFindMoreSuggestions(personId, options);
  }
  // Rest is identical (job tracking, results dialog)
}
```

**Benefits**:
- ✅ **ZERO UI component changes** (reuses everything)
- ✅ Seamless migration (swap endpoint, keep workflow)
- ✅ Job progress tracking works as-is
- ✅ FindMoreResultsDialog loads from DB normally
- ✅ Accept/reject actions unchanged
- ✅ Easy A/B testing (toggle between modes)

**Trade-offs**:
- ⚠️ Creates database records (like current system)
- ⚠️ Potential duplicate suggestions if both methods used
- ⚠️ Backend complexity (two similar job implementations)

**Deduplication Strategy**:
```python
# Before creating suggestion, check for existing
existing = await db.execute(
    select(FaceSuggestion).where(
        FaceSuggestion.face_instance_id == face_id,
        FaceSuggestion.suggested_person_id == person_id,
        FaceSuggestion.status == 'pending'
    )
)
if existing.scalar_one_or_none():
    duplicates_skipped += 1
    continue

# Create new suggestion
await db.execute(
    insert(FaceSuggestion).values(
        face_instance_id=face_id,
        suggested_person_id=person_id,
        confidence=score,
        source_face_id=centroid_id,  # NEW: Use centroid ID
        discovery_method='centroid',  # NEW: Track method
        status='pending'
    )
)
```

**Migration Path**:
1. Add `discovery_method` enum field to `FaceSuggestion` (nullable, default NULL)
   - Values: `'dynamic_prototype'`, `'centroid'`, `'auto_assign'`
2. Create `find_more_centroid_suggestions_job()` in `queue/face_jobs.py`
3. Add endpoint `POST /faces/suggestions/persons/{id}/find-more-centroid`
4. Update frontend: Add mode toggle to FindMoreDialog
5. Deploy and monitor (A/B test centroid vs dynamic)

---

### Option B: Client-Side Transformation (Direct Display)

**Approach**: Transform centroid suggestions to FaceSuggestion format in frontend.

**Implementation**:
```typescript
// NEW: CentroidResultsAdapter.svelte
function transformCentroidSuggestion(
  centroidSuggestion: CentroidSuggestion,
  personId: string,
  personName: string,
  tempId: number  // Generated client-side
): FaceSuggestion {
  return {
    id: tempId,  // Temporary ID for selection tracking
    faceInstanceId: centroidSuggestion.faceInstanceId,
    suggestedPersonId: personId,
    personName: personName,
    confidence: centroidSuggestion.score,
    status: 'pending',
    faceThumbnailUrl: centroidSuggestion.thumbnailUrl,
    fullImageUrl: `/api/v1/images/${centroidSuggestion.assetId}/full`,
    sourceFaceId: centroidSuggestion.matchedCentroid,
    createdAt: new Date().toISOString(),

    // Leave optional fields null (fetch on-demand if needed)
    path: '',
    bboxX: null, bboxY: null, bboxW: null, bboxH: null,
    detectionConfidence: null,
    qualityScore: null,
    reviewedAt: null,

    // Multi-prototype fields (not applicable for centroids)
    isMultiPrototypeMatch: false,
    aggregateConfidence: null,
    prototypeMatchCount: null,
    matchingPrototypeIds: null,
    prototypeScores: null
  };
}

// Accept action calls different endpoint
async function handleAcceptCentroid(suggestion: FaceSuggestion) {
  // NO FaceSuggestion record exists, so directly assign
  await assignFaceToPerson(suggestion.faceInstanceId, suggestion.suggestedPersonId);
}
```

**Benefits**:
- ✅ No database bloat (no FaceSuggestion records)
- ✅ Immediate results (synchronous)
- ✅ Clear separation of concerns

**Trade-offs**:
- ❌ Moderate UI changes (new dialog/adapter component)
- ❌ Different accept/reject logic (direct assignment vs DB update)
- ❌ Can't use FindMoreResultsDialog as-is
- ❌ Lost audit trail (no record of what was suggested)
- ❌ Can't review suggestions later (ephemeral)

---

### Option C: Replace Prototype System Entirely

**Approach**: Migrate entire suggestions system to centroids.

**Implementation**:
1. Deprecate `find_more_suggestions_job()` (dynamic prototypes)
2. Update `assign_new_faces()` to use centroids instead of prototypes
3. Remove prototype search from `FaceAssigner`
4. Keep FaceSuggestion records (now always from centroids)

**Benefits**:
- ✅ Single, optimized approach (no confusion)
- ✅ Simpler codebase (one algorithm)
- ✅ Faster suggestions generation

**Trade-offs**:
- ❌ Loss of prototype-based flexibility (edge cases)
- ❌ Large refactoring effort (affects core assignment logic)
- ❌ Breaking change (requires careful migration)
- ❌ Centroid staleness becomes critical issue

**Recommendation**: NOT recommended for initial integration (too disruptive).

---

## 5. Recommended Integration Flow

### 5.1 Proposed User Journey (Option A)

**Scenario**: User wants to find more suggestions for "John Doe"

**Step 1**: Navigate to `/faces/suggestions` page
- Sees pending suggestions grouped by person
- "John Doe" group has 5 pending suggestions

**Step 2**: Click "Find More" button in John Doe's group
- Opens `FindMoreDialog` (updated)
- **NEW**: Dropdown to select search method:
  - **Centroid (Recommended)** ← Default
  - Dynamic Prototypes (Original)

**Step 3**: Configure centroid search
- Min Similarity: 0.65 (slider, default from config)
- Max Results: 200 (dropdown)
- Click "Find Suggestions"

**Step 4**: Background job execution
- POST `/api/v1/faces/suggestions/persons/{id}/find-more-centroid`
- Job enqueued, returns `{ jobId, progressKey }`
- `jobProgressStore` tracks progress via SSE-like polling
- Progress bar shows: "Searching for matches..." (no percentage for centroids)

**Step 5**: Job completion
- Dialog shows results:
  - **42 new suggestions created**
  - Centroid: `global` (1 centroid used)
  - 68 candidates found, 26 duplicates skipped
- User clicks "View Suggestions"

**Step 6**: Review suggestions
- `FindMoreResultsDialog` opens (unchanged)
- Loads suggestions via `listGroupedSuggestions({ personId: "...", status: "pending" })`
- Displays 42 new suggestions in `SuggestionGroupCard`
- **NEW**: Badge shows "Centroid Match" (from `discovery_method` field)

**Step 7**: Accept suggestions
- User bulk selects 35 suggestions
- Clicks "Accept All"
- Updates `FaceSuggestion` records: `status='accepted'`, `person_id` assigned
- **Exactly same as current workflow**

### 5.2 Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ USER INTERACTION                                                 │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreDialog (ENHANCED)                                        │
│ - Search Method: [Centroid ▼] (new dropdown)                    │
│ - Min Similarity: 0.65                                           │
│ - Max Results: 200                                               │
│ - Click "Find Suggestions"                                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FRONTEND API CALL                                                │
│ startFindMoreCentroidSuggestions(personId, options)             │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ BACKEND: find_more_centroid_suggestions_job() (NEW)             │
│                                                                  │
│ 1. Get/Compute Centroid                                         │
│    - Check staleness → rebuild if needed                        │
│    - Load centroid vector from Qdrant                           │
│                                                                  │
│ 2. Search Faces with Centroid                                   │
│    - Single Qdrant search (fast!)                               │
│    - Filter: score >= minSimilarity, unassignedOnly=true        │
│    - Deduplicate by faceInstanceId                              │
│                                                                  │
│ 3. CREATE FaceSuggestion Records                                │
│    FOR EACH match:                                              │
│      - Check existing suggestion (skip duplicates)              │
│      - INSERT INTO face_suggestions:                            │
│          face_instance_id = match.faceInstanceId                │
│          suggested_person_id = personId                         │
│          confidence = match.score                               │
│          source_face_id = centroid_id                           │
│          discovery_method = 'centroid'                          │
│          status = 'pending'                                     │
│                                                                  │
│ 4. Return Job Result                                            │
│    {                                                             │
│      suggestionsCreated: 42,                                    │
│      centroidsUsed: 1,                                          │
│      candidatesFound: 68,                                       │
│      duplicatesSkipped: 26                                      │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FRONTEND: Job Progress Tracking                                 │
│ jobProgressStore.trackJob(jobId, progressKey, ...)             │
│ - Polls Redis for job status                                    │
│ - Updates FindMoreDialog with progress                          │
│ - Receives job.result on completion                             │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreDialog - Results Display                                │
│ - Shows: 42 suggestions created, 1 centroid used                │
│ - User clicks "View Suggestions"                                │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreResultsDialog (UNCHANGED)                               │
│ - Calls: listGroupedSuggestions({ personId, status: pending }) │
│ - Loads FaceSuggestion records FROM DATABASE                    │
│ - Renders via SuggestionGroupCard                               │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ SuggestionGroupCard (ENHANCED)                                   │
│ - Display suggestions with thumbnails                            │
│ - NEW: Badge shows "Centroid Match" if discovery_method='centroid'│
│ - Actions: Accept, Reject, Bulk operations (UNCHANGED)          │
│ - Updates FaceSuggestion.status in DB (UNCHANGED)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Technical Changes Required

### 6.1 Backend Changes

#### Database Migration
```python
# Migration: Add discovery_method field to face_suggestions
op.add_column(
    'face_suggestions',
    sa.Column(
        'discovery_method',
        sa.Enum('dynamic_prototype', 'centroid', 'auto_assign', name='discovery_method_enum'),
        nullable=True
    )
)

# Backfill existing records (optional)
op.execute(
    "UPDATE face_suggestions SET discovery_method = 'dynamic_prototype' WHERE discovery_method IS NULL"
)
```

#### New Background Job
**File**: `src/image_search_service/queue/face_jobs.py`

```python
@job('default', timeout=300)
async def find_more_centroid_suggestions_job(
    person_id: UUID,
    min_similarity: float = 0.65,
    max_results: int = 200,
    unassigned_only: bool = True
) -> dict:
    """Find face suggestions using centroid-based search."""

    async with async_session_maker() as db:
        # 1. Get/compute centroids
        centroids_result = await compute_centroids_for_person(
            db, person_id, force_rebuild=False
        )
        if not centroids_result.centroids:
            raise ValueError("No centroids available")

        centroid = centroids_result.centroids[0]  # Use global centroid

        # 2. Search faces with centroid
        centroid_qdrant = _get_centroid_qdrant()
        face_qdrant = _get_face_qdrant()

        centroid_vector = await centroid_qdrant.get_centroid_vector(
            str(centroid.centroid_id)
        )

        scored_points = await face_qdrant.search_faces_with_centroid(
            centroid_vector=centroid_vector,
            limit=max_results * 2,  # Over-fetch for filtering
            score_threshold=min_similarity
        )

        # 3. Filter unassigned faces
        suggestions_created = 0
        duplicates_skipped = 0
        candidates_found = len(scored_points)

        for point in scored_points:
            face_id = UUID(point.id)

            # Check if already suggested
            existing = await db.execute(
                select(FaceSuggestion).where(
                    FaceSuggestion.face_instance_id == face_id,
                    FaceSuggestion.suggested_person_id == person_id,
                    FaceSuggestion.status == 'pending'
                )
            )
            if existing.scalar_one_or_none():
                duplicates_skipped += 1
                continue

            # Check if unassigned
            if unassigned_only:
                face_instance = await db.get(FaceInstance, face_id)
                if face_instance and face_instance.person_id is not None:
                    continue

            # Create suggestion
            suggestion = FaceSuggestion(
                face_instance_id=face_id,
                suggested_person_id=person_id,
                confidence=point.score,
                source_face_id=centroid.centroid_id,  # Use centroid as source
                discovery_method='centroid',
                status='pending'
            )
            db.add(suggestion)
            suggestions_created += 1

            if suggestions_created >= max_results:
                break

        await db.commit()

        return {
            'suggestionsCreated': suggestions_created,
            'centroidsUsed': 1,
            'candidatesFound': candidates_found,
            'duplicatesSkipped': duplicates_skipped
        }
```

#### New API Endpoint
**File**: `src/image_search_service/api/routes/face_suggestions.py`

```python
@router.post(
    "/persons/{person_id}/find-more-centroid",
    response_model=FindMoreJobResponse
)
async def find_more_centroid_suggestions(
    person_id: UUID,
    request: FindMoreCentroidRequest,
    db: AsyncSession = Depends(get_db)
) -> FindMoreJobResponse:
    """
    Find face suggestions using centroid-based search.

    Faster and more consistent than dynamic prototype search.
    """
    # Validate person exists
    person = await db.get(Person, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")

    # Check minimum faces
    labeled_count = await get_person_labeled_face_count(db, person_id)
    if labeled_count < 2:
        raise HTTPException(
            status_code=422,
            detail=f"Person has only {labeled_count} faces, need at least 2"
        )

    # Enqueue job
    job_uuid = uuid4()
    progress_key = f"find_more:progress:{person_id}:{job_uuid}"

    job = queue.enqueue(
        find_more_centroid_suggestions_job,
        person_id=person_id,
        min_similarity=request.min_similarity,
        max_results=request.max_results,
        unassigned_only=request.unassigned_only,
        job_id=str(job_uuid),
        meta={'progress_key': progress_key}
    )

    return FindMoreJobResponse(
        job_id=str(job.id),
        person_id=str(person_id),
        person_name=person.name or "Unknown",
        prototype_count=1,  # Centroids use 1 "prototype" (the centroid)
        labeled_face_count=labeled_count,
        status="queued",
        progress_key=progress_key
    )
```

#### New Request Schema
```python
class FindMoreCentroidRequest(BaseModel):
    min_similarity: float = Field(default=0.65, ge=0.5, le=0.95)
    max_results: int = Field(default=200, ge=1, le=500)
    unassigned_only: bool = Field(default=True)
```

### 6.2 Frontend Changes

#### Update FindMoreDialog
**File**: `src/lib/components/faces/FindMoreDialog.svelte`

**Changes**:
1. Add search mode toggle (already exists in current implementation!)
2. Call appropriate endpoint based on mode
3. Job tracking works identically for both

**Code** (Current implementation already supports this!):
```typescript
let searchMode = $state<'centroid' | 'dynamic'>('centroid');

async function handleSubmit() {
  if (searchMode === 'centroid') {
    response = await startFindMoreCentroidSuggestions(personId, {
      minSimilarity: parseFloat(selectedThresholdStr),
      maxResults: 200,
      unassignedOnly: true
    });
  } else {
    response = await startFindMoreSuggestions(personId, {
      prototypeCount: actualCount,
      minConfidence: parseFloat(selectedThresholdStr)
    });
  }
  // Identical job tracking from here
}
```

#### Add API Client Function
**File**: `src/lib/api/faces.ts`

```typescript
export async function startFindMoreCentroidSuggestions(
  personId: string,
  options: {
    minSimilarity: number;
    maxResults: number;
    unassignedOnly: boolean;
  }
): Promise<FindMoreJobResponse> {
  const response = await fetch(
    `/api/v1/faces/suggestions/persons/${personId}/find-more-centroid`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        minSimilarity: options.minSimilarity,
        maxResults: options.maxResults,
        unassignedOnly: options.unassignedOnly
      })
    }
  );

  if (!response.ok) {
    throw new Error(`Failed to start centroid search: ${response.statusText}`);
  }

  return response.json();
}
```

#### Optional: Add Discovery Method Badge
**File**: `src/lib/components/faces/SuggestionGroupCard.svelte`

```svelte
<!-- In suggestion thumbnail display -->
{#if suggestion.discoveryMethod === 'centroid'}
  <span class="badge badge-centroid" title="Found via centroid search">
    Centroid Match
  </span>
{:else if suggestion.discoveryMethod === 'dynamic_prototype'}
  <span class="badge badge-prototype" title="Found via dynamic prototype">
    Prototype Match
  </span>
{/if}
```

---

## 7. Migration Timeline

### Phase 1: Backend Implementation (3-4 hours)
- [ ] Create database migration for `discovery_method` field
- [ ] Implement `find_more_centroid_suggestions_job()` in `queue/face_jobs.py`
- [ ] Add `FindMoreCentroidRequest` schema
- [ ] Create endpoint `POST /faces/suggestions/persons/{id}/find-more-centroid`
- [ ] Add API client function `startFindMoreCentroidSuggestions()`
- [ ] Run migrations: `make migrate`
- [ ] Test endpoint manually via Swagger

### Phase 2: Frontend Integration (1-2 hours)
- [ ] Verify FindMoreDialog already has mode toggle (it does!)
- [ ] Update API client function in `lib/api/faces.ts`
- [ ] Update API contract docs: `docs/api-contract.md`
- [ ] Regenerate TypeScript types: `npm run gen:api`
- [ ] Test FindMoreDialog with centroid mode

### Phase 3: UI Enhancements (1 hour)
- [ ] Add discovery method badge to `SuggestionGroupCard`
- [ ] Update CSS for badge styling
- [ ] Test suggestion display with both methods

### Phase 4: Testing (2 hours)
- [ ] Manual testing: Person detail page
- [ ] Manual testing: Suggestions page
- [ ] Manual testing: Bulk accept with auto-find-more
- [ ] Edge cases: No faces, stale centroids, duplicate suggestions
- [ ] Performance testing: Large person datasets (1000+ faces)

### Phase 5: Documentation (1 hour)
- [ ] Update README with centroid approach
- [ ] Document configuration options
- [ ] Add admin guide: When to use centroids vs prototypes
- [ ] Update API documentation

**Total Estimated Time**: 8-10 hours

---

## 8. Performance Comparison

### Dynamic Prototypes (Current)
- **Latency**: 5-30 seconds (depends on prototype count)
- **Consistency**: Variable (random sampling each run)
- **Database Queries**: N searches (one per prototype)
- **Qdrant Searches**: N vector searches
- **Scalability**: Slows with more labeled faces

### Centroids (New)
- **Latency**: <1 second (single search, cached centroid)
- **Consistency**: Deterministic (same centroid each time)
- **Database Queries**: 1 centroid lookup + 1 suggestion batch insert
- **Qdrant Searches**: 1 vector search
- **Scalability**: Fast regardless of face count

**Expected Improvement**: **10-30x faster** for persons with 100+ faces

---

## 9. Known Limitations and Future Work

### Current Limitations

1. **Single Centroid Only**
   - Currently uses global centroid (average of all faces)
   - Doesn't leverage cluster-based centroids (HDBSCAN)
   - **Future**: Multi-centroid search for persons with diverse appearances

2. **No Prototype Exclusion**
   - `exclude_prototypes` filter not implemented (missing `is_prototype` field)
   - **Future**: Add migration to mark prototype faces

3. **Discovery Method Filtering**
   - Can't filter suggestions by how they were discovered
   - **Future**: Add filter dropdown in UI (All / Centroid / Prototype)

4. **Centroid Staleness Monitoring**
   - No dashboard showing which persons have stale centroids
   - **Future**: Admin panel with centroid health metrics

### Future Enhancements

1. **Hybrid Scoring**
   - Combine centroid score + prototype scores for better confidence
   - **Algorithm**: `final_score = max(centroid_score, max(prototype_scores))`

2. **Active Learning**
   - Track acceptance rates by discovery method
   - Adjust thresholds based on user feedback
   - Auto-switch to better method per person

3. **Background Centroid Rebuild**
   - Periodic job to rebuild stale centroids
   - Triggered after bulk face assignments
   - **Frequency**: Daily or after N new assignments

4. **Multi-Centroid Support**
   - Use cluster-based centroids for diverse persons
   - Search with each cluster centroid, merge results
   - **Use case**: Persons with appearance changes (age, beard, glasses)

---

## 10. Decision Matrix

### When to Use Centroid vs Prototype Search

| Criterion | Use Centroids | Use Dynamic Prototypes |
|-----------|--------------|----------------------|
| **Person Face Count** | 50+ faces | 10-49 faces |
| **Appearance Diversity** | Consistent appearance | Diverse (age changes, accessories) |
| **Speed Priority** | High (need fast results) | Low (can wait 30s) |
| **Consistency Need** | High (reproducible results) | Low (exploration mode) |
| **Centroid Staleness** | Fresh (< 7 days) | Stale (> 30 days) |
| **First-Time Search** | Yes (faster initial run) | No (less flexible) |
| **Follow-Up Search** | Yes (cached centroid) | Yes (sample different prototypes) |

**Recommended Default**: **Centroid** (95% of use cases)

**When to Use Dynamic**:
- Person has <50 faces AND diverse appearance
- Exploring edge cases (low confidence threshold)
- Centroid consistently returns poor results (rare)

---

## 11. Success Metrics

### Performance Metrics
- **Latency**: Centroid search completes in <1s (vs 5-30s for prototypes)
- **Cache Hit Rate**: Centroids reused without rebuild in >80% of searches
- **Throughput**: Support 10+ concurrent find-more requests

### Quality Metrics
- **Precision**: Acceptance rate >= prototype method (target: >70%)
- **Recall**: Unique faces found (centroid vs prototype comparison)
- **User Satisfaction**: Preference survey (faster results vs slightly lower recall)

### Operational Metrics
- **Centroid Coverage**: >90% of active persons have fresh centroids
- **Job Failure Rate**: <5% of find-more jobs fail
- **Database Growth**: FaceSuggestion table growth rate (monitor duplication)

---

## 12. Rollout Plan

### Week 1-2: Parallel Deployment
- Deploy backend hybrid endpoint alongside existing
- Enable centroid mode in FindMoreDialog for beta users (via feature flag)
- Monitor: Success rate, latency, user feedback

### Week 3-4: Gradual Rollout
- **Criteria-Based Default**:
  - Persons with >200 faces: Default to centroid
  - Persons with 50-200 faces: Show both options, recommend centroid
  - Persons with <50 faces: Show both options, no recommendation
- A/B test: 50% centroid, 50% dynamic (same person groups)

### Week 5-6: Optimization
- Analyze acceptance rates by method
- Optimize centroid computation (caching, clustering)
- Add background centroid rebuild job

### Month 3+: Deprecation (Optional)
- If centroid proves superior, make it default everywhere
- Keep dynamic as "Advanced Mode" option
- Remove UI toggle, auto-select best method per person

---

## 13. Conclusion

**Recommended Integration Path**: **Option A - Backend Hybrid Endpoint**

**Summary**:
- ✅ **Minimal Frontend Changes**: Reuse all existing UI components
- ✅ **Seamless User Experience**: Same workflow, faster results
- ✅ **Easy Migration**: Gradual rollout with A/B testing
- ✅ **Backward Compatible**: Dynamic prototypes still available
- ✅ **Performance Gain**: 10-30x faster for large person datasets
- ✅ **Audit Trail**: FaceSuggestion records preserve what was suggested

**Key Innovation**: Bridging centroid-based search with existing suggestion workflow by creating FaceSuggestion records from centroid results, enabling complete UI reuse.

**Next Steps**:
1. Implement backend hybrid endpoint (3-4 hours)
2. Update frontend API client (1 hour)
3. Test both search methods (2 hours)
4. Deploy and monitor (ongoing)

**Estimated Total Effort**: 8-10 hours for complete integration

---

## Appendix A: API Contract Example

### Request: Start Centroid-Based Find More

```http
POST /api/v1/faces/suggestions/persons/550e8400-e29b-41d4-a716-446655440000/find-more-centroid
Content-Type: application/json

{
  "minSimilarity": 0.65,
  "maxResults": 200,
  "unassignedOnly": true
}
```

### Response: Job Info

```json
{
  "jobId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "personId": "550e8400-e29b-41d4-a716-446655440000",
  "personName": "John Doe",
  "prototypeCount": 1,
  "labeledFaceCount": 234,
  "status": "queued",
  "progressKey": "find_more:progress:550e8400-e29b-41d4-a716-446655440000:f47ac10b-58cc-4372-a567-0e02b2c3d479"
}
```

### Job Result (via progress tracking)

```json
{
  "suggestionsCreated": 42,
  "centroidsUsed": 1,
  "candidatesFound": 68,
  "duplicatesSkipped": 26
}
```

---

## Appendix B: Database Schema Changes

### Migration: Add discovery_method field

```python
"""Add discovery_method to face_suggestions

Revision ID: abc123def456
Revises: previous_revision
Create Date: 2026-01-17 12:00:00.000000
"""

from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    # Create enum type
    op.execute(
        "CREATE TYPE discovery_method_enum AS ENUM ('dynamic_prototype', 'centroid', 'auto_assign')"
    )

    # Add column
    op.add_column(
        'face_suggestions',
        sa.Column('discovery_method', sa.Enum('dynamic_prototype', 'centroid', 'auto_assign', name='discovery_method_enum'), nullable=True)
    )

    # Optional: Backfill existing records
    op.execute(
        "UPDATE face_suggestions SET discovery_method = 'dynamic_prototype' WHERE discovery_method IS NULL"
    )

def downgrade() -> None:
    op.drop_column('face_suggestions', 'discovery_method')
    op.execute("DROP TYPE discovery_method_enum")
```

---

**Document Version**: 1.0
**Author**: Research Agent
**Last Updated**: 2026-01-17
