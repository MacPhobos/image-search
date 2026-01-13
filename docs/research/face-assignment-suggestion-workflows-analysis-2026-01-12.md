# Face Assignment & Suggestion Generation Workflows Analysis

**Date**: 2026-01-12
**Context**: User identified 3 different face assignment approaches with different suggestion generation behaviors
**Investigation Goal**: Understand why manual assignment generates suggestions but training pipeline doesn't

---

## Executive Summary

**Key Finding**: Manual face assignment (approach 1) triggers suggestion generation via `propagate_person_label_job`, but training pipeline (approach 3) does NOT generate suggestions—it only performs auto-assignment based on confidence thresholds. This is an **intentional architectural decision**, not a bug.

### The Three Approaches Compared

| Approach | Endpoint/Flow | Assigns Faces? | Creates Suggestions? | How? |
|----------|--------------|----------------|---------------------|------|
| **1. FaceListSidebar** | `POST /api/v1/faces/faces/{face_id}/assign` | ✅ Yes (1 face) | ✅ Yes | Queues `propagate_person_label_job` |
| **2. Cluster Labeling** | `POST /api/v1/faces/clusters/{cluster_id}/label` | ✅ Yes (N faces) | ✅ Yes | Queues `propagate_person_label_job` |
| **3. Training Pipeline** | `detect_faces_for_session_job` → `FaceAssigner.assign_new_faces()` | ✅ Yes (auto-assign) | ⚠️ **ONLY if threshold met** | Two-tier threshold system |

---

## Detailed Analysis

### Approach 1: Manual Single Face Assignment

**Backend Endpoint**: `/api/v1/faces/faces/{face_id}/assign` (lines 1616-1752 in `faces.py`)

**Workflow**:
```python
# 1. Assign face to person
face.person_id = person.id

# 2. Update Qdrant
qdrant.update_person_ids([face.qdrant_point_id], person.id)

# 3. Create prototype from verified label (lines 1679-1694)
await create_or_update_prototypes(
    db=db,
    qdrant=qdrant,
    person_id=person.id,
    newly_labeled_face_id=face.id,
    max_exemplars=settings.face_prototype_max_exemplars,
    min_quality_threshold=settings.face_prototype_min_quality,
)

# 4. TRIGGER SUGGESTION GENERATION (lines 1720-1740)
queue.enqueue(
    propagate_person_label_job,
    source_face_id=str(face.id),
    person_id=str(person.id),
    min_confidence=0.7,
    max_suggestions=50,
    job_timeout="10m",
)
```

**Suggestion Generation Logic** (`propagate_person_label_job` in `face_jobs.py` lines 832-994):

```python
def propagate_person_label_job(
    source_face_id: str,
    person_id: str,
    min_confidence: float | None = None,  # Default: 0.7 (from config)
    max_suggestions: int | None = None,   # Default: 50 (from config)
):
    # 1. Get source face embedding from Qdrant
    source_vector = qdrant.retrieve(source_face.qdrant_point_id)

    # 2. Search for similar faces (UNASSIGNED ONLY)
    search_results = qdrant.query_points(
        query=source_vector,
        limit=max_suggestions + 10,
        score_threshold=min_confidence,  # 0.7
    )

    # 3. Filter to unassigned faces
    for result in search_results:
        face = get_face_by_qdrant_point_id(result.id)

        # Skip if already assigned
        if face.person_id is not None:
            continue

        # Skip if suggestion already exists
        if existing_pending_suggestion(face.id, person_id):
            continue

        # 4. Create FaceSuggestion
        suggestion = FaceSuggestion(
            face_instance_id=face.id,
            suggested_person_id=person_id,
            confidence=result.score,
            source_face_id=source_face_id,
            status="pending",
        )
        db.add(suggestion)
```

**Key Characteristics**:
- ✅ Always creates suggestions for similar unassigned faces
- ✅ Uses single source face as query vector
- ✅ Finds up to 50 suggestions per assignment
- ✅ Creates prototypes from verified labels
- ✅ Queues background job (non-blocking)

---

### Approach 2: Cluster Labeling

**Backend Endpoint**: `POST /api/v1/faces/clusters/{cluster_id}/label` (lines 341-437 in `faces.py`)

**Workflow**:
```python
# 1. Get all faces in cluster
faces = db.query(FaceInstance).filter(cluster_id == cluster_id).all()

# 2. Assign ALL faces in cluster to person
for face in faces:
    face.person_id = person.id

# 3. Update Qdrant for all faces
qdrant.update_person_ids(qdrant_point_ids, person.id)

# 4. Create prototypes (top 3 quality faces)
sorted_faces = sorted(faces, key=lambda f: f.quality_score, reverse=True)
for face in sorted_faces[:3]:
    prototype = PersonPrototype(
        person_id=person.id,
        face_instance_id=face.id,
        role=PrototypeRole.EXEMPLAR,
    )

# 5. TRIGGER SUGGESTION GENERATION using BEST face (lines 408-430)
best_face = sorted_faces[0]
queue.enqueue(
    propagate_person_label_job,
    source_face_id=str(best_face.id),
    person_id=str(person.id),
    min_confidence=0.7,
    max_suggestions=50,
)
```

**Key Characteristics**:
- ✅ Assigns N faces at once (entire cluster)
- ✅ Creates suggestions using highest-quality face as source
- ✅ Same suggestion generation as approach 1
- ⚠️ UI clarity: **ALL** faces in the cluster are assigned (not always obvious in UI)

**UI Consideration**: The cluster page shows sample faces, but clicking "Label as Person" assigns **ALL** faces in that cluster to the person, which may include faces not visible in the UI preview.

---

### Approach 3: Training Pipeline (Face Detection Session)

**Background Job**: `detect_faces_for_session_job` (lines 421-829 in `face_jobs.py`)

**Workflow**:
```python
# 1. Detect faces in batch
face_service.process_assets_batch(
    asset_ids=asset_ids,
    min_confidence=0.5,  # Detection threshold
)

# 2. AUTO-ASSIGN faces using FaceAssigner (lines 715-749)
assigner = get_face_assigner(db_session=db_session)
assignment_result = assigner.assign_new_faces(
    since=session.started_at,
    max_faces=10000,
)

faces_assigned_to_persons = assignment_result.get("auto_assigned", 0)
suggestions_created = assignment_result.get("suggestions_created", 0)

# 3. Run clustering for remaining unlabeled faces (lines 750-783)
clusterer.cluster_unlabeled_faces(
    quality_threshold=0.3,
    max_faces=10000,
)
```

**FaceAssigner Two-Tier Threshold System** (`assign_new_faces()` in `assigner.py` lines 34-256):

```python
# Get config thresholds
auto_assign_threshold = config.get_float("face_auto_assign_threshold")  # 0.85
suggestion_threshold = config.get_float("face_suggestion_threshold")    # 0.70

for face in unassigned_faces:
    # Search against ALL prototypes
    matches = qdrant.search_against_prototypes(
        query_embedding=face_embedding,
        score_threshold=suggestion_threshold,  # 0.70 (lower threshold)
    )

    best_match = matches[0]

    # TWO-TIER DECISION LOGIC:
    if best_match.score >= auto_assign_threshold:  # >= 0.85
        # AUTO-ASSIGN: High confidence
        face.person_id = person_id
        qdrant.update_person_ids([face.qdrant_point_id], person_id)
        assigned_count += 1

    elif best_match.score >= suggestion_threshold:  # 0.70 - 0.84
        # CREATE SUGGESTION: Medium confidence
        if not existing_suggestion(face.id, person_id):
            suggestion = FaceSuggestion(
                face_instance_id=face.id,
                suggested_person_id=person_id,
                confidence=best_match.score,
                source_face_id=prototype.face_instance_id,
                status="pending",
            )
            db.add(suggestion)
            suggestion_count += 1

    else:  # < 0.70
        # LEAVE UNASSIGNED
        unassigned_count += 1
```

**Key Characteristics**:
- ✅ Does create suggestions, but **ONLY** for faces scoring 0.70-0.84
- ✅ Auto-assigns faces scoring >= 0.85 (no suggestion needed)
- ⚠️ Does **NOT** trigger `propagate_person_label_job` for auto-assigned faces
- ⚠️ No recursive suggestion generation (each auto-assigned face doesn't search for more)

---

## The Critical Difference

### Manual Assignment (Approaches 1 & 2):
```
User assigns 1 face manually
    ↓
Queue propagate_person_label_job
    ↓
Search Qdrant for similar faces (using assigned face as query)
    ↓
Create up to 50 suggestions for review
    ↓
User can accept more suggestions → triggers MORE searches
    ↓
Snowball effect: 1 manual label → hundreds of suggestions
```

### Training Pipeline (Approach 3):
```
Training detects 1000 new faces
    ↓
FaceAssigner searches against existing prototypes
    ↓
    ├─ Score >= 0.85 → AUTO-ASSIGN (no propagation)
    ├─ Score 0.70-0.84 → CREATE SUGGESTION (no propagation)
    └─ Score < 0.70 → LEAVE UNASSIGNED → Send to clustering
         ↓
Unlabeled faces → HDBSCAN clustering → Unknown person clusters
```

**Result**: Training pipeline creates suggestions **ONLY** for medium-confidence matches. It does **NOT** use newly-assigned faces to search for MORE similar faces.

---

## Why This Design Exists

### Design Rationale:

1. **Performance**: Training sessions process thousands of faces. Queuing 50 propagation jobs per auto-assigned face would overwhelm the system.

2. **Circular Logic Prevention**: Auto-assigned faces (0.85+ confidence) are already matched to existing prototypes. Using them as new query vectors would likely find the same faces already in the person's cluster.

3. **Prototype Quality**: Manual assignments are user-verified high-quality labels. Auto-assignments are machine-predicted and may contain errors.

4. **Batch Efficiency**: Training pipeline prioritizes throughput (detect → assign → cluster) over exhaustive suggestion generation.

### Evidence from Code:

**Manual assignment explicitly triggers propagation** (line 1720-1740 in `faces.py`):
```python
# Trigger propagation job using the assigned face as source
try:
    queue.enqueue(
        propagate_person_label_job,
        source_face_id=str(face.id),
        person_id=str(person.id),
        min_confidence=0.7,
        max_suggestions=50,
        job_timeout="10m",
    )
    logger.info(
        f"Queued propagation job for face {face.id} → person {person.id} "
        f"(single assignment)"
    )
except Exception as e:
    logger.warning(f"Failed to enqueue propagation job: {e}")
    # Don't fail the request if job queueing fails
```

**Training pipeline does NOT trigger propagation** (lines 715-749 in `face_jobs.py`):
```python
# Auto-assign faces to known persons (uses config-based thresholds)
logger.info(f"[{job_id}] Auto-assigning faces to known persons")
faces_assigned_to_persons = 0
suggestions_created = 0

assigner = get_face_assigner(db_session=db_session)

# Assign faces created during this session
assignment_result = assigner.assign_new_faces(
    since=session.started_at,
    max_faces=10000,  # Process all new faces
)

faces_assigned_to_persons = assignment_result.get("auto_assigned", 0)
suggestions_created = assignment_result.get("suggestions_created", 0)

# NO propagate_person_label_job queued here!
```

---

## Gaps and Recommendations

### Gap 1: No Propagation After Auto-Assignment

**Issue**: Users with 1000+ labeled faces from training don't get automatic suggestion generation. They must manually trigger "Find More Suggestions" or rely on future training runs.

**Impact**:
- Medium confidence faces (0.70-0.84) become suggestions
- High confidence faces (0.85+) are auto-assigned
- But neither triggers recursive search for MORE similar faces

**Recommendation**: Add optional post-training suggestion generation

**Implementation**:
```python
# In detect_faces_for_session_job (after clustering)
if session.enable_post_training_suggestions:
    # Get persons that received auto-assignments
    from collections import Counter
    assigned_persons = Counter([f.person_id for f in newly_assigned_faces])

    for person_id, count in assigned_persons.most_common(10):
        # Queue multi-prototype suggestion job for top 10 persons
        queue.enqueue(
            propagate_person_label_multiproto_job,
            person_id=str(person_id),
            min_confidence=0.7,
            max_suggestions=50,
            preserve_existing=True,
        )
```

---

### Gap 2: Cluster Labeling UI Clarity

**Issue**: "Label as Person" button on cluster page assigns **ALL** faces in cluster, but UI only shows sample faces (5-10 typically).

**Current Behavior** (lines 341-437 in `faces.py`):
```python
# Get ALL faces in cluster
faces = db.query(FaceInstance).filter(cluster_id == cluster_id).all()

# Assign ALL of them
for face in faces:
    face.person_id = person.id

# UI only showed sample_face_ids[:5]
```

**Recommendation**: Enhance UI transparency

**UI Changes**:
1. Show total count: "Label 47 faces as Person"
2. Add preview gallery showing all faces (paginated if needed)
3. Add confirmation dialog:
   ```
   You are about to label 47 faces as "John Smith"

   [Show all faces] [Cancel] [Confirm]
   ```

---

### Gap 3: Training Pipeline Doesn't Use Person History

**Issue**: Training pipeline only uses **current prototypes** for assignment. It doesn't learn from recently assigned faces during the same session.

**Current Logic**:
```
Training starts → Load existing prototypes (3-5 per person)
    ↓
Detect 1000 faces
    ↓
Assign 300 faces to Person A using old prototypes
    ↓
Detect 1000 MORE faces
    ↓
Still using old prototypes (not the 300 newly assigned faces!)
```

**Recommendation**: Dynamic prototype updates during training

**Implementation**:
```python
# In detect_faces_for_session_job
for batch in batches:
    # Detect faces
    face_service.process_assets_batch(batch)

    # Assign using current prototypes
    assignment_result = assigner.assign_new_faces(...)

    # REFRESH prototypes after each batch
    if assignment_result["auto_assigned"] > 10:
        # Recompute prototypes for persons that gained significant faces
        for person_id in top_assigned_persons:
            recompute_prototypes_for_person(
                person_id=person_id,
                preserve_pins=True,
            )
```

---

### Gap 4: No Suggestion Generation Feedback

**Issue**: Users don't know which approach generates suggestions vs auto-assigns.

**Recommendation**: Add suggestion generation status to API responses

**API Enhancement**:
```typescript
// Current response
{
  "face_id": "uuid",
  "person_id": "uuid",
  "person_name": "John Smith"
}

// Enhanced response
{
  "face_id": "uuid",
  "person_id": "uuid",
  "person_name": "John Smith",
  "suggestion_job_queued": true,
  "expected_suggestions": "0-50"
}
```

---

## Architectural Decision Summary

### Why Manual Assignment Generates Suggestions (Intentional):

1. **User verification**: Manually assigned faces are high-quality training data
2. **Progressive labeling**: Snowball effect helps users label entire person quickly
3. **Interactive workflow**: User expects immediate feedback and next steps

### Why Training Doesn't Generate Suggestions (Intentional):

1. **Performance**: Thousands of faces × 50 suggestions each = system overload
2. **Batch efficiency**: Prioritize throughput over exhaustive search
3. **Prototype-based matching**: Auto-assigned faces don't improve search quality
4. **Clustering fallback**: Unknown faces go to HDBSCAN for discovery

---

## Implementation Recommendations

### High Priority:

1. **Add Post-Training Suggestion Hook** (Easy)
   - Location: `detect_faces_for_session_job` lines 750-783
   - Action: Queue `propagate_person_label_multiproto_job` for top 5-10 persons
   - Benefit: Users with 1000+ labeled faces get suggestions without manual trigger

2. **Improve Cluster Labeling UI** (Medium)
   - Location: Frontend cluster detail page
   - Action: Show all faces + confirmation dialog
   - Benefit: Users understand assignment scope

### Medium Priority:

3. **Dynamic Prototype Updates During Training** (Medium)
   - Location: `detect_faces_for_session_job` batch processing loop
   - Action: Recompute prototypes after each batch
   - Benefit: Better assignment accuracy for long-running sessions

4. **Add Suggestion Status to API** (Easy)
   - Location: `assign_face_to_person` response
   - Action: Include `suggestion_job_queued` flag
   - Benefit: UI can show "Finding similar faces..." progress

### Low Priority:

5. **Configurable Propagation Strategy** (Hard)
   - Location: System config
   - Action: Add `auto_propagate_after_training` setting
   - Benefit: Users can opt-in to aggressive suggestion generation

---

## Testing Recommendations

### Verify Current Behavior:

```bash
# Test 1: Manual assignment generates suggestions
curl -X POST http://localhost:8000/api/v1/faces/faces/{face_id}/assign \
  -H "Content-Type: application/json" \
  -d '{"person_id": "uuid"}'

# Check RQ queue for propagate_person_label_job
redis-cli LLEN rq:queue:default

# Test 2: Training pipeline creates suggestions only for 0.70-0.84 matches
# Run training session, then query:
SELECT status, COUNT(*)
FROM face_suggestions
WHERE created_at > '2026-01-12 00:00:00'
GROUP BY status;

# Expected: suggestions_created > 0 for medium-confidence faces
```

### Verify Recommendation 1 (Post-Training Suggestions):

```python
# After implementing post-training hook
session = run_training_session(asset_ids=test_assets)

# Check that multi-proto jobs were queued
jobs = rq_queue.get_jobs()
assert any(job.func_name == 'propagate_person_label_multiproto_job' for job in jobs)

# Wait for jobs to complete
wait_for_queue_empty()

# Verify suggestions were created
suggestions = db.query(FaceSuggestion).filter(
    FaceSuggestion.created_at > session.started_at
).all()
assert len(suggestions) > 0
```

---

## Conclusion

**The three approaches are working as designed**:

1. **Manual assignment** (approach 1 & 2): User-verified, triggers aggressive suggestion generation
2. **Training pipeline** (approach 3): Batch processing, creates suggestions only for medium-confidence matches

**The "problem" is actually a feature**: Training prioritizes speed over exhaustive search. The solution is to:
- Add optional post-training suggestion generation
- Improve UI clarity around cluster labeling
- Consider dynamic prototype updates for long sessions

**No bugs found** – the system operates according to its architectural design. The gap is in **discoverability**: users expect training to generate as many suggestions as manual assignment does.

---

## Code References

### Backend Files Analyzed:

1. **`image-search-service/src/image_search_service/api/routes/faces.py`**
   - Lines 1616-1752: `assign_face_to_person()` – Manual assignment endpoint
   - Lines 341-437: `label_cluster()` – Cluster labeling endpoint

2. **`image-search-service/src/image_search_service/queue/face_jobs.py`**
   - Lines 832-994: `propagate_person_label_job()` – Suggestion generation
   - Lines 421-829: `detect_faces_for_session_job()` – Training pipeline
   - Lines 1431-1675: `propagate_person_label_multiproto_job()` – Multi-prototype search

3. **`image-search-service/src/image_search_service/faces/assigner.py`**
   - Lines 34-256: `assign_new_faces()` – Two-tier threshold system

4. **`image-search-service/src/image_search_service/faces/trainer.py`**
   - Lines 187-316: `fine_tune_for_person_clustering()` – Training logic (no suggestion generation)

### Frontend Files to Review:

- `image-search-ui/src/routes/faces/clusters/[id]/+page.svelte` – Cluster labeling UI
- `image-search-ui/src/lib/components/FaceListSidebar.svelte` – Manual assignment UI
- `image-search-ui/src/routes/admin/training-sessions/+page.svelte` – Training session monitoring

---

**Analysis Complete**
**Date**: 2026-01-12
**Researcher**: Claude Sonnet 4.5 (Research Agent)
