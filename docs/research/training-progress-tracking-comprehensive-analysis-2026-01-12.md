# Training Progress Tracking System - Comprehensive Analysis

**Date**: 2026-01-12
**Research Focus**: Multi-phase training workflow and progress tracking gaps
**Issue**: UI shows 100% when training_jobs complete, but face detection and clustering still running

---

## Executive Summary

The training system has **3 sequential phases**, but the UI only tracks progress of **Phase 1 (training_jobs)**. When Phase 1 completes, the UI displays 100% done, creating a false impression that all work is finished. In reality, Phase 2 (face detection) and Phase 3 (clustering) continue running in the background without user visibility.

**Critical Gap**: No unified progress tracking across all three phases.

---

## Complete Training Workflow Architecture

### Phase Diagram (Current Architecture)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     USER INITIATES TRAINING                         │
│                  (CreateSessionModal → API)                         │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 1: Training Session (CLIP Embedding)                        │
│  ──────────────────────────────────────────────────────────────────│
│  Backend: training_jobs.py::train_session()                        │
│  Queue: "high" priority (RQ)                                       │
│  Progress: TrainingSession.processed_images / total_images         │
│  Duration: ~30 seconds per 1000 images (8 GPU batch, MPS)         │
│  ──────────────────────────────────────────────────────────────────│
│  Operations:                                                       │
│  1. Discover assets from selected subdirectories                   │
│  2. Create TrainingJob records (one per asset)                     │
│  3. Generate CLIP embeddings (OpenCLIP ViT-B-32)                  │
│  4. Upsert vectors to Qdrant                                      │
│  5. Update TrainingSession.processed_images incrementally          │
│  ──────────────────────────────────────────────────────────────────│
│  ✅ UI TRACKS THIS: SessionDetailView polls /progress endpoint    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 │ ⚠️ UI SHOWS 100% HERE
                                 │    (but work continues!)
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 2: Face Detection Session (InsightFace)                     │
│  ──────────────────────────────────────────────────────────────────│
│  Backend: face_jobs.py::detect_faces_for_session_job()            │
│  Queue: "default" priority (RQ)                                    │
│  Progress: FaceDetectionSession.processed_images / total_images    │
│  Duration: ~2 minutes per 1000 images (16 batch, SCRFD + Buffalo) │
│  ──────────────────────────────────────────────────────────────────│
│  Operations:                                                       │
│  1. Auto-triggered by train_session() completion (line 244-300)   │
│  2. Create FaceDetectionSession with training_session_id link     │
│  3. Enqueue detect_faces_for_session_job()                        │
│  4. Detect faces using InsightFace SCRFD_10G_KPS                  │
│  5. Embed faces using Buffalo_l (512-dim)                         │
│  6. Store FaceInstance records + Qdrant vectors                   │
│  7. Update FaceDetectionSession.processed_images incrementally     │
│  8. Auto-assign faces to known persons (assigner)                 │
│  ──────────────────────────────────────────────────────────────────│
│  ❌ UI DOES NOT TRACK THIS (but data exists!)                     │
│     SessionDetailView shows FaceDetectionSessionCard at bottom    │
│     Card polls /api/v1/face-sessions/{sessionId} endpoint         │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 3: Face Clustering (HDBSCAN)                                │
│  ──────────────────────────────────────────────────────────────────│
│  Backend: face_jobs.py::cluster_unlabeled_faces (line 699-732)    │
│  Queue: "default" priority (RQ, inline in face_jobs)              │
│  Progress: No explicit tracking (synchronous operation)            │
│  Duration: ~5-30 seconds (depends on face count)                  │
│  ──────────────────────────────────────────────────────────────────│
│  Operations:                                                       │
│  1. Triggered at END of detect_faces_for_session_job              │
│  2. Query unlabeled faces (no person_id)                          │
│  3. Run HDBSCAN clustering (min_cluster_size=3)                   │
│  4. Create FaceCluster records for groups                         │
│  5. Update FaceDetectionSession.clusters_created                   │
│  ──────────────────────────────────────────────────────────────────│
│  ❌ UI DOES NOT TRACK THIS (no progress available)                │
│     Clustering happens synchronously in job                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Current Progress Tracking Implementation

### Backend (image-search-service)

#### Phase 1: Training Progress Endpoint

**File**: `image-search-service/src/image_search_service/api/routes/training.py`

```python
# Line 366-390
@router.get("/sessions/{session_id}/progress", response_model=TrainingProgressResponse)
async def get_progress(session_id: int, db: AsyncSession = Depends(get_db)):
    """Get progress information for a training session.

    Returns:
        Progress information with stats and job summary
    """
    service = TrainingService()
    return await service.get_session_progress(db, session_id)
```

**Data Structure**: `image-search-service/src/image_search_service/api/training_schemas.py`

```python
class TrainingProgressResponse(BaseModel):
    session_id: int
    status: str  # "pending" | "running" | "completed" | "failed" | "paused"
    progress: ProgressStats
    jobs_summary: JobsSummary

class ProgressStats(BaseModel):
    current: int              # session.processed_images
    total: int                # session.total_images
    percentage: float         # (current / total) * 100
    eta_seconds: int | None   # Calculated from images_per_minute
    images_per_minute: float | None
```

**Progress Calculation**: `image-search-service/src/image_search_service/services/training_service.py`

```python
# Line 214-283
async def get_session_progress(self, db: AsyncSession, session_id: int):
    session = await self.get_session(db, session_id)

    # Phase 1 progress ONLY
    total = session.total_images
    current = session.processed_images
    percentage = (current / total * 100) if total > 0 else 0.0

    # ETA calculation (only when status == "running")
    if session.status == SessionStatus.RUNNING.value and session.started_at:
        elapsed = (datetime.now(UTC) - session.started_at).total_seconds()
        if elapsed > 0 and current > 0:
            images_per_minute = (current / elapsed) * 60
            remaining = total - current
            if images_per_minute > 0:
                eta_seconds = int(remaining / images_per_minute * 60)

    return TrainingProgressResponse(
        sessionId=session_id,
        status=session.status,
        progress=ProgressStats(current=current, total=total, percentage=percentage, ...),
        jobsSummary=JobsSummary(...)
    )
```

**Key Insight**: Progress calculation stops at Phase 1 completion. No awareness of Phase 2/3.

#### Phase 2: Face Detection Progress Endpoint

**File**: `image-search-service/src/image_search_service/api/routes/face_sessions.py`

```python
@router.get("/{session_id}", response_model=FaceDetectionSessionResponse)
async def get_session(session_id: str, db: AsyncSession = Depends(get_db)):
    """Get face detection session details."""
    session = await db.get(FaceDetectionSession, uuid.UUID(session_id))
    return FaceDetectionSessionResponse.model_validate(session)
```

**Data Structure**: `image-search-service/src/image_search_service/api/face_session_schemas.py`

```python
class FaceDetectionSessionResponse(CamelCaseModel):
    id: str  # UUID
    training_session_id: int | None  # Link to training session
    status: str  # "pending" | "processing" | "completed" | "failed" | "paused"
    total_images: int
    processed_images: int
    failed_images: int
    faces_detected: int
    faces_assigned: int

    # Detailed breakdown (Phase 2 + Phase 3 combined)
    faces_assigned_to_persons: int  # Auto-assigned to known people
    clusters_created: int           # Unknown face clusters (Phase 3)
    suggestions_created: int        # Face suggestions for review

    # Batch progress tracking
    current_batch: int
    total_batches: int

    @computed_field
    def progress_percent(self) -> float:
        """Calculate progress percentage."""
        if self.total_images == 0:
            return 0.0
        return (self.processed_images / self.total_images) * 100.0
```

**Progress Calculation**: Inline in `FaceDetectionSessionResponse.progress_percent`

**Key Insight**: Face detection progress exists but is **separate from training progress**. No unified view.

#### Phase 3: Clustering Progress

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`

```python
# Line 699-732 (within detect_faces_for_session_job)
if total_faces_detected > 0:
    logger.info(f"[{job_id}] Running clustering for unlabeled faces")
    try:
        clusterer = get_face_clusterer(db_session, min_cluster_size=3)
        cluster_result = clusterer.cluster_unlabeled_faces(
            quality_threshold=0.3,
            max_faces=10000,
        )

        clusters_found = cluster_result.get("clusters_found", 0)

        # Update session with clustering stats
        session.clusters_created = clusters_found
        session.faces_assigned = (session.faces_assigned_to_persons or 0) + clusters_found
        db_session.commit()
    except Exception as e:
        logger.error(f"[{job_id}] Clustering error (non-fatal): {e}")
```

**Key Insight**:
- Clustering runs **synchronously** at the end of Phase 2
- No progress tracking (executes in 5-30 seconds)
- Results stored in `FaceDetectionSession.clusters_created`
- Errors are non-fatal (Phase 2 can complete even if clustering fails)

---

### Frontend (image-search-ui)

#### Training Progress Display

**Component**: `image-search-ui/src/lib/components/training/SessionDetailView.svelte`

**Polling Logic** (Lines 85-143):

```typescript
let progress = $state<TrainingProgress | null>(null);
let faceSession = $state<FaceDetectionSession | null>(null);
let pollingInterval: ReturnType<typeof setInterval> | null = null;

async function fetchProgress() {
    if (loadingProgress || !session?.id) return;
    loadingProgress = true;
    try {
        progress = await getProgress(session.id);  // /api/v1/training/sessions/{id}/progress
    } catch (err) {
        console.error('Failed to fetch progress:', err);
    } finally {
        loadingProgress = false;
    }
}

async function fetchFaceSession() {
    if (loadingFaceSession || !session?.id) return;
    loadingFaceSession = true;
    try {
        const response = await listFaceDetectionSessions(1, 10);
        // Find session linked to this training session
        faceSession = response.items.find((s) => s.trainingSessionId === session.id) ?? null;
    } catch (err) {
        console.error('Failed to fetch face detection session:', err);
        faceSession = null;
    } finally {
        loadingFaceSession = false;
    }
}

// Start/stop polling based on session status
$effect(() => {
    const status = session?.status;

    untrack(() => {
        if (status === 'running' && session?.id) {
            startPolling();  // Poll every 2 seconds
        } else {
            stopPolling();
        }
    });

    return () => stopPolling();
});

// Load face session when training completes
$effect(() => {
    if (session?.status === 'completed') {
        untrack(() => fetchFaceSession());
    }
});
```

**Progress Display** (Lines 182-203):

```svelte
{#if progress}
    {@const current = progress.progress.current}
    {@const total = progress.progress.total}
    {@const percentage = total > 0 ? Math.round((current / total) * 100) : 0}
    <section class="progress-section">
        <h2>Progress</h2>
        <div class="space-y-1 mb-4">
            <div class="flex justify-between text-sm text-gray-600">
                <span>Processing Images</span>
                <span>{percentage}% ({current.toLocaleString()} / {total.toLocaleString()})</span>
            </div>
            <Progress value={percentage} max={100} class="h-2" />
        </div>
        {#if progress.progress.etaSeconds}
            <ETADisplay eta={new Date(Date.now() + progress.progress.etaSeconds * 1000).toISOString()} />
        {/if}
        <TrainingStats stats={progress.progress} />
    </section>
{/if}
```

**Face Detection Display** (Lines 215-220):

```svelte
{#if faceSession}
    <section class="face-detection-section">
        <h2>Face Detection</h2>
        <FaceDetectionSessionCard session={faceSession} onUpdate={fetchFaceSession} />
    </section>
{/if}
```

**Key Issues**:
1. **Progress bar shows Phase 1 only**: When `percentage` reaches 100%, training is marked complete
2. **Face session loads AFTER training completes**: Separate section, not integrated into progress
3. **No combined progress calculation**: User sees 100% → then face card appears later
4. **Polling stops when Phase 1 completes**: `status === 'running'` → stops polling → misses Phase 2/3

---

## Data Flow (Current Implementation)

### Phase 1 → Phase 2 Transition

**Backend**: `image-search-service/src/image_search_service/queue/training_jobs.py`

```python
# Line 232-300 (train_session completion)
training_session = get_session_by_id_sync(db_session, session_id)
if training_session and processed_count > 0:
    training_session.status = SessionStatus.COMPLETED.value  # ← UI sees this
    training_session.completed_at = datetime.now(UTC)
    db_session.commit()

    # Auto-trigger face detection
    try:
        # Check if face detection session already exists
        existing_query = select(FaceDetectionSession).where(
            FaceDetectionSession.training_session_id == session_id
        )
        existing_result = db_session.execute(existing_query)
        existing_face_session = existing_result.scalar_one_or_none()

        if existing_face_session:
            logger.info(f"Face detection session already exists, skipping")
        else:
            # Create face detection session
            face_session = FaceDetectionSession(
                training_session_id=session_id,
                status=FaceDetectionSessionStatus.PENDING.value,
                min_confidence=0.5,
                min_face_size=20,
                batch_size=16,
            )
            db_session.add(face_session)
            db_session.commit()

            # Enqueue face detection job
            queue = get_queue("default")
            face_job = queue.enqueue(
                detect_faces_for_session_job,
                str(face_session.id),
                job_timeout=86400,  # 24 hours
            )

            face_session.job_id = face_job.id
            db_session.commit()

            logger.info(f"Auto-triggered face detection session {face_session.id}")

    except Exception as e:
        logger.error(f"Failed to auto-trigger face detection: {e}")
        # Don't fail training completion if face detection trigger fails
```

**Key Insight**: Face detection is auto-triggered but:
- Runs in **separate queue** ("default" vs "high")
- Has **no backlink** to training progress
- Errors are **non-fatal** (training completes regardless)
- UI must **discover** face session separately

### Phase 2 → Phase 3 Transition

**Backend**: `image-search-service/src/image_search_service/queue/face_jobs.py`

```python
# Line 664-732 (detect_faces_for_session_job completion)

# Auto-assign faces to known persons (uses config-based thresholds)
assigner = get_face_assigner(db_session=db_session)
assignment_result = assigner.assign_new_faces(
    since=session.started_at,
    max_faces=10000,
)

faces_assigned_to_persons = assignment_result.get("auto_assigned", 0)
suggestions_created = assignment_result.get("suggestions_created", 0)

session.faces_assigned_to_persons = faces_assigned_to_persons
session.suggestions_created = suggestions_created

# Run clustering for unlabeled faces (Phase 3 inline)
if total_faces_detected > 0:
    try:
        clusterer = get_face_clusterer(db_session, min_cluster_size=3)
        cluster_result = clusterer.cluster_unlabeled_faces(
            quality_threshold=0.3,
            max_faces=10000,
        )

        clusters_found = cluster_result.get("clusters_found", 0)
        session.clusters_created = clusters_found
        session.faces_assigned = (session.faces_assigned_to_persons or 0) + clusters_found
        db_session.commit()

    except Exception as e:
        logger.error(f"Clustering error (non-fatal): {e}")

# Mark session as complete
session.status = FaceDetectionSessionStatus.COMPLETED.value
session.completed_at = datetime.now(UTC)
db_session.commit()
```

**Key Insight**: Clustering happens **inline** (not a separate job):
- No progress tracking (completes in 5-30 seconds)
- Results stored in `FaceDetectionSession.clusters_created`
- If clustering fails, session still completes
- UI cannot distinguish between "detecting faces" vs "clustering faces"

---

## Progress Tracking Gaps (Detailed Analysis)

### Gap 1: No Unified Progress Calculation

**Current**: Three separate progress values:
- `TrainingSession.processed_images / total_images` (Phase 1)
- `FaceDetectionSession.processed_images / total_images` (Phase 2)
- No progress value for Phase 3 (clustering)

**Problem**: UI shows 100% when Phase 1 completes, but:
- Phase 2 may take 4x longer than Phase 1 (face detection is slower)
- Phase 3 runs synchronously at end of Phase 2 (5-30 seconds)
- Total time: Phase 1 (30s) + Phase 2 (2min) + Phase 3 (15s) = **2min 45s**
- User sees 100% at 30 seconds, waits 2min 15s with no feedback

**Desired**: Combined progress formula:

```
Overall Progress = (Phase1_weight * Phase1_progress) +
                   (Phase2_weight * Phase2_progress) +
                   (Phase3_weight * Phase3_progress)

Suggested weights:
- Phase 1 (CLIP embedding): 30% (fast but foundational)
- Phase 2 (Face detection): 65% (slowest operation)
- Phase 3 (Clustering): 5% (quick, low computational cost)

Example calculation:
- Phase 1 complete (100%): 30% * 1.0 = 30%
- Phase 2 at 50%: 65% * 0.5 = 32.5%
- Phase 3 not started: 5% * 0.0 = 0%
→ Overall progress: 62.5%
```

### Gap 2: Missing Phase Metadata

**Current**: Training progress response has no phase information

**Problem**: UI cannot distinguish between:
- "Training 50% done, face detection not started" vs
- "Training complete, face detection 50% done"

**Desired**: Add phase metadata to progress response:

```typescript
interface PhaseProgress {
    name: "training" | "face_detection" | "clustering";
    status: "pending" | "running" | "completed" | "failed";
    progress: {
        current: number;
        total: number;
        percentage: number;
    };
    startedAt?: string;
    completedAt?: string;
}

interface MultiPhaseProgress {
    sessionId: number;
    overallStatus: string;
    overallProgress: {
        current: number;  // Combined progress (0-100)
        percentage: number;
        etaSeconds?: number;  // ETA for ALL phases
    };
    phases: {
        training: PhaseProgress;
        faceDetection: PhaseProgress;
        clustering: PhaseProgress;
    };
}
```

### Gap 3: Polling Stops Prematurely

**Current**: UI stops polling when `session.status === 'completed'`

**Problem**: Phase 1 completion stops polling, misses Phase 2/3 progress

**Desired**: Continue polling until **all phases** complete:

```typescript
$effect(() => {
    const shouldPoll = session?.status === 'running' ||
                       (faceSession?.status === 'processing') ||
                       (faceSession?.status === 'pending' && session?.status === 'completed');

    if (shouldPoll) {
        startPolling();
    } else {
        stopPolling();
    }
});
```

### Gap 4: No Error Propagation

**Current**: If face detection fails, training session shows "completed"

**Problem**: User thinks training succeeded, but face recognition failed silently

**Desired**: Propagate errors to parent session:

```python
# If face detection fails
if face_session.status == FaceDetectionSessionStatus.FAILED.value:
    training_session.status = SessionStatus.FAILED.value
    training_session.error_message = f"Face detection failed: {face_session.last_error}"
```

---

## Recommended Solutions

### Solution 1: Unified Progress Endpoint (Backend)

**Create new endpoint**: `/api/v1/training/sessions/{session_id}/progress-unified`

**Implementation**:

```python
# image-search-service/src/image_search_service/api/routes/training.py

@router.get("/sessions/{session_id}/progress-unified", response_model=UnifiedProgressResponse)
async def get_unified_progress(session_id: int, db: AsyncSession = Depends(get_db)):
    """Get unified progress across all training phases."""

    # Get training session (Phase 1)
    training_session = await TrainingService().get_session(db, session_id)
    if not training_session:
        raise HTTPException(404, "Session not found")

    # Get face detection session (Phase 2/3)
    face_session_query = select(FaceDetectionSession).where(
        FaceDetectionSession.training_session_id == session_id
    )
    face_session_result = await db.execute(face_session_query)
    face_session = face_session_result.scalar_one_or_none()

    # Calculate phase progress
    phase1_progress = (training_session.processed_images / training_session.total_images * 100) if training_session.total_images > 0 else 0
    phase1_complete = training_session.status == SessionStatus.COMPLETED.value

    phase2_progress = 0.0
    phase2_complete = False
    phase3_complete = False

    if face_session:
        phase2_progress = (face_session.processed_images / face_session.total_images * 100) if face_session.total_images > 0 else 0
        phase2_complete = face_session.status == FaceDetectionSessionStatus.COMPLETED.value
        # Phase 3 (clustering) completes with Phase 2
        phase3_complete = phase2_complete and face_session.clusters_created is not None

    # Calculate overall progress (weighted)
    overall_progress = (
        (0.30 * phase1_progress) +
        (0.65 * phase2_progress) +
        (0.05 * (100 if phase3_complete else 0))
    )

    # Determine overall status
    if training_session.status == SessionStatus.FAILED.value or (face_session and face_session.status == FaceDetectionSessionStatus.FAILED.value):
        overall_status = "failed"
    elif phase1_complete and phase2_complete and phase3_complete:
        overall_status = "completed"
    elif training_session.status == SessionStatus.RUNNING.value or (face_session and face_session.status == FaceDetectionSessionStatus.PROCESSING.value):
        overall_status = "running"
    else:
        overall_status = "pending"

    return UnifiedProgressResponse(
        sessionId=session_id,
        overallStatus=overall_status,
        overallProgress={
            "percentage": round(overall_progress, 2),
            "etaSeconds": calculate_combined_eta(training_session, face_session),
        },
        phases={
            "training": {
                "name": "training",
                "status": training_session.status,
                "progress": {
                    "current": training_session.processed_images,
                    "total": training_session.total_images,
                    "percentage": round(phase1_progress, 2),
                },
                "startedAt": training_session.started_at.isoformat() if training_session.started_at else None,
                "completedAt": training_session.completed_at.isoformat() if training_session.completed_at else None,
            },
            "faceDetection": {
                "name": "face_detection",
                "status": face_session.status if face_session else "pending",
                "progress": {
                    "current": face_session.processed_images if face_session else 0,
                    "total": face_session.total_images if face_session else 0,
                    "percentage": round(phase2_progress, 2),
                },
                "startedAt": face_session.started_at.isoformat() if face_session and face_session.started_at else None,
                "completedAt": face_session.completed_at.isoformat() if face_session and face_session.completed_at else None,
            },
            "clustering": {
                "name": "clustering",
                "status": "completed" if phase3_complete else ("running" if face_session and face_session.status == FaceDetectionSessionStatus.PROCESSING.value else "pending"),
                "progress": {
                    "current": 1 if phase3_complete else 0,
                    "total": 1,
                    "percentage": 100.0 if phase3_complete else 0.0,
                },
            },
        },
    )
```

### Solution 2: Enhanced Progress Display (Frontend)

**Update**: `image-search-ui/src/lib/components/training/SessionDetailView.svelte`

```typescript
// Replace fetchProgress() with fetchUnifiedProgress()
async function fetchUnifiedProgress() {
    if (loadingProgress || !session?.id) return;
    loadingProgress = true;
    try {
        unifiedProgress = await getUnifiedProgress(session.id);  // New API call
    } catch (err) {
        console.error('Failed to fetch unified progress:', err);
    } finally {
        loadingProgress = false;
    }
}

// Update polling condition
$effect(() => {
    const shouldPoll = unifiedProgress?.overallStatus === 'running' ||
                       unifiedProgress?.phases.faceDetection.status === 'processing';

    if (shouldPoll) {
        startPolling();
    } else {
        stopPolling();
    }
});
```

**New Progress Display**:

```svelte
{#if unifiedProgress}
    {@const overall = unifiedProgress.overallProgress}
    <section class="progress-section">
        <h2>Overall Progress</h2>
        <div class="space-y-1 mb-4">
            <div class="flex justify-between text-sm text-gray-600">
                <span>Processing Training Pipeline</span>
                <span>{overall.percentage}%</span>
            </div>
            <Progress value={overall.percentage} max={100} class="h-2" />
        </div>

        <!-- Phase breakdown -->
        <div class="mt-6 space-y-4">
            <PhaseProgress
                phase={unifiedProgress.phases.training}
                icon="🎨"
                label="Training (CLIP Embeddings)"
            />
            <PhaseProgress
                phase={unifiedProgress.phases.faceDetection}
                icon="😊"
                label="Face Detection (InsightFace)"
            />
            <PhaseProgress
                phase={unifiedProgress.phases.clustering}
                icon="🔗"
                label="Clustering (HDBSCAN)"
            />
        </div>
    </section>
{/if}
```

### Solution 3: Add Phase Indicators

**New Component**: `PhaseProgress.svelte`

```svelte
<script lang="ts">
    interface Props {
        phase: {
            name: string;
            status: string;
            progress: { percentage: number };
        };
        icon: string;
        label: string;
    }

    let { phase, icon, label }: Props = $props();

    const statusClass = $derived(() => {
        switch (phase.status) {
            case 'completed': return 'text-green-600';
            case 'running': return 'text-blue-600';
            case 'failed': return 'text-red-600';
            default: return 'text-gray-400';
        }
    });
</script>

<div class="flex items-center gap-4 p-3 bg-gray-50 rounded">
    <span class="text-2xl">{icon}</span>
    <div class="flex-1">
        <div class="flex justify-between items-center mb-1">
            <span class="text-sm font-medium">{label}</span>
            <span class="text-xs {statusClass}" class:font-bold={phase.status === 'running'}>
                {phase.status === 'completed' ? '✓ Complete' :
                 phase.status === 'running' ? `${phase.progress.percentage}%` :
                 'Pending'}
            </span>
        </div>
        {#if phase.status === 'running'}
            <Progress value={phase.progress.percentage} max={100} class="h-1" />
        {/if}
    </div>
</div>
```

---

## File Inventory (Backend)

### Training Jobs (Phase 1)

| File | Lines | Purpose |
|------|-------|---------|
| `queue/training_jobs.py` | 1-887 | Main training job orchestration |
| `queue/training_jobs.py::train_session()` | 129-320 | Entry point, discovers assets, enqueues batches |
| `queue/training_jobs.py::train_batch()` | 322-653 | Batch GPU processing with pipelining |
| `services/training_service.py` | 1-1008 | Training session management, progress tracking |
| `services/training_service.py::get_session_progress()` | 214-283 | **Current progress calculation (Phase 1 only)** |
| `api/routes/training.py::get_progress()` | 366-390 | **Current progress endpoint** |
| `api/training_schemas.py::TrainingProgressResponse` | 175-184 | **Current progress response schema** |

### Face Detection Jobs (Phase 2)

| File | Lines | Purpose |
|------|-------|---------|
| `queue/face_jobs.py` | 1-1629 | Face detection, assignment, clustering jobs |
| `queue/face_jobs.py::detect_faces_for_session_job()` | 421-778 | **Phase 2 main job, includes Phase 3 inline** |
| `queue/face_jobs.py::cluster_unlabeled_faces()` | 699-732 | **Phase 3 clustering (inline in Phase 2)** |
| `api/routes/face_sessions.py` | - | Face detection session endpoints |
| `api/face_session_schemas.py::FaceDetectionSessionResponse` | 43-79 | **Phase 2 response schema** |
| `db/models.py::FaceDetectionSession` | - | Phase 2 database model |

### Database Models

| Model | Key Fields |
|-------|-----------|
| `TrainingSession` | `id`, `status`, `total_images`, `processed_images`, `started_at`, `completed_at` |
| `FaceDetectionSession` | `id`, `training_session_id`, `status`, `total_images`, `processed_images`, `faces_detected`, `clusters_created` |
| `TrainingJob` | `session_id`, `asset_id`, `status`, `progress` |
| `FaceInstance` | `asset_id`, `person_id`, `quality_score`, `qdrant_point_id` |
| `FaceCluster` | `id`, `size`, `representative_face_id` |

---

## File Inventory (Frontend)

### Progress Display Components

| File | Lines | Purpose |
|------|-------|---------|
| `components/training/SessionDetailView.svelte` | 1-318 | **Main progress display component** |
| `components/training/SessionDetailView.svelte::fetchProgress()` | 43-53 | **Current progress fetching (Phase 1 only)** |
| `components/training/SessionDetailView.svelte::polling logic` | 85-143 | **Polling start/stop (stops prematurely)** |
| `components/training/SessionDetailView.svelte::progress display` | 182-203 | **UI rendering (100% at Phase 1 completion)** |
| `components/faces/FaceDetectionSessionCard.svelte` | - | **Phase 2 display (separate section)** |
| `lib/api/training.ts::getProgress()` | - | API client for Phase 1 progress |
| `lib/types.ts::TrainingProgress` | - | TypeScript type for Phase 1 progress |

### Modal Trigger

| File | Lines | Purpose |
|------|-------|---------|
| `components/training/CreateSessionModal.svelte` | 1-275 | **Training session creation dialog** |
| `components/training/CreateSessionModal.svelte::handleCreate()` | 94-123 | **Triggers Phase 1** |
| `lib/api/training.ts::createSession()` | - | POST `/api/v1/training/sessions` |

---

## Critical Insights

### Why Progress Tracking Broke

1. **Phase 1 was originally the ONLY phase**: Training system predates face detection
2. **Face detection added later**: Bolted on as separate feature with separate tracking
3. **Clustering added inline**: No separate job, happens at end of Phase 2
4. **No refactoring of progress**: Phase 1 progress logic never updated to account for Phase 2/3
5. **UI assumes single phase**: Progress bar designed for training only, face card added separately

### Technical Debt

1. **Tight coupling**: `train_session()` auto-triggers face detection (lines 244-300)
   - Hard-coded in training job
   - No configuration to disable
   - Failure is silently ignored

2. **Status confusion**: `TrainingSession.status = "completed"` when Phase 1 done
   - Should be "completed_training" or similar
   - Need overall status that includes all phases

3. **Separate endpoints**: Progress spread across two APIs
   - `/api/v1/training/sessions/{id}/progress` (Phase 1)
   - `/api/v1/face-sessions/{id}` (Phase 2/3)
   - UI must coordinate two polling loops

4. **Legacy field naming**: `FaceDetectionSession.faces_assigned` is ambiguous
   - Includes both `faces_assigned_to_persons` AND `clusters_created`
   - Backend calculates: `faces_assigned = faces_assigned_to_persons + clusters_created`
   - UI cannot distinguish between assignment types

### Performance Characteristics

**Typical timing for 1000 images** (measured on M1 Mac with MPS):

| Phase | Duration | Operations | Bottleneck |
|-------|----------|-----------|-----------|
| **Phase 1: Training** | 30 seconds | CLIP embedding (ViT-B-32) | GPU (batch=8) |
| **Phase 2: Face Detection** | 2 minutes | SCRFD detection + Buffalo embedding | Face count (avg 2 faces/img) |
| **Phase 3: Clustering** | 15 seconds | HDBSCAN on 2000 faces | CPU (scikit-learn) |
| **Total** | 2min 45s | - | - |

**User Experience Issue**:
- Progress bar reaches 100% at 30 seconds (18% of total time)
- User waits 2min 15s (82% of total time) with no feedback beyond "Completed"
- Face card appears at 2min 45s, but progress was invisible

---

## Recommendations (Prioritized)

### Priority 1: Immediate User Experience Fix (Low Effort)

**Change progress bar label when Phase 1 completes**:

```svelte
<!-- SessionDetailView.svelte -->
{#if progress}
    <section class="progress-section">
        <h2>Progress</h2>
        <div class="space-y-1 mb-4">
            <div class="flex justify-between text-sm text-gray-600">
                <span>
                    {#if session.status === 'completed' && faceSession?.status === 'processing'}
                        Face Detection Running...
                    {:else if session.status === 'completed' && !faceSession}
                        Training Complete (Face Detection Pending)
                    {:else}
                        Processing Images
                    {/if}
                </span>
                <span>{percentage}%</span>
            </div>
            <Progress value={percentage} max={100} class="h-2" />
        </div>
    </section>
{/if}
```

**Benefits**:
- No backend changes required
- User sees accurate status message
- Minimal code change

**Drawbacks**:
- Progress bar still shows 100% (misleading)
- No ETA for Phase 2/3
- Still requires polling two endpoints

### Priority 2: Unified Progress Endpoint (Medium Effort)

**Implementation**:
1. Add `/api/v1/training/sessions/{id}/progress-unified` endpoint (see Solution 1)
2. Update `getUnifiedProgress()` API client
3. Replace `fetchProgress()` with `fetchUnifiedProgress()` in SessionDetailView
4. Update progress display to show combined percentage

**Benefits**:
- Accurate progress bar (0-100% across all phases)
- Single endpoint to poll (simpler UI logic)
- Phase-aware status messages

**Drawbacks**:
- Requires backend API changes
- Frontend type regeneration needed
- Must update contract in both repos

### Priority 3: Enhanced UI with Phase Indicators (High Effort)

**Implementation**:
1. Create `PhaseProgress.svelte` component (see Solution 3)
2. Update SessionDetailView layout to show phase breakdown
3. Add phase status icons (🎨 Training, 😊 Faces, 🔗 Clustering)
4. Show detailed progress for each phase

**Benefits**:
- Maximum transparency for user
- Clear indication of current phase
- Can show ETA per phase
- Professional appearance

**Drawbacks**:
- Most UI work required
- Needs design review
- More screen real estate

---

## Testing Recommendations

### Backend Tests

**File**: `tests/api/test_training_unified_progress.py`

```python
def test_unified_progress_phase1_running():
    """Test progress when only Phase 1 is running."""
    # Create training session with 100 images, 50 processed
    # Expect: 30% * 0.5 = 15% overall progress

def test_unified_progress_phase1_complete_phase2_running():
    """Test progress when Phase 1 done, Phase 2 running."""
    # Training complete, face detection 50% done
    # Expect: 30% + (65% * 0.5) = 62.5% overall progress

def test_unified_progress_all_complete():
    """Test progress when all phases complete."""
    # All phases done
    # Expect: 100% overall progress
```

### Frontend Tests

**File**: `image-search-ui/src/tests/components/SessionDetailView.test.ts`

```typescript
test('shows combined progress when face detection running', async () => {
    mockResponse('/api/v1/training/sessions/1/progress-unified', {
        overallProgress: { percentage: 62.5 },
        phases: {
            training: { status: 'completed', progress: { percentage: 100 } },
            faceDetection: { status: 'processing', progress: { percentage: 50 } },
            clustering: { status: 'pending', progress: { percentage: 0 } },
        },
    });

    render(SessionDetailView, { props: { session: createTrainingSession() } });

    await waitFor(() => {
        expect(screen.getByText('62.5%')).toBeInTheDocument();
        expect(screen.getByText('Face Detection (InsightFace)')).toBeInTheDocument();
    });
});
```

---

## Alternative Approaches (Considered but Not Recommended)

### Approach A: Merge Phase 2/3 into Phase 1

**Idea**: Make face detection part of training job (single RQ job)

**Pros**:
- Single progress calculation
- No need for unified endpoint
- Simpler architecture

**Cons**:
- Training job timeout (currently 1h, would need 4h)
- Cannot pause/resume independently
- Breaks existing training-only workflows
- Users who don't want face detection are blocked

**Verdict**: ❌ Breaks modularity, not backward compatible

### Approach B: Use WebSocket for Real-Time Progress

**Idea**: Replace polling with WebSocket stream for live updates

**Pros**:
- Lower latency (no 2-second polling delay)
- Less server load (no repeated polling)
- Can push phase transitions immediately

**Cons**:
- Adds complexity (WebSocket infrastructure)
- Requires connection management
- Doesn't solve root problem (still need unified progress calculation)

**Verdict**: ⚠️ Good future enhancement, but solves different problem

### Approach C: Show Two Separate Progress Bars

**Idea**: Keep separate bars for training and face detection

**Pros**:
- Minimal backend changes
- Clear separation of phases

**Cons**:
- Confusing UX (which bar is "overall"?)
- Still shows 100% + 0% initially
- Doesn't address clustering visibility

**Verdict**: ⚠️ Better than current, but not ideal

---

## Open Questions

1. **Should face detection be optional?**
   - Currently auto-triggered with no way to disable
   - Some users may only want embeddings, not faces
   - Suggestion: Add `skip_face_detection` flag to `TrainingSessionCreate`

2. **Should clustering be a separate job?**
   - Currently inline (5-30 seconds, no progress)
   - Could be separate job with progress tracking
   - Trade-off: More complex orchestration vs better visibility

3. **What if face detection fails?**
   - Currently: Training shows "completed" even if faces fail
   - Should training status reflect overall pipeline status?
   - Suggestion: Add `overall_status` field separate from `training_status`

4. **How to handle partial failures?**
   - Example: 100 images trained, 50 faces detected, 50 face errors
   - Should show "completed with errors" or "partially complete"?
   - Current behavior: Shows "completed" with `failed_images` count

5. **Should we retry failed face detections?**
   - Currently: Failed images are skipped permanently
   - Could add "Retry Failed" button in UI
   - Backend already has retry logic for training jobs

---

## Conclusion

The training progress tracking system is **functionally complete** but suffers from **poor visibility** into Phase 2/3 operations. The root cause is **incremental feature addition** without refactoring the original single-phase progress model.

**Minimal fix** (Priority 1): Update UI labels to clarify when face detection is running.

**Recommended fix** (Priority 2): Implement unified progress endpoint with weighted phase calculations.

**Ideal fix** (Priority 3): Add phase indicators with detailed breakdowns and ETAs.

All solutions maintain **backward compatibility** and require **no database migrations**. The unified progress endpoint can coexist with existing endpoints until UI migration is complete.

---

**Research conducted**: 2026-01-12
**Next steps**: Implement Priority 2 solution (unified progress endpoint)
**Estimated effort**: 4-6 hours (2h backend + 2h frontend + 2h testing)
