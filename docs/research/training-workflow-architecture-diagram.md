# Training System Architecture - Visual Diagram

## Current Multi-Phase Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         USER INTERACTION LAYER                          │
│  ────────────────────────────────────────────────────────────────────  │
│  CreateSessionModal.svelte → API POST /api/v1/training/sessions        │
│  User selects: root_path, subdirectories, category                     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      TRAINING SESSION RECORD                            │
│  ────────────────────────────────────────────────────────────────────  │
│  Database: training_sessions table                                     │
│  Status: pending → running → completed                                  │
│  Fields: id, total_images, processed_images, started_at, completed_at  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: TRAINING JOBS (CLIP Embeddings)              ✅ UI TRACKED   │
│═════════════════════════════════════════════════════════════════════════│
│                                                                         │
│  Entry Point: training_jobs.py::train_session(session_id)              │
│  Queue: "high" priority (RQ)                                           │
│  Worker: image_search_service.queue.worker                             │
│  Duration: ~30 seconds per 1000 images                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Discover Assets                                              │   │
│  │    - Query TrainingSubdirectory records                         │   │
│  │    - Filter by selected=True                                    │   │
│  │    - Find all image files matching extensions                   │   │
│  │    Result: List of asset IDs                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. Create TrainingJob Records                                   │   │
│  │    - One TrainingJob per asset                                  │   │
│  │    - Initial status: pending                                    │   │
│  │    - Stored in training_jobs table                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. Process in Batches (training_jobs.py::train_batch)          │   │
│  │    - Batch size: 32 images (configurable)                       │   │
│  │    - GPU batch size: 8 (for embedding)                          │   │
│  │    - Pipelined I/O: 4 parallel image loaders                    │   │
│  │                                                                  │   │
│  │    For each batch:                                              │   │
│  │    a. Load images from disk (ThreadPoolExecutor)                │   │
│  │    b. Generate CLIP embeddings (OpenCLIP ViT-B-32)              │   │
│  │       Device: MPS (Mac) or CUDA (GPU) or CPU                    │   │
│  │       Output: 512-dimensional float32 vectors                   │   │
│  │    c. Calculate embedding checksums (SHA256)                    │   │
│  │    d. Upsert to Qdrant collection "images"                      │   │
│  │       Vector + payload {path, created_at, category_id}          │   │
│  │    e. Update TrainingJob status → completed                     │   │
│  │    f. Increment session.processed_images                        │   │
│  │    g. Create TrainingEvidence record (metadata)                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 4. Progress Tracking (Updated Every Batch)                      │   │
│  │    - TrainingSession.processed_images incremented               │   │
│  │    - API: GET /api/v1/training/sessions/{id}/progress          │   │
│  │    - Response: {current, total, percentage, eta_seconds}        │   │
│  │    - UI polls every 2 seconds                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 5. Completion (training_jobs.py line 232-242)                   │   │
│  │    - Set session.status = "completed"                           │   │
│  │    - Set session.completed_at = now()                           │   │
│  │    - Commit to database                                         │   │
│  │    ⚠️  UI SHOWS 100% HERE (but work continues below!)           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
                               │  Auto-Trigger (line 244-300)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: FACE DETECTION SESSION                    ❌ UI NOT TRACKED  │
│═════════════════════════════════════════════════════════════════════════│
│                                                                         │
│  Entry Point: face_jobs.py::detect_faces_for_session_job(session_id)   │
│  Queue: "default" priority (RQ)                                        │
│  Worker: image_search_service.queue.worker                             │
│  Duration: ~2 minutes per 1000 images (depends on face count)          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Create FaceDetectionSession Record                           │   │
│  │    - training_session_id → links back to Phase 1                │   │
│  │    - status: pending → processing → completed                   │   │
│  │    - min_confidence: 0.5, min_face_size: 20px                   │   │
│  │    - batch_size: 16 images                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. Discover Assets to Process                                   │   │
│  │    - If training_session_id provided:                           │   │
│  │      Query TrainingEvidence.asset_id WHERE session_id=X         │   │
│  │    - Else: Query all ImageAsset NOT IN FaceInstance.asset_id    │   │
│  │    - Store asset IDs as JSON in session.asset_ids_json          │   │
│  │    - Resume support: session.current_asset_index tracks progress│   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. Process in Batches (batch_size=16)                           │   │
│  │                                                                  │   │
│  │    For each batch:                                              │   │
│  │    a. Load images from disk (ThreadPoolExecutor)                │   │
│  │    b. Detect faces (InsightFace SCRFD_10G_KPS)                  │   │
│  │       Model: RetinaFace architecture, 10G params                │   │
│  │       Output: [{bbox, confidence, landmarks}]                   │   │
│  │    c. Filter by min_confidence and min_face_size                │   │
│  │    d. Embed face crops (InsightFace Buffalo_l)                  │   │
│  │       Model: ArcFace ResNet-100, 512-dim output                 │   │
│  │       Normalization: L2 norm = 1.0                              │   │
│  │    e. Create FaceInstance records (database)                    │   │
│  │       Fields: asset_id, bbox, confidence, quality_score         │   │
│  │    f. Upsert to Qdrant collection "faces"                       │   │
│  │       Vector + payload {face_id, asset_id, person_id}           │   │
│  │    g. Increment session.processed_images                        │   │
│  │    h. Check pause/cancel status (resume support)                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 4. Auto-Assignment to Known Persons (line 664-696)             │   │
│  │    - Uses FaceAssigner service                                  │   │
│  │    - Query all faces created since session.started_at           │   │
│  │    - Compare embeddings to PersonPrototype centroids            │   │
│  │    - Threshold: face_suggestion_threshold (config)              │   │
│  │    - If similarity > threshold: Auto-assign face.person_id      │   │
│  │    - Else: Create FaceSuggestion for user review                │   │
│  │    - Update session.faces_assigned_to_persons                   │   │
│  │    - Update session.suggestions_created                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 5. Progress Tracking (Available but UI Doesn't Poll)            │   │
│  │    - FaceDetectionSession.processed_images incremented           │   │
│  │    - API: GET /api/v1/face-sessions/{id}                        │   │
│  │    - Response: {processed_images, faces_detected, ...}          │   │
│  │    - ⚠️  UI only checks AFTER Phase 1 completes                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
                               │  Inline (no separate job)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: FACE CLUSTERING (HDBSCAN)                 ❌ NO TRACKING     │
│═════════════════════════════════════════════════════════════════════════│
│                                                                         │
│  Execution: Within detect_faces_for_session_job (line 699-732)         │
│  Queue: N/A (synchronous, not a separate job)                          │
│  Duration: ~5-30 seconds (depends on face count)                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Query Unlabeled Faces                                        │   │
│  │    - SELECT * FROM face_instances WHERE person_id IS NULL       │   │
│  │    - Filter by quality_threshold (default: 0.3)                 │   │
│  │    - Limit: 10,000 faces                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. Run HDBSCAN Clustering                                       │   │
│  │    - Algorithm: Hierarchical Density-Based Clustering           │   │
│  │    - Parameters:                                                │   │
│  │      * min_cluster_size: 3 (minimum faces per cluster)          │   │
│  │      * metric: cosine distance                                  │   │
│  │      * cluster_selection_method: eom (excess of mass)           │   │
│  │    - Input: 512-dim face embeddings (from Buffalo_l)            │   │
│  │    - Output: cluster labels (-1 for noise)                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. Create FaceCluster Records                                   │   │
│  │    - One FaceCluster per cluster_id (excluding -1)              │   │
│  │    - Update FaceInstance.cluster_id for grouped faces           │   │
│  │    - Select representative face (highest quality_score)         │   │
│  │    - Update session.clusters_created = cluster_count            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 4. Update Session (line 725-732)                                │   │
│  │    - session.clusters_created = clusters_found                  │   │
│  │    - session.faces_assigned = faces_assigned_to_persons +       │   │
│  │                               clusters_created                  │   │
│  │    - session.status = "completed"                               │   │
│  │    - session.completed_at = now()                               │   │
│  │    - ⚠️  No separate progress tracking                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     ALL PHASES COMPLETE                                 │
│  ────────────────────────────────────────────────────────────────────  │
│  UI: FaceDetectionSessionCard appears (but progress was invisible)     │
│  Result: Trained images + detected faces + clustered unknowns          │
│  Time elapsed: ~2m 45s for 1000 images (M1 Mac MPS)                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Progress Data Flow (Current)

```
Frontend: SessionDetailView.svelte
    │
    │ Poll every 2s while status="running"
    │
    ↓
API: GET /api/v1/training/sessions/{id}/progress
    │
    │ Returns Phase 1 progress ONLY
    │
    ↓
Backend: training_service.py::get_session_progress()
    │
    │ Query: training_sessions table
    │
    ↓
Response:
{
  "sessionId": 123,
  "status": "completed",  ← UI stops polling here
  "progress": {
    "current": 1000,
    "total": 1000,
    "percentage": 100.0  ← UI shows 100% done
  }
}

⚠️  At this point:
- Phase 2 (face detection) is running
- Phase 3 (clustering) is queued
- User sees "Completed" but 82% of work remains
```

## Proposed Unified Progress (Solution)

```
Frontend: SessionDetailView.svelte
    │
    │ Poll every 2s while ANY phase is active
    │
    ↓
API: GET /api/v1/training/sessions/{id}/progress-unified
    │
    │ Returns ALL phase progress
    │
    ↓
Backend: NEW unified progress calculation
    │
    │ Query: training_sessions + face_detection_sessions
    │ Calculate: weighted phase progress
    │
    ↓
Response:
{
  "sessionId": 123,
  "overallStatus": "running",  ← Accurate status
  "overallProgress": {
    "percentage": 62.5  ← Accurate progress
  },
  "phases": {
    "training": {
      "status": "completed",
      "progress": { "percentage": 100.0 }
    },
    "faceDetection": {
      "status": "processing",  ← Currently running
      "progress": { "percentage": 50.0 }
    },
    "clustering": {
      "status": "pending",
      "progress": { "percentage": 0.0 }
    }
  }
}

✅ User sees:
- "Processing Training Pipeline: 62.5%"
- "Face Detection (InsightFace): 50%"
- Clear indication work is ongoing
```

## Key Architectural Insights

1. **Phase Coupling**: Training job auto-triggers face detection (line 244-300)
   - Hard-coded coupling
   - No way to disable face detection
   - Non-fatal failure (training completes even if face trigger fails)

2. **Status Ambiguity**: `training_sessions.status = "completed"` means:
   - ✅ CLIP embeddings complete
   - ❌ Face detection NOT started yet
   - Solution: Need `overall_status` vs `training_status`

3. **Inline Clustering**: Phase 3 runs synchronously in Phase 2 job
   - No separate job, no progress tracking
   - Fast enough (5-30s) to not warrant separate job
   - But invisible to user

4. **Separate Data Models**: No unified tracking table
   - `training_sessions` (Phase 1)
   - `face_detection_sessions` (Phase 2/3)
   - Linked by `training_session_id` FK
   - No single "overall progress" field

5. **UI Polling Logic**: Stops when Phase 1 completes
   - Condition: `session.status === 'running'`
   - Should be: `overallStatus === 'running'`
   - Causes invisible Phase 2/3 execution

---

**Created**: 2026-01-12
**Purpose**: Visual reference for training system architecture and progress tracking gaps
