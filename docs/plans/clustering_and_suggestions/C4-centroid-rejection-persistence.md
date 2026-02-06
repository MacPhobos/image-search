# C4: Persist Centroid Suggestion Rejections

**Priority**: Critical
**Severity**: Critical (Data Loss / UX Regression)
**Estimated Effort**: 3-5 days
**Date**: 2026-02-06

---

## References

| Artifact | Path |
|----------|------|
| Research: Executive Synthesis | `docs/research/clustering_and_suggestions/00-executive-synthesis.md` |
| Research: UI/UX Analysis | `docs/research/clustering_and_suggestions/01-ui-ux-analysis.md` |
| Research: Backend Analysis | `docs/research/clustering_and_suggestions/02-backend-analysis.md` |
| Research: Devil's Advocate | `docs/research/clustering_and_suggestions/04-devils-advocate-analysis.md` |
| Frontend: CentroidResultsDialog | `image-search-ui/src/lib/components/faces/CentroidResultsDialog.svelte` |
| Frontend: SuggestionDetailModal | `image-search-ui/src/lib/components/faces/SuggestionDetailModal.svelte` |
| Frontend: Faces API Client | `image-search-ui/src/lib/api/faces.ts` |
| Backend: Face Suggestions Routes | `image-search-service/src/image_search_service/api/routes/face_suggestions.py` |
| Backend: Face Jobs | `image-search-service/src/image_search_service/queue/face_jobs.py` |
| Backend: DB Models | `image-search-service/src/image_search_service/db/models.py` |

---

## 1. Problem Statement

When a user rejects a centroid suggestion in the CentroidResultsDialog, the rejection is purely cosmetic -- it only clears a local Svelte state variable. No backend API call is made to persist the rejection. As a result:

1. Refreshing the page or reopening the dialog brings back previously rejected suggestions.
2. Users must repeatedly reject the same false-positive faces, leading to frustration and eroded trust.
3. The rejection data is lost, so the system cannot learn from user feedback to improve future suggestions.

This contrasts with the regular FaceSuggestion rejection flow, which persists rejections to the database with a `REJECTED` status and a `reviewed_at` timestamp.

---

## 2. Root Cause Analysis

### The Two Suggestion Systems

The codebase has two distinct suggestion pipelines with different persistence models:

**Pipeline A: Database-Backed FaceSuggestion (regular suggestions)**
- Suggestions are created as `FaceSuggestion` records in PostgreSQL (`face_suggestions` table) with `status = "pending"`.
- The reject endpoint (`POST /api/v1/faces/suggestions/{id}/reject`) updates `status` to `"rejected"` and sets `reviewed_at`.
- The frontend calls `rejectSuggestion(suggestionId)` which hits this endpoint.
- Rejections are durable -- the suggestion never reappears.

**Pipeline B: Centroid Suggestions (transient results)**
- The `find_more_centroid_suggestions_job` (line 1810 in `face_jobs.py`) searches Qdrant using a person's centroid embedding and returns matches directly.
- Results are returned as `CentroidSuggestion` objects -- a frontend-only type with no database backing.
- There is NO backend endpoint to reject a centroid suggestion.
- The frontend `handleDetailReject()` in `CentroidResultsDialog.svelte` only clears local state.

### Root Cause Trace

1. **User clicks "Reject Primary" in the SuggestionDetailModal** (stacked on top of CentroidResultsDialog).

2. **SuggestionDetailModal.svelte (line 294-301)** calls the `onReject` callback:
   ```typescript
   async function handleReject() {
       if (!suggestion || isActionLoading) return;
       isActionLoading = true;
       try {
           await onReject(suggestion);
           onClose();
       } finally {
           isActionLoading = false;
       }
   }
   ```

3. **CentroidResultsDialog.svelte (lines 221-226)** provides the `onReject` handler:
   ```typescript
   async function handleDetailReject() {
       // For centroid suggestions, "reject" means just removing from the list
       // We don't have a reject endpoint for centroid suggestions
       toast.info('Suggestion removed from list');
       selectedSuggestionForDetail = null;
   }
   ```
   This is the **root cause**: no backend call is made. The comment explicitly acknowledges the missing endpoint.

4. **No rejection is persisted.** The suggestion is removed from the detail modal's view but remains in the underlying `suggestions` array. The next time the centroid search runs, the same face will be returned again.

### Why the Regular Flow Works But Centroid Does Not

| Aspect | Regular Suggestions | Centroid Suggestions |
|--------|-------------------|---------------------|
| Storage | `FaceSuggestion` table in PostgreSQL | Not stored; returned directly from Qdrant search |
| ID Type | Integer auto-increment (`id` column) | No persistent ID; uses `faceInstanceId` |
| Reject Endpoint | `POST /suggestions/{id}/reject` (line 574 in `face_suggestions.py`) | Does not exist |
| Frontend Reject Call | `rejectSuggestion(suggestionId: number)` (line 930 in `faces.ts`) | `toast.info('Suggestion removed')` only |
| Persistence | `status = "rejected"`, `reviewed_at` set | None -- local state only |

---

## 3. Current Behavior vs Expected Behavior

### Current Behavior

1. User opens CentroidResultsDialog for a person.
2. System shows N suggestions from centroid search.
3. User clicks a suggestion thumbnail, opening SuggestionDetailModal.
4. User clicks "Reject Primary".
5. A toast says "Suggestion removed from list".
6. The detail modal closes. The suggestion remains in the grid.
7. Closing and reopening the dialog, or running a new centroid search, returns the same rejected face.
8. The user has to reject the same face again indefinitely.

### Expected Behavior

1. User opens CentroidResultsDialog for a person.
2. System shows N suggestions from centroid search, **excluding previously rejected faces**.
3. User clicks a suggestion thumbnail, opening SuggestionDetailModal.
4. User clicks "Reject Primary".
5. A toast says "Suggestion rejected".
6. The face is **persisted as rejected** in the database.
7. The suggestion is removed from the current grid view.
8. Future centroid searches **exclude this face** for this person.
9. The rejection is visible in the person's history/audit trail.

---

## 4. Implementation Plan

### Approach Selection

There are two viable approaches:

**Option A: Create centroid suggestions as FaceSuggestion records (Recommended)**
- When centroid search returns results, create `FaceSuggestion` records with `status = "pending"` before returning to the frontend.
- Reuse the existing `reject_suggestion` endpoint.
- Pro: Unified model, full audit trail, reuses existing infrastructure.
- Con: Adds database writes to the centroid search flow.

**Option B: Lightweight rejection tracking table**
- Create a new `centroid_rejections` table that stores `(person_id, face_instance_id, rejected_at)`.
- Filter centroid search results against this table.
- Pro: Minimal schema change, no coupling to FaceSuggestion.
- Con: Creates a parallel tracking system, less reuse.

**Decision: Option A** -- Creating `FaceSuggestion` records from centroid results unifies the two suggestion systems, reuses the existing reject/accept infrastructure, and provides full auditability. The database write overhead is acceptable given the typical result set size (10-200 suggestions).

### Step 1: Backend -- Create FaceSuggestion Records from Centroid Results

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Function**: `find_more_centroid_suggestions_job` (line 1810)
**Current behavior (around line 2037-2039)**: The job already has a section labeled "Creating suggestions from matches" but the actual creation logic needs to be verified.

Read the section after the search results to understand the current suggestion creation:

The job at lines 2037-2039 shows progress update "Creating suggestions from {len(search_results)} matches". The job should already be creating `FaceSuggestion` records from centroid search results. If it is, the issue is solely in the frontend. If it is not, we need to add creation logic.

**Action**: Verify and ensure the job creates `FaceSuggestion` records for each centroid match. Each record needs:
- `face_instance_id`: The matched face's UUID
- `suggested_person_id`: The person whose centroid was used
- `confidence`: The cosine similarity score from Qdrant
- `source_face_id`: Use the face_instance_id itself (centroid-based, no specific source face)
- `status`: `"pending"`

If records ARE being created (likely based on the progress message), skip to Step 2.

If records are NOT being created, add:

```python
# After filtering search results, create FaceSuggestion records
from image_search_service.db.models import FaceSuggestion, FaceSuggestionStatus

suggestions_created = 0
for result in filtered_results:
    face_instance_id = uuid_lib.UUID(result.payload["face_instance_id"])

    # Check for existing pending/rejected suggestion for this face+person
    existing = db_session.execute(
        select(FaceSuggestion).where(
            FaceSuggestion.face_instance_id == face_instance_id,
            FaceSuggestion.suggested_person_id == person_uuid,
            FaceSuggestion.status.in_([
                FaceSuggestionStatus.PENDING.value,
                FaceSuggestionStatus.REJECTED.value,
            ]),
        )
    ).scalar_one_or_none()

    if existing:
        continue  # Skip duplicates and previously rejected

    suggestion = FaceSuggestion(
        face_instance_id=face_instance_id,
        suggested_person_id=person_uuid,
        confidence=result.score,
        source_face_id=face_instance_id,  # Self-reference for centroid-based
        status=FaceSuggestionStatus.PENDING.value,
    )
    db_session.add(suggestion)
    suggestions_created += 1

db_session.commit()
```

**Critical**: The deduplication query MUST check for `REJECTED` status to prevent re-creating suggestions that the user already rejected. This is the key fix.

### Step 2: Backend -- Return Suggestion IDs in Centroid Results

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`
**Context**: The endpoint `start_find_more_centroid_suggestions` (line 841) returns centroid suggestions. The response schema needs to include the `FaceSuggestion.id` so the frontend can call the reject endpoint.

**Action**: Update the centroid suggestion response to include the database suggestion ID.

Add a new response field to the centroid suggestion schema:

```python
# In api/schemas.py or wherever CentroidSuggestionItem is defined
class CentroidSuggestionItem(BaseModel):
    face_instance_id: str
    asset_id: str
    score: float
    matched_centroid: str
    thumbnail_url: str | None
    suggestion_id: int | None = None  # NEW: FaceSuggestion.id for reject/accept
```

Update the job to include `suggestion_id` in the returned results by querying the newly created `FaceSuggestion` records after creation.

### Step 3: Frontend -- Update CentroidSuggestion Type

**File**: `image-search-ui/src/lib/api/faces.ts`
**Lines**: 1536-1542

**Before**:
```typescript
export interface CentroidSuggestion {
    faceInstanceId: string;
    assetId: string;
    score: number;
    matchedCentroid: string;
    thumbnailUrl: string | null;
}
```

**After**:
```typescript
export interface CentroidSuggestion {
    faceInstanceId: string;
    assetId: string;
    score: number;
    matchedCentroid: string;
    thumbnailUrl: string | null;
    suggestionId: number | null;  // NEW: Database ID for reject/accept API calls
}
```

### Step 4: Frontend -- Fix handleDetailReject to Call Backend

**File**: `image-search-ui/src/lib/components/faces/CentroidResultsDialog.svelte`
**Lines**: 221-226

**Before**:
```typescript
async function handleDetailReject() {
    // For centroid suggestions, "reject" means just removing from the list
    // We don't have a reject endpoint for centroid suggestions
    toast.info('Suggestion removed from list');
    selectedSuggestionForDetail = null;
}
```

**After**:
```typescript
async function handleDetailReject() {
    if (!selectedSuggestionForDetail) return;

    // Find the centroid suggestion to get the database suggestion ID
    const centroidSuggestion = suggestions.find(
        s => s.faceInstanceId === selectedSuggestionForDetail!.faceInstanceId
    );

    if (centroidSuggestion?.suggestionId) {
        try {
            await rejectSuggestion(centroidSuggestion.suggestionId);
            toast.success('Suggestion rejected');
        } catch (e) {
            console.error('Failed to reject suggestion:', e);
            toast.error('Failed to reject suggestion');
            selectedSuggestionForDetail = null;
            return;
        }
    } else {
        // Fallback for suggestions without database backing
        toast.info('Suggestion removed from list');
    }

    selectedSuggestionForDetail = null;

    // Call onComplete to refresh data
    if (onComplete) {
        onComplete();
    }
}
```

Add the import at the top of the script:
```typescript
import {
    toAbsoluteUrl,
    assignFaceToPerson,
    rejectSuggestion,  // NEW
    type CentroidSuggestion,
    type FaceSuggestion
} from '$lib/api/faces';
```

### Step 5: Frontend -- Remove Rejected Suggestions from Grid

**File**: `image-search-ui/src/lib/components/faces/CentroidResultsDialog.svelte`

After a successful rejection, the suggestion should be removed from the local `suggestions` array to provide immediate visual feedback.

Add a new derived that filters out rejected suggestions:

```typescript
// Track locally rejected face IDs for immediate UI feedback
let rejectedIds = $state<Set<string>>(new Set());

// Filter out rejected suggestions from display
const visibleSuggestions = $derived.by(() => {
    return suggestions.filter(s => !rejectedIds.has(s.faceInstanceId));
});
```

Update `handleDetailReject` to add to `rejectedIds`:
```typescript
// After successful rejection:
rejectedIds.add(selectedSuggestionForDetail!.faceInstanceId);
rejectedIds = new Set(rejectedIds); // Trigger reactivity
```

Update the template to use `visibleSuggestions` instead of `suggestions` in the grid and count displays:
- Line 260: `{#if visibleSuggestions.length === 0}` (instead of `suggestions.length`)
- Line 275: `{visibleSuggestions.length} suggestion{visibleSuggestions.length === 1 ? '' : 's'} found`
- Line 302: `{#each sortedSuggestions as suggestion (suggestion.faceInstanceId)}` -- update `sortedSuggestions` to derive from `visibleSuggestions`

### Step 6: Backend -- Filter Rejected Faces from Centroid Search

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Function**: `find_more_centroid_suggestions_job` (line 1810)

After retrieving search results from Qdrant, filter out faces that have already been rejected for this person:

```python
# After getting search_results from centroid_qdrant.search_faces_with_centroid()
# Filter out already-rejected suggestions
rejected_face_ids_query = select(FaceSuggestion.face_instance_id).where(
    FaceSuggestion.suggested_person_id == person_uuid,
    FaceSuggestion.status == FaceSuggestionStatus.REJECTED.value,
)
rejected_result = db_session.execute(rejected_face_ids_query)
rejected_face_ids = {str(row[0]) for row in rejected_result.all()}

filtered_results = [
    r for r in search_results
    if r.payload and r.payload.get("face_instance_id") not in rejected_face_ids
]

logger.info(
    f"[{job_id}] Filtered {len(search_results) - len(filtered_results)} "
    f"previously rejected faces, {len(filtered_results)} remaining"
)
```

---

## 5. Data Repair

No data repair migration is needed. Existing centroid suggestion rejections were never persisted, so there is no corrupt data. The fix is purely additive -- new rejections will be persisted going forward.

However, users may notice that previously "rejected" suggestions reappear after the fix is deployed (since those rejections were never saved). This is expected behavior and should be communicated in release notes.

---

## 6. Verification Checklist

### Manual Testing

- [ ] Open CentroidResultsDialog for a person with centroid suggestions.
- [ ] Click a suggestion thumbnail to open SuggestionDetailModal.
- [ ] Click "Reject Primary".
- [ ] Verify toast says "Suggestion rejected" (not "Suggestion removed from list").
- [ ] Verify the rejected suggestion disappears from the grid.
- [ ] Close and reopen the dialog. Verify the rejected suggestion does NOT reappear.
- [ ] Run a new centroid search for the same person. Verify the rejected face is excluded.
- [ ] Check the database: verify a `FaceSuggestion` record exists with `status = "rejected"` and `reviewed_at` is set.

### Automated Tests

**Backend tests** (`tests/api/test_face_suggestions.py`):
- [ ] Test that centroid job creates `FaceSuggestion` records with `status = "pending"`.
- [ ] Test that centroid job excludes faces with existing `REJECTED` suggestions for the person.
- [ ] Test that centroid job excludes faces with existing `PENDING` suggestions (no duplicates).
- [ ] Test that rejecting a centroid-created suggestion via `POST /suggestions/{id}/reject` works correctly.

**Frontend tests** (`src/tests/components/CentroidResultsDialog.test.ts`):
- [ ] Test that `handleDetailReject` calls `rejectSuggestion` when `suggestionId` is available.
- [ ] Test that `handleDetailReject` falls back to toast when `suggestionId` is null.
- [ ] Test that rejected suggestion is visually removed from grid.
- [ ] Test that `onComplete` callback is called after rejection.

### Edge Cases

- [ ] Reject a suggestion, accept another suggestion for the same face with a different person.
- [ ] Reject all suggestions in a centroid result set. Verify "No centroid suggestions found" message.
- [ ] Test with a person who has no active centroid (on-demand computation path).
- [ ] Concurrent rejection: two users rejecting the same suggestion simultaneously.

---

## 7. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Database write overhead in centroid search | Low | Low | Batch inserts; typical result sets are 10-200 items |
| Breaking existing FaceSuggestion queries | Medium | Medium | Centroid suggestions use same schema; ensure status filtering is correct in all existing queries |
| Frontend type mismatch after adding `suggestionId` | Low | Low | Field is nullable; old responses without it will parse as `null` |
| Performance regression in centroid job | Low | Low | Rejected-face filter query uses indexed column (`suggested_person_id` + `status`) |
| Backward compatibility during rolling deploy | Medium | Low | New frontend deployed before backend will still show "Suggestion removed" toast (graceful degradation) |

### Rollback Plan

If issues are detected after deployment:
1. The frontend can be reverted to the previous version (cosmetic-only rejection).
2. Any `FaceSuggestion` records created by the centroid job can be identified by their `source_face_id` pattern (self-referencing) and deleted if needed.
3. No database migration is required, so no schema rollback is needed.

---

## 8. Estimated Effort

| Task | Effort |
|------|--------|
| Step 1: Backend -- Verify/add FaceSuggestion creation in centroid job | 0.5 day |
| Step 2: Backend -- Return suggestion IDs in centroid response | 0.5 day |
| Step 3: Frontend -- Update CentroidSuggestion type | 0.25 day |
| Step 4: Frontend -- Fix handleDetailReject | 0.5 day |
| Step 5: Frontend -- Remove rejected from grid | 0.5 day |
| Step 6: Backend -- Filter rejected faces from search | 0.5 day |
| Testing (backend + frontend) | 1 day |
| Code review + QA | 0.5 day |
| **Total** | **~4 days** |

---

## 9. File Change Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `image-search-service/.../queue/face_jobs.py` | Modify | Add FaceSuggestion creation + rejection filtering in `find_more_centroid_suggestions_job` |
| `image-search-service/.../api/routes/face_suggestions.py` | Modify | Include `suggestion_id` in centroid suggestion response |
| `image-search-service/.../api/schemas.py` | Modify | Add `suggestion_id` field to centroid suggestion schema |
| `image-search-ui/src/lib/api/faces.ts` | Modify | Add `suggestionId` to `CentroidSuggestion` interface |
| `image-search-ui/.../faces/CentroidResultsDialog.svelte` | Modify | Fix `handleDetailReject` to call backend; add rejection tracking |
| `tests/api/test_face_suggestions.py` | Modify | Add tests for centroid suggestion persistence |
| `src/tests/components/CentroidResultsDialog.test.ts` | Add | Component tests for rejection behavior |
