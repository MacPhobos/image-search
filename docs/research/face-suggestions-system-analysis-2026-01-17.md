# Face Suggestions System Analysis

**Date**: 2026-01-17
**Purpose**: Comprehensive analysis of how face labeling suggestions are computed, stored, and displayed
**Scope**: Backend computation pipeline and frontend UI integration

---

## 1. Overview

The **Face Suggestions System** automatically generates face-to-person assignment recommendations by comparing unassigned face embeddings against known person prototypes using vector similarity search. Suggestions are displayed in a group-based UI where users can review, accept, or reject assignments in bulk.

### High-Level Flow

```
Unassigned Face → Vector Search → Match Prototypes → Apply Thresholds → Create Suggestion/Auto-Assign
                                                                              ↓
                                                        Database (face_suggestions table)
                                                                              ↓
                                                    UI Fetch (grouped by person) → Review & Action
```

---

## 2. Backend Architecture

### 2.1 Core Components

| Component | File | Purpose |
|-----------|------|---------|
| **FaceAssigner** | `src/image_search_service/faces/assigner.py` | Core logic for matching faces to persons via prototypes |
| **Database Model** | `src/image_search_service/db/models.py` | `FaceSuggestion` table schema |
| **API Routes** | `src/image_search_service/api/routes/face_suggestions.py` | RESTful endpoints for suggestion CRUD |
| **Background Jobs** | `src/image_search_service/queue/face_jobs.py` | Async suggestion generation via RQ |
| **Qdrant Client** | `src/image_search_service/vector/face_qdrant.py` | Vector similarity search |

### 2.2 Suggestion Generation Algorithm

**Trigger Points**:
1. **Incremental Assignment Job** - Processes new unassigned faces (`assign_new_faces()`)
2. **Face Detection Session** - After detecting faces in new images
3. **Find More Job** - User-triggered search for additional matches (`find_more_suggestions_job()`)
4. **Centroid Search** - Fast batch matching using person centroids (`find_more_centroid_suggestions_job()`)

**Two-Tier Threshold System**:

```python
# From ConfigService (database-stored settings)
auto_assign_threshold = 0.85   # High confidence → automatic assignment
suggestion_threshold = 0.70     # Medium confidence → manual review suggestion

# Decision tree per face:
if best_match_score >= auto_assign_threshold:
    → Auto-assign face.person_id = person_id (no suggestion created)
elif best_match_score >= suggestion_threshold:
    → Create FaceSuggestion(status='pending') for manual review
else:
    → Leave face unassigned
```

**Processing Steps** (`FaceAssigner.assign_new_faces()`):

1. **Fetch Unassigned Faces**
   ```python
   query = select(FaceInstance).where(
       FaceInstance.person_id.is_(None),
       FaceInstance.cluster_id.is_(None)
   ).limit(max_faces)
   ```

2. **Retrieve Face Embedding**
   - Query Qdrant using `qdrant_point_id` (UUID stored in PostgreSQL)
   - Returns 512-dimensional buffalo_l embedding vector

3. **Search Against Prototypes**
   ```python
   matches = qdrant.search_against_prototypes(
       query_embedding=embedding,
       limit=3,  # max_matches_per_face
       score_threshold=suggestion_threshold  # 0.70
   )
   ```

4. **Extract Person ID**
   - From Qdrant payload: `best_match.payload.get("person_id")`
   - Fallback: Query `PersonPrototype` table using `qdrant_point_id`

5. **Apply Threshold Logic**
   - **Auto-assign** (≥0.85): Batch update `face_instances.person_id` + Qdrant payload
   - **Suggest** (≥0.70): Create `FaceSuggestion` record with `status='pending'`
   - **Skip** (<0.70): Increment `unassigned_count`

6. **Commit Changes**
   - Database: Batch insert suggestions, update auto-assigned faces
   - Qdrant: Update `person_id` payload for auto-assigned faces

### 2.3 Database Schema

**Table**: `face_suggestions`

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer (PK) | Auto-incrementing suggestion ID |
| `face_instance_id` | UUID (FK) | Face to be labeled |
| `suggested_person_id` | UUID (FK) | Recommended person assignment |
| `confidence` | Float | Cosine similarity score (0.0-1.0) |
| `source_face_id` | UUID (FK) | Prototype face that triggered the match |
| `status` | String | `pending`, `accepted`, `rejected`, `expired` |
| `created_at` | DateTime | Auto-generated timestamp |
| `reviewed_at` | DateTime (nullable) | When user accepted/rejected |
| **Multi-Prototype Fields** (v1.10+) |
| `matching_prototype_ids` | JSONB (nullable) | Array of prototype UUIDs that matched |
| `prototype_scores` | JSONB (nullable) | Map of `{prototype_id: score}` |
| `aggregate_confidence` | Float (nullable) | Max/avg score across prototypes |
| `prototype_match_count` | Integer (nullable) | Number of prototypes matched |

**Indexes**:
- `(face_instance_id, suggested_person_id, status)` - Prevent duplicate pending suggestions
- `(suggested_person_id, status)` - Fast filtering by person and status
- `(status, created_at)` - Pagination queries

**Relationships**:
- `face_instance_id` → `face_instances.id` (CASCADE delete)
- `suggested_person_id` → `persons.id` (CASCADE delete)
- `source_face_id` → `face_instances.id` (CASCADE delete - the prototype face)

### 2.4 API Endpoints

**Base Path**: `/api/v1/faces/suggestions`

#### 2.4.1 List Suggestions (Grouped)

**Endpoint**: `GET /api/v1/faces/suggestions?grouped=true`

**Query Parameters**:
```typescript
{
  grouped: true,                    // Enable group-based pagination
  page: 1,                         // Page number (1-indexed)
  groupsPerPage: 20,               // Persons per page (default from config)
  suggestionsPerGroup: 20,         // Suggestions per person (default from config)
  status?: 'pending' | 'accepted' | 'rejected',
  personId?: string                // Filter by person UUID
}
```

**Response Schema**:
```typescript
{
  groups: [
    {
      personId: string,
      personName: string | null,
      suggestionCount: number,      // Total suggestions for this person
      maxConfidence: number,         // Highest confidence in group
      suggestions: [                 // Top N suggestions (limited by suggestionsPerGroup)
        {
          id: number,
          faceInstanceId: string,
          suggestedPersonId: string,
          confidence: number,
          status: 'pending' | 'accepted' | 'rejected',
          faceThumbnailUrl: string,   // /api/v1/images/{asset_id}/thumbnail
          fullImageUrl: string,       // /api/v1/images/{asset_id}/full
          path: string,               // Original filesystem path
          bboxX/Y/W/H: number,        // Face bounding box coordinates
          // Multi-prototype fields
          matchingPrototypeIds?: string[],
          prototypeScores?: { [prototypeId: string]: number },
          aggregateConfidence?: number,
          prototypeMatchCount?: number
        }
      ]
    }
  ],
  totalGroups: number,              // Total number of persons with suggestions
  totalSuggestions: number,         // Grand total across all persons
  page: number,
  groupsPerPage: number,
  suggestionsPerGroup: number
}
```

**Backend Implementation** (`_list_suggestions_grouped()`):

1. **Group Aggregation Query**:
   ```sql
   SELECT
     suggested_person_id,
     COUNT(id) as count,
     MAX(confidence) as max_conf
   FROM face_suggestions
   WHERE status = 'pending'  -- if status filter applied
   GROUP BY suggested_person_id
   ORDER BY MAX(confidence) DESC
   OFFSET (page-1)*groupsPerPage LIMIT groupsPerPage
   ```

2. **Per-Group Suggestions Query** (with row limiting):
   ```sql
   SELECT *, ROW_NUMBER() OVER (
     PARTITION BY suggested_person_id
     ORDER BY confidence DESC
   ) as rn
   FROM face_suggestions
   WHERE suggested_person_id IN (...)
   AND rn <= suggestionsPerGroup
   ```

3. **Batch Data Enrichment**:
   - Load all `Person` records in single query
   - Load all `FaceInstance` records (for bounding boxes)
   - Load all `ImageAsset` records (for file paths)
   - Build response objects with JOIN-like logic in Python

#### 2.4.2 Accept Suggestion

**Endpoint**: `POST /api/v1/faces/suggestions/{suggestion_id}/accept`

**Effect**:
1. Update `face_instances.person_id = suggested_person_id`
2. Update `face_suggestions.status = 'accepted'`
3. Update `face_suggestions.reviewed_at = NOW()`
4. Commit transaction

**Response**: Updated `FaceSuggestion` object

#### 2.4.3 Reject Suggestion

**Endpoint**: `POST /api/v1/faces/suggestions/{suggestion_id}/reject`

**Effect**:
1. Update `face_suggestions.status = 'rejected'`
2. Update `face_suggestions.reviewed_at = NOW()`
3. Face remains unassigned (`person_id = NULL`)

**Response**: Updated `FaceSuggestion` object

#### 2.4.4 Bulk Action

**Endpoint**: `POST /api/v1/faces/suggestions/bulk-action`

**Request Body**:
```json
{
  "suggestionIds": [123, 456, 789],
  "action": "accept" | "reject",
  "autoFindMore": true,              // Trigger "Find More" jobs after accepting
  "findMorePrototypeCount": 50       // Number of prototypes for Find More
}
```

**Response**:
```json
{
  "processed": 3,
  "failed": 0,
  "errors": [],
  "findMoreJobs": [                   // If autoFindMore=true
    {
      "personId": "uuid",
      "jobId": "rq-job-id",
      "progressKey": "find_more:progress:person_id:job_uuid"
    }
  ]
}
```

**Auto-Find-More Feature** (when `action='accept'` and `autoFindMore=true`):
- Collects unique `person_id`s from accepted suggestions
- Enqueues `find_more_suggestions_job()` for each person
- Returns job tracking info for SSE progress monitoring

#### 2.4.5 Find More Suggestions (Manual Trigger)

**Endpoint**: `POST /api/v1/faces/suggestions/persons/{person_id}/find-more`

**Request Body**:
```json
{
  "prototypeCount": 50,       // Number of random prototypes to sample (default 50)
  "maxSuggestions": 100,      // Max suggestions to create (default 100)
  "minConfidence": 0.70       // Similarity threshold (uses config default if omitted)
}
```

**Background Job** (`find_more_suggestions_job()`):

1. **Sample Prototypes**:
   - Randomly select N labeled faces for person (weighted by quality)
   - Load embeddings from Qdrant
   - Use as temporary search prototypes (doesn't modify `person_prototypes` table)

2. **Search Qdrant**:
   - For each prototype, search for similar unassigned faces
   - Deduplicate results across prototypes
   - Rank by similarity score

3. **Create Suggestions**:
   - Insert `FaceSuggestion` records for top matches
   - Skip faces already having pending suggestions for this person
   - Commit and return count

**Progress Tracking**:
- Redis key: `find_more:progress:{person_id}:{job_uuid}`
- SSE endpoint: `/api/v1/faces/suggestions/persons/{person_id}/find-more-events`
- Phases: `queued` → `sampling_prototypes` → `searching` → `creating_suggestions` → `complete`

#### 2.4.6 Find More Centroid (Faster Alternative)

**Endpoint**: `POST /api/v1/faces/suggestions/persons/{person_id}/find-more-centroid`

**Request Body**:
```json
{
  "minSimilarity": 0.65,       // Lower threshold for centroid (centroids are more robust)
  "maxResults": 200,           // Max suggestions to create
  "unassignedOnly": true       // Only search unassigned faces
}
```

**Advantages**:
- Single vector search (1 query vs N for multi-prototype)
- Consistent results (centroid is stable)
- Faster for persons with many labeled faces

**Process**:
1. Compute/retrieve person centroid (average of all labeled face embeddings)
2. Single Qdrant search using centroid vector
3. Create suggestions from top matches

---

## 3. Frontend Architecture

### 3.1 Key Components

| Component | File | Purpose |
|-----------|------|---------|
| **Suggestions Page** | `src/routes/faces/suggestions/+page.svelte` | Main page with filters, pagination, bulk actions |
| **SuggestionGroupCard** | `src/lib/components/faces/SuggestionGroupCard.svelte` | Person-grouped suggestion display |
| **SuggestionDetailModal** | `src/lib/components/faces/SuggestionDetailModal.svelte` | Full-screen face detail with assignment options |
| **SuggestionThumbnail** | `src/lib/components/faces/SuggestionThumbnail.svelte` | Individual face thumbnail with selection |
| **RecentlyAssignedPanel** | `src/lib/components/faces/RecentlyAssignedPanel.svelte` | Sidebar showing recent assignments with undo |
| **API Client** | `src/lib/api/faces.ts` | TypeScript API functions |

### 3.2 Page State Management

**Svelte 5 Runes-Based State** (`+page.svelte`):

```typescript
let groupedResponse = $state<GroupedSuggestionsResponse | null>(null);
let settings = $state<FaceSuggestionSettings>({ groupsPerPage: 20, itemsPerGroup: 20 });
let groupsPerPage = $state<10 | 20 | 50>(10);  // User preference (persisted to localStorage)
let page = $state(1);
let statusFilter = $state<string>('pending');
let personFilter = $state<string | null>(null);
let selectedIds = $state<Set<number>>(new Set());
let recentAssignments = $state<RecentAssignment[]>([]);
```

**localStorage Integration**:
```typescript
import { localSettings } from '$lib/stores/localSettings.svelte';

const GROUPS_PER_PAGE_KEY = 'suggestions.groupsPerPage';
let groupsPerPage = $state<GroupsPerPageOption>(
  localSettings.get<GroupsPerPageOption>(GROUPS_PER_PAGE_KEY, 10)
);

// On change:
localSettings.set(GROUPS_PER_PAGE_KEY, newValue);
```

### 3.3 Data Flow

**Initialization** (`onMount`):
```typescript
onMount(async () => {
  settings = await getFaceSuggestionSettings();  // Fetch config defaults
  await loadPersons();                           // Fetch all persons for filter dropdown
  await loadSuggestions();                       // Fetch first page
});
```

**Reactive Loading** (`$effect`):
```typescript
$effect(() => {
  // Reload when filters, page, or groupsPerPage changes
  if (statusFilter !== undefined || page || personFilter !== undefined || groupsPerPage) {
    loadSuggestions();
  }
});
```

**API Call** (`loadSuggestions()`):
```typescript
async function loadSuggestions() {
  groupedResponse = await listGroupedSuggestions({
    page,
    groupsPerPage: groupsPerPage,
    suggestionsPerGroup: settings.itemsPerGroup,
    status: statusFilter || undefined,
    personId: personFilter || undefined
  });
}
```

### 3.4 UI Features

#### 3.4.1 Filters and Pagination

**Status Filter**:
```html
<select bind:value={statusFilter}>
  <option value="">All</option>
  <option value="pending">Pending</option>
  <option value="accepted">Accepted</option>
  <option value="rejected">Rejected</option>
</select>
```

**Person Filter** (searchable dropdown):
```html
<PersonSearchBar
  {persons}
  selectedPersonId={personFilter}
  onSelect={handlePersonSelect}
  placeholder="Filter by person..."
/>
```

**Groups Per Page** (persisted to localStorage):
```html
<select value={groupsPerPage} onchange={(e) => handleGroupsPerPageChange(...)}>
  <option value={10}>10 persons</option>
  <option value={20}>20 persons</option>
  <option value={50}>50 persons</option>
</select>
```

**Pagination Controls**:
```typescript
const totalPages = $derived(
  groupedResponse ? Math.ceil(groupedResponse.totalGroups / groupsPerPage) : 0
);

// Previous/Next buttons bound to page state
```

#### 3.4.2 Bulk Selection and Actions

**Selection State**:
```typescript
let selectedIds = $state<Set<number>>(new Set());

// Per-suggestion checkbox:
function handleSelect(id: number, selected: boolean) {
  if (selected) selectedIds.add(id);
  else selectedIds.delete(id);
  selectedIds = new Set(selectedIds);  // Trigger reactivity
}

// Select all in group:
function handleSelectAllInGroup(ids: number[], selected: boolean) {
  ids.forEach(id => selected ? selectedIds.add(id) : selectedIds.delete(id));
  selectedIds = new Set(selectedIds);
}

// Page-level select all:
function selectAll() {
  const pendingIds = groupedResponse.groups
    .flatMap(g => g.suggestions.filter(s => s.status === 'pending').map(s => s.id));
  selectedIds = new Set(pendingIds);
}
```

**Bulk Actions**:
```typescript
async function handleBulkAction(action: 'accept' | 'reject') {
  // Track assignments BEFORE API call (for RecentlyAssignedPanel)
  if (action === 'accept') {
    acceptedSuggestions.forEach(s => trackAssignment(s));
  }

  const response = await bulkSuggestionAction([...selectedIds], action, {
    autoFindMore: action === 'accept',
    findMorePrototypeCount: 50
  });

  // Handle auto-triggered Find More jobs
  if (response.findMoreJobs?.length > 0) {
    handleFindMoreJobs(response.findMoreJobs);
  }

  await loadSuggestions();  // Refresh page
  selectedIds = new Set();  // Clear selection
}
```

#### 3.4.3 SuggestionGroupCard Component

**Props**:
```typescript
interface SuggestionGroup {
  personId: string;
  personName: string | null;
  suggestions: FaceSuggestion[];
  pendingCount: number;           // Count of pending suggestions for this person
  labeledFaceCount?: number;      // Total labeled faces (for context)
}
```

**Features**:
- **Person Header**: Name, pending count, labeled face count badge
- **Suggestion Grid**: Thumbnail gallery with selection checkboxes
- **Multi-Prototype Badge**: Highlights suggestions matched by multiple prototypes
- **Confidence Display**: Shows similarity score (colored by threshold)
- **Bulk Actions**: "Accept All" and "Reject All" buttons for group
- **Find More Button**: Triggers centroid-based search for additional suggestions

**Sorting Logic**:
```typescript
const sortedSuggestions = $derived.by(() => {
  return [...group.suggestions].sort((a, b) => {
    // Multi-prototype matches first
    const aMulti = a.isMultiPrototypeMatch ? 1 : 0;
    const bMulti = b.isMultiPrototypeMatch ? 1 : 0;
    if (bMulti !== aMulti) return bMulti - aMulti;

    // Then by aggregate confidence (fallback to regular confidence)
    const aConf = a.aggregateConfidence ?? a.confidence;
    const bConf = b.aggregateConfidence ?? b.confidence;
    return bConf - aConf;
  });
});
```

#### 3.4.4 SuggestionDetailModal

**Purpose**: Full-screen view for detailed face review

**Features**:
- Full-resolution image with face bounding box overlay
- Person info and confidence score
- Accept/Reject buttons
- Alternative assignment options ("All Faces" list showing other persons)
- Photo metadata (path, dimensions, camera info if available)

**State Management**:
```typescript
let selectedSuggestion = $state<FaceSuggestion | null>(null);

function handleThumbnailClick(suggestion: FaceSuggestion) {
  selectedSuggestion = suggestion;  // Opens modal
}

function handleModalClose() {
  selectedSuggestion = null;  // Closes modal
}
```

#### 3.4.5 Recently Assigned Panel

**Purpose**: Sidebar showing last 10 assignments with undo capability

**State**:
```typescript
interface RecentAssignment {
  faceId: string;
  personId: string;
  personName: string;
  thumbnailUrl: string;
  photoFilename: string;
  assignedAt: Date;
}

let recentAssignments = $state<RecentAssignment[]>([]);

function trackAssignment(suggestion: FaceSuggestion) {
  const assignment: RecentAssignment = { /* ... */ };
  recentAssignments = [assignment, ...recentAssignments].slice(0, 10);
}
```

**Undo Action**:
```typescript
async function handleRecentUndo(faceId: string) {
  await unassignFace(faceId);  // API call to remove person_id
  recentAssignments = recentAssignments.filter(a => a.faceId !== faceId);
  await loadSuggestions();     // Refresh to show face in suggestions again
}
```

### 3.5 Thumbnail Caching

**Batch Loading Strategy** (`thumbnailCache` store):

```typescript
import { thumbnailCache } from '$lib/stores/thumbnailCache.svelte';

$effect(() => {
  if (groupedResponse) {
    const assetIds = extractAssetIds(groupedResponse);  // Parse URLs
    if (assetIds.length > 0) {
      thumbnailCache.fetchBatch(assetIds);  // Single API call for all thumbnails
    }
  }
});

function extractAssetIds(response: GroupedSuggestionsResponse): number[] {
  const assetIds: number[] = [];
  for (const group of response.groups) {
    for (const suggestion of group.suggestions) {
      if (suggestion.faceThumbnailUrl) {
        const match = suggestion.faceThumbnailUrl.match(/\/images\/(\d+)\/thumbnail/);
        if (match) assetIds.push(parseInt(match[1], 10));
      }
    }
  }
  return [...new Set(assetIds)];  // Deduplicate
}
```

**Benefits**:
- Single `/api/v1/images/thumbnails/batch` POST request
- Returns map of `{assetId: dataUri}` for immediate display
- Cached in Svelte store (persists across navigation)

---

## 4. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          SUGGESTION GENERATION                           │
└─────────────────────────────────────────────────────────────────────────┘

Face Detection Job             Find More Job               Centroid Search
       │                              │                           │
       v                              v                           v
   Detect Faces                Sample Prototypes          Compute Centroid
       │                              │                           │
       v                              v                           v
   Get Embeddings              Load Embeddings            Load Centroid Embedding
       │                              │                           │
       └──────────────────┬───────────┴───────────────────────────┘
                          v
                 FaceAssigner.assign_new_faces()
                          │
                          v
            ┌─────────────────────────────┐
            │  For each unassigned face:  │
            │  1. Get embedding           │
            │  2. Search prototypes       │
            │  3. Apply thresholds        │
            └─────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              v                       v
      Score >= 0.85           0.70 <= Score < 0.85
    (Auto-Assign)              (Create Suggestion)
              │                       │
              v                       v
   face_instances.person_id    INSERT face_suggestions
   Qdrant payload update           status='pending'
              │                       │
              └───────────┬───────────┘
                          v
                  PostgreSQL Commit

┌─────────────────────────────────────────────────────────────────────────┐
│                            UI DISPLAY FLOW                               │
└─────────────────────────────────────────────────────────────────────────┘

User Navigates to /faces/suggestions
              │
              v
    GET /api/v1/faces/suggestions?grouped=true&page=1
              │
              v
   Backend: _list_suggestions_grouped()
              │
        ┌─────┴─────┬─────────────┬──────────────┐
        v           v             v              v
   Group by    Fetch Top N   Load Person    Load Face Data
   Person ID   per Group     Records        & Image Paths
        │           │             │              │
        └─────┬─────┴─────────────┴──────────────┘
              v
    Response: GroupedSuggestionsResponse
              │
              v
    Frontend: groupedResponse state updated
              │
        ┌─────┴─────┬───────────────────┐
        v           v                   v
  Render Grid   Extract Asset IDs   Display Filters
                     │
                     v
         thumbnailCache.fetchBatch()
                     │
                     v
    POST /api/v1/images/thumbnails/batch
                     │
                     v
         Thumbnails displayed in grid

┌─────────────────────────────────────────────────────────────────────────┐
│                        ACCEPT/REJECT FLOW                                │
└─────────────────────────────────────────────────────────────────────────┘

User selects suggestions → selectedIds Set<number>
              │
              v
    Click "Accept All" or "Reject All"
              │
              v
    POST /api/v1/faces/suggestions/bulk-action
    {
      suggestionIds: [123, 456],
      action: 'accept',
      autoFindMore: true
    }
              │
        ┌─────┴─────┐
        v           v
   Accept Path  Reject Path
        │           │
        v           v
   Update      Update status
   person_id   reviewed_at
   status
   reviewed_at
        │
        └─────┬─────┘
              v
   Auto Find More? (if action='accept' && autoFindMore=true)
              │
              v
   Enqueue find_more_suggestions_job() for each person
              │
              v
   Return findMoreJobs[] with progress keys
              │
              v
   Frontend: jobProgressStore.trackJob()
              │
              v
   SSE subscription to progress updates
              │
              v
   Toast notifications on completion
              │
              v
   Reload suggestions (new ones from Find More)
```

---

## 5. Key Files Reference

### Backend

| File | Purpose | Lines of Interest |
|------|---------|-------------------|
| `src/image_search_service/faces/assigner.py` | Core suggestion algorithm | 34-256 (assign_new_faces) |
| `src/image_search_service/db/models.py` | FaceSuggestion table schema | 734-790 |
| `src/image_search_service/api/routes/face_suggestions.py` | REST endpoints | 314-383 (list), 502-572 (accept), 651-752 (bulk) |
| `src/image_search_service/queue/face_jobs.py` | Background job functions | 147-250 (assign_faces_job) |
| `src/image_search_service/vector/face_qdrant.py` | Qdrant search operations | `search_against_prototypes()` |
| `src/image_search_service/services/config_service.py` | Threshold configuration | `face_auto_assign_threshold`, `face_suggestion_threshold` |

### Frontend

| File | Purpose | Lines of Interest |
|------|---------|-------------------|
| `src/routes/faces/suggestions/+page.svelte` | Main suggestions page | 1-682 (full component) |
| `src/lib/components/faces/SuggestionGroupCard.svelte` | Person-grouped display | 45-58 (sorting logic) |
| `src/lib/components/faces/SuggestionDetailModal.svelte` | Face detail view | Modal with accept/reject actions |
| `src/lib/components/faces/RecentlyAssignedPanel.svelte` | Undo panel | Recent assignments sidebar |
| `src/lib/api/faces.ts` | API client functions | 899-970 (suggestions API) |
| `src/lib/stores/thumbnailCache.svelte.ts` | Thumbnail caching | Batch loading implementation |
| `src/lib/stores/localSettings.svelte.ts` | localStorage persistence | UI preferences |

---

## 6. Technical Details

### 6.1 Similarity Scoring

**Vector Similarity**:
- **Algorithm**: Cosine similarity (dot product of normalized vectors)
- **Embedding Model**: InsightFace buffalo_l (512-dimensional)
- **Score Range**: 0.0 (completely different) to 1.0 (identical)
- **Typical Thresholds**:
  - 0.85+ → Same person (auto-assign)
  - 0.70-0.85 → Likely same person (suggest for review)
  - 0.50-0.70 → Possible match (not shown)
  - <0.50 → Different person

**Multi-Prototype Matching** (v1.10+):
- If face matches multiple prototypes for same person → higher confidence
- `aggregate_confidence` = max(scores) or weighted average
- `prototype_match_count` used for UI sorting (prefer multi-matches)

### 6.2 Performance Optimizations

**Backend**:
1. **Batch Processing**: Process faces in batches (default 1000)
2. **Query Optimization**: Window functions for row limiting per group
3. **Lazy Loading**: JOINs avoided; batch load related data
4. **Qdrant Indexing**: HNSW index for sub-second vector search
5. **Database Indexes**: Compound indexes on `(suggested_person_id, status, confidence)`

**Frontend**:
1. **Grouped Pagination**: Reduces network overhead (20 persons vs 400 suggestions)
2. **Batch Thumbnails**: Single API call for all visible thumbnails
3. **localStorage Caching**: Persist user preferences (groupsPerPage)
4. **Svelte Reactivity**: Minimal re-renders with `$derived` and `$effect`
5. **Thumbnail Cache**: In-memory store prevents redundant downloads

### 6.3 Configuration

**Database-Stored Settings** (`config` table):

| Key | Default | Description |
|-----|---------|-------------|
| `face_auto_assign_threshold` | 0.85 | Minimum score for automatic assignment |
| `face_suggestion_threshold` | 0.70 | Minimum score for manual suggestion |
| `face_suggestion_groups_per_page` | 20 | Default persons per page in UI |
| `face_suggestion_items_per_group` | 20 | Default suggestions per person |

**User Preferences** (localStorage):

| Key | Default | Description |
|-----|---------|-------------|
| `suggestions.groupsPerPage` | 10 | User's preferred pagination size |

---

## 7. Edge Cases and Error Handling

### Backend

1. **No Prototypes**: Returns `{"status": "no_prototypes", "processed": 0}`
2. **No Unassigned Faces**: Returns `{"status": "no_new_faces", "processed": 0}`
3. **Duplicate Suggestions**: Checked before insert (unique index on `(face_instance_id, suggested_person_id, status='pending')`)
4. **Orphaned Suggestions**: CASCADE delete when face or person deleted
5. **Threshold Edge Cases**: `<0.70` leaves face unassigned (may be picked up by clustering)

### Frontend

1. **Empty State**: Shows "No suggestions found" message
2. **Loading State**: Spinner during API calls
3. **Error State**: Red error banner with message
4. **Pagination Bounds**: Disable Previous/Next buttons at limits
5. **Selection Persistence**: Clear selection after bulk action completes

---

## 8. Future Enhancements

### Planned Features

1. **Active Learning**: Let users correct wrong suggestions to improve model
2. **Confidence Calibration**: Adjust thresholds based on acceptance rates
3. **Temporal Filtering**: Show suggestions from specific date ranges
4. **Quality Filtering**: Filter suggestions by face quality score
5. **Prototype Visualization**: Show which prototype triggered each suggestion
6. **Batch Export**: Export accepted/rejected suggestions for analysis
7. **Suggestion Expiry**: Auto-expire old suggestions after N days
8. **Conflict Resolution**: Handle cases where face matches multiple persons

### Performance Improvements

1. **Incremental Updates**: WebSocket push for new suggestions
2. **Infinite Scroll**: Replace pagination with virtual scrolling
3. **Optimistic UI**: Update UI before API confirmation
4. **Background Prefetch**: Load next page while viewing current

---

## 9. Related Documentation

- **Backend**: `image-search-service/CLAUDE.md`
- **Frontend**: `image-search-ui/CLAUDE.md`
- **API Contract**: `docs/api-contract.md` (section: Face Suggestions)
- **Database Schema**: `db/migrations/versions/` (see FaceSuggestion table migrations)
- **Face Detection**: `docs/research/face-detection-person-labeling-analysis-2025-12-25.md`
- **Prototypes System**: `docs/research/face-person-architecture-2025-12-26.md`

---

**Document Version**: 1.0
**Author**: Research Agent
**Last Updated**: 2026-01-17
