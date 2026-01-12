# Training Progress Tracking - Quick Reference

**Problem**: UI shows 100% when training completes, but face detection and clustering still run invisibly.

## Three-Phase Workflow

```
Phase 1: Training (CLIP Embeddings)     [30s]  ←─ UI tracks this ✅
    ↓ Auto-triggers
Phase 2: Face Detection (InsightFace)   [2m]   ←─ UI ignores this ❌
    ↓ Inline at end
Phase 3: Face Clustering (HDBSCAN)      [15s]  ←─ UI ignores this ❌

Total: ~2m 45s, but UI shows 100% at 30 seconds
```

## Current Architecture

**Backend**:
- `train_session()` → auto-triggers → `detect_faces_for_session_job()` → inline clustering
- Two separate progress endpoints:
  - `/api/v1/training/sessions/{id}/progress` (Phase 1 only)
  - `/api/v1/face-sessions/{id}` (Phase 2/3 combined)

**Frontend**:
- `SessionDetailView.svelte` polls Phase 1 endpoint
- Stops polling when `status === 'completed'` (misses Phase 2/3)
- Shows face card separately AFTER training completes

## Quick Fixes

### Fix 1: Update UI Labels (No Backend Changes)

```svelte
{#if session.status === 'completed' && faceSession?.status === 'processing'}
    Face Detection Running...
{:else if session.status === 'completed'}
    Training Complete (Face Detection Pending)
{:else}
    Processing Images
{/if}
```

### Fix 2: Unified Progress Endpoint (Recommended)

**Backend**: New endpoint `/api/v1/training/sessions/{id}/progress-unified`

Returns:
```json
{
  "overallProgress": { "percentage": 62.5 },
  "phases": {
    "training": { "status": "completed", "progress": { "percentage": 100 } },
    "faceDetection": { "status": "processing", "progress": { "percentage": 50 } },
    "clustering": { "status": "pending", "progress": { "percentage": 0 } }
  }
}
```

**Calculation**:
```python
overall = (0.30 * phase1_pct) + (0.65 * phase2_pct) + (0.05 * phase3_pct)
```

**Frontend**: Replace `fetchProgress()` with `fetchUnifiedProgress()`, update polling logic.

## Key Files

**Backend**:
- `queue/training_jobs.py::train_session()` (Phase 1, auto-triggers Phase 2)
- `queue/face_jobs.py::detect_faces_for_session_job()` (Phase 2 + inline Phase 3)
- `services/training_service.py::get_session_progress()` (current progress calculation)

**Frontend**:
- `components/training/SessionDetailView.svelte` (main progress display)
- `components/training/SessionDetailView.svelte::fetchProgress()` (polling logic)

## Recommendations

1. **Immediate** (1h): Implement Fix 1 (label changes)
2. **Short-term** (4-6h): Implement Fix 2 (unified endpoint)
3. **Long-term**: Add phase indicators with icons (🎨 Training, 😊 Faces, 🔗 Clustering)

---

See `docs/research/training-progress-tracking-comprehensive-analysis-2026-01-12.md` for full analysis.
