# Centroid Find More Integration Plan

**Date**: 2026-01-16
**Status**: DRAFT - Awaiting Review
**Research**: See `docs/research/find-more-centroid-integration-analysis-2026-01-16.md`

---

## Executive Summary

This plan integrates centroid-based suggestions into the existing "Find More" workflow with **minimal UI changes** and **maximum code reuse**.

### Recommended Approach: Backend Hybrid Endpoint (Option C)

**Why?**
- **Zero changes** to FindMoreResultsDialog and SuggestionGroupCard
- Reuses existing job tracking and progress infrastructure
- Easy A/B testing via simple UI toggle
- Gradual migration path to centroid-first approach

---

## Current State vs. Proposed State

| Aspect | Current (Dynamic Prototypes) | Proposed (Centroid) |
|--------|------------------------------|---------------------|
| **Search Method** | Sample random labeled faces | Use computed centroid embedding |
| **Speed** | 5-30 seconds (job) | <500ms search + job overhead |
| **FaceSuggestion Records** | Created in job | Created in job (same) |
| **UI Flow** | FindMoreDialog → Job → FindMoreResultsDialog | Same (no changes) |
| **Code Reuse** | - | 100% UI reuse |

---

## Implementation Plan

### Phase 1: Backend - New Centroid Job Endpoint (1-2 days)

#### 1.1 Create Job Function

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`

```python
def find_more_centroid_suggestions_job(
    person_id: str,
    min_similarity: float = 0.65,
    max_results: int = 200,
    unassigned_only: bool = True,
) -> dict:
    """
    Find more suggestions using person centroid (faster than dynamic prototypes).

    Flow:
    1. Compute/retrieve centroid for person
    2. Search Qdrant faces collection using centroid
    3. Create FaceSuggestion records for matches
    4. Return job result with counts
    """
```

**Job Result**:
```python
{
    "suggestionsCreated": 42,
    "centroidsUsed": 1,
    "candidatesFound": 68,
    "duplicatesSkipped": 26,  # Existing pending suggestions
}
```

#### 1.2 Create API Endpoint

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

```python
@router.post("/persons/{person_id}/find-more-centroid")
async def find_more_centroid_suggestions(
    person_id: UUID,
    request: FindMoreCentroidRequest,
    db: AsyncSession = Depends(get_async_session),
) -> FindMoreJobResponse:
    """
    Start centroid-based Find More job.

    Uses person centroid for faster, more accurate matching.
    Returns same response format as dynamic prototype endpoint.
    """
```

**Request Schema**:
```python
class FindMoreCentroidRequest(BaseModel):
    min_similarity: float = Field(default=0.65, ge=0.5, le=0.95)
    max_results: int = Field(default=200, ge=1, le=500)
    unassigned_only: bool = True
```

**Response**: Reuse existing `FindMoreJobResponse` (compatible with frontend)

#### 1.3 Deduplication Logic

Before creating FaceSuggestion, check for existing pending suggestion:

```python
# In job function
existing = await db.execute(
    select(FaceSuggestion).where(
        FaceSuggestion.face_instance_id == face_id,
        FaceSuggestion.suggested_person_id == person_id,
        FaceSuggestion.status == FaceSuggestionStatus.PENDING,
    )
)
if existing.scalar_one_or_none():
    duplicates_skipped += 1
    continue

# Create new suggestion with centroid as source
suggestion = FaceSuggestion(
    face_instance_id=face_id,
    suggested_person_id=person_id,
    confidence=score,
    source_face_id=centroid_id,  # Centroid ID instead of prototype face
    status=FaceSuggestionStatus.PENDING,
)
```

### Phase 2: Frontend - Mode Toggle (1 hour)

#### 2.1 Update API Client

**File**: `image-search-ui/src/lib/api/faces.ts`

```typescript
// Add new function
export async function startFindMoreCentroidSuggestions(
    personId: string,
    options: {
        minSimilarity?: number;
        maxResults?: number;
        unassignedOnly?: boolean;
    } = {}
): Promise<FindMoreJobResponse> {
    const response = await fetch(
        `${API_BASE}/faces/suggestions/persons/${personId}/find-more-centroid`,
        {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                min_similarity: options.minSimilarity ?? 0.65,
                max_results: options.maxResults ?? 200,
                unassigned_only: options.unassignedOnly ?? true,
            }),
        }
    );
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
}
```

#### 2.2 Add Mode Toggle to FindMoreDialog

**File**: `image-search-ui/src/lib/components/faces/FindMoreDialog.svelte`

Add search method selector:

```svelte
<script lang="ts">
    // ... existing code ...

    // NEW: Search method selection
    let searchMethod = $state<'dynamic' | 'centroid'>('centroid'); // Default to centroid
</script>

<!-- Add to options section -->
<div class="space-y-2">
    <label class="block text-sm font-medium">Search Method</label>
    <select bind:value={searchMethod} class="w-full border rounded p-2">
        <option value="centroid">Centroid (Faster, Recommended)</option>
        <option value="dynamic">Dynamic Prototypes (Original)</option>
    </select>
    <p class="text-xs text-gray-500">
        {#if searchMethod === 'centroid'}
            Uses computed centroid embedding for faster, more consistent results.
        {:else}
            Samples random labeled faces. Better for diverse appearances.
        {/if}
    </p>
</div>

<!-- Update submit handler -->
<script lang="ts">
async function handleSubmit() {
    if (searchMethod === 'centroid') {
        const response = await startFindMoreCentroidSuggestions(personId, {
            minSimilarity: parseFloat(selectedThresholdStr),
            maxResults: 200,
            unassignedOnly: true,
        });
        // Same job tracking as before
        jobProgressStore.trackJob(response.jobId, response.progressKey, ...);
    } else {
        // Existing dynamic prototype flow
        const response = await startFindMoreSuggestions(personId, { ... });
        jobProgressStore.trackJob(...);
    }
}
</script>
```

### Phase 3: Remove ComputeCentroidsDialog Button (Optional)

Once centroid is integrated into FindMoreDialog, the separate "Compute Centroids" button can be:
- **Option A**: Keep as "Advanced" option for power users
- **Option B**: Remove from person detail page (simpler UX)

**Recommendation**: Option A initially, evaluate based on usage.

---

## Files to Change

### Backend

| File | Change | Effort |
|------|--------|--------|
| `api/routes/face_suggestions.py` | Add `find-more-centroid` endpoint | 2h |
| `api/face_suggestion_schemas.py` | Add `FindMoreCentroidRequest` schema | 30m |
| `queue/face_jobs.py` | Add `find_more_centroid_suggestions_job` | 3h |
| `docs/api-contract.md` | Document new endpoint | 30m |

### Frontend

| File | Change | Effort |
|------|--------|--------|
| `lib/api/faces.ts` | Add `startFindMoreCentroidSuggestions()` | 30m |
| `components/faces/FindMoreDialog.svelte` | Add search method toggle | 1h |
| `docs/api-contract.md` | Sync from backend | 10m |

### No Changes Required

- `FindMoreResultsDialog.svelte` - Works as-is (loads from DB)
- `SuggestionGroupCard.svelte` - Works as-is (displays FaceSuggestion)
- Job progress tracking - Works as-is (same response format)

---

## API Contract Update

Add to `docs/api-contract.md` (v1.18.0):

```markdown
### Find More Suggestions (Centroid)

**POST /api/v1/faces/suggestions/persons/{person_id}/find-more-centroid**

Uses person centroid for faster suggestion discovery.

**Request Body:**
```json
{
    "minSimilarity": 0.65,
    "maxResults": 200,
    "unassignedOnly": true
}
```

**Response:** Same as `/find-more` endpoint (FindMoreJobResponse)
```

---

## Testing Strategy

### Backend Tests

```python
# tests/api/test_find_more_centroid.py

async def test_find_more_centroid_creates_suggestions():
    """Job creates FaceSuggestion records from centroid search."""

async def test_find_more_centroid_deduplication():
    """Existing pending suggestions are skipped."""

async def test_find_more_centroid_auto_computes_centroid():
    """Centroid computed if missing or stale."""

async def test_find_more_centroid_response_format():
    """Response matches FindMoreJobResponse schema."""
```

### Frontend Tests

```typescript
// tests/components/FindMoreDialog.test.ts

test('shows search method toggle', async () => {
    // Verify toggle is visible with both options
});

test('centroid mode calls correct endpoint', async () => {
    // Select centroid mode, submit, verify API call
});

test('dynamic mode calls original endpoint', async () => {
    // Select dynamic mode, submit, verify original API call
});
```

---

## Migration Strategy

### Week 1-2: Beta Testing
- Deploy new endpoint
- Enable toggle for beta users via feature flag
- Monitor: success rate, latency, user feedback

### Week 3-4: Gradual Rollout
- Default to centroid for users with >100 labeled faces
- Keep dynamic as fallback option
- A/B test: Compare acceptance rates

### Month 2+: Evaluation
- If centroid performs better: Make it default
- If dynamic has edge cases: Keep both options
- Consider removing toggle if centroid clearly superior

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Search Latency | <500ms | Backend timing logs |
| Job Completion | <5s total | Job duration |
| Acceptance Rate | >=70% | FaceSuggestion status tracking |
| User Preference | >80% choose centroid | UI analytics |

---

## Timeline

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| Phase 1: Backend | 1-2 days | New endpoint + job |
| Phase 2: Frontend | 1 hour | Mode toggle |
| Phase 3: Testing | 1 day | Backend + frontend tests |
| Phase 4: Deploy | 1 day | Beta rollout |
| **Total** | **3-4 days** | |

---

## Decision Points for Review

1. **Default Mode**: Should centroid be default or opt-in?
   - Recommendation: Default to centroid (faster, better)

2. **Prototype Count**: Should we hide prototype count when centroid selected?
   - Recommendation: Yes, simplify UI for centroid mode

3. **ComputeCentroidsDialog**: Keep or remove from person page?
   - Recommendation: Keep as "Advanced" option initially

4. **Discovery Method Tracking**: Add field to FaceSuggestion?
   - Recommendation: Yes, for future analysis (optional for v1)

---

**Ready for Review**

Please confirm:
1. Approach (Option C: Backend Hybrid Endpoint)
2. Default search method (centroid vs. dynamic)
3. UI simplification level
4. Timeline expectations

Then I'll proceed with implementation.
