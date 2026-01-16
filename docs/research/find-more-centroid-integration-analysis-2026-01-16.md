# Find More Suggestions - Centroid Integration Analysis

**Date**: 2026-01-16
**Purpose**: Analyze current "Find More" implementation to plan integration with centroid-based approach
**Status**: Research Complete

---

## Executive Summary

The "Find More Suggestions" feature uses dynamic prototype sampling to discover additional face matches. This research analyzes the current implementation across frontend and backend to enable integration with the new centroid-based approach.

**Key Findings**:
- Current: Job-based approach creates FaceSuggestion records via background worker
- New: Centroid endpoint returns suggestions directly (no DB records)
- Integration Path: Adapt FindMoreResultsDialog to accept direct suggestions OR create FaceSuggestion records from centroid results
- Reusability: SuggestionGroupCard can display both types with minor schema alignment

---

## 1. Frontend Components Analysis

### 1.1 FindMoreDialog.svelte

**Location**: `image-search-ui/src/lib/components/faces/FindMoreDialog.svelte`

**Purpose**: Dialog for configuring and launching "Find More" job

**Props Interface**:
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

**Key Features**:
- **Prototype Count Selection**: 10, 50, 100, 1000, or "All" faces
- **Similarity Threshold**: 0.55-0.70 range (configurable via admin API)
- **Job Progress Tracking**: Uses `jobProgressStore` with SSE-like polling
- **Results Display**: Shows 4-metric summary (created, candidates, prototypes used, duplicates)
- **Results Action**: Opens FindMoreResultsDialog on "View Suggestions" click

**State Flow**:
1. **Options** → User configures prototype count and threshold
2. **Computing** → API call starts job, receives `jobId` and `progressKey`
3. **Progress** → Tracks job via `jobProgressStore.trackJob()`
4. **Results** → Shows summary stats from job completion
5. **View** → Opens FindMoreResultsDialog with `personId` filter

**API Integration**:
```typescript
// Calls: POST /api/v1/faces/suggestions/persons/{personId}/find-more
const response = await startFindMoreSuggestions(personId, {
  prototypeCount: actualCount,
  minConfidence: parseFloat(selectedThresholdStr)
});
// Returns: { jobId, progressKey, personId, personName, ... }
```

**Job Result Schema**:
```typescript
jobResults = {
  suggestionsCreated: number;  // NEW FaceSuggestion records
  prototypesUsed: number;      // Random samples used
  candidatesFound: number;     // Total matches found
  duplicatesSkipped: number;   // Already-existing suggestions
}
```

---

### 1.2 FindMoreResultsDialog.svelte

**Location**: `image-search-ui/src/lib/components/faces/FindMoreResultsDialog.svelte`

**Purpose**: Display and review suggestions created by Find More job

**Props Interface**:
```typescript
interface Props {
  open: boolean;
  personId: string;
  personName: string;
  onClose: () => void;
}
```

**Key Features**:
- **Auto-loads suggestions** via `listGroupedSuggestions({ personId, status: 'pending' })`
- **Single-group display** (first group only: `groupsPerPage: 1, suggestionsPerGroup: 100`)
- **Delegates to SuggestionGroupCard** for actual suggestion display
- **Actions**: Accept/reject individual or bulk via SuggestionGroupCard callbacks

**Data Flow**:
```
FindMoreResultsDialog (loads from DB)
    ↓
listGroupedSuggestions({ personId: "...", status: "pending" })
    ↓
GroupedSuggestionsResponse { groups: [...] }
    ↓
SuggestionGroupCard (renders suggestions with actions)
```

**Critical Dependency**: Expects FaceSuggestion records to exist in database

---

### 1.3 SuggestionGroupCard.svelte

**Location**: `image-search-ui/src/lib/components/faces/SuggestionGroupCard.svelte`

**Purpose**: Reusable component for displaying and managing face suggestions for a person

**Props Interface**:
```typescript
interface SuggestionGroup {
  personId: string;
  personName: string | null;
  suggestions: FaceSuggestion[];
  pendingCount: number;
  labeledFaceCount?: number;  // Optional, enables "Find More" button
}

interface Props {
  group: SuggestionGroup;
  selectedIds: Set<number>;
  onSelect: (id: number, selected: boolean) => void;
  onSelectAllInGroup: (ids: number[], selected: boolean) => void;
  onAcceptAll: (ids: number[]) => Promise<void>;
  onRejectAll: (ids: number[]) => Promise<void>;
  onThumbnailClick: (suggestion: FaceSuggestion) => void;
  onFindMoreComplete?: () => void;  // Optional callback
}
```

**Key Features**:
- **Bulk Selection**: Checkbox for select-all with indeterminate state
- **Sorting**: Multi-prototype matches first, then by confidence
- **Actions**: Accept All, Reject All, individual selection
- **Find More Integration**: Shows "Find More" button if `labeledFaceCount >= 10`
- **Internal Find More Dialog**: Launches FindMoreDialog inline

**FaceSuggestion Schema** (Expected):
```typescript
interface FaceSuggestion {
  id: number;                          // Primary key for DB record
  faceInstanceId: string;              // UUID
  suggestedPersonId: string;           // UUID
  confidence: number;                  // 0.0-1.0
  status: 'pending' | 'accepted' | 'rejected' | 'expired';
  faceThumbnailUrl: string | null;
  personName: string | null;

  // Multi-prototype scoring (optional)
  isMultiPrototypeMatch?: boolean;
  aggregateConfidence?: number | null;
  prototypeMatchCount?: number | null;
  matchingPrototypeIds?: string[] | null;
  prototypeScores?: Record<string, number> | null;

  // Bounding box (for detail modal)
  bboxX: number | null;
  bboxY: number | null;
  bboxW: number | null;
  bboxH: number | null;

  // Other metadata
  path: string;
  fullImageUrl: string | null;
  detectionConfidence: number | null;
  qualityScore: number | null;
  createdAt: string;
  reviewedAt: string | null;
  sourceFaceId: string;
}
```

**Reusability for Centroid Suggestions**: HIGH
- Core display logic is data-agnostic
- Needs `id` field for selection tracking
- Could adapt to work with transformed centroid data

---

### 1.4 ComputeCentroidsDialog.svelte

**Location**: `image-search-ui/src/lib/components/faces/ComputeCentroidsDialog.svelte`

**Purpose**: Dialog for computing centroids and retrieving centroid-based suggestions

**Props Interface**:
```typescript
interface Props {
  open: boolean;
  personId: string;
  personName: string;
  labeledFaceCount: number;
  onClose: () => void;
  onSuggestionsReady?: (suggestions: CentroidSuggestion[]) => void;
}
```

**Key Features**:
- **Options Step**: Configure clustering, min similarity (0.5-0.85), max results, unassigned-only
- **Computing Step**: Calls `computeCentroids()` API
- **Searching Step**: Calls `getCentroidSuggestions()` API
- **Results Step**: Shows centroid summary + suggestion count
- **Callback**: `onSuggestionsReady(suggestions)` passes raw centroid suggestions to parent

**State Flow**:
```
options → computing → searching → results
```

**API Integration**:
```typescript
// Step 1: Compute centroids
const result = await computeCentroids(personId, {
  forceRebuild: false,
  enableClustering: enableClustering && canCluster,
  minFaces: 2
});
centroids = result.centroids;

// Step 2: Get suggestions using centroids
const suggestionsResult = await getCentroidSuggestions(personId, {
  minSimilarity,
  maxResults,
  unassignedOnly,
  excludePrototypes: true,
  autoRebuild: false
});
suggestions = suggestionsResult.suggestions;
```

**CentroidSuggestion Schema** (From API):
```typescript
interface CentroidSuggestion {
  faceInstanceId: string;    // UUID (note: different from FaceSuggestion.id)
  assetId: string;           // Asset ID as string
  score: number;             // Similarity score 0.0-1.0
  matchedCentroid: string;   // Centroid ID that matched
  thumbnailUrl: string | null;
}
```

**Integration Gap**:
- Returns `CentroidSuggestion[]` via callback
- Does NOT create FaceSuggestion records in database
- Parent component must decide what to do with suggestions

---

## 2. Backend Endpoints Analysis

### 2.1 Find More Endpoint (Current)

**Endpoint**: `POST /api/v1/faces/suggestions/persons/{person_id}/find-more`
**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

**Request Schema**:
```python
class FindMoreSuggestionsRequest(BaseModel):
    prototype_count: int = Field(default=50, ge=10, le=1000)
    max_suggestions: int = Field(default=100, ge=1, le=500)
    min_confidence: float | None = Field(default=None, ge=0.3, le=1.0)
```

**Response Schema**:
```python
class FindMoreJobResponse(BaseModel):
    job_id: str              # RQ job ID (UUID)
    person_id: str           # Person UUID
    person_name: str
    prototype_count: int     # Actual count used
    labeled_face_count: int  # Total faces available
    status: str              # "queued"
    progress_key: str        # Redis key for progress tracking
```

**Behavior**:
1. **Validates** person exists and has >= 10 labeled faces
2. **Enqueues** `find_more_suggestions_job` with parameters
3. **Returns** job info immediately (non-blocking)
4. **Job creates** FaceSuggestion records in database asynchronously
5. **Frontend polls** job progress via `progress_key`

**Job Implementation** (Background):
- Samples random labeled faces (weighted by quality)
- Searches Qdrant for similar faces using each prototype
- Deduplicates results
- Creates FaceSuggestion records with `source_face_id` = prototype used
- Tracks: `suggestionsCreated`, `prototypesUsed`, `candidatesFound`, `duplicatesSkipped`

**Key Characteristics**:
- ✅ Asynchronous (non-blocking)
- ✅ Creates persistent FaceSuggestion records
- ✅ Integrates with existing suggestion review workflow
- ✅ Progress tracking built-in
- ❌ No clustering/centroid optimization
- ❌ Re-runs from scratch each time

---

### 2.2 Centroid Suggestions Endpoint (New)

**Endpoint**: `POST /api/v1/faces/centroids/persons/{person_id}/suggestions`
**File**: `image-search-service/src/image_search_service/api/routes/face_centroids.py`

**Request Schema**:
```python
class CentroidSuggestionRequest(BaseModel):
    min_similarity: float = Field(default=0.65, ge=0.0, le=1.0)
    max_results: int = Field(default=200, ge=1, le=500)
    unassigned_only: bool = Field(default=True)
    exclude_prototypes: bool = Field(default=True)
    auto_rebuild: bool = Field(default=True)  # Rebuild if stale
```

**Response Schema**:
```python
class CentroidSuggestionResponse(BaseModel):
    person_id: UUID
    centroids_used: list[str]        # Centroid IDs used in search
    suggestions: list[CentroidSuggestion]
    total_found: int
    rebuilt_centroids: bool          # Whether centroids were rebuilt

class CentroidSuggestion(BaseModel):
    face_instance_id: UUID
    asset_id: str
    score: float
    matched_centroid: str
    thumbnail_url: str | None
```

**Behavior**:
1. **Gets or computes** centroid (auto-rebuilds if stale and `auto_rebuild=True`)
2. **Searches** faces collection in Qdrant using centroid vector
3. **Filters** results (unassigned only, exclude prototypes)
4. **Returns** suggestions immediately (synchronous)
5. **Does NOT** create FaceSuggestion database records

**Key Characteristics**:
- ✅ Synchronous (immediate results)
- ✅ Uses optimized centroid representation
- ✅ Cached centroids (rebuild only when stale)
- ✅ Auto-rebuild support
- ❌ No FaceSuggestion records created
- ❌ No progress tracking (fast enough to be synchronous)
- ❌ No job result summary

---

## 3. Data Flow Comparison

### 3.1 Current Find More Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ User Interaction                                                 │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreDialog                                                   │
│ - Configure: prototypeCount, minConfidence                      │
│ - Click "Find Suggestions"                                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Frontend API Call                                                │
│ startFindMoreSuggestions(personId, { prototypeCount, ... })    │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Backend: POST /faces/suggestions/persons/{id}/find-more         │
│ - Validates person (>= 10 faces)                                │
│ - Enqueues find_more_suggestions_job()                          │
│ - Returns: { jobId, progressKey, ... }                          │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Background Worker (RQ)                                           │
│ find_more_suggestions_job:                                       │
│ 1. Sample random labeled faces (weighted by quality)            │
│ 2. For each prototype:                                           │
│    - Search Qdrant for similar faces                            │
│    - Filter by minConfidence                                     │
│ 3. Deduplicate results                                           │
│ 4. CREATE FaceSuggestion records in DB                          │
│ 5. Update progress in Redis                                     │
│ 6. Return: { suggestionsCreated, prototypesUsed, ... }          │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Frontend Progress Tracking                                       │
│ jobProgressStore.trackJob(jobId, progressKey, ...)             │
│ - Polls job status via progressKey                              │
│ - Updates FindMoreDialog UI with progress                        │
│ - Receives completion with job.result                            │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreDialog - Results Display                                │
│ - Shows: suggestionsCreated, candidatesFound, etc.              │
│ - User clicks "View Suggestions"                                │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreResultsDialog                                            │
│ - Calls: listGroupedSuggestions({ personId, status: pending })  │
│ - Loads FaceSuggestion records FROM DATABASE                    │
│ - Renders via SuggestionGroupCard                               │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ SuggestionGroupCard                                              │
│ - Display suggestions with thumbnails                            │
│ - Actions: Accept, Reject, Bulk operations                      │
│ - Updates FaceSuggestion status in DB                           │
└─────────────────────────────────────────────────────────────────┘
```

**Key Points**:
- Creates **persistent** FaceSuggestion records
- Uses **job queue** for async processing
- Progress tracking via Redis
- Results displayed from DB query

---

### 3.2 Proposed Centroid Flow (Option A: Direct Display)

```
┌─────────────────────────────────────────────────────────────────┐
│ User Interaction                                                 │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ ComputeCentroidsDialog                                           │
│ - Configure: minSimilarity, maxResults, unassignedOnly          │
│ - Click "Compute & Search"                                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Compute Centroids                                        │
│ computeCentroids(personId, { forceRebuild, enableClustering })  │
│ → Backend computes/retrieves centroid                           │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Get Suggestions                                          │
│ getCentroidSuggestions(personId, { minSimilarity, ... })        │
│ → Backend searches Qdrant with centroid                         │
│ → Returns: CentroidSuggestion[] (NO DB records)                 │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ ComputeCentroidsDialog - Results                                │
│ - Shows: N suggestions found, centroid summary                  │
│ - User clicks "View Suggestions"                                │
│ - Calls: onSuggestionsReady(centroidSuggestions)                │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ NEW: CentroidResultsDialog (NEEDS TO BE BUILT)                  │
│ - Receives: CentroidSuggestion[] directly                       │
│ - Transforms to SuggestionGroup format                          │
│ - Maps: faceInstanceId → temporary "id" for selection           │
│ - Renders via ADAPTED SuggestionGroupCard                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ SuggestionGroupCard (ADAPTED)                                    │
│ - Display centroid suggestions with thumbnails                  │
│ - Actions call DIFFERENT endpoints:                             │
│   Accept → assignFaceToPerson(faceInstanceId, personId)         │
│   Reject → (no-op or track in local storage)                    │
│ - NO FaceSuggestion DB records                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key Points**:
- **NO** persistent FaceSuggestion records
- **Synchronous** (fast centroid search)
- Results passed directly via callback
- Requires new dialog or adapter component

---

### 3.3 Proposed Centroid Flow (Option B: Create FaceSuggestions)

```
┌─────────────────────────────────────────────────────────────────┐
│ User Interaction                                                 │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ ComputeCentroidsDialog                                           │
│ - Configure: minSimilarity, maxResults, unassignedOnly          │
│ - Click "Compute & Search"                                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ Backend: NEW Endpoint                                            │
│ POST /faces/suggestions/persons/{id}/find-more-centroid         │
│ 1. Compute/retrieve centroids                                    │
│ 2. Search Qdrant with centroid                                  │
│ 3. CREATE FaceSuggestion records for results                    │
│ 4. Return: { suggestionsCreated, centroidUsed, ... }            │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ ComputeCentroidsDialog - Results                                │
│ - Shows: N suggestions created                                  │
│ - User clicks "View Suggestions"                                │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ FindMoreResultsDialog (REUSED)                                   │
│ - Calls: listGroupedSuggestions({ personId, status: pending })  │
│ - Loads FaceSuggestion records FROM DATABASE                    │
│ - Renders via SuggestionGroupCard                               │
│ - UNCHANGED - works exactly as before                           │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ SuggestionGroupCard (UNCHANGED)                                  │
│ - Display suggestions with thumbnails                            │
│ - Actions: Accept, Reject, Bulk operations                      │
│ - Updates FaceSuggestion status in DB                           │
└─────────────────────────────────────────────────────────────────┘
```

**Key Points**:
- Creates **persistent** FaceSuggestion records (like current)
- **Synchronous** or async (depending on implementation)
- Reuses existing FindMoreResultsDialog
- Minimal frontend changes required

---

## 4. API Response Schema Comparison

### 4.1 Find More Job Response

**Endpoint**: `POST /api/v1/faces/suggestions/persons/{person_id}/find-more`

```typescript
interface FindMoreJobResponse {
  jobId: string;              // "550e8400-e29b-41d4-a716-446655440000"
  personId: string;           // UUID
  personName: string;         // "John Doe"
  prototypeCount: number;     // 50 (actual count used)
  labeledFaceCount: number;   // 234 (total available)
  status: string;             // "queued"
  progressKey: string;        // "find_more:progress:{person_id}:{job_id}"
}

// Job Result (from progress tracking):
interface FindMoreJobResult {
  suggestionsCreated: number;   // 42
  prototypesUsed: number;       // 50
  candidatesFound: number;      // 68
  duplicatesSkipped: number;    // 26
}
```

**Usage Pattern**:
```typescript
// 1. Start job
const jobInfo = await startFindMoreSuggestions(personId, { prototypeCount: 50 });

// 2. Track progress
jobProgressStore.trackJob(
  jobInfo.jobId,
  jobInfo.progressKey,
  personId,
  personName,
  onComplete,  // Receives job.result with counts
  onError
);

// 3. View results (from DB)
await listGroupedSuggestions({ personId, status: 'pending' });
```

---

### 4.2 Centroid Suggestions Response

**Endpoint**: `POST /api/v1/faces/centroids/persons/{person_id}/suggestions`

```typescript
interface CentroidSuggestionResponse {
  personId: string;                    // UUID
  centroidsUsed: string[];             // ["centroid-abc123"]
  suggestions: CentroidSuggestion[];   // Direct array
  totalFound: number;                  // 42
  rebuiltCentroids: boolean;           // true if centroid was stale
}

interface CentroidSuggestion {
  faceInstanceId: string;    // UUID - NO "id" field!
  assetId: string;           // Asset ID (string)
  score: number;             // 0.85
  matchedCentroid: string;   // "centroid-abc123"
  thumbnailUrl: string | null;
}
```

**Usage Pattern**:
```typescript
// 1. Compute centroids (optional, can auto-rebuild)
await computeCentroids(personId, { forceRebuild: false });

// 2. Get suggestions (synchronous)
const result = await getCentroidSuggestions(personId, {
  minSimilarity: 0.65,
  maxResults: 200,
  unassignedOnly: true
});

// 3. Use suggestions directly
// result.suggestions is ready to display
```

---

### 4.3 Schema Alignment Requirements

To use SuggestionGroupCard with centroid suggestions, we need:

**Missing Fields in CentroidSuggestion**:
| Field | Required By | Solution |
|-------|-------------|----------|
| `id` | Selection tracking | Generate temporary ID client-side OR use `faceInstanceId` as key |
| `suggestedPersonId` | Header display | Pass as prop from parent context |
| `personName` | Header display | Pass as prop from parent context |
| `status` | Filtering logic | Always "pending" for direct suggestions |
| `confidence` | Sorting/display | Use `score` field (rename) |
| `path` | Detail modal | Fetch from asset metadata OR skip |
| `fullImageUrl` | Detail modal | Construct from `assetId` |
| `bboxX/Y/W/H` | Detail modal bounding boxes | Fetch from FaceInstance OR skip initially |
| `detectionConfidence` | Metadata display | Fetch from FaceInstance OR skip |
| `qualityScore` | Metadata display | Fetch from FaceInstance OR skip |
| `createdAt` | Timestamp display | Use `new Date().toISOString()` (client-side) |

**Transformation Function Needed**:
```typescript
function transformCentroidToFaceSuggestion(
  centroidSuggestion: CentroidSuggestion,
  personId: string,
  personName: string
): FaceSuggestion {
  return {
    id: generateTempId(),  // or use faceInstanceId as string key
    faceInstanceId: centroidSuggestion.faceInstanceId,
    suggestedPersonId: personId,
    personName: personName,
    confidence: centroidSuggestion.score,  // Rename
    status: 'pending',
    faceThumbnailUrl: centroidSuggestion.thumbnailUrl,
    fullImageUrl: `/api/v1/images/${centroidSuggestion.assetId}/full`,
    path: '', // Leave empty or fetch later
    sourceFaceId: centroidSuggestion.matchedCentroid,  // Use centroid ID
    createdAt: new Date().toISOString(),
    reviewedAt: null,

    // Optional fields - can be null initially
    bboxX: null,
    bboxY: null,
    bboxW: null,
    bboxH: null,
    detectionConfidence: null,
    qualityScore: null,

    // Multi-prototype fields
    isMultiPrototypeMatch: false,
    aggregateConfidence: null,
    prototypeMatchCount: null,
    matchingPrototypeIds: null,
    prototypeScores: null
  };
}
```

---

## 5. Integration Points Analysis

### 5.1 Where "Find More" is Used

**Locations Found**:

1. **`/people/[personId]/+page.svelte`** - Person detail page
   - Shows "Find More Suggestions" button in actions area
   - Condition: Person has `faceCount >= 10`
   - Opens FindMoreDialog
   - Handles completion via `handleFindMoreComplete()`

2. **`/faces/suggestions/+page.svelte`** - Suggestions review page
   - Auto-triggers Find More after bulk accept via `bulkSuggestionAction()`
   - Configuration: `autoFindMore: true, findMorePrototypeCount: 50`
   - Tracks jobs via `handleFindMoreJobs(response.findMoreJobs)`
   - Shows toast on job completion

3. **`SuggestionGroupCard.svelte`** - Within suggestion groups
   - Shows "Find More" button in group header
   - Condition: `group.labeledFaceCount >= 10`
   - Launches FindMoreDialog inline
   - Refreshes parent on completion via `onFindMoreComplete` callback

**Common Pattern**:
- All locations use FindMoreDialog component
- All track job progress via jobProgressStore
- All refresh suggestions after completion

---

### 5.2 Centroid Integration Touch Points

To integrate centroids with existing "Find More" locations, we need:

#### Option A: Dual-Mode FindMoreDialog
**Modify FindMoreDialog to support both modes:**

```typescript
interface Props {
  open: boolean;
  personId: string;
  personName: string;
  labeledFaceCount: number;
  onClose: () => void;
  onComplete: (suggestionsFound: number) => void;
  mode?: 'dynamic-prototypes' | 'centroid';  // NEW
}
```

**Behavior Changes**:
- `mode === 'dynamic-prototypes'`: Current behavior (job-based)
- `mode === 'centroid'`:
  - No job tracking
  - Immediate API call to compute + search
  - Pass suggestions directly to results dialog
  - OR create FaceSuggestions and redirect to standard flow

**Pros**:
- Single component for both modes
- Consistent UX across approaches
- Easy A/B testing

**Cons**:
- Increased complexity in FindMoreDialog
- Mixed concerns (job-based vs direct)

---

#### Option B: Separate CentroidFindMoreDialog
**Create new component for centroid approach:**

```typescript
// NEW Component
interface CentroidFindMoreDialogProps {
  open: boolean;
  personId: string;
  personName: string;
  labeledFaceCount: number;
  onClose: () => void;
  onComplete: (suggestionsFound: number) => void;
}
```

**Reuse**: Can delegate to ComputeCentroidsDialog for computation

**Pros**:
- Clean separation of concerns
- No changes to existing Find More flow
- Can coexist during transition

**Cons**:
- Code duplication between dialogs
- Users see two different "Find More" features

---

#### Option C: Backend Hybrid Endpoint
**Create new endpoint that bridges both approaches:**

```
POST /api/v1/faces/suggestions/persons/{person_id}/find-more-centroid

Request:
{
  "min_similarity": 0.65,
  "max_results": 200,
  "unassigned_only": true,
  "enable_clustering": false
}

Response: FindMoreJobResponse (same as current)
{
  "job_id": "...",
  "person_id": "...",
  "person_name": "...",
  "status": "queued",
  "progress_key": "..."
}

Background Job:
1. Compute/retrieve centroids
2. Search Qdrant with centroid
3. CREATE FaceSuggestion records for results
4. Return job result with counts
```

**Pros**:
- **ZERO frontend changes** to FindMoreDialog or FindMoreResultsDialog
- Reuses existing job tracking infrastructure
- Seamless migration path (swap endpoint URL)
- Users get improved algorithm transparently

**Cons**:
- Creates FaceSuggestion records (might be redundant with prototype-based)
- Loses some flexibility of direct centroid access
- Backend becomes more complex

---

## 6. Recommendations

### 6.1 Short-Term Integration (Recommended: Option C)

**Approach**: Backend Hybrid Endpoint + Existing UI

**Implementation Steps**:

1. **Backend** (1-2 days):
   - Create new endpoint: `POST /faces/suggestions/persons/{id}/find-more-centroid`
   - Implement job: `find_more_centroid_suggestions_job()`
   - Job logic:
     ```python
     async def find_more_centroid_suggestions_job(person_id, min_similarity, max_results):
         # 1. Compute/get centroid
         centroid = await compute_centroids_for_person(...)

         # 2. Search faces with centroid
         scored_points = centroid_qdrant.search_faces_with_centroid(...)

         # 3. Create FaceSuggestion records (like current find-more)
         for point in scored_points:
             create_face_suggestion(
                 face_instance_id=point.face_id,
                 suggested_person_id=person_id,
                 confidence=point.score,
                 source_face_id=centroid.centroid_id  # Use centroid ID
             )

         # 4. Return job result
         return {
             "suggestionsCreated": len(scored_points),
             "centroidsUsed": 1,
             "candidatesFound": len(scored_points),
             "duplicatesSkipped": 0
         }
     ```
   - Response: Standard `FindMoreJobResponse`

2. **Frontend** (1 hour):
   - Add toggle to FindMoreDialog: "Use Centroid" checkbox
   - When enabled, call new endpoint instead of current
   - Everything else stays the same (progress tracking, results display)

**Benefits**:
- ✅ Minimal frontend changes (add 1 checkbox)
- ✅ Reuses existing UI components
- ✅ Easy to A/B test (same user flow)
- ✅ Gradual migration path

**Drawbacks**:
- ⚠️ Creates duplicate FaceSuggestion records if both methods used
- ⚠️ Backend complexity increases (two similar jobs)

---

### 6.2 Medium-Term Evolution

**Approach**: Unified "Find More" with Mode Selection

**Implementation Steps**:

1. **Backend** (2-3 days):
   - Merge both approaches into single endpoint with `mode` parameter
   - Add field to FaceSuggestion: `discovery_method` (enum: 'dynamic_prototype', 'centroid')
   - Job deduplicates based on face_instance_id + suggested_person_id

2. **Frontend** (1-2 days):
   - Update FindMoreDialog with "Search Method" selector
   - Options: "Dynamic Prototypes", "Centroid (Faster)", "Both"
   - Update SuggestionGroupCard to show discovery method badge
   - Add filtering in FindMoreResultsDialog by discovery method

**Benefits**:
- ✅ Single unified workflow
- ✅ Users can choose optimal method
- ✅ Deduplication prevents redundant suggestions
- ✅ Audit trail of which method found which matches

---

### 6.3 Long-Term Vision

**Approach**: Centroid-First with Prototype Fallback

**Philosophy**: Centroids as primary, dynamic prototypes as enhancement

**Implementation Steps**:

1. **Backend** (3-5 days):
   - Always compute/maintain centroids for persons with >= 50 faces
   - "Find More" uses centroids by default
   - Optionally add: "Explore More" button for dynamic prototype sampling
   - Background job: Periodically rebuild stale centroids

2. **Frontend** (2-3 days):
   - FindMoreDialog becomes CentroidDialog (simplified)
   - Remove prototype count selection (centroids are auto-computed)
   - Add "Advanced: Sample Additional Prototypes" option
   - Results show: "Found via Centroid" vs "Found via Sampling"

**Benefits**:
- ✅ Optimized performance (centroids are cached)
- ✅ Simpler user experience (fewer options)
- ✅ Best of both worlds (centroid speed + prototype coverage)

**Challenges**:
- ⚠️ Requires robust centroid staleness detection
- ⚠️ Migration path for existing users
- ⚠️ More backend infrastructure

---

## 7. Implementation Checklist (Option C: Recommended)

### Backend Tasks

- [ ] Create `find_more_centroid_suggestions_job()` in `queue/face_jobs.py`
- [ ] Add endpoint: `POST /faces/suggestions/persons/{id}/find-more-centroid` in `api/routes/face_suggestions.py`
- [ ] Request schema: `FindMoreCentroidRequest` (min_similarity, max_results, unassigned_only)
- [ ] Response schema: Reuse `FindMoreJobResponse`
- [ ] Job logic:
  - [ ] Compute/retrieve centroid via `compute_centroids_for_person()`
  - [ ] Search faces with `centroid_qdrant.search_faces_with_centroid()`
  - [ ] Create `FaceSuggestion` records for results
  - [ ] Track progress in Redis
  - [ ] Return counts: suggestionsCreated, centroidsUsed, candidatesFound
- [ ] Add `discovery_method` field to `FaceSuggestion` model (optional)
- [ ] Migration: Add `discovery_method` column (nullable, default NULL)
- [ ] Tests:
  - [ ] Test job execution with valid person
  - [ ] Test centroid auto-rebuild when stale
  - [ ] Test FaceSuggestion creation
  - [ ] Test deduplication of existing suggestions

### Frontend Tasks

- [ ] Update `startFindMoreSuggestions()` in `lib/api/faces.ts` to accept `mode` parameter
- [ ] Add UI toggle to FindMoreDialog:
  ```svelte
  <Label>Search Method</Label>
  <Select bind:value={searchMode}>
    <option value="dynamic">Dynamic Prototypes (Original)</option>
    <option value="centroid">Centroid (Faster, Recommended)</option>
  </Select>
  ```
- [ ] Update `handleSubmit()` to call appropriate endpoint based on mode
- [ ] **No changes needed** to FindMoreResultsDialog (works as-is)
- [ ] **No changes needed** to SuggestionGroupCard (works as-is)
- [ ] Optional: Add "Discovery Method" badge to suggestion display
- [ ] Tests:
  - [ ] Test FindMoreDialog with both modes
  - [ ] Test job tracking for centroid mode
  - [ ] Test results display (unchanged)

### Documentation Tasks

- [ ] Update `docs/api-contract.md` with new endpoint
- [ ] Update frontend types: `npm run gen:api`
- [ ] Add user-facing docs: "Find More: Centroid vs Dynamic Prototypes"
- [ ] Update admin guide: When to use which method

---

## 8. Technical Debt Considerations

### 8.1 FaceSuggestion Record Duplication

**Issue**: Two methods can create suggestions for same face

**Impact**: Database bloat, confusion in UI

**Solutions**:
1. **Deduplication in Job**: Check for existing pending suggestion before creating
2. **Unique Constraint**: Add DB constraint on (face_instance_id, suggested_person_id, status='pending')
3. **Discovery Method Field**: Track which method found the match

**Recommended**: Option 2 + Option 3

---

### 8.2 Centroid Staleness Management

**Issue**: Centroids can become stale when faces added/removed

**Current Approach**: `auto_rebuild` flag in suggestions endpoint

**Gaps**:
- No background job to rebuild stale centroids
- No alerting when centroids are consistently stale

**Solutions**:
1. **Background Worker**: Periodic task to rebuild centroids for active persons
2. **Trigger on Assignment**: Rebuild centroid after bulk face assignments
3. **Staleness Dashboard**: Admin UI showing centroid health

**Recommended**: Option 1 + Option 2

---

### 8.3 Job Tracking Infrastructure

**Issue**: Job progress tracking tied to Redis keys

**Limitations**:
- Progress lost if Redis restarted
- No historical job data
- Difficult to debug failed jobs

**Solutions**:
1. **Job Status Table**: Store job metadata in Postgres
2. **Job History**: Keep last 30 days of job results
3. **Failed Job Queue**: Separate queue for retries

**Recommended**: Option 1 (for persistence)

---

## 9. Testing Strategy

### 9.1 Backend Testing

**Unit Tests** (`tests/unit/`):
- [ ] `test_compute_centroids()` - Centroid computation logic
- [ ] `test_search_with_centroid()` - Qdrant search functionality
- [ ] `test_find_more_centroid_job()` - Job execution
- [ ] `test_faceSuggestion_deduplication()` - Unique constraint handling

**Integration Tests** (`tests/api/`):
- [ ] `test_find_more_centroid_endpoint()` - API contract
- [ ] `test_centroid_auto_rebuild()` - Staleness detection
- [ ] `test_suggestion_creation()` - DB record creation

**Performance Tests**:
- [ ] Centroid search latency (target: <500ms for 50k faces)
- [ ] Job throughput (concurrent find-more requests)

---

### 9.2 Frontend Testing

**Component Tests** (`tests/components/`):
- [ ] `FindMoreDialog.test.ts` - Mode selection UI
- [ ] `FindMoreResultsDialog.test.ts` - Unchanged (regression test)
- [ ] `SuggestionGroupCard.test.ts` - Unchanged (regression test)

**Integration Tests**:
- [ ] Full flow: Launch FindMoreDialog → Track job → View results
- [ ] Mode switching: Verify correct endpoint called
- [ ] Error handling: Network failures, job failures

**User Acceptance Testing**:
- [ ] Users can select centroid mode
- [ ] Results appear within 5 seconds (centroid mode)
- [ ] Accept/reject actions work correctly

---

## 10. Migration Path

### Phase 1: Parallel Deployment (Week 1-2)
- Deploy new centroid endpoint alongside existing
- Add UI toggle for beta users
- Monitor: Success rate, latency, user feedback

### Phase 2: Gradual Rollout (Week 3-4)
- Enable centroid by default for users with >200 faces
- Keep dynamic prototypes as fallback
- A/B test: 50% centroid, 50% dynamic

### Phase 3: Optimization (Week 5-6)
- Analyze performance metrics
- Optimize centroid computation (caching, clustering)
- Add background centroid rebuild job

### Phase 4: Deprecation (Optional, Month 3+)
- If centroid approach proves superior, deprecate dynamic prototypes
- Keep dynamic as manual "Explore More" option
- Remove UI toggle, centroids become default

---

## 11. Open Questions

1. **Should centroid suggestions create FaceSuggestion records?**
   - **Recommended**: Yes (Option C) for short-term consistency
   - Long-term: Consider removing records and using ephemeral suggestions

2. **How to handle person with both centroid and prototype suggestions?**
   - **Recommended**: Add `discovery_method` field to deduplicate and track
   - UI: Show badge indicating discovery method

3. **Should ComputeCentroidsDialog replace FindMoreDialog entirely?**
   - **Recommended**: No, keep both during transition
   - Long-term: Merge into unified dialog with mode selection

4. **What's the threshold for using centroids vs prototypes?**
   - **Recommended**: Centroids for persons with >= 50 faces (faster)
   - Dynamic prototypes for persons with 10-49 faces (more flexible)

5. **Should centroid suggestions be reviewable like current suggestions?**
   - **Recommended**: Yes, use same FaceSuggestion workflow
   - Alternative: Direct accept/reject without pending queue (faster but less traceable)

---

## 12. Success Metrics

### Performance Metrics
- **Latency**: Centroid search completes in <500ms (vs 5-30s for dynamic prototypes)
- **Cache Hit Rate**: Centroid reused without rebuild in >80% of searches
- **Throughput**: Support 10+ concurrent find-more requests

### Quality Metrics
- **Precision**: Acceptance rate >= current method (target: >70%)
- **Recall**: Unique faces found (centroid vs dynamic comparison)
- **User Satisfaction**: Net Promoter Score for find-more feature

### Operational Metrics
- **Centroid Coverage**: >90% of active persons have fresh centroids
- **Job Failure Rate**: <5% of find-more jobs fail
- **Database Growth**: FaceSuggestion table growth rate (monitor duplication)

---

## Conclusion

The centroid-based approach offers significant performance improvements over dynamic prototype sampling. The recommended integration path (Option C: Backend Hybrid Endpoint) provides:

✅ **Minimal Frontend Changes** - Reuse existing UI components
✅ **Seamless Migration** - Gradual rollout with fallback to dynamic prototypes
✅ **User Choice** - Toggle between modes for A/B testing
✅ **Backward Compatibility** - Existing suggestion workflow unchanged

Next step: Implement backend hybrid endpoint and add UI toggle to FindMoreDialog for beta testing.
