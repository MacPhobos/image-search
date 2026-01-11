# Plan: Dynamic Prototype Selection for "Find More Suggestions"

> **Status**: Planned
> **Created**: 2026-01-10
> **Updated**: 2026-01-10 (added critical safeguards)
> **Estimated Effort**: ~8 days

## Summary

Enhance the face suggestion system to find additional suggestions by sampling labeled faces as temporary prototypes. This enables discovery of faces that don't match the fixed prototype set but may still belong to the same person.

### Problem Statement

The current approach uses a fixed set of prototypes per person to find suggestions. While this works well initially, over time as more faces are labeled to a person, we might be missing opportunities to find additional suggestions. Faces that look different from the prototypes (different angles, lighting, ages) may never be found.

### Solution

Use a dynamic set of temporary prototypes randomly sampled from the person's labeled faces to search for additional similar faces. This feature:
- Only runs when explicitly requested by the user ("Find More Suggestions" button)
- Uses the same similarity threshold as the existing suggestion process
- Does NOT modify the user's configured prototypes
- Supports configurable sample sizes (10, 50, 100, 1000, all)

## Confirmed Design Decisions

- **Progress Reporting**: SSE (Server-Sent Events) - simple, one-way, built for this use case
- **Prototype Selection**: Quality + Diversity weighted - prefer high-quality faces from different photos/times
- **Accept All Flow**: Auto-trigger find-more job after bulk accept (configurable, default 50 prototypes)
- **Default Prototype Count**: 50 (good balance of coverage vs speed)

---

## Implementation Phases

### Phase 1: Backend - Core Job & Endpoint (Days 1-2)

#### 1.1 New Job Function

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`

Add `find_more_suggestions_job()` following the existing `propagate_person_label_multiproto_job()` pattern:

```python
def find_more_suggestions_job(
    person_id: str,
    prototype_count: int = 50,  # 10, 50, 100, 1000, or -1 for all
    min_confidence: float | None = None,  # Uses SystemConfig if None
    max_suggestions: int = 100,
    progress_key: str | None = None,  # Redis key for progress
) -> dict[str, Any]:
    """Find more face suggestions using random sampling of labeled faces.

    Selection Strategy (Quality + Diversity):
    1. Get all labeled faces for the person (excluding existing prototypes)
    2. Score each face: quality_score * 0.7 + diversity_bonus * 0.3
       - diversity_bonus based on unique asset_id distribution
    3. Weighted random selection of top N faces
    4. For each selected face, query Qdrant for similar unknown faces
    5. Aggregate results using MAX score across all matches
    6. Create FaceSuggestion records (skip duplicates with existing pending)
    7. Update progress in Redis after each prototype

    Args:
        person_id: UUID string of the person
        prototype_count: Number of faces to sample as prototypes (default: 50)
                        Use -1 for all available faces
        min_confidence: Similarity threshold (default: from SystemConfig)
        max_suggestions: Maximum new suggestions to create
        progress_key: Redis key for progress updates (optional)

    Returns:
        dict with: status, suggestions_created, prototypes_used,
        candidates_found, duplicates_skipped
    """
```

**Algorithm Details (Quality + Diversity Selection)**:

```python
def score_face_for_selection(face: FaceInstance, asset_counts: dict[int, int]) -> float:
    """Score a face for selection as temporary prototype.

    Higher scores = more likely to be selected.
    Balances quality (70%) with diversity (30%).
    """
    quality = face.quality_score or 0.5

    # Track how many times we've used faces from this asset
    asset_id = face.asset_id
    if asset_id not in asset_counts:
        asset_counts[asset_id] = 0
    usage_count = asset_counts[asset_id]

    # Penalize repeated use of same asset (prefer diverse photos)
    diversity_penalty = min(0.3, usage_count * 0.1)
    diversity_bonus = 0.3 - diversity_penalty

    return quality * 0.7 + diversity_bonus
```

**Progress Update Pattern**:

```python
def update_job_progress(redis_client, key: str, phase: str, current: int, total: int, message: str):
    """Update job progress in Redis for SSE streaming."""
    progress = {
        "phase": phase,
        "current": current,
        "total": total,
        "message": message,
        "timestamp": datetime.now(UTC).isoformat()
    }
    redis_client.set(key, json.dumps(progress), ex=3600)  # 1 hour TTL
```

**Progress Phases**:
- `selecting` - Scoring and selecting faces from labeled pool
- `searching` - Querying Qdrant with each temporary prototype
- `creating` - Writing new FaceSuggestion records
- `completed` - Job finished successfully
- `failed` - Job encountered an error

#### 1.2 New Pydantic Schemas

**File**: `image-search-service/src/image_search_service/api/face_session_schemas.py`

```python
from pydantic import Field
from .schemas import CamelCaseModel

class FindMoreSuggestionsRequest(CamelCaseModel):
    """Request to find more suggestions using random face sampling."""
    prototype_count: int = Field(
        default=50,
        ge=10,
        le=1000,
        description="Number of labeled faces to sample as temporary prototypes"
    )
    max_suggestions: int = Field(
        default=100,
        ge=1,
        le=500,
        description="Maximum number of new suggestions to create"
    )

class FindMoreJobResponse(CamelCaseModel):
    """Response when starting a find-more job."""
    job_id: str
    person_id: str
    person_name: str
    prototype_count: int
    labeled_face_count: int  # Total available for sampling
    status: str  # "queued"
    progress_key: str  # Redis key for progress

class JobProgress(CamelCaseModel):
    """Progress update for any background job (SSE payload)."""
    phase: str
    current: int
    total: int
    message: str
    timestamp: str  # ISO format

class JobResult(CamelCaseModel):
    """Final result when job completes."""
    status: str  # "completed" or "failed"
    suggestions_created: int | None = None
    prototypes_used: int | None = None
    candidates_found: int | None = None
    duplicates_skipped: int | None = None
    error: str | None = None
```

#### 1.3 New API Endpoint

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

```python
from uuid import UUID
from rq import Queue
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from ..face_session_schemas import FindMoreSuggestionsRequest, FindMoreJobResponse
from ...db.session import get_db
from ...db.models import Person, FaceInstance
from ...queue.face_jobs import find_more_suggestions_job

@router.post(
    "/persons/{person_id}/find-more",
    response_model=FindMoreJobResponse,
    status_code=201,
    summary="Start job to find more suggestions using dynamic prototypes",
    description="""
    Samples random labeled faces (weighted by quality and diversity)
    as temporary prototypes to search for additional similar faces.

    Does NOT modify the person's configured prototypes.
    Uses the same similarity threshold as normal suggestion generation.
    """,
)
async def start_find_more_suggestions(
    person_id: UUID,
    request: FindMoreSuggestionsRequest,
    db: AsyncSession = Depends(get_db),
) -> FindMoreJobResponse:
    """Start a background job to find more suggestions for a person."""

    # Validate person exists
    person = await db.get(Person, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")

    # Count available labeled faces
    labeled_count = await db.scalar(
        select(func.count()).where(
            FaceInstance.person_id == person_id
        )
    )

    if labeled_count < 10:
        raise HTTPException(
            status_code=400,
            detail=f"Person has only {labeled_count} labeled faces. Minimum 10 required."
        )

    # Adjust prototype count if "all" requested
    actual_count = min(request.prototype_count, labeled_count)

    # Generate progress key
    import uuid
    job_uuid = str(uuid.uuid4())
    progress_key = f"find_more:progress:{person_id}:{job_uuid}"

    # Enqueue job
    queue = Queue(connection=get_redis())
    job = queue.enqueue(
        find_more_suggestions_job,
        str(person_id),
        actual_count,
        None,  # min_confidence (use system default)
        request.max_suggestions,
        progress_key,
        job_id=job_uuid,
    )

    return FindMoreJobResponse(
        job_id=job.id,
        person_id=str(person_id),
        person_name=person.name,
        prototype_count=actual_count,
        labeled_face_count=labeled_count,
        status="queued",
        progress_key=progress_key,
    )
```

#### 1.4 SSE Progress Endpoint

**File**: `image-search-service/src/image_search_service/api/routes/jobs.py` (new file)

```python
"""Job progress and status endpoints."""

import asyncio
import json
from fastapi import APIRouter, HTTPException
from fastapi.responses import StreamingResponse
from redis import Redis

from ...core.config import settings

router = APIRouter(prefix="/jobs", tags=["jobs"])

def get_redis() -> Redis:
    """Get Redis client (lazy initialization)."""
    return Redis.from_url(settings.redis_url)

@router.get(
    "/events",
    summary="Stream job progress via Server-Sent Events",
    description="""
    Opens an SSE connection that streams progress updates for a background job.

    Pass the `progress_key` returned from the job creation endpoint.

    Events:
    - `progress`: Periodic progress updates
    - `complete`: Final result when job finishes
    - `error`: Error message if job fails

    The stream automatically closes when the job completes or fails.
    """,
)
async def stream_job_events(progress_key: str) -> StreamingResponse:
    """Stream real-time progress updates for a background job."""

    redis = get_redis()

    # Verify the key exists (job was started)
    if not redis.exists(progress_key):
        raise HTTPException(status_code=404, detail="Job not found or expired")

    async def event_generator():
        """Generate SSE events from Redis progress updates."""
        last_data = None
        poll_interval = 1.0  # seconds
        max_polls = 600  # 10 minutes max

        for _ in range(max_polls):
            # Get current progress
            data = redis.get(progress_key)

            if data:
                data_str = data.decode() if isinstance(data, bytes) else data

                # Only send if changed
                if data_str != last_data:
                    last_data = data_str
                    progress = json.loads(data_str)

                    # Determine event type
                    phase = progress.get("phase", "")
                    if phase == "completed":
                        yield f"event: complete\ndata: {data_str}\n\n"
                        return
                    elif phase == "failed":
                        yield f"event: error\ndata: {data_str}\n\n"
                        return
                    else:
                        yield f"event: progress\ndata: {data_str}\n\n"

            await asyncio.sleep(poll_interval)

        # Timeout
        yield f"event: error\ndata: {{\"error\": \"Timeout waiting for job\"}}\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )

@router.get(
    "/status",
    summary="Get current job status (non-streaming)",
)
async def get_job_status(progress_key: str) -> dict:
    """Get current status of a background job."""

    redis = get_redis()

    data = redis.get(progress_key)
    if not data:
        raise HTTPException(status_code=404, detail="Job not found or expired")

    data_str = data.decode() if isinstance(data, bytes) else data
    return json.loads(data_str)
```

**Redis Key Structure**:
```
find_more:progress:{person_id}:{job_id} -> JSON {phase, current, total, message, timestamp}
TTL: 3600 seconds (1 hour)
```

**Duplicate Suggestion Prevention**:

To prevent race conditions when multiple jobs might find the same face:

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

def create_suggestion_if_not_exists(
    session: Session,
    face_id: int,
    person_id: str,
    confidence: float,
    source: str = "find_more"
) -> bool:
    """Create suggestion only if no pending suggestion exists for this face+person.

    Returns True if created, False if duplicate.
    """
    stmt = pg_insert(FaceSuggestion).values(
        face_id=face_id,
        suggested_person_id=person_id,
        confidence=confidence,
        status="pending",
        source=source,
    ).on_conflict_do_nothing(
        index_elements=["face_id", "suggested_person_id"],
        where=FaceSuggestion.status == "pending"
    )
    result = session.execute(stmt)
    return result.rowcount > 0
```

**Required Migration**: Add partial unique index to prevent duplicate pending suggestions:

```sql
CREATE UNIQUE INDEX uq_face_suggestion_pending
ON face_suggestions (face_id, suggested_person_id)
WHERE status = 'pending';
```

---

### Phase 2: Frontend - Dialog & Store (Days 2-3)

#### 2.1 Job Progress Store

**File**: `image-search-ui/src/lib/stores/jobProgressStore.svelte.ts` (new file)

```typescript
/**
 * Svelte 5 runes-based store for tracking background job progress.
 * Supports multiple concurrent jobs with SSE connections.
 */

import { getApiBaseUrl } from '$lib/api/client';

export interface JobProgress {
  jobId: string;
  personId: string;
  personName: string;
  phase: string;
  current: number;
  total: number;
  message: string;
  status: 'queued' | 'running' | 'completed' | 'failed';
  error?: string;
  result?: {
    suggestionsCreated?: number;
    prototypesUsed?: number;
    candidatesFound?: number;
    duplicatesSkipped?: number;
  };
}

class JobProgressStore {
  // Reactive state using Svelte 5 runes
  jobs = $state<Map<string, JobProgress>>(new Map());
  private eventSources = new Map<string, EventSource>();
  private pollingIntervals = new Map<string, ReturnType<typeof setInterval>>();

  // Browsers limit SSE connections to ~6 per domain
  private readonly MAX_SSE_CONNECTIONS = 4;

  /**
   * Start tracking a job via SSE (preferred) or polling (fallback).
   * Falls back to polling when SSE connection limit reached.
   * @param progressKey The progress_key returned from job creation endpoint
   * @returns Cleanup function to stop tracking
   */
  trackJob(
    jobId: string,
    progressKey: string,  // Use progressKey from job creation response
    personId: string,
    personName: string,
    onComplete?: (job: JobProgress) => void,
    onError?: (error: string) => void
  ): () => void {
    // Initialize job state
    const initialState: JobProgress = {
      jobId,
      personId,
      personName,
      phase: 'queued',
      current: 0,
      total: 100,
      message: 'Queued...',
      status: 'queued',
    };
    this.jobs.set(jobId, initialState);

    // Fall back to polling if SSE connection limit reached
    if (this.eventSources.size >= this.MAX_SSE_CONNECTIONS) {
      return this.trackJobViaPolling(jobId, progressKey, onComplete, onError);
    }

    // Open SSE connection (preferred) - uses progressKey query parameter
    const baseUrl = getApiBaseUrl();
    const eventSource = new EventSource(
      `${baseUrl}/api/v1/jobs/events?progress_key=${encodeURIComponent(progressKey)}`
    );
    this.eventSources.set(jobId, eventSource);

    eventSource.addEventListener('progress', (event) => {
      const data = JSON.parse(event.data);
      const current = this.jobs.get(jobId);
      if (current) {
        this.jobs.set(jobId, {
          ...current,
          phase: data.phase,
          current: data.current,
          total: data.total,
          message: data.message,
          status: 'running',
        });
      }
    });

    eventSource.addEventListener('complete', (event) => {
      const data = JSON.parse(event.data);
      const current = this.jobs.get(jobId);
      if (current) {
        const completed: JobProgress = {
          ...current,
          phase: 'completed',
          current: data.current || data.total,
          total: data.total,
          message: data.message || 'Completed',
          status: 'completed',
          result: {
            suggestionsCreated: data.suggestionsCreated,
            prototypesUsed: data.prototypesUsed,
            candidatesFound: data.candidatesFound,
            duplicatesSkipped: data.duplicatesSkipped,
          },
        };
        this.jobs.set(jobId, completed);
        onComplete?.(completed);
      }
      this.cleanup(jobId);
    });

    eventSource.addEventListener('error', (event) => {
      let errorMessage = 'Unknown error';
      if (event instanceof MessageEvent) {
        try {
          const data = JSON.parse(event.data);
          errorMessage = data.error || data.message || 'Job failed';
        } catch {
          errorMessage = 'Connection error';
        }
      }

      const current = this.jobs.get(jobId);
      if (current) {
        this.jobs.set(jobId, {
          ...current,
          status: 'failed',
          error: errorMessage,
        });
      }
      onError?.(errorMessage);
      this.cleanup(jobId);
    });

    // Return cleanup function
    return () => this.cleanup(jobId);
  }

  /**
   * Get current state of a job.
   */
  getJob(jobId: string): JobProgress | undefined {
    return this.jobs.get(jobId);
  }

  /**
   * Check if any jobs are currently running.
   */
  get hasRunningJobs(): boolean {
    return [...this.jobs.values()].some(
      (j) => j.status === 'queued' || j.status === 'running'
    );
  }

  /**
   * Get all running jobs.
   */
  get runningJobs(): JobProgress[] {
    return [...this.jobs.values()].filter(
      (j) => j.status === 'queued' || j.status === 'running'
    );
  }

  /**
   * Polling fallback when SSE connections exhausted.
   */
  private trackJobViaPolling(
    jobId: string,
    progressKey: string,
    onComplete?: (job: JobProgress) => void,
    onError?: (error: string) => void
  ): () => void {
    const baseUrl = getApiBaseUrl();
    const pollInterval = 2000; // 2 seconds

    const interval = setInterval(async () => {
      try {
        const response = await fetch(
          `${baseUrl}/api/v1/jobs/status?progress_key=${encodeURIComponent(progressKey)}`
        );
        if (!response.ok) {
          throw new Error('Job not found');
        }

        const data = await response.json();
        const current = this.jobs.get(jobId);
        if (!current) return;

        if (data.phase === 'completed') {
          const completed: JobProgress = {
            ...current,
            phase: 'completed',
            current: data.current || data.total,
            total: data.total,
            message: data.message || 'Completed',
            status: 'completed',
            result: {
              suggestionsCreated: data.suggestionsCreated,
              prototypesUsed: data.prototypesUsed,
              candidatesFound: data.candidatesFound,
              duplicatesSkipped: data.duplicatesSkipped,
            },
          };
          this.jobs.set(jobId, completed);
          onComplete?.(completed);
          this.cleanup(jobId);
        } else if (data.phase === 'failed') {
          this.jobs.set(jobId, { ...current, status: 'failed', error: data.error });
          onError?.(data.error || 'Job failed');
          this.cleanup(jobId);
        } else {
          this.jobs.set(jobId, {
            ...current,
            phase: data.phase,
            current: data.current,
            total: data.total,
            message: data.message,
            status: 'running',
          });
        }
      } catch (err) {
        const message = err instanceof Error ? err.message : 'Polling error';
        onError?.(message);
        this.cleanup(jobId);
      }
    }, pollInterval);

    this.pollingIntervals.set(jobId, interval);
    return () => this.cleanup(jobId);
  }

  private cleanup(jobId: string) {
    const eventSource = this.eventSources.get(jobId);
    if (eventSource) {
      eventSource.close();
      this.eventSources.delete(jobId);
    }
    const interval = this.pollingIntervals.get(jobId);
    if (interval) {
      clearInterval(interval);
      this.pollingIntervals.delete(jobId);
    }
  }

  /**
   * Clean up all connections (call on unmount).
   */
  destroy() {
    for (const [jobId] of this.eventSources) {
      this.cleanup(jobId);
    }
    for (const [jobId] of this.pollingIntervals) {
      this.cleanup(jobId);
    }
    this.jobs.clear();
  }
}

// Export singleton instance
export const jobProgressStore = new JobProgressStore();
```

#### 2.2 FindMoreDialog Component

**File**: `image-search-ui/src/lib/components/faces/FindMoreDialog.svelte` (new file)

```svelte
<script lang="ts">
  import * as Dialog from '$lib/components/ui/dialog';
  import { Button } from '$lib/components/ui/button';
  import { Progress } from '$lib/components/ui/progress';
  import { RadioGroup, RadioGroupItem } from '$lib/components/ui/radio-group';
  import { Label } from '$lib/components/ui/label';
  import { toast } from 'svelte-sonner';
  import { startFindMoreSuggestions, type FindMoreJobResponse } from '$lib/api/faces';
  import { jobProgressStore, type JobProgress } from '$lib/stores/jobProgressStore.svelte';

  interface Props {
    open: boolean;
    personId: string;
    personName: string;
    labeledFaceCount: number;
    onClose: () => void;
    onComplete: (suggestionsFound: number) => void;
  }

  let { open, personId, personName, labeledFaceCount, onClose, onComplete }: Props = $props();

  // Prototype count options
  const PROTOTYPE_OPTIONS = [
    { value: 10, label: '10 faces', description: 'Quick scan (~5 seconds)' },
    { value: 50, label: '50 faces', description: 'Balanced (~15 seconds)' },
    { value: 100, label: '100 faces', description: 'Thorough (~30 seconds)' },
    { value: 1000, label: '1000 faces', description: 'Comprehensive (~2 minutes)' },
    { value: -1, label: 'All faces', description: `All ${labeledFaceCount} labeled faces` },
  ];

  let selectedCount = $state(50);
  let isSubmitting = $state(false);
  let currentJob = $state<JobProgress | null>(null);
  let cleanupFn: (() => void) | null = null;

  // Filter options based on available faces
  const availableOptions = $derived(
    PROTOTYPE_OPTIONS.filter(opt => opt.value === -1 || opt.value <= labeledFaceCount)
  );

  // Progress percentage
  const progressPercent = $derived(
    currentJob ? Math.round((currentJob.current / currentJob.total) * 100) : 0
  );

  async function handleSubmit() {
    if (isSubmitting || currentJob) return;

    isSubmitting = true;

    try {
      const actualCount = selectedCount === -1 ? labeledFaceCount : selectedCount;
      const response = await startFindMoreSuggestions(personId, {
        prototypeCount: actualCount,
      });

      // Start tracking progress using progressKey from response
      cleanupFn = jobProgressStore.trackJob(
        response.jobId,
        response.progressKey,  // Use progressKey from job creation response
        personId,
        personName,
        handleJobComplete,
        handleJobError
      );

      // Update local state to show progress
      currentJob = jobProgressStore.getJob(response.jobId) || null;

      toast.loading(`Finding more suggestions for ${personName}...`, {
        id: `find-more-${response.jobId}`,
      });

    } catch (err) {
      const message = err instanceof Error ? err.message : 'Failed to start job';
      toast.error(message);
    } finally {
      isSubmitting = false;
    }
  }

  function handleJobComplete(job: JobProgress) {
    const count = job.result?.suggestionsCreated || 0;

    toast.success(
      count > 0
        ? `Found ${count} new suggestions for ${personName}!`
        : `No new suggestions found for ${personName}`,
      { id: `find-more-${job.jobId}` }
    );

    currentJob = null;
    cleanupFn = null;
    onComplete(count);
    onClose();
  }

  function handleJobError(error: string) {
    toast.error(`Failed: ${error}`, { id: currentJob ? `find-more-${currentJob.jobId}` : undefined });
    currentJob = null;
    cleanupFn = null;
  }

  function handleClose() {
    // Job continues in background if running
    if (currentJob) {
      toast.info(`Job continues in background for ${personName}`);
    }
    cleanupFn?.();
    currentJob = null;
    onClose();
  }

  // Track job progress reactively
  $effect(() => {
    if (currentJob?.jobId) {
      const updated = jobProgressStore.getJob(currentJob.jobId);
      if (updated) {
        currentJob = updated;
      }
    }
  });
</script>

<Dialog.Root {open} onOpenChange={(isOpen) => !isOpen && handleClose()}>
  <Dialog.Content class="max-w-md">
    <Dialog.Header>
      <Dialog.Title>Find More Suggestions</Dialog.Title>
      <Dialog.Description>
        Search for additional faces similar to {personName}'s labeled faces
      </Dialog.Description>
    </Dialog.Header>

    <div class="py-4 space-y-4">
      {#if !currentJob}
        <!-- Selection UI -->
        <div class="text-sm text-muted-foreground">
          {personName} has <strong>{labeledFaceCount}</strong> labeled faces available for sampling.
        </div>

        <RadioGroup bind:value={selectedCount} class="space-y-2">
          {#each availableOptions as option}
            <div class="flex items-center space-x-2">
              <RadioGroupItem value={option.value} id={`opt-${option.value}`} />
              <Label for={`opt-${option.value}`} class="flex-1 cursor-pointer">
                <span class="font-medium">{option.label}</span>
                <span class="text-xs text-muted-foreground ml-2">{option.description}</span>
              </Label>
            </div>
          {/each}
        </RadioGroup>

        <div class="text-xs text-muted-foreground">
          Higher counts provide better coverage but take longer to process.
        </div>
      {:else}
        <!-- Progress UI -->
        <div class="space-y-3">
          <div class="flex justify-between text-sm">
            <span>{currentJob.message}</span>
            <span class="text-muted-foreground">{progressPercent}%</span>
          </div>
          <Progress value={progressPercent} class="h-2" />
          <div class="text-xs text-muted-foreground text-center">
            {currentJob.current} / {currentJob.total} prototypes processed
          </div>
        </div>
      {/if}
    </div>

    <Dialog.Footer class="gap-2">
      <Button variant="outline" onclick={handleClose}>
        {currentJob ? 'Close' : 'Cancel'}
      </Button>
      {#if !currentJob}
        <Button onclick={handleSubmit} disabled={isSubmitting}>
          {isSubmitting ? 'Starting...' : 'Find Suggestions'}
        </Button>
      {/if}
    </Dialog.Footer>
  </Dialog.Content>
</Dialog.Root>
```

#### 2.3 API Functions

**File**: `image-search-ui/src/lib/api/faces.ts` (add to existing file)

```typescript
// Add these interfaces
export interface FindMoreSuggestionsOptions {
  prototypeCount?: number;
  maxSuggestions?: number;
}

export interface FindMoreJobResponse {
  jobId: string;
  personId: string;
  personName: string;
  prototypeCount: number;
  labeledFaceCount: number;
  status: string;
  progressKey: string;
}

// Add this function
/**
 * Start a job to find more suggestions using dynamic prototypes.
 */
export async function startFindMoreSuggestions(
  personId: string,
  options?: FindMoreSuggestionsOptions
): Promise<FindMoreJobResponse> {
  const body: Record<string, number> = {};
  if (options?.prototypeCount !== undefined) {
    body.prototypeCount = options.prototypeCount;
  }
  if (options?.maxSuggestions !== undefined) {
    body.maxSuggestions = options.maxSuggestions;
  }

  return apiRequest<FindMoreJobResponse>(
    `/api/v1/faces/suggestions/persons/${encodeURIComponent(personId)}/find-more`,
    {
      method: 'POST',
      body: JSON.stringify(body),
    }
  );
}
```

---

### Phase 3: Integration (Days 3-4)

#### 3.1 Face Suggestions View Integration

**File**: `image-search-ui/src/lib/components/faces/SuggestionGroupCard.svelte`

Add "Find More" button in the group header:

```svelte
<script lang="ts">
  // Add import
  import FindMoreDialog from './FindMoreDialog.svelte';

  // Add state
  let showFindMoreDialog = $state(false);

  // Add handler
  function handleFindMoreComplete(count: number) {
    // Trigger refresh of suggestions for this person
    // This would call the parent's refresh function
  }
</script>

<!-- In the header section, add button -->
<div class="flex items-center gap-2">
  <Button
    variant="outline"
    size="sm"
    onclick={() => showFindMoreDialog = true}
    title="Find more suggestions using additional prototypes"
  >
    <Search class="h-4 w-4 mr-1" />
    Find More
  </Button>

  <!-- Existing Accept All / Reject All buttons -->
</div>

<!-- Add dialog at end of component -->
{#if showFindMoreDialog}
  <FindMoreDialog
    open={showFindMoreDialog}
    personId={group.personId}
    personName={group.personName || 'Unknown'}
    labeledFaceCount={getLabeledFaceCount(group.personId)}
    onClose={() => showFindMoreDialog = false}
    onComplete={handleFindMoreComplete}
  />
{/if}
```

#### 3.2 Person Detail View Integration

**File**: `image-search-ui/src/routes/people/[personId]/+page.svelte`

Add "Find More Suggestions" button in the header:

```svelte
<script lang="ts">
  // Add import
  import FindMoreDialog from '$lib/components/faces/FindMoreDialog.svelte';

  // Add state
  let showFindMoreDialog = $state(false);

  // Handler
  function handleFindMoreComplete(count: number) {
    // Refresh suggestions tab if visible
    toast.success(`Found ${count} new suggestions!`);
    // Could also trigger a data refresh
  }
</script>

<!-- In header actions area -->
<Button
  variant="outline"
  onclick={() => showFindMoreDialog = true}
>
  <Search class="h-4 w-4 mr-1" />
  Find More Suggestions
</Button>

<!-- Dialog -->
{#if showFindMoreDialog && person}
  <FindMoreDialog
    open={showFindMoreDialog}
    personId={person.id}
    personName={person.name}
    labeledFaceCount={person.faceCount}
    onClose={() => showFindMoreDialog = false}
    onComplete={handleFindMoreComplete}
  />
{/if}
```

---

### Phase 4: Accept All Enhancement (Day 4)

#### 4.1 Backend Enhancement

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

Modify `bulk_suggestion_action` endpoint:

```python
class BulkSuggestionActionRequest(CamelCaseModel):
    suggestion_ids: list[int]
    action: str = Field(..., pattern="^(accept|reject)$")
    auto_find_more: bool = Field(
        default=False,
        description="Auto-trigger find-more job after accepting"
    )
    find_more_prototype_count: int = Field(
        default=50,
        ge=10,
        le=1000,
        description="Prototype count for auto-triggered find-more jobs"
    )

class FindMoreJobInfo(CamelCaseModel):
    """Info about an auto-triggered find-more job."""
    person_id: str
    job_id: str
    progress_key: str

class BulkSuggestionActionResponse(CamelCaseModel):
    processed: int
    failed: int
    errors: list[str] = []
    find_more_jobs: list[FindMoreJobInfo] | None = Field(
        default=None,
        description="List of auto-triggered find-more jobs with progress keys"
    )
```

Implementation:

```python
@router.post("/bulk-action", response_model=BulkSuggestionActionResponse)
async def bulk_suggestion_action(
    request: BulkSuggestionActionRequest,
    db: AsyncSession = Depends(get_db),
) -> BulkSuggestionActionResponse:
    """Process bulk accept/reject of suggestions."""

    processed = 0
    failed = 0
    errors: list[str] = []
    affected_person_ids: set[str] = set()

    # Existing bulk processing logic...
    for suggestion_id in request.suggestion_ids:
        # ... process each suggestion
        if request.action == "accept" and suggestion:
            affected_person_ids.add(str(suggestion.suggested_person_id))

    # Auto-trigger find-more if requested
    find_more_jobs: list[FindMoreJobInfo] | None = None

    if request.auto_find_more and request.action == "accept" and affected_person_ids:
        find_more_jobs = []
        queue = Queue(connection=get_redis())

        for person_id in affected_person_ids:
            job_uuid = str(uuid.uuid4())
            progress_key = f"find_more:progress:{person_id}:{job_uuid}"

            job = queue.enqueue(
                find_more_suggestions_job,
                person_id,
                request.find_more_prototype_count,
                None,  # min_confidence
                100,   # max_suggestions
                progress_key,
                job_id=job_uuid,
            )

            find_more_jobs.append(FindMoreJobInfo(
                person_id=person_id,
                job_id=job.id,
                progress_key=progress_key,
            ))

    return BulkSuggestionActionResponse(
        processed=processed,
        failed=failed,
        errors=errors[:10],  # Limit error list
        find_more_jobs=find_more_jobs,
    )
```

#### 4.2 Frontend Enhancement

**File**: `image-search-ui/src/routes/faces/suggestions/+page.svelte`

Handle auto-find-more response:

```typescript
async function handleBulkAccept() {
  const response = await bulkSuggestionAction(
    [...selectedIds],
    'accept',
    { autoFindMore: true, findMorePrototypeCount: 50 }
  );

  // Track any auto-triggered jobs
  // Response includes both jobIds and progressKeys for each person
  if (response.findMoreJobs) {
    for (const job of response.findMoreJobs) {
      const personName = getPersonName(job.personId); // helper to lookup name

      jobProgressStore.trackJob(
        job.jobId,
        job.progressKey,  // Use progressKey from response
        job.personId,
        personName,
        (completedJob) => {
          toast.success(`Found ${completedJob.result?.suggestionsCreated || 0} new suggestions for ${personName}`);
          loadSuggestions(); // Refresh
        },
        (error) => {
          toast.error(`Find more failed for ${personName}: ${error}`);
        }
      );

      toast.loading(`Finding more suggestions for ${personName}...`, {
        id: `find-more-${job.jobId}`,
      });
    }
  }

  // Existing refresh logic
  await loadSuggestions();
}
```

---

### Phase 5: API Contract Sync (Day 5)

🔴 **CRITICAL**: Per monorepo Golden Rule, API changes require contract synchronization.

#### 5.1 Update Backend Contract

**File**: `image-search-service/docs/api-contract.md`

Add documentation for:
- `POST /api/v1/faces/suggestions/persons/{person_id}/find-more`
- `GET /api/v1/jobs/{job_id}/events` (SSE)
- `GET /api/v1/jobs/{job_id}/status`
- Enhanced `POST /api/v1/faces/suggestions/bulk-action` with auto-find-more fields

#### 5.2 Copy Contract to Frontend

```bash
cp image-search-service/docs/api-contract.md image-search-ui/docs/api-contract.md
```

#### 5.3 Regenerate TypeScript Types

```bash
cd image-search-ui
npm run gen:api
```

#### 5.4 Verify Generated Types

After running `gen:api`, verify that the generated types in `src/lib/api/generated.ts` include:
- `FindMoreSuggestionsRequest`
- `FindMoreJobResponse`
- `JobProgress`
- Updated `BulkSuggestionActionRequest` with `autoFindMore` field
- Updated `BulkSuggestionActionResponse` with `findMoreJobIds` field

#### 5.5 Update Manual API Functions (if needed)

If `npm run gen:api` doesn't auto-generate client functions, verify manual interfaces in `src/lib/api/faces.ts` match the generated types exactly.

---

## Files to Create/Modify

### Backend (image-search-service)

| File | Action | Description |
|------|--------|-------------|
| `src/.../queue/face_jobs.py` | Modify | Add `find_more_suggestions_job()` (~150 lines) |
| `src/.../api/face_session_schemas.py` | Modify | Add request/response schemas (~50 lines) |
| `src/.../api/routes/face_suggestions.py` | Modify | Add `/find-more` endpoint, enhance bulk-action (~100 lines) |
| `src/.../api/routes/jobs.py` | Create | SSE progress endpoint (~80 lines) |
| `src/.../main.py` | Modify | Register jobs router |
| `db/migrations/versions/xxx_add_pending_suggestion_index.py` | Create | Partial unique index migration |
| `tests/unit/test_find_more_job.py` | Create | Job unit tests |
| `tests/api/test_find_more_endpoint.py` | Create | API tests |
| `docs/api-contract.md` | Modify | Add new endpoint documentation |

### Frontend (image-search-ui)

| File | Action | Description |
|------|--------|-------------|
| `src/lib/stores/jobProgressStore.svelte.ts` | Create | Progress tracking store (~150 lines) |
| `src/lib/components/faces/FindMoreDialog.svelte` | Create | Reusable dialog (~200 lines) |
| `src/lib/api/faces.ts` | Modify | Add API functions (~30 lines) |
| `src/lib/components/faces/SuggestionGroupCard.svelte` | Modify | Add Find More button (~20 lines) |
| `src/routes/people/[personId]/+page.svelte` | Modify | Add Find More button (~20 lines) |
| `src/routes/faces/suggestions/+page.svelte` | Modify | Handle auto-find-more response (~40 lines) |
| `src/tests/components/FindMoreDialog.test.ts` | Create | Component tests |
| `src/tests/stores/jobProgressStore.test.ts` | Create | Store tests |
| `docs/api-contract.md` | Sync | Copy from backend (must match) |

---

## API Contract Changes

### New: POST /api/v1/faces/suggestions/persons/{person_id}/find-more

**Request**:
```json
{
  "prototypeCount": 50,
  "maxSuggestions": 100
}
```

**Response (201)**:
```json
{
  "jobId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "personId": "550e8400-e29b-41d4-a716-446655440000",
  "personName": "John Doe",
  "prototypeCount": 50,
  "labeledFaceCount": 234,
  "status": "queued",
  "progressKey": "find_more:progress:550e8400-e29b-41d4-a716-446655440000:a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### New: GET /api/v1/jobs/events?progress_key={key} (SSE)

**Query Parameters**:
- `progress_key` (required): The `progressKey` from job creation response

**Response**: Server-Sent Events stream

```
event: progress
data: {"phase":"searching","current":25,"total":50,"message":"Processing prototype 25/50","timestamp":"2026-01-10T15:30:00Z"}

event: progress
data: {"phase":"creating","current":42,"total":50,"message":"Creating suggestions...","timestamp":"2026-01-10T15:30:15Z"}

event: complete
data: {"phase":"completed","current":50,"total":50,"message":"Found 42 new suggestions","suggestionsCreated":42,"prototypesUsed":50,"candidatesFound":156,"duplicatesSkipped":114,"timestamp":"2026-01-10T15:30:20Z"}
```

### New: GET /api/v1/jobs/status?progress_key={key}

**Query Parameters**:
- `progress_key` (required): The `progressKey` from job creation response

**Response**:
```json
{
  "phase": "searching",
  "current": 25,
  "total": 50,
  "message": "Processing prototype 25/50",
  "timestamp": "2026-01-10T15:30:00Z"
}
```

### Enhanced: POST /api/v1/faces/suggestions/bulk-action

**Request** (new optional fields):
```json
{
  "suggestionIds": [1, 2, 3, 4, 5],
  "action": "accept",
  "autoFindMore": true,
  "findMorePrototypeCount": 50
}
```

**Response** (new optional field):
```json
{
  "processed": 5,
  "failed": 0,
  "errors": [],
  "findMoreJobs": [
    {
      "personId": "550e8400-e29b-41d4-a716-446655440000",
      "jobId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "progressKey": "find_more:progress:550e8400-e29b-41d4-a716-446655440000:a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    },
    {
      "personId": "661f9511-f3ac-52e5-b827-557766551111",
      "jobId": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "progressKey": "find_more:progress:661f9511-f3ac-52e5-b827-557766551111:b2c3d4e5-f6a7-8901-bcde-f12345678901"
    }
  ]
}
```

---

## Verification Plan

### Backend Tests

**File**: `tests/unit/test_find_more_job.py`

1. `test_quality_diversity_selection_produces_varied_faces`
   - Given 100 faces from 20 assets
   - When selecting 20 prototypes
   - Then at least 15 different assets should be represented

2. `test_excludes_existing_prototypes_from_selection`
   - Given person with 5 prototypes and 50 labeled faces
   - When running find-more job
   - Then none of the 5 prototype faces should be selected

3. `test_skips_duplicate_suggestions`
   - Given existing pending suggestions for faces A, B, C
   - When find-more finds faces A, B, C, D, E
   - Then only D, E should be created as new suggestions

4. `test_uses_threshold_from_system_config`
   - Given SystemConfig.face_suggestion_threshold = 0.65
   - When running job with min_confidence=None
   - Then Qdrant search uses 0.65 threshold

5. `test_updates_redis_progress`
   - When processing 10 prototypes
   - Then Redis key should be updated 10+ times
   - And progress.current should increase

6. `test_concurrent_jobs_do_not_create_duplicate_suggestions`
   - Given two concurrent find-more jobs for same person
   - When both find face X as a match
   - Then only one FaceSuggestion record is created (due to partial unique index)

**File**: `tests/api/test_find_more_endpoint.py`

1. `test_start_find_more_returns_job_info`
2. `test_start_find_more_validates_prototype_count_range`
3. `test_start_find_more_returns_404_for_unknown_person`
4. `test_start_find_more_requires_minimum_labeled_faces`
5. `test_bulk_action_with_auto_find_more_queues_jobs`
6. `test_sse_endpoint_streams_progress_updates`

### Frontend Tests

**File**: `src/tests/components/FindMoreDialog.test.ts`

1. `test_renders_all_prototype_options`
2. `test_disables_options_exceeding_labeled_count`
3. `test_shows_labeled_face_count_in_description`
4. `test_calls_api_with_selected_prototype_count`
5. `test_shows_progress_bar_during_job`
6. `test_calls_onComplete_with_suggestion_count`
7. `test_closes_dialog_on_cancel`
8. `test_shows_toast_on_completion`

**File**: `src/tests/stores/jobProgressStore.test.ts`

1. `test_tracks_multiple_concurrent_jobs`
2. `test_updates_job_state_from_sse_events`
3. `test_calls_onComplete_callback`
4. `test_calls_onError_callback`
5. `test_cleans_up_event_source_on_destroy`
6. `test_falls_back_to_polling_when_sse_limit_reached`
7. `test_polling_updates_job_state_correctly`
8. `test_cleans_up_polling_intervals_on_destroy`

### Manual E2E Testing

1. **Suggestions View Flow**:
   - Navigate to Face Suggestions page
   - Click "Find More" on a person's suggestion card
   - Select "50 faces" option
   - Click "Find Suggestions"
   - Verify progress bar updates
   - Verify toast notification on completion
   - Verify suggestions list refreshes with new items

2. **People View Flow**:
   - Navigate to People page
   - Select a person with >10 labeled faces
   - Click "Find More Suggestions" button
   - Complete same flow as above

3. **Accept All with Auto-Trigger**:
   - In Face Suggestions, accept all suggestions for a person
   - Verify toast shows "Finding more suggestions..."
   - Verify progress updates
   - Verify new suggestions appear after completion

4. **Edge Cases**:
   - Person with < 10 labeled faces → Should show error
   - Select "All" option → Should use all available faces
   - Close dialog during progress → Job continues, toast updates

---

## Configuration Options

### Backend (SystemConfig table)

| Key | Default | Description |
|-----|---------|-------------|
| `find_more_default_prototype_count` | 50 | Default prototype count when not specified |
| `find_more_max_suggestions` | 100 | Maximum suggestions per job |

### Frontend (localSettings store)

| Key | Default | Description |
|-----|---------|-------------|
| `findMore.lastPrototypeCount` | 50 | Remember user's last selection |
| `findMore.autoTriggerOnAcceptAll` | true | Default for auto-trigger checkbox |

---

## Estimated Effort

| Phase | Description | Days |
|-------|-------------|------|
| Phase 1 | Backend - Core Job & Endpoint | 2 |
| Phase 2 | Frontend - Dialog & Store | 2 |
| Phase 3 | Integration (both views) | 1 |
| Phase 4 | Accept All Enhancement | 1 |
| Phase 5 | API Contract Sync | 0.5 |
| Testing | Unit, integration, manual E2E | 1.5 |
| **Total** | | **8 days** |

---

## Future Enhancements

1. **User Preference Storage**: Remember last-used prototype count per person
2. **Batch Mode**: Queue find-more for multiple persons at once
3. **Smart Defaults**: Adjust default count based on labeled face count
4. **Cancellation**: Add ability to cancel running job via SSE
5. **History**: Show history of find-more runs and their results
