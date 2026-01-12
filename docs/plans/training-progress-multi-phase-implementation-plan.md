# Training Progress Multi-Phase Tracking - Implementation Plan

**Date**: 2026-01-12
**Issue**: UI shows 100% when training_jobs complete, but face detection and clustering still run invisibly
**Research Document**: `docs/research/training-progress-tracking-comprehensive-analysis-2026-01-12.md`

---

## Executive Summary

This plan addresses the false completion indicator shown to users when training Phase 1 completes (30 seconds) while face detection Phase 2 (2 minutes) and clustering Phase 3 (15 seconds) continue invisibly. The solution implements unified progress tracking across all three phases with accurate 0-100% progress calculation.

**Recommended Approach**: Three-phase implementation starting with immediate UX fix, followed by unified backend endpoint, and enhanced UI with phase indicators.

---

## Problem Statement

### Current Behavior
```
User initiates training → UI shows progress 0% → 100%
                                                   ↓
                                    "Training Complete" ✓
                                    (but 82% of work still running!)
                                                   ↓
                        Face detection runs invisibly (2 min)
                                                   ↓
                        Clustering runs invisibly (15 sec)
                                                   ↓
                        Face card appears (user surprise)
```

### Root Cause
1. Training system originally had only Phase 1 (CLIP embeddings)
2. Face detection (Phase 2) added later as separate feature
3. Clustering (Phase 3) added inline in Phase 2 job
4. UI progress calculation never refactored to include all phases
5. Polling stops when `session.status === 'completed'` (after Phase 1)

### Performance Data (1000 images, M1 Mac)
- **Phase 1 (Training)**: 30 seconds → 18% of total time
- **Phase 2 (Face Detection)**: 2 minutes → 73% of total time
- **Phase 3 (Clustering)**: 15 seconds → 9% of total time
- **Total**: 2 minutes 45 seconds

**User sees 100% complete at 30 seconds (18% actual progress)**

---

## Implementation Plan

### 🟢 Phase 1: Immediate UX Fix (1-2 hours)
**Goal**: Clarify what's happening without backend changes
**Effort**: 1-2 hours
**Priority**: P0 (Can deploy immediately)

#### Changes Required

**Frontend Only** (`image-search-ui/`)

1. **Update SessionDetailView.svelte** (Lines 182-203)
   ```svelte
   <!-- Current: Generic "Processing Images" label -->
   <!-- New: Context-aware label based on phase -->

   {#if progress}
       {@const current = progress.progress.current}
       {@const total = progress.progress.total}
       {@const percentage = total > 0 ? Math.round((current / total) * 100) : 0}

       <section class="progress-section">
           <h2>Progress</h2>
           <div class="space-y-1 mb-4">
               <div class="flex justify-between text-sm text-gray-600">
                   <span>
                       {#if session.status === 'completed' && faceSession?.status === 'processing'}
                           🔄 Face Detection Running...
                       {:else if session.status === 'completed' && faceSession?.status === 'pending'}
                           ⏳ Training Complete - Face Detection Starting...
                       {:else if session.status === 'completed' && !faceSession}
                           ✓ Training Complete - Face Detection Pending
                       {:else if session.status === 'running'}
                           🎨 Processing Training Images
                       {:else}
                           Processing Images
                       {/if}
                   </span>
                   <span>{percentage}%</span>
               </div>
               <Progress value={percentage} max={100} class="h-2" />
           </div>

           {#if progress.progress.etaSeconds && session.status === 'running'}
               <ETADisplay eta={new Date(Date.now() + progress.progress.etaSeconds * 1000).toISOString()} />
           {/if}

           <!-- Show informational message when training complete but faces pending -->
           {#if session.status === 'completed' && (faceSession?.status === 'processing' || faceSession?.status === 'pending')}
               <div class="text-sm text-blue-600 bg-blue-50 p-2 rounded mt-2">
                   ℹ️ Face detection is running in the background. This may take several minutes depending on the number of images.
               </div>
           {/if}

           <TrainingStats stats={progress.progress} />
       </section>
   {/if}
   ```

2. **Extend Polling Logic** (Lines 285-297)
   ```typescript
   // Current: Stops polling when training completes
   // New: Continue polling while face session is active

   $effect(() => {
       const status = session?.status;
       const faceStatus = faceSession?.status;

       untrack(() => {
           // Keep polling if:
           // 1. Training is running, OR
           // 2. Face session exists and is processing/pending
           const shouldPoll = status === 'running' ||
                             faceStatus === 'processing' ||
                             (faceStatus === 'pending' && status === 'completed');

           if (shouldPoll && session?.id) {
               startPolling();
           } else {
               stopPolling();
           }
       });

       return () => stopPolling();
   });
   ```

#### Acceptance Criteria
- ✅ Label changes to "Face Detection Running..." when Phase 1 completes
- ✅ Informational message explains background processing
- ✅ Polling continues until face session completes
- ✅ Progress bar still shows 100% (expected - not fixing calculation yet)
- ✅ No backend changes required
- ✅ Tests pass (`npm run test`)

#### Testing
```bash
cd image-search-ui

# Run component tests
npm run test -- SessionDetailView.test.ts

# Manual test:
# 1. Start training session (1000 images)
# 2. Wait for training to complete (30 seconds)
# 3. Verify label changes to "Face Detection Running..."
# 4. Verify polling continues (network tab shows requests)
# 5. Wait for face detection to complete (2 minutes)
# 6. Verify face card appears
# 7. Verify polling stops
```

---

### 🟡 Phase 2: Unified Progress Endpoint (4-6 hours)
**Goal**: Accurate 0-100% progress across all phases
**Effort**: 4-6 hours (2h backend + 2h frontend + 2h testing)
**Priority**: P1 (Recommended solution)

#### Changes Required

**Backend** (`image-search-service/`)

1. **Add new schema** to `src/image_search_service/api/training_schemas.py`
   ```python
   from pydantic import BaseModel, Field, computed_field
   from typing import Literal

   class PhaseProgress(BaseModel):
       """Progress information for a single training phase."""
       name: Literal["training", "face_detection", "clustering"]
       status: str  # "pending" | "running" | "completed" | "failed" | "paused"
       progress: ProgressStats
       started_at: str | None = None
       completed_at: str | None = None

   class OverallProgress(BaseModel):
       """Overall progress across all phases."""
       percentage: float = Field(..., ge=0, le=100)
       eta_seconds: int | None = None
       current_phase: str  # "training" | "face_detection" | "clustering" | "completed"

   class UnifiedProgressResponse(BaseModel):
       """Unified progress response combining all training phases."""
       session_id: int
       overall_status: str  # "pending" | "running" | "completed" | "failed" | "paused"
       overall_progress: OverallProgress
       phases: dict[str, PhaseProgress]  # Keys: "training", "faceDetection", "clustering"

       class Config:
           alias_generator = to_camel
           populate_by_name = True
   ```

2. **Add unified progress method** to `src/image_search_service/services/training_service.py`
   ```python
   async def get_session_progress_unified(
       self, db: AsyncSession, session_id: int
   ) -> UnifiedProgressResponse:
       """Get unified progress across all training phases.

       Progress weights:
       - Phase 1 (Training): 30% (fast GPU operation)
       - Phase 2 (Face Detection): 65% (slowest operation)
       - Phase 3 (Clustering): 5% (quick CPU operation)
       """
       # Get training session (Phase 1)
       training_session = await self.get_session(db, session_id)

       # Get face detection session (Phase 2/3)
       face_session_query = select(FaceDetectionSession).where(
           FaceDetectionSession.training_session_id == session_id
       )
       face_session_result = await db.execute(face_session_query)
       face_session = face_session_result.scalar_one_or_none()

       # Calculate phase progress
       phase1_pct = (
           (training_session.processed_images / training_session.total_images * 100)
           if training_session.total_images > 0
           else 0.0
       )
       phase1_complete = training_session.status == SessionStatus.COMPLETED.value

       phase2_pct = 0.0
       phase2_complete = False
       phase3_complete = False

       if face_session:
           phase2_pct = (
               (face_session.processed_images / face_session.total_images * 100)
               if face_session.total_images > 0
               else 0.0
           )
           phase2_complete = face_session.status == FaceDetectionSessionStatus.COMPLETED.value
           # Phase 3 completes with Phase 2 (inline operation)
           phase3_complete = phase2_complete and face_session.clusters_created is not None

       # Calculate overall progress (weighted)
       overall_pct = (0.30 * phase1_pct) + (0.65 * phase2_pct) + (0.05 * (100 if phase3_complete else 0))

       # Determine current phase
       if phase3_complete:
           current_phase = "completed"
       elif face_session and face_session.status == FaceDetectionSessionStatus.PROCESSING.value:
           current_phase = "face_detection"
       elif phase1_complete:
           current_phase = "face_detection"  # Pending or starting
       else:
           current_phase = "training"

       # Determine overall status
       if (training_session.status == SessionStatus.FAILED.value or
           (face_session and face_session.status == FaceDetectionSessionStatus.FAILED.value)):
           overall_status = "failed"
       elif phase1_complete and phase2_complete and phase3_complete:
           overall_status = "completed"
       elif (training_session.status == SessionStatus.RUNNING.value or
             (face_session and face_session.status == FaceDetectionSessionStatus.PROCESSING.value)):
           overall_status = "running"
       else:
           overall_status = "pending"

       # Calculate combined ETA
       eta_seconds = self._calculate_combined_eta(training_session, face_session)

       return UnifiedProgressResponse(
           session_id=session_id,
           overall_status=overall_status,
           overall_progress=OverallProgress(
               percentage=round(overall_pct, 2),
               eta_seconds=eta_seconds,
               current_phase=current_phase,
           ),
           phases={
               "training": PhaseProgress(
                   name="training",
                   status=training_session.status,
                   progress=ProgressStats(
                       current=training_session.processed_images,
                       total=training_session.total_images,
                       percentage=round(phase1_pct, 2),
                       eta_seconds=None,
                       images_per_minute=None,
                   ),
                   started_at=training_session.started_at.isoformat() if training_session.started_at else None,
                   completed_at=training_session.completed_at.isoformat() if training_session.completed_at else None,
               ),
               "faceDetection": PhaseProgress(
                   name="face_detection",
                   status=face_session.status if face_session else "pending",
                   progress=ProgressStats(
                       current=face_session.processed_images if face_session else 0,
                       total=face_session.total_images if face_session else training_session.total_images,
                       percentage=round(phase2_pct, 2),
                       eta_seconds=None,
                       images_per_minute=None,
                   ),
                   started_at=face_session.started_at.isoformat() if face_session and face_session.started_at else None,
                   completed_at=face_session.completed_at.isoformat() if face_session and face_session.completed_at else None,
               ),
               "clustering": PhaseProgress(
                   name="clustering",
                   status="completed" if phase3_complete else (
                       "running" if face_session and face_session.status == FaceDetectionSessionStatus.PROCESSING.value else "pending"
                   ),
                   progress=ProgressStats(
                       current=1 if phase3_complete else 0,
                       total=1,
                       percentage=100.0 if phase3_complete else 0.0,
                       eta_seconds=None,
                       images_per_minute=None,
                   ),
               ),
           },
       )

   def _calculate_combined_eta(
       self,
       training_session,
       face_session
   ) -> int | None:
       """Calculate ETA for all remaining phases."""
       # If training is running, calculate based on training rate
       if training_session.status == SessionStatus.RUNNING.value and training_session.started_at:
           elapsed = (datetime.now(UTC) - training_session.started_at).total_seconds()
           current = training_session.processed_images
           total = training_session.total_images

           if elapsed > 0 and current > 0:
               images_per_sec = current / elapsed
               remaining_training = total - current

               # Estimate Phase 1 remaining
               phase1_remaining = remaining_training / images_per_sec if images_per_sec > 0 else 0

               # Estimate Phase 2 (face detection is ~4x slower than training)
               phase2_estimate = total * 4  # Conservative estimate

               # Estimate Phase 3 (clustering is fixed ~15 seconds)
               phase3_estimate = 15

               return int(phase1_remaining + phase2_estimate + phase3_estimate)

       # If face detection is running, calculate based on face detection rate
       if face_session and face_session.status == FaceDetectionSessionStatus.PROCESSING.value and face_session.started_at:
           elapsed = (datetime.now(UTC) - face_session.started_at).total_seconds()
           current = face_session.processed_images
           total = face_session.total_images

           if elapsed > 0 and current > 0:
               images_per_sec = current / elapsed
               remaining_faces = total - current

               phase2_remaining = remaining_faces / images_per_sec if images_per_sec > 0 else 0
               phase3_estimate = 15

               return int(phase2_remaining + phase3_estimate)

       return None
   ```

3. **Add new endpoint** to `src/image_search_service/api/routes/training.py`
   ```python
   @router.get(
       "/sessions/{session_id}/progress-unified",
       response_model=UnifiedProgressResponse,
       summary="Get unified progress across all training phases",
       description=(
           "Returns combined progress information for training (CLIP embeddings), "
           "face detection (InsightFace), and clustering (HDBSCAN) phases. "
           "Progress is weighted: 30% training, 65% face detection, 5% clustering."
       ),
   )
   async def get_unified_progress(
       session_id: int,
       db: AsyncSession = Depends(get_db),
   ):
       """Get unified progress across all training phases.

       This endpoint provides a single progress value (0-100%) that accounts for:
       - Phase 1: CLIP embedding generation (30% weight)
       - Phase 2: Face detection with InsightFace (65% weight)
       - Phase 3: HDBSCAN clustering (5% weight)

       Returns detailed progress for each phase along with overall status.
       """
       service = TrainingService()
       return await service.get_session_progress_unified(db, session_id)
   ```

4. **Update API contract** in `image-search-service/docs/api-contract.md`
   ```markdown
   ### Get Unified Training Progress

   Get combined progress across all training phases (training, face detection, clustering).

   **Endpoint**: `GET /api/v1/training/sessions/{sessionId}/progress-unified`

   **Path Parameters**:
   - `sessionId` (integer): Training session ID

   **Response** (200 OK):
   ```json
   {
     "sessionId": 123,
     "overallStatus": "running",
     "overallProgress": {
       "percentage": 62.5,
       "etaSeconds": 180,
       "currentPhase": "face_detection"
     },
     "phases": {
       "training": {
         "name": "training",
         "status": "completed",
         "progress": {
           "current": 1000,
           "total": 1000,
           "percentage": 100.0
         },
         "startedAt": "2026-01-12T10:00:00Z",
         "completedAt": "2026-01-12T10:00:30Z"
       },
       "faceDetection": {
         "name": "face_detection",
         "status": "processing",
         "progress": {
           "current": 500,
           "total": 1000,
           "percentage": 50.0
         },
         "startedAt": "2026-01-12T10:00:35Z",
         "completedAt": null
       },
       "clustering": {
         "name": "clustering",
         "status": "pending",
         "progress": {
           "current": 0,
           "total": 1,
           "percentage": 0.0
         }
       }
     }
   }
   ```

   **Progress Weights**:
   - Training (CLIP embeddings): 30%
   - Face Detection (InsightFace): 65%
   - Clustering (HDBSCAN): 5%

   **Overall Status Values**:
   - `pending`: No phases started
   - `running`: At least one phase is running
   - `completed`: All phases completed successfully
   - `failed`: At least one phase failed
   - `paused`: Training was paused
   ```

5. **Copy contract to frontend**
   ```bash
   cp image-search-service/docs/api-contract.md image-search-ui/docs/api-contract.md
   ```

**Frontend** (`image-search-ui/`)

6. **Regenerate TypeScript types**
   ```bash
   cd image-search-ui
   npm run gen:api  # Generates src/lib/api/generated.ts from OpenAPI
   ```

7. **Add API client method** to `src/lib/api/training.ts`
   ```typescript
   import type { UnifiedProgressResponse } from './generated';

   export async function getUnifiedProgress(sessionId: number): Promise<UnifiedProgressResponse> {
       const response = await fetch(`/api/v1/training/sessions/${sessionId}/progress-unified`);
       if (!response.ok) {
           throw new Error(`Failed to fetch unified progress: ${response.statusText}`);
       }
       return response.json();
   }
   ```

8. **Update SessionDetailView.svelte** (Lines 43-143)
   ```typescript
   <script lang="ts">
   import { getUnifiedProgress } from '$lib/api/training';
   import type { UnifiedProgressResponse } from '$lib/api/generated';

   let unifiedProgress = $state<UnifiedProgressResponse | null>(null);
   let loadingProgress = $state(false);

   async function fetchUnifiedProgress() {
       if (loadingProgress || !session?.id) return;
       loadingProgress = true;
       try {
           unifiedProgress = await getUnifiedProgress(session.id);
       } catch (err) {
           console.error('Failed to fetch unified progress:', err);
       } finally {
           loadingProgress = false;
       }
   }

   function startPolling() {
       if (pollingInterval) return;
       pollingInterval = setInterval(fetchUnifiedProgress, 2000);
       fetchUnifiedProgress(); // Immediate fetch
   }

   function stopPolling() {
       if (pollingInterval) {
           clearInterval(pollingInterval);
           pollingInterval = null;
       }
   }

   // Start/stop polling based on overall status
   $effect(() => {
       const overallStatus = unifiedProgress?.overallStatus;

       untrack(() => {
           if (overallStatus === 'running' && session?.id) {
               startPolling();
           } else {
               stopPolling();
           }
       });

       return () => stopPolling();
   });

   // Initial load
   $effect(() => {
       if (session?.id) {
           untrack(() => fetchUnifiedProgress());
       }
   });
   </script>
   ```

9. **Update progress display** (Lines 182-203)
   ```svelte
   {#if unifiedProgress}
       {@const overall = unifiedProgress.overallProgress}
       {@const trainingPhase = unifiedProgress.phases.training}
       {@const facePhase = unifiedProgress.phases.faceDetection}
       {@const clusterPhase = unifiedProgress.phases.clustering}

       <section class="progress-section">
           <h2>Overall Progress</h2>
           <div class="space-y-1 mb-4">
               <div class="flex justify-between text-sm text-gray-600">
                   <span>
                       {#if overall.currentPhase === 'training'}
                           🎨 Training - Generating CLIP Embeddings
                       {:else if overall.currentPhase === 'face_detection'}
                           😊 Face Detection - Analyzing Faces
                       {:else if overall.currentPhase === 'clustering'}
                           🔗 Clustering - Grouping Similar Faces
                       {:else}
                           ✓ All Phases Complete
                       {/if}
                   </span>
                   <span class="font-semibold">{overall.percentage.toFixed(1)}%</span>
               </div>
               <Progress value={overall.percentage} max={100} class="h-2" />
           </div>

           {#if overall.etaSeconds}
               <ETADisplay eta={new Date(Date.now() + overall.etaSeconds * 1000).toISOString()} />
           {/if}

           <!-- Phase breakdown (collapsible details) -->
           <details class="mt-4">
               <summary class="text-sm text-gray-600 cursor-pointer hover:text-gray-800">
                   View Phase Details
               </summary>
               <div class="mt-2 space-y-2 text-sm">
                   <PhaseProgressBar phase={trainingPhase} label="Training (CLIP)" icon="🎨" />
                   <PhaseProgressBar phase={facePhase} label="Face Detection" icon="😊" />
                   <PhaseProgressBar phase={clusterPhase} label="Clustering" icon="🔗" />
               </div>
           </details>
       </section>
   {/if}
   ```

10. **Create PhaseProgressBar component** at `src/lib/components/training/PhaseProgressBar.svelte`
    ```svelte
    <script lang="ts">
    import { Progress } from '$lib/components/ui/progress';
    import type { PhaseProgress } from '$lib/api/generated';

    interface Props {
        phase: PhaseProgress;
        label: string;
        icon: string;
    }

    let { phase, label, icon }: Props = $props();

    const statusClass = $derived(() => {
        switch (phase.status) {
            case 'completed':
                return 'text-green-600';
            case 'running':
            case 'processing':
                return 'text-blue-600';
            case 'failed':
                return 'text-red-600';
            default:
                return 'text-gray-400';
        }
    });

    const statusText = $derived(() => {
        switch (phase.status) {
            case 'completed':
                return '✓ Complete';
            case 'running':
            case 'processing':
                return `${phase.progress.percentage.toFixed(0)}%`;
            case 'failed':
                return '✗ Failed';
            default:
                return 'Pending';
        }
    });
    </script>

    <div class="flex items-center gap-3 p-2 bg-gray-50 rounded">
        <span class="text-xl">{icon}</span>
        <div class="flex-1">
            <div class="flex justify-between items-center mb-1">
                <span class="text-xs font-medium">{label}</span>
                <span class="text-xs {statusClass()}" class:font-bold={phase.status === 'running' || phase.status === 'processing'}>
                    {statusText()}
                </span>
            </div>
            {#if phase.status === 'running' || phase.status === 'processing'}
                <Progress value={phase.progress.percentage} max={100} class="h-1" />
            {/if}
        </div>
    </div>
    ```

#### Acceptance Criteria
- ✅ New endpoint `/api/v1/training/sessions/{id}/progress-unified` returns unified progress
- ✅ Overall progress is weighted: 30% training + 65% face detection + 5% clustering
- ✅ API contract updated in both `image-search-service` and `image-search-ui`
- ✅ TypeScript types regenerated from OpenAPI spec
- ✅ Frontend displays accurate 0-100% progress across all phases
- ✅ Progress bar shows combined percentage (not 100% at Phase 1 completion)
- ✅ Phase details available via collapsible section
- ✅ ETA accounts for all remaining phases
- ✅ Polling continues until all phases complete
- ✅ Backend tests pass (`make test`)
- ✅ Frontend tests pass (`npm run test`)
- ✅ No breaking changes to existing `/progress` endpoint

#### Testing

**Backend Tests** (`image-search-service/tests/api/test_training_unified_progress.py`):
```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_unified_progress_phase1_running(async_client: AsyncClient):
    """Test progress when only Phase 1 is running."""
    # Create training session with 100 images, 50 processed
    # Expected: 30% * 0.5 = 15% overall progress

    response = await async_client.get("/api/v1/training/sessions/1/progress-unified")
    assert response.status_code == 200

    data = response.json()
    assert data["overallProgress"]["percentage"] == 15.0
    assert data["overallProgress"]["currentPhase"] == "training"
    assert data["phases"]["training"]["status"] == "running"
    assert data["phases"]["faceDetection"]["status"] == "pending"

@pytest.mark.asyncio
async def test_unified_progress_phase1_complete_phase2_running(async_client: AsyncClient):
    """Test progress when Phase 1 done, Phase 2 running."""
    # Training complete (100%), face detection 50% done
    # Expected: 30% + (65% * 0.5) = 62.5% overall progress

    response = await async_client.get("/api/v1/training/sessions/1/progress-unified")
    assert response.status_code == 200

    data = response.json()
    assert data["overallProgress"]["percentage"] == 62.5
    assert data["overallProgress"]["currentPhase"] == "face_detection"
    assert data["phases"]["training"]["status"] == "completed"
    assert data["phases"]["faceDetection"]["status"] == "processing"

@pytest.mark.asyncio
async def test_unified_progress_all_complete(async_client: AsyncClient):
    """Test progress when all phases complete."""
    # All phases done
    # Expected: 100% overall progress

    response = await async_client.get("/api/v1/training/sessions/1/progress-unified")
    assert response.status_code == 200

    data = response.json()
    assert data["overallProgress"]["percentage"] == 100.0
    assert data["overallStatus"] == "completed"
    assert data["phases"]["training"]["status"] == "completed"
    assert data["phases"]["faceDetection"]["status"] == "completed"
    assert data["phases"]["clustering"]["status"] == "completed"
```

**Frontend Tests** (`image-search-ui/src/tests/components/SessionDetailView.test.ts`):
```typescript
import { render, screen, waitFor } from '@testing-library/svelte';
import { describe, test, expect, vi } from 'vitest';
import SessionDetailView from '$lib/components/training/SessionDetailView.svelte';
import { mockResponse } from '../helpers/mockFetch';
import { createTrainingSession } from '../helpers/fixtures';

describe('SessionDetailView - Unified Progress', () => {
    test('shows combined progress when face detection running', async () => {
        mockResponse('/api/v1/training/sessions/1/progress-unified', {
            sessionId: 1,
            overallStatus: 'running',
            overallProgress: {
                percentage: 62.5,
                etaSeconds: 120,
                currentPhase: 'face_detection',
            },
            phases: {
                training: {
                    name: 'training',
                    status: 'completed',
                    progress: { current: 1000, total: 1000, percentage: 100.0 },
                },
                faceDetection: {
                    name: 'face_detection',
                    status: 'processing',
                    progress: { current: 500, total: 1000, percentage: 50.0 },
                },
                clustering: {
                    name: 'clustering',
                    status: 'pending',
                    progress: { current: 0, total: 1, percentage: 0.0 },
                },
            },
        });

        render(SessionDetailView, {
            props: { session: createTrainingSession({ id: 1, status: 'running' }) },
        });

        await waitFor(() => {
            expect(screen.getByText('62.5%')).toBeInTheDocument();
            expect(screen.getByText(/Face Detection - Analyzing Faces/i)).toBeInTheDocument();
        });
    });

    test('continues polling while face detection runs', async () => {
        const mockFetch = vi.fn();
        global.fetch = mockFetch;

        mockFetch.mockResolvedValue({
            ok: true,
            json: async () => ({
                overallStatus: 'running',
                overallProgress: { percentage: 50, currentPhase: 'face_detection' },
                phases: { /* ... */ },
            }),
        });

        render(SessionDetailView, {
            props: { session: createTrainingSession({ id: 1, status: 'running' }) },
        });

        await waitFor(() => expect(mockFetch).toHaveBeenCalled());

        // Wait 4 seconds (should call at 0s, 2s, 4s = 3 times)
        await new Promise((resolve) => setTimeout(resolve, 4100));

        expect(mockFetch.mock.calls.length).toBeGreaterThanOrEqual(3);
    });

    test('shows phase breakdown when expanded', async () => {
        mockResponse('/api/v1/training/sessions/1/progress-unified', {
            /* ... */
        });

        render(SessionDetailView, {
            props: { session: createTrainingSession({ id: 1 }) },
        });

        await waitFor(() => screen.getByText('View Phase Details'));

        const details = screen.getByText('View Phase Details').closest('details');
        details?.click();

        await waitFor(() => {
            expect(screen.getByText(/Training \(CLIP\)/i)).toBeInTheDocument();
            expect(screen.getByText(/Face Detection/i)).toBeInTheDocument();
            expect(screen.getByText(/Clustering/i)).toBeInTheDocument();
        });
    });
});
```

---

### 🔵 Phase 3: Enhanced UI with Phase Indicators (Optional, 3-4 hours)
**Goal**: Professional phase breakdown with icons and detailed stats
**Effort**: 3-4 hours
**Priority**: P2 (Nice-to-have)

This phase is **optional** and can be implemented later if desired. It enhances the visual presentation of the unified progress with:
- Horizontal phase indicators (🎨 → 😊 → 🔗)
- Per-phase progress bars and ETAs
- Visual state transitions (pending → running → complete)
- Detailed statistics per phase (images processed, faces detected, clusters created)

**Implementation details available in research document** (lines 743-788).

---

## API Contract Synchronization Workflow

### 🔴 CRITICAL: Follow This Exact Sequence

```bash
# STEP 1: Update backend contract
cd image-search-service
vim docs/api-contract.md  # Add unified progress endpoint spec

# STEP 2: Implement backend changes
vim src/image_search_service/api/training_schemas.py  # Add Pydantic schemas
vim src/image_search_service/services/training_service.py  # Add service method
vim src/image_search_service/api/routes/training.py  # Add endpoint

# STEP 3: Run backend quality checks
make lint
make typecheck
make test

# STEP 4: Verify OpenAPI generation
make dev  # Start server
curl http://localhost:8000/openapi.json | jq '.paths."/api/v1/training/sessions/{sessionId}/progress-unified"'

# STEP 5: Copy contract to frontend (MUST BE IDENTICAL)
cp image-search-service/docs/api-contract.md image-search-ui/docs/api-contract.md

# STEP 6: Regenerate frontend types (MANDATORY)
cd ../image-search-ui
npm run gen:api  # Generates src/lib/api/generated.ts from OpenAPI spec

# STEP 7: Implement frontend changes
vim src/lib/api/training.ts  # Add API client method
vim src/lib/components/training/SessionDetailView.svelte  # Update UI
vim src/lib/components/training/PhaseProgressBar.svelte  # New component

# STEP 8: Run frontend quality checks
npm run lint
npm run test

# STEP 9: Manual end-to-end test
# Terminal 1: Backend
cd image-search-service && make dev
# Terminal 2: Frontend
cd image-search-ui && npm run dev
# Browser: Create training session, observe unified progress
```

### Contract Change Checklist
- [ ] Update `image-search-service/docs/api-contract.md`
- [ ] Implement Pydantic schemas in backend
- [ ] Implement service method in backend
- [ ] Add API endpoint in backend
- [ ] Backend tests pass (`make lint && make typecheck && make test`)
- [ ] Verify OpenAPI spec includes new endpoint
- [ ] Copy contract to `image-search-ui/docs/api-contract.md`
- [ ] Regenerate TypeScript types (`npm run gen:api`)
- [ ] Implement frontend API client
- [ ] Update frontend components
- [ ] Frontend tests pass (`npm run lint && npm run test`)
- [ ] Manual E2E test successful

---

## Quality Standards

### Backend (`image-search-service`)
```bash
make lint       # Ruff linting (PEP 8 + more)
make typecheck  # MyPy strict mode (100% coverage)
make test       # Pytest (must work WITHOUT external services)
make format     # Ruff auto-fix + format
```

**Requirements**:
- ✅ 100% type coverage (mypy strict mode)
- ✅ No import-time side effects
- ✅ Tests pass without Postgres/Redis/Qdrant
- ✅ Every feature includes tests in same PR

### Frontend (`image-search-ui`)
```bash
npm run lint    # ESLint 9 + typescript-eslint
npm run test    # Vitest + Testing Library
npm run format  # Prettier + prettier-plugin-svelte
```

**Requirements**:
- ✅ NEVER manually edit `src/lib/api/generated.ts` (auto-generated)
- ✅ Use Testing Library queries (semantic, not test IDs)
- ✅ Centralized mocking via `tests/helpers/mockFetch.ts`
- ✅ Tests pass without backend running
- ✅ Every feature includes tests in same PR

---

## Rollout Strategy

### Phase 1 (Immediate - 1-2 hours)
1. Deploy frontend changes only (no backend changes)
2. Users see improved status messages immediately
3. Polling continues during face detection
4. Low risk (no API changes)

### Phase 2 (Short-term - 4-6 hours)
1. Deploy backend changes first (new endpoint is additive)
2. Verify new endpoint works with manual testing
3. Deploy frontend changes (switch to unified endpoint)
4. Monitor for errors in production logs
5. Old `/progress` endpoint remains available (backward compatible)

### Phase 3 (Optional - 3-4 hours)
1. Deploy enhanced UI as incremental improvement
2. A/B test with subset of users (if desired)
3. Gather feedback on phase indicators

---

## Backward Compatibility

### Existing Endpoints Remain Unchanged
- `/api/v1/training/sessions/{id}/progress` (Phase 1 only) - KEEP
- `/api/v1/face-sessions/{id}` (Phase 2/3 combined) - KEEP

### New Endpoint is Additive
- `/api/v1/training/sessions/{id}/progress-unified` - NEW

### Migration Path
1. Phase 1: Frontend still uses old endpoints (no backend changes)
2. Phase 2: Frontend switches to new unified endpoint
3. Future: Old endpoints can be deprecated after migration period (6+ months)

---

## Performance Considerations

### Backend
- Single database query for training session
- Single database query for face detection session
- No additional computational overhead (calculations are simple arithmetic)
- Response size: ~1KB JSON (negligible)

### Frontend
- Polling frequency unchanged (2 seconds)
- Same number of API calls per polling cycle (1 call instead of 2)
- Reduced network overhead (unified response vs separate requests)
- UI rendering performance unchanged (same component structure)

---

## Monitoring and Observability

### Backend Logging
```python
# Add to training_service.py
logger.info(
    f"Unified progress calculated for session {session_id}: "
    f"overall={overall_pct:.1f}%, phase1={phase1_pct:.1f}%, "
    f"phase2={phase2_pct:.1f}%, phase3={100 if phase3_complete else 0}%"
)
```

### Frontend Logging
```typescript
// Add to SessionDetailView.svelte
console.debug(
    'Unified progress fetched:',
    unifiedProgress.overallProgress.percentage,
    'Current phase:',
    unifiedProgress.overallProgress.currentPhase
);
```

### Metrics to Track
- Unified progress endpoint response time (should be < 100ms)
- Polling frequency (ensure 2-second interval maintained)
- Frontend render performance (ensure no regression)
- Error rate for unified progress endpoint

---

## Risk Assessment

### Low Risk
- Phase 1 (UI labels) - No backend changes, easily reversible
- Phase 2 (unified endpoint) - Additive change, no breaking changes

### Medium Risk
- TypeScript type regeneration - Could break if OpenAPI spec malformed
- Mitigation: Test type generation in dev environment first

### Negligible Risk
- Phase 3 (enhanced UI) - Optional, cosmetic changes only

### Rollback Plan
- Phase 1: Revert frontend commit (no backend involved)
- Phase 2: Frontend falls back to old `/progress` endpoint (if issues)
- Backend endpoint can remain (no harm if unused)

---

## Open Questions

1. **Should face detection be optional?**
   - Currently auto-triggered with no way to disable
   - Future enhancement: Add `skip_face_detection` flag to `TrainingSessionCreate`
   - Decision: Out of scope for this implementation (can add later)

2. **Should clustering be a separate job?**
   - Currently inline (5-30 seconds, no progress tracking)
   - Trade-off: More complex orchestration vs better visibility
   - Decision: Keep inline for now (complexity not justified by short duration)

3. **What if face detection fails?**
   - Currently: Training shows "completed" even if faces fail
   - Proposed: `overallStatus` reflects all phases (implemented in Phase 2)
   - Decision: Implemented in unified endpoint

4. **Should we show warnings for slow progress?**
   - Example: "Face detection taking longer than expected"
   - Decision: Out of scope (can add later based on user feedback)

---

## Success Metrics

### User Experience
- ✅ Users see accurate progress from 0% to 100% (not false 100% at 30 seconds)
- ✅ Users understand what phase is currently running
- ✅ Users have realistic ETA for complete pipeline
- ✅ No confusion about "completed" status while work continues

### Technical
- ✅ Unified progress calculation is accurate (within 5% of actual time)
- ✅ Polling continues until all phases complete
- ✅ No increase in error rate after deployment
- ✅ API response time < 100ms for unified endpoint

### Testing
- ✅ Backend test coverage for unified progress endpoint > 80%
- ✅ Frontend test coverage for SessionDetailView > 80%
- ✅ Manual E2E test passes for 1000-image training session

---

## References

- **Comprehensive Analysis**: `docs/research/training-progress-tracking-comprehensive-analysis-2026-01-12.md`
- **Quick Summary**: `TRAINING_PROGRESS_SUMMARY.md`
- **Architecture Diagram**: `docs/research/training-workflow-architecture-diagram.md`
- **Monorepo Guide**: `CLAUDE.md`
- **Backend Guide**: `image-search-service/CLAUDE.md`
- **Frontend Guide**: `image-search-ui/CLAUDE.md`
- **API Contract**: `docs/api-contract.md` (in both subprojects)

---

## Estimated Timeline

### Phase 1: Immediate UX Fix
- **Effort**: 1-2 hours
- **Tasks**: Update UI labels, extend polling logic
- **Testing**: 30 minutes
- **Total**: 1.5-2.5 hours

### Phase 2: Unified Progress Endpoint
- **Backend**: 2 hours (schemas, service, endpoint, tests)
- **Frontend**: 2 hours (API client, component updates, tests)
- **Testing**: 2 hours (E2E, verification, contract sync)
- **Total**: 6 hours

### Phase 3: Enhanced UI (Optional)
- **Frontend**: 3 hours (PhaseProgressBar, layout, styling)
- **Testing**: 1 hour
- **Total**: 4 hours

**Overall Timeline**: 11.5-12.5 hours across all phases (Phase 3 optional)

---

## Next Steps

1. **Review this plan** with team/stakeholders
2. **Prioritize phases** (Phase 1 immediate, Phase 2 recommended, Phase 3 optional)
3. **Assign implementation** to appropriate developers
4. **Schedule deployment windows** for each phase
5. **Set up monitoring** for unified progress endpoint
6. **Plan user communication** about improved progress tracking

---

**Plan Created**: 2026-01-12
**Plan Status**: Ready for Review
**Recommended Action**: Implement Phase 1 immediately, schedule Phase 2 for next sprint
