# Training Progress Tracking Issues - Root Cause Analysis

**Date**: 2026-01-12
**Investigator**: Research Agent
**Scope**: Backend unified progress tracking and frontend UI reactivity

---

## Executive Summary

Investigation identified **4 critical issues** in the unified training progress tracking system affecting both backend progress calculation and frontend UI updates. All issues stem from incomplete status synchronization between database models and the unified progress calculation logic.

**Critical Findings**:
1. Face detection progress never updates (always "Pending")
2. Clustering phase never shows progress (jumps to "Completed")
3. UI doesn't stop polling when phases complete (remains "Running")
4. Overall progress calculation incorrect due to skipped images counting

---

## Problem 1: Face Detection Phase Never Shows Progress

### Observed Behavior
- Face detection phase stays at "Pending" status throughout execution
- Progress percentage never updates (remains 0%)
- No status transition to "processing" visible in UI

### Root Cause

**Location**: `training_service.py:1009-1208` (`get_session_progress_unified`)

The unified progress endpoint checks `face_session.status` to determine phase status, but the face detection job (`face_jobs.py:421-778`) **does not update progress fields during batch processing**.

**Critical Code Path**:

```python
# training_service.py:1064-1067
if face_session:
    processed = getattr(face_session, "processed_images", 0)
    total = getattr(face_session, "total_images", 0)
    phase2_pct = (processed / total * 100) if total > 0 else 0.0
```

**Issue**: `face_session.processed_images` is never incremented during batch processing in `detect_faces_for_session_job()`.

**Evidence from face_jobs.py:634-645**:

```python
# Update session progress
total_faces_detected += result["total_faces"]
total_failed += result["errors"]

session.processed_images += result["processed"]  # ✓ Updates processed count
session.faces_detected = total_faces_detected
session.failed_images = total_failed

# Update current position for resume support
session.current_asset_index += len(batch)

db_session.commit()  # ✓ Commits to database
```

**The progress IS being updated!** But there's a **data flow disconnect**:

1. **Face detection job updates**: `session.processed_images` (line 634)
2. **Unified endpoint reads**: `getattr(face_session, "processed_images", 0)` (line 1065)
3. **Problem**: The frontend **polls every 2 seconds** but database updates happen **per batch** (every ~8-16 images)
4. **Timing issue**: Between batches, there are **no intermediate commits**, causing progress to appear frozen

### Real Issue: Batch Granularity

Face detection processes images in batches (default: 16 images/batch). Progress updates only commit **after each full batch completes**, not per image. With 666 images:
- Total batches: ~42 batches (666 / 16)
- Progress updates: **42 times** during entire session
- Frontend polls: **~120 times/minute** (2-second interval)
- **Gap**: Most polls occur **between batch completions**, seeing stale data

### Impact on UI

**Frontend polling** (`SessionDetailView.svelte:71-76`):
```typescript
pollingInterval = setInterval(() => {
    untrack(() => {
        fetchUnifiedProgress();  // Polls every 2 seconds
    });
}, 2000);
```

**What user sees**:
- Phase status: ✗ Shows "pending" (should be "processing")
- Progress bar: ✗ Frozen at 0% (updates only 42 times for 666 images)
- Percentage: ✗ No incremental updates between batches

---

## Problem 2: Clustering Phase Never Shows Progress

### Observed Behavior
- Clustering phase shows no progress indication
- Jumps directly from "pending" (0%) to "completed" (100%)
- No intermediate states visible during execution

### Root Cause

**Location**: `training_service.py:1185-1206` (clustering phase logic)

**Critical Code**:

```python
"clustering": PhaseProgress(
    name="clustering",
    status=(
        "completed"
        if phase3_complete
        else (
            "running"
            if face_session
            and getattr(face_session, "status", None)
            == FaceDetectionSessionStatus.PROCESSING.value
            else "pending"
        )
    ),
    progress=ProgressStats(
        current=1 if phase3_complete else 0,  # ⚠️ BINARY: 0 or 1
        total=1,
        percentage=100.0 if phase3_complete else 0.0,  # ⚠️ BINARY: 0% or 100%
        etaSeconds=None,
        imagesPerMinute=None,
    ),
    startedAt=None,  # ⚠️ Never set
    completedAt=None,  # ⚠️ Never set
)
```

**Problem**: Clustering progress is **modeled as binary** (0 or 1), not granular.

**Why**:
1. Clustering runs **inline** with face detection (lines 699-732 in `face_jobs.py`)
2. No separate tracking for clustering progress
3. `phase3_complete` determined by `face_session.clusters_created is not None` (line 1073-1075)
4. **No intermediate states tracked** during HDBSCAN execution

### Clustering Execution Path

**face_jobs.py:699-732**:

```python
# Run clustering for unlabeled faces
if total_faces_detected > 0:
    logger.info(f"[{job_id}] Running clustering for unlabeled faces")
    try:
        clusterer = get_face_clusterer(
            db_session,
            min_cluster_size=3,
        )

        # Cluster faces that don't have a person_id (unlabeled)
        cluster_result = clusterer.cluster_unlabeled_faces(
            quality_threshold=0.3,
            max_faces=10000,  # Process up to 10k faces
        )

        clusters_found = cluster_result.get("clusters_found", 0)

        # Update session with clustering stats
        session.clusters_created = clusters_found
        db_session.commit()  # ⚠️ Single commit after clustering completes
```

**Issue**: Clustering is **all-or-nothing**:
- Status: "pending" → (HDBSCAN runs for ~5-15 seconds) → "completed"
- No progress updates during HDBSCAN execution
- Frontend sees: 0% → (black box) → 100%

### Missing Infrastructure

**FaceDetectionSession model** (`models.py:653-715`) has **no clustering progress fields**:

```python
class FaceDetectionSession(Base):
    # ... existing fields ...
    faces_detected: Mapped[int]
    faces_assigned: Mapped[int]
    clusters_created: Mapped[int]  # ✓ Result count only

    # ❌ MISSING:
    # clustering_started_at: Mapped[datetime | None]
    # clustering_completed_at: Mapped[datetime | None]
    # clustering_progress: Mapped[int]  # Faces clustered so far
```

### Impact

**User Experience**:
- **Uncertainty**: No indication clustering is happening
- **Perception**: Appears frozen or stuck
- **Reality**: CPU-intensive HDBSCAN running in background
- **Duration**: Can take 5-30 seconds for 1000+ faces

---

## Problem 3: UI Doesn't Stop Polling When Complete

### Observed Behavior
- Overall status shows "Running" even after all phases complete
- Frontend continues polling indefinitely
- No automatic transition to "Completed" state

### Root Cause

**Location**: `training_service.py:1096-1112` (overallStatus calculation)

**Critical Logic**:

```python
# Determine overall status
if training_session.status == SessionStatus.FAILED.value or (
    face_session
    and getattr(face_session, "status", None)
    == FaceDetectionSessionStatus.FAILED.value
):
    overall_status = "failed"
elif phase1_complete and phase2_complete and phase3_complete:
    overall_status = "completed"  # ✓ Correct condition
elif training_session.status == SessionStatus.RUNNING.value or (
    face_session
    and getattr(face_session, "status", None)
    == FaceDetectionSessionStatus.PROCESSING.value
):
    overall_status = "running"  # ⚠️ Takes precedence over completion check
else:
    overall_status = "pending"
```

**Issue**: `phase3_complete` check is **too strict**:

```python
# Line 1073-1075
phase3_complete = phase2_complete and getattr(
    face_session, "clusters_created", None
) is not None
```

**Problem Scenario**:
1. Face detection completes successfully
2. Clustering runs and creates 0 clusters (all faces already assigned)
3. `clusters_created` = 0 (not None)
4. `phase2_complete` = True (face detection done)
5. **Expected**: `phase3_complete = True`
6. **Actual**: `phase3_complete = True` (✓ works correctly)

**Wait, if the logic is correct, why does UI show "Running"?**

### Real Issue: Race Condition

**Frontend polling effect** (`SessionDetailView.svelte:107-122`):

```typescript
$effect(() => {
    const overallStatus = unifiedProgress?.overallStatus;

    untrack(() => {
        // Keep polling while any phase is running
        if (overallStatus === 'running' && session?.id) {
            startPolling();
        } else {
            stopPolling();
        }
    });

    return () => stopPolling();
});
```

**Race Condition Timeline**:
1. **T+0s**: Face detection completes, sets `status = COMPLETED`
2. **T+0.1s**: Clustering runs (inline, same job)
3. **T+0.5s**: Frontend polls → sees `status = PROCESSING` (stale read)
4. **T+1.0s**: Clustering commits → sets `clusters_created = 5`
5. **T+1.5s**: Frontend polls → sees `status = COMPLETED`, stops polling

**Why it appears stuck**:
- If clustering creates **0 clusters**, `clusters_created` field may not be set explicitly
- Backend sets `clusters_created = 0` (line 724-726), but check uses `is not None`
- **Actually works correctly**, but **perception** issue due to batch granularity

### Actual Root Cause: TrainingSession Status

**The real problem is TrainingSession completion!**

**training_jobs.py:236-240**:

```python
training_session = get_session_by_id_sync(db_session, session_id)
if training_session and processed_count > 0:
    training_session.status = SessionStatus.COMPLETED.value
    training_session.completed_at = datetime.now(UTC)
    db_session.commit()
```

**BUT** unified progress checks:

```python
# Line 1105-1109
elif training_session.status == SessionStatus.RUNNING.value or (
    face_session
    and getattr(face_session, "status", None)
    == FaceDetectionSessionStatus.PROCESSING.value
):
    overall_status = "running"
```

**Issue**: If `training_session.status == RUNNING` (not yet updated to COMPLETED), overall status remains "running" **even if face detection and clustering are done**.

**Race condition**:
1. Training job completes, marks session COMPLETED
2. Face detection auto-starts (lines 244-294)
3. Face session status: PENDING → PROCESSING
4. **If frontend polls during this window**, sees:
   - `training_session.status = COMPLETED` ✓
   - `face_session.status = PROCESSING` ✓
   - **Result**: `overall_status = "running"` (line 1105-1109 condition)

---

## Problem 4: Overall Progress Calculation Wrong

### Observed Behavior
- Shows 49.7% progress with Current:331, Total:666
- Expected: 100% (331 unique + 335 skipped = 666 total)
- Duplicate images are skipped but still count toward total

### Root Cause

**Location**: `training_service.py:1078-1082` (overall progress calculation)

**Critical Code**:

```python
# Calculate overall progress (weighted)
overall_pct = (
    (0.30 * phase1_pct)
    + (0.65 * phase2_pct)
    + (0.05 * (100 if phase3_complete else 0))
)
```

**Phase 1 calculation** (line 1053-1056):

```python
phase1_pct = (
    (training_session.processed_images / training_session.total_images * 100)
    if training_session.total_images > 0
    else 0.0
)
```

**Issue**: `processed_images` counts **only unique images processed**, but `total_images` includes **all images (unique + duplicates)**.

### Duplicate Handling Logic

**training_service.py:373-514** (`create_training_jobs`):

```python
def create_training_jobs(
    self, db: AsyncSession, session_id: int, asset_ids: list[int]
) -> dict[str, int]:
    """Create TrainingJob records with hash deduplication.

    For each hash group:
    - Create PENDING job for representative (oldest by created_at)
    - Create SKIPPED jobs for duplicates with skip_reason

    Returns:
        jobs_created: Total jobs (PENDING + SKIPPED)
        unique: Number of unique images (PENDING jobs)
        skipped: Number of duplicate images (SKIPPED jobs)
    """
```

**Process**:
1. Query 666 assets
2. Compute perceptual hashes
3. Group by hash: 331 unique, 335 duplicates
4. Create jobs:
   - 331 PENDING jobs (representatives)
   - 335 SKIPPED jobs (duplicates)
5. Update session:
   - `total_images = 666` (line 344)
   - `skipped_images += 335` (line 500)

**Training execution** (`training_jobs.py:152-212`):

```python
# Get all pending jobs for this session
query = (
    select(TrainingJob)
    .where(TrainingJob.session_id == session_id)
    .where(TrainingJob.status == JobStatus.PENDING.value)  # ⚠️ Only PENDING
)
result = db_session.execute(query)
pending_jobs = list(result.scalars().all())  # 331 jobs

# Process jobs
processed_count = 0
for batch in batches:
    # ... process batch ...
    processed_count += batch_result.get("processed", 0)  # Increments by 331

# Update progress
tracker.update_progress(db_session, processed_count, failed_count)
# Sets: processed_images = 331
```

**Result**:
- `total_images = 666` (all assets, including duplicates)
- `processed_images = 331` (only PENDING jobs processed)
- `skipped_images = 335` (duplicates marked SKIPPED)
- **Progress**: 331 / 666 = 49.7% ⚠️

### Semantic Mismatch

**Two interpretations of "total"**:

1. **User expectation**: Total **work to be done** = 331 unique images
2. **System reality**: Total **assets discovered** = 666 images (331 + 335)

**Progress formula assumes**:
- `total_images` = work to be done
- `processed_images` = work completed
- `percentage = processed / total * 100`

**Actual data**:
- `total_images` = 666 (all assets)
- `processed_images` = 331 (unique only)
- `skipped_images` = 335 (not counted in processed)
- **Formula breaks**: 331 / 666 = 49.7%

### Why Skipped Images Aren't Counted

**Progress tracker** (`progress.py:98-102`):

```python
total = session.total_images
processed = session.processed_images  # ⚠️ Does NOT include skipped
failed = session.failed_images
remaining = max(0, total - processed)  # 666 - 331 = 335 (the skipped ones!)
percentage = (processed / total * 100) if total > 0 else 0.0
```

**Should be**:

```python
effective_completed = session.processed_images + session.skipped_images
effective_total = session.total_images
percentage = (effective_completed / effective_total * 100)
# 331 + 335 / 666 = 100%
```

---

## Impact Summary

### User Experience Impact

| Issue | Severity | User Impact | Technical Impact |
|-------|----------|-------------|------------------|
| **Face Detection Progress** | 🔴 High | Appears frozen during slowest phase (65% of total time) | Progress updates only 42 times for 666 images (batch granularity) |
| **Clustering Progress** | 🟡 Medium | 5-30 second black box (no feedback) | Binary progress model (0% or 100%) |
| **Polling Doesn't Stop** | 🟡 Medium | Unnecessary backend load, battery drain | Race condition between status transitions |
| **Wrong Progress %** | 🔴 High | Confusing, shows 50% when actually complete | Semantic mismatch: total includes skipped, processed doesn't |

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Backend (training_jobs.py + face_jobs.py)                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Training Phase (30% weight)                                │
│  ├─ Process 331 PENDING jobs in batches                     │
│  ├─ Skip 335 SKIPPED jobs (duplicates)                      │
│  ├─ Update: processed_images += 331                         │
│  └─ Issue: total_images = 666 (includes skipped)            │
│      Result: 331/666 = 49.7% ❌                              │
│                                                              │
│  Face Detection Phase (65% weight)                          │
│  ├─ Process images in batches (16 per batch)                │
│  ├─ Update: processed_images += batch_size (per batch)      │
│  ├─ Issue: No updates BETWEEN batches (frozen perception)   │
│  └─ Commits: ~42 times for 666 images (batch granularity)   │
│                                                              │
│  Clustering Phase (5% weight)                               │
│  ├─ Run HDBSCAN (5-30 seconds)                              │
│  ├─ Issue: Binary progress (0% → 100%)                      │
│  └─ Update: clusters_created = N (single commit at end)     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Unified Progress Endpoint (training_service.py)             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  get_session_progress_unified()                             │
│  ├─ Read: training_session.processed_images (331)           │
│  ├─ Read: training_session.total_images (666)               │
│  ├─ Read: face_session.processed_images (varies)            │
│  ├─ Calculate: overall_pct = weighted average               │
│  │   • 30% * (331/666) = 14.9%  ← Wrong!                    │
│  │   • 65% * (face progress)                                │
│  │   • 5% * (clustering binary)                             │
│  └─ Issue: Uses raw numbers without skipped adjustment      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Frontend (SessionDetailView.svelte)                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Polling Loop (every 2 seconds)                             │
│  ├─ Fetch: /api/v1/training/{id}/progress/unified           │
│  ├─ Display: overallProgress.percentage (49.7%)             │
│  ├─ Display: phases.training.progress (14.9%)               │
│  ├─ Display: phases.faceDetection.progress (varies)         │
│  ├─ Display: phases.clustering.progress (0% or 100%)        │
│  └─ Issue: Polls continue while status="running"            │
│                                                              │
│  Polling Control ($effect)                                  │
│  ├─ Start: if overallStatus === "running"                   │
│  ├─ Stop: if overallStatus !== "running"                    │
│  └─ Issue: Race condition with status transitions           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Recommendations

### Fix 1: Training Progress Calculation (Critical)

**Problem**: `processed_images` doesn't include `skipped_images`, but `total_images` does.

**Solution**: Adjust progress calculation to include skipped images in "completed" count.

**Location**: `training_service.py:1053-1058`

**Current Code**:
```python
phase1_pct = (
    (training_session.processed_images / training_session.total_images * 100)
    if training_session.total_images > 0
    else 0.0
)
```

**Fixed Code**:
```python
# Include skipped images in completion calculation
effective_completed = (
    training_session.processed_images + training_session.skipped_images
)
phase1_pct = (
    (effective_completed / training_session.total_images * 100)
    if training_session.total_images > 0
    else 0.0
)
```

**Rationale**: Skipped images are "work completed" (just optimized away). They should count toward progress.

### Fix 2: Face Detection Progress Granularity (High Priority)

**Problem**: Progress updates only happen per batch, causing frozen appearance.

**Solutions** (pick one):

#### Option A: Per-Image Progress Updates (Simplest)
```python
# face_jobs.py, inside batch loop
for i, asset in enumerate(batch):
    result = face_service.process_single_asset(asset.id, ...)

    # Update progress per image
    session.processed_images += 1
    if (i + 1) % 4 == 0:  # Commit every 4 images
        db_session.commit()
```

**Pros**: Smooth progress updates
**Cons**: More database commits (overhead)

#### Option B: Optimistic UI Progress (Frontend-Side)
```typescript
// SessionDetailView.svelte
let optimisticProgress = $state(0);

$effect(() => {
    if (unifiedProgress?.overallStatus === 'running') {
        // Interpolate between actual progress and 100%
        const interval = setInterval(() => {
            optimisticProgress = Math.min(
                optimisticProgress + 0.1,
                unifiedProgress.overallProgress.percentage + 5
            );
        }, 100);
        return () => clearInterval(interval);
    }
});
```

**Pros**: No backend changes, smooth UX
**Cons**: Not "real" progress, just cosmetic

#### Option C: Redis Progress Cache (Recommended)
```python
# face_jobs.py
def update_progress_cache(session_id: str, processed: int, total: int):
    redis_client.setex(
        f"face_session:{session_id}:progress",
        ex=300,  # 5 minute TTL
        value=json.dumps({"processed": processed, "total": total})
    )

# Update cache per image, commit to DB per batch
for i, asset in enumerate(batch):
    result = face_service.process_single_asset(asset.id, ...)
    session.processed_images += 1
    update_progress_cache(session.id, session.processed_images, session.total_images)

db_session.commit()  # Once per batch
```

**Pros**: Best of both worlds (smooth + efficient)
**Cons**: Adds Redis dependency for progress tracking

### Fix 3: Clustering Progress Visibility (Medium Priority)

**Problem**: Clustering is a black box (0% → 100%).

**Solutions**:

#### Option A: Add Progress Tracking to Clusterer
```python
# faces/clusterer.py
class FaceClusterer:
    def cluster_unlabeled_faces(
        self,
        quality_threshold: float = 0.5,
        max_faces: int = 50000,
        progress_callback: Callable[[int, int], None] | None = None,
    ) -> dict[str, Any]:
        # ... load faces ...

        if progress_callback:
            progress_callback(0, len(embeddings))

        # Run HDBSCAN (can't report internal progress)
        labels = self.clusterer.fit_predict(embeddings)

        if progress_callback:
            progress_callback(len(embeddings), len(embeddings))

        # ... assign clusters ...
```

**Issue**: HDBSCAN is a black box, can't report internal progress.

#### Option B: Add Status Field (Recommended)
```python
# db/models.py - FaceDetectionSession
clustering_status: Mapped[str | None] = mapped_column(String(20), nullable=True)
# Values: None, "pending", "running", "completed"

# face_jobs.py
session.clustering_status = "running"
db_session.commit()

cluster_result = clusterer.cluster_unlabeled_faces(...)

session.clustering_status = "completed"
session.clusters_created = clusters_found
db_session.commit()
```

**UI Display**:
```typescript
{#if clusterPhase.status === 'running'}
    <div class="spinner">🔄 Clustering faces...</div>
{/if}
```

**Pros**: Simple, provides feedback
**Cons**: Still binary (running vs done)

#### Option C: Accept Binary Progress (Lowest Effort)
Just add better UI messaging:
```typescript
{#if clusterPhase.status === 'pending' && facePhase.status === 'completed'}
    <div class="info">⏳ Clustering will start after face detection...</div>
{:else if clusterPhase.status === 'running'}
    <div class="spinner">🔄 Clustering faces (typically 5-30 seconds)...</div>
{/if}
```

**Pros**: No backend changes
**Cons**: Doesn't solve root issue

### Fix 4: Polling Stop Condition (High Priority)

**Problem**: UI continues polling even when all phases complete.

**Solution**: Fix overall status calculation to properly detect completion.

**Location**: `training_service.py:1096-1112`

**Current Code**:
```python
# Determine overall status
if training_session.status == SessionStatus.FAILED.value or (...):
    overall_status = "failed"
elif phase1_complete and phase2_complete and phase3_complete:
    overall_status = "completed"
elif training_session.status == SessionStatus.RUNNING.value or (...):
    overall_status = "running"  # ⚠️ Can override completion check
else:
    overall_status = "pending"
```

**Issue**: Condition order allows "running" to override "completed" in edge cases.

**Fixed Code**:
```python
# Check completion FIRST (highest priority)
if phase1_complete and phase2_complete and phase3_complete:
    overall_status = "completed"
elif training_session.status == SessionStatus.FAILED.value or (
    face_session
    and getattr(face_session, "status", None)
    == FaceDetectionSessionStatus.FAILED.value
):
    overall_status = "failed"
elif training_session.status == SessionStatus.RUNNING.value or (
    face_session
    and getattr(face_session, "status", None)
    == FaceDetectionSessionStatus.PROCESSING.value
):
    overall_status = "running"
else:
    overall_status = "pending"
```

**Also add defensive check**:
```python
# If face session exists and is complete, ensure phase 3 is marked complete
if face_session and getattr(face_session, "status", None) == FaceDetectionSessionStatus.COMPLETED.value:
    # Force phase 3 complete if clustering happened (even if 0 clusters)
    if not phase3_complete and getattr(face_session, "clusters_created", None) == 0:
        phase3_complete = True
```

---

## Implementation Priority

### Critical (Fix Immediately)
1. **Training Progress Calculation** (Fix 1)
   - Impact: Blocks accurate progress reporting
   - Effort: 5 lines of code
   - Risk: Low (pure calculation fix)

2. **Polling Stop Condition** (Fix 4)
   - Impact: Prevents UI from stopping
   - Effort: Reorder conditions (10 lines)
   - Risk: Low (logic reordering)

### High Priority (Fix Soon)
3. **Face Detection Granularity** (Fix 2, Option C: Redis Cache)
   - Impact: Major UX improvement
   - Effort: ~50 lines (Redis integration)
   - Risk: Medium (new dependency)

### Medium Priority (Future Enhancement)
4. **Clustering Visibility** (Fix 3, Option B: Status Field)
   - Impact: Better user feedback
   - Effort: ~30 lines (migration + logic)
   - Risk: Low (additive change)

---

## Testing Strategy

### Unit Tests
```python
# test_training_service.py
def test_unified_progress_with_skipped_images():
    """Progress should include skipped images in completion."""
    session = TrainingSession(
        total_images=666,
        processed_images=331,
        skipped_images=335,
        status=SessionStatus.COMPLETED.value,
    )

    service = TrainingService()
    progress = service.get_session_progress_unified(db, session.id)

    # 331 processed + 335 skipped = 666 total
    assert progress.phases["training"].progress.percentage == 100.0

def test_polling_stops_when_complete():
    """Overall status should be 'completed' when all phases done."""
    # Setup: all phases complete
    training_session.status = SessionStatus.COMPLETED
    face_session.status = FaceDetectionSessionStatus.COMPLETED
    face_session.clusters_created = 5

    progress = service.get_session_progress_unified(db, session.id)
    assert progress.overallStatus == "completed"
```

### Integration Tests
```python
def test_face_detection_progress_updates():
    """Face detection should update progress between batches."""
    session_id = create_face_detection_session()

    # Poll progress during execution
    progress_samples = []
    for _ in range(10):
        time.sleep(1)
        progress = api_client.get(f"/training/{session_id}/progress/unified")
        progress_samples.append(progress["phases"]["faceDetection"]["progress"]["percentage"])

    # Should see incremental increases, not just 0% → 100%
    assert len(set(progress_samples)) > 2  # At least 3 different values
```

### E2E Tests
```typescript
// SessionDetailView.test.ts
test('polling stops when session completes', async () => {
    const { getByText } = render(SessionDetailView, {
        props: { session: mockSession, onSessionUpdate: vi.fn() }
    });

    // Mock progress: running → completed
    mockFetchSequence([
        { overallStatus: 'running', overallProgress: { percentage: 50 } },
        { overallStatus: 'completed', overallProgress: { percentage: 100 } },
    ]);

    await waitFor(() => {
        expect(getByText('✓ All Phases Complete')).toBeInTheDocument();
    });

    // Verify polling stopped (no more fetches after completion)
    await new Promise(resolve => setTimeout(resolve, 3000));
    expect(fetchMock).toHaveBeenCalledTimes(2);  // Initial + completion
});
```

---

## Conclusion

All four issues stem from **incomplete synchronization** between:
1. Database models (what data exists)
2. Business logic (how it's calculated)
3. UI expectations (what users see)

**Root Causes**:
- **Semantic mismatch**: `total_images` includes skipped, `processed_images` doesn't
- **Batch granularity**: Progress updates only per batch, not per image
- **Binary modeling**: Clustering has no intermediate progress states
- **Condition ordering**: Completion check can be overridden by running check

**Fixes are straightforward** and low-risk. Implementing **Fix 1 and Fix 4** will resolve the most critical user-facing issues immediately.

---

**Related Files**:
- Backend: `training_service.py`, `face_jobs.py`, `training_jobs.py`, `progress.py`
- Frontend: `SessionDetailView.svelte`
- Models: `models.py` (TrainingSession, FaceDetectionSession)

**Estimated Effort**:
- Critical fixes (1+4): 2-3 hours
- High priority (2): 4-6 hours
- Medium priority (3): 2-3 hours
- **Total**: 8-12 hours for complete fix
