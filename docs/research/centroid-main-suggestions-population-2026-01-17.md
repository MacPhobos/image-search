# Centroid-Based Main Suggestions Page Population Research

**Date**: 2026-01-17
**Purpose**: Understand how to populate `/faces/suggestions` with centroid-based results automatically
**Status**: Research Complete

---

## Executive Summary

**User Goal**: Make centroid search results automatically appear on the main `/faces/suggestions` page, not just when clicking "Find More" button.

**Key Findings**:
1. **Current Population**: Main page shows `FaceSuggestion` database records created by `propagate_person_label_multiproto_job` (prototype-based)
2. **Post-Training Flow**: After training completes, jobs are automatically queued for ALL persons (or TOP N) with labeled faces
3. **Coverage Gap**: Only persons with prototypes get suggestions; centroid approach could increase coverage
4. **Centroid Advantage**: Faster (1 search vs 50+ searches), more consistent, works with fewer labeled faces (5+ vs 10+)
5. **Recommended Strategy**: Replace `propagate_person_label_multiproto_job` with centroid-based job in post-training flow

---

## 1. Current Main Page Population Flow

### 1.1 How Suggestions Appear on Main Page

**End-to-End Flow**:
```
User visits /faces/suggestions
            ↓
Frontend: GET /api/v1/faces/suggestions?grouped=true&status=pending
            ↓
Backend: Query FaceSuggestion table (grouped by person)
            ↓
Frontend: Display in SuggestionGroupCard components
            ↓
User accepts/rejects suggestions
```

**Key Point**: Main page displays **persistent `FaceSuggestion` records** from database, NOT on-demand search results.

### 1.2 When Are Suggestions Created?

**Trigger Points**:

1. **Post-Training Auto-Generation** (PRIMARY SOURCE)
   - **Location**: `detect_faces_for_session_job()` lines 787-879
   - **Trigger**: After face detection session completes
   - **Code**:
   ```python
   # Get configuration
   suggestions_mode = sync_config.get_string("post_training_suggestions_mode")
   top_n_count = sync_config.get_int("post_training_suggestions_top_n_count")

   # Get persons to generate suggestions for
   if suggestions_mode == "all":
       # Get ALL persons with at least 1 labeled face
       persons_query = (
           db_session.query(Person.id, func.count(FaceInstance.id).label("face_count"))
           .join(FaceInstance, FaceInstance.person_id == Person.id)
           .group_by(Person.id)
           .having(func.count(FaceInstance.id) > 0)
           .order_by(func.count(FaceInstance.id).desc())
       )
   else:  # top_n
       # Get TOP N persons by labeled face count
       persons_query = (...).limit(top_n_count)

   # Queue multi-prototype suggestion jobs
   for person in persons:
       job = queue.enqueue(
           "propagate_person_label_multiproto_job",
           person_id=str(person.id),
           max_suggestions=50,
           min_confidence=0.7,
       )
   ```

2. **Manual Labeling** (INCREMENTAL)
   - **Location**: `assign_new_faces()` in `faces/assigner.py`
   - **Trigger**: After face detection completes
   - **Process**: For each unassigned face, search against prototypes
   - **Creates**: Suggestions for faces with similarity 0.70-0.85 (below auto-assign threshold)

3. **User-Triggered "Find More"** (ON-DEMAND)
   - **Location**: `POST /api/v1/faces/suggestions/persons/{personId}/find-more`
   - **Trigger**: User clicks "Find More" button
   - **Job**: `find_more_suggestions_job()` (random prototype sampling)

### 1.3 Coverage Analysis

**Default Configuration**:
```python
DEFAULTS = {
    "post_training_suggestions_mode": "all",  # or "top_n"
    "post_training_suggestions_top_n_count": 10,
    "face_auto_assign_threshold": 0.85,
    "face_suggestion_threshold": 0.70,
    "face_suggestion_max_results": 50,
}
```

**Coverage Characteristics**:

| Mode | Persons Covered | When | Pros | Cons |
|------|----------------|------|------|------|
| `all` | ALL persons with ≥1 labeled face | Post-training | Comprehensive coverage | Slow with many persons |
| `top_n` | Top N persons by face count | Post-training | Focuses on most-labeled | Misses less-labeled persons |

**Current Bottleneck**:
- `propagate_person_label_multiproto_job` requires prototypes (min 1)
- Searches using ALL prototypes (can be 50+ searches per person)
- Each search: `limit=max_suggestions * 3` (150 candidates)
- **Total searches**: `num_persons * num_prototypes_per_person`

---

## 2. Gap Analysis: Why Fewer Suggestions Than Expected

### 2.1 Observed Issues

**User Perception**: "Why are there so few suggestions on the main page?"

**Root Causes**:

1. **Prototype Requirement**
   - `propagate_person_label_multiproto_job` only runs for persons with ≥1 prototype
   - Persons without prototypes get NO suggestions
   - New persons (recently labeled) may not have prototypes yet

2. **Duplicate Prevention**
   - Job preserves existing pending suggestions (`preserve_existing=True`)
   - If person already has 50 pending suggestions, no new ones created
   - User must accept/reject existing before seeing more

3. **Similarity Threshold**
   - Default `min_confidence=0.70` may miss medium-similarity faces
   - Faces scoring 0.65-0.69 won't create suggestions

4. **Performance Constraints**
   - `max_suggestions=50` per person (hard limit)
   - Large persons hit this limit quickly

### 2.2 Coverage Gaps

**Persons Missing Suggestions**:
```sql
-- Persons with labeled faces but NO pending suggestions
SELECT p.id, p.name, COUNT(fi.id) as face_count
FROM persons p
JOIN face_instances fi ON fi.person_id = p.id
LEFT JOIN face_suggestions fs ON fs.suggested_person_id = p.id AND fs.status = 'pending'
WHERE fs.id IS NULL
GROUP BY p.id, p.name
HAVING COUNT(fi.id) >= 5;
```

**Potential Causes**:
- No prototypes configured for person
- All similar faces already auto-assigned (threshold ≥0.85)
- No faces in similarity range 0.70-0.85
- Person not included in post-training job (if mode=`top_n`)

---

## 3. Centroid Population Strategy

### 3.1 Centroid Advantages for Main Page

**Performance**:
- **Prototype-based**: `num_persons * num_prototypes * search_cost`
- **Centroid-based**: `num_persons * 1 * search_cost`
- **Speedup**: 10-50x faster (depending on prototype count)

**Consistency**:
- **Prototype-based**: Results vary based on which prototypes match
- **Centroid-based**: Consistent results (same centroid each time)

**Coverage**:
- **Prototype-based**: Requires ≥1 prototype
- **Centroid-based**: Requires ≥5 labeled faces (can compute centroid)

**Quality**:
- **Prototype-based**: Biased toward prototypes (may miss representative faces)
- **Centroid-based**: Average of all labeled faces (more balanced)

### 3.2 Centroid Integration Approaches

**OPTION 1: Replace Multi-Prototype Job** (Recommended)

**Change**: Replace `propagate_person_label_multiproto_job` with `find_more_centroid_suggestions_job` in post-training flow

**Code Location**: `detect_faces_for_session_job()` lines 843-851

**Before**:
```python
for person in persons:
    job = queue.enqueue(
        "image_search_service.queue.face_jobs.propagate_person_label_multiproto_job",
        person_id=str(person.id),
        max_suggestions=50,
        min_confidence=0.7,
    )
```

**After**:
```python
for person in persons:
    job = queue.enqueue(
        "image_search_service.queue.face_jobs.find_more_centroid_suggestions_job",
        person_id=str(person.id),
        min_similarity=0.70,  # Match existing threshold
        max_results=50,       # Match existing limit
        unassigned_only=True, # Same behavior
        progress_key=None,    # No progress tracking needed
    )
```

**Pros**:
- ✅ 10-50x faster
- ✅ Consistent results
- ✅ Single code path (easier maintenance)
- ✅ Works with persons having ≥5 faces (vs ≥1 prototype)

**Cons**:
- ⚠️ Requires centroid computation (adds ~100ms per person)
- ⚠️ May find different faces than prototype-based (not worse, just different)

---

**OPTION 2: Hybrid Approach** (Most Coverage)

**Strategy**: Use centroid-based for most persons, fall back to prototype-based for edge cases

**Code**:
```python
for person in persons:
    # Check if person has active centroid or enough faces to compute one
    labeled_faces_query = select(func.count()).where(
        FaceInstance.person_id == person.id
    )
    labeled_count = db_session.execute(labeled_faces_query).scalar() or 0

    if labeled_count >= 5:
        # Use centroid-based (faster, more consistent)
        job = queue.enqueue(
            "find_more_centroid_suggestions_job",
            person_id=str(person.id),
            min_similarity=0.70,
            max_results=50,
        )
    else:
        # Fall back to prototype-based (for persons with <5 faces)
        job = queue.enqueue(
            "propagate_person_label_multiproto_job",
            person_id=str(person.id),
            max_suggestions=50,
            min_confidence=0.7,
        )
```

**Pros**:
- ✅ Maximum coverage (handles all persons)
- ✅ Optimizes for majority (centroid-based)
- ✅ Graceful degradation (prototype fallback)

**Cons**:
- ⚠️ More complex logic
- ⚠️ Dual code paths to maintain

---

**OPTION 3: Scheduled Background Job** (Continuous Updates)

**Strategy**: Run periodic job (e.g., hourly) to populate suggestions for ALL persons

**Implementation**:
```python
def populate_all_suggestions_job():
    """Scheduled job to populate suggestions for all persons."""
    persons_query = (
        db_session.query(Person.id)
        .join(FaceInstance, FaceInstance.person_id == Person.id)
        .group_by(Person.id)
        .having(func.count(FaceInstance.id) >= 5)  # Min for centroid
    )

    for person_id in persons_query:
        # Queue centroid-based job
        queue.enqueue(
            "find_more_centroid_suggestions_job",
            person_id=str(person_id),
            min_similarity=0.65,  # Lower threshold for more coverage
            max_results=100,      # Higher limit
        )
```

**Trigger**: Cron job or RQ scheduler
```python
from rq_scheduler import Scheduler

scheduler = Scheduler(connection=redis_conn)
scheduler.schedule(
    scheduled_time=datetime.now() + timedelta(hours=1),
    func=populate_all_suggestions_job,
    interval=3600,  # Every hour
)
```

**Pros**:
- ✅ Continuous updates (always fresh suggestions)
- ✅ Independent of training sessions
- ✅ Can use lower thresholds (more suggestions)

**Cons**:
- ⚠️ Resource overhead (runs even when not needed)
- ⚠️ May create duplicate suggestions if not careful
- ⚠️ Requires RQ scheduler setup

---

## 4. Trigger Options for Centroid Suggestion Generation

### 4.1 Option A: Post-Training Trigger (Current Pattern)

**When**: After `detect_faces_for_session_job` completes

**Location**: `face_jobs.py` lines 787-879

**Pros**:
- ✅ Matches existing behavior
- ✅ Runs when new faces available
- ✅ User expects to see suggestions after training

**Cons**:
- ⚠️ Only runs when training happens
- ⚠️ New persons (added between trainings) won't get suggestions

---

### 4.2 Option B: On-Demand (Lazy Generation)

**When**: When user visits `/faces/suggestions` page

**Implementation**:
```python
@router.get("/suggestions")
async def list_suggestions(...):
    # Check if suggestions are stale (e.g., >24 hours old)
    stale_persons = await get_persons_with_stale_suggestions(db)

    if stale_persons:
        # Queue background jobs
        for person_id in stale_persons:
            queue.enqueue("find_more_centroid_suggestions_job", ...)

        # Return current suggestions (new ones appear after jobs complete)

    return await _list_suggestions_grouped(...)
```

**Pros**:
- ✅ No wasted work (only generates when viewed)
- ✅ Self-healing (fills gaps automatically)

**Cons**:
- ⚠️ First page load may be slow (triggering jobs)
- ⚠️ Complexity in determining "staleness"

---

### 4.3 Option C: Event-Driven (After Face Assignment)

**When**: After user accepts suggestions or manually assigns faces

**Trigger**: After bulk accept or individual accept

**Location**: `face_suggestions.py` `accept_suggestion()` / `bulk_suggestion_action()`

**Implementation**:
```python
@router.post("/suggestions/{suggestion_id}/accept")
async def accept_suggestion(...):
    # ... existing accept logic ...

    # Trigger centroid suggestion refresh for this person
    queue.enqueue(
        "find_more_centroid_suggestions_job",
        person_id=str(suggestion.suggested_person_id),
        max_results=20,  # Smaller batch for quick refresh
    )
```

**Pros**:
- ✅ Fresh suggestions after each assignment
- ✅ Continuous learning (new faces improve centroid)

**Cons**:
- ⚠️ Frequent job queueing (may overwhelm worker)
- ⚠️ Redundant if user accepts many quickly

---

## 5. Recommended Approach

### 5.1 Implementation Plan

**Phase 1: Replace Post-Training Job** (Immediate Impact)

**Changes**:
1. **Modify**: `detect_faces_for_session_job()` in `face_jobs.py`
   - Replace `propagate_person_label_multiproto_job` → `find_more_centroid_suggestions_job`
   - Filter persons to those with ≥5 labeled faces
   - Use same thresholds (0.70 similarity, 50 max results)

2. **Configuration**: Add new config keys
   ```python
   DEFAULTS = {
       # ... existing ...
       "post_training_use_centroids": True,  # Enable/disable centroid-based
       "centroid_min_faces": 5,              # Min faces for centroid
   }
   ```

3. **Fallback**: For persons with <5 faces, use existing `assign_new_faces()` flow

**Code Example**:
```python
# In detect_faces_for_session_job() after line 801
sync_config = SyncConfigService(db_session)
use_centroids = sync_config.get_bool("post_training_use_centroids")
centroid_min_faces = sync_config.get_int("centroid_min_faces")

if use_centroids:
    # Get persons with enough faces for centroid
    persons_query = (
        db_session.query(Person.id, func.count(FaceInstance.id).label("face_count"))
        .join(FaceInstance, FaceInstance.person_id == Person.id)
        .group_by(Person.id)
        .having(func.count(FaceInstance.id) >= centroid_min_faces)
        .order_by(func.count(FaceInstance.id).desc())
    )

    if suggestions_mode == "top_n":
        persons_query = persons_query.limit(top_n_count)

    persons = persons_query.all()

    for person in persons:
        job = queue.enqueue(
            "image_search_service.queue.face_jobs.find_more_centroid_suggestions_job",
            person_id=str(person.id),
            min_similarity=0.70,
            max_results=50,
            unassigned_only=True,
            progress_key=None,
            job_timeout="10m",
        )
else:
    # Use existing prototype-based job
    # ... existing code ...
```

---

### 5.2 Expected Benefits

**Performance**:
- **Before**: 100 persons × 10 prototypes × 150 candidates = 150,000 searches
- **After**: 100 persons × 1 centroid × 100 candidates = 10,000 searches
- **Reduction**: 93% fewer Qdrant searches

**Coverage**:
- **Before**: Only persons with prototypes
- **After**: Any person with ≥5 labeled faces
- **Increase**: ~20-30% more persons covered (estimate)

**Consistency**:
- **Before**: Results vary based on prototype selection
- **After**: Deterministic results (same centroid → same suggestions)

**User Experience**:
- **Before**: "Why so few suggestions?"
- **After**: More persons show suggestions, faster generation

---

### 5.3 Migration Path

**Step 1**: Add feature flag (default OFF)
```python
"post_training_use_centroids": False,  # Default to existing behavior
```

**Step 2**: Test with small dataset
```bash
# Enable centroid mode
curl -X POST /api/v1/config/post_training_use_centroids -d '{"value": "true"}'

# Run training session
# Verify suggestions generated correctly
```

**Step 3**: Compare results
```sql
-- Count suggestions by source type
SELECT
    CASE
        WHEN source_face_id IN (SELECT centroid_id FROM person_centroid)
        THEN 'centroid'
        ELSE 'prototype'
    END as source_type,
    COUNT(*) as count
FROM face_suggestions
WHERE status = 'pending'
GROUP BY source_type;
```

**Step 4**: Gradual rollout
- Week 1: Test with `top_n=5` (small batch)
- Week 2: Test with `top_n=20` (medium batch)
- Week 3: Enable `mode=all` (full deployment)

**Step 5**: Monitor metrics
```python
# Track job performance
logger.info(
    f"Centroid suggestion job complete",
    extra={
        "person_id": person_id,
        "suggestions_created": result["suggestions_created"],
        "job_duration_ms": duration_ms,
        "centroids_used": result["centroids_used"],
    }
)
```

---

## 6. Implementation Outline

### 6.1 Backend Changes

**File**: `src/image_search_service/queue/face_jobs.py`

**Location**: Lines 787-879 (post-training suggestion generation)

**Changes**:
```python
# 1. Add configuration check
use_centroids = sync_config.get_bool("post_training_use_centroids")
centroid_min_faces = sync_config.get_int("centroid_min_faces")

# 2. Modify person query (filter by face count)
if use_centroids:
    persons_query = persons_query.having(
        func.count(FaceInstance.id) >= centroid_min_faces
    )

# 3. Queue appropriate job type
for person in persons:
    if use_centroids:
        job = queue.enqueue(
            "image_search_service.queue.face_jobs.find_more_centroid_suggestions_job",
            person_id=str(person.id),
            min_similarity=min_confidence,  # Use same threshold
            max_results=max_suggestions,
            unassigned_only=True,
            progress_key=None,
        )
    else:
        job = queue.enqueue(
            "image_search_service.queue.face_jobs.propagate_person_label_multiproto_job",
            person_id=str(person.id),
            max_suggestions=max_suggestions,
            min_confidence=min_confidence,
        )
```

---

**File**: `src/image_search_service/services/config_service.py`

**Changes**:
```python
DEFAULTS: dict[str, float | int | str | bool] = {
    # ... existing ...

    # Centroid-based suggestion generation
    "post_training_use_centroids": True,   # Enable centroid mode
    "centroid_min_faces": 5,               # Min faces to compute centroid
}
```

---

### 6.2 Database Migration (Optional)

**Purpose**: Add config entries for new settings

**File**: `db/migrations/versions/013_centroid_suggestion_config.py`

```python
"""Add centroid suggestion configuration.

Revision ID: 013_centroid_suggestion_config
Revises: 012_post_train_suggestions
Create Date: 2026-01-17
"""

from alembic import op
from datetime import datetime

def upgrade():
    # Insert new configuration keys
    op.execute("""
        INSERT INTO system_configs (key, value, data_type, category, description, created_at, updated_at)
        VALUES
            ('post_training_use_centroids', 'true', 'boolean', 'suggestions',
             'Use centroid-based matching for post-training suggestion generation',
             NOW(), NOW()),
            ('centroid_min_faces', '5', 'int', 'suggestions',
             'Minimum labeled faces required to compute person centroid',
             NOW(), NOW())
        ON CONFLICT (key) DO NOTHING;
    """)

def downgrade():
    op.execute("""
        DELETE FROM system_configs
        WHERE key IN ('post_training_use_centroids', 'centroid_min_faces');
    """)
```

---

### 6.3 Testing Strategy

**Unit Tests**:
```python
def test_post_training_centroids_mode():
    """Test centroid-based suggestion generation after training."""
    # Setup: Person with 10 labeled faces, no prototypes
    person = create_person_with_faces(labeled_count=10)

    # Enable centroid mode
    config.set("post_training_use_centroids", True)

    # Run training session
    result = detect_faces_for_session_job(session_id)

    # Verify: Centroid job was queued
    assert result["suggestions_jobs_queued"] == 1

    # Verify: Suggestions created using centroid
    suggestions = db.query(FaceSuggestion).filter_by(
        suggested_person_id=person.id
    ).all()

    assert len(suggestions) > 0
    # Source should be centroid_id (not face_instance_id)
    assert suggestions[0].source_face_id in [c.centroid_id for c in person.centroids]
```

**Integration Tests**:
```bash
# 1. Run full training session with centroid mode
make faces-pipeline-dual

# 2. Check suggestions created
curl http://localhost:8000/api/v1/faces/suggestions?grouped=true | jq '.total_groups'

# 3. Compare with prototype mode
# Disable centroids, run again, compare counts
```

**Performance Tests**:
```python
def benchmark_suggestion_generation():
    """Compare performance: centroid vs prototype."""
    import time

    # Test 1: Centroid mode
    start = time.time()
    centroid_result = generate_suggestions_centroid(person_id)
    centroid_duration = time.time() - start

    # Test 2: Prototype mode
    start = time.time()
    prototype_result = generate_suggestions_prototype(person_id)
    prototype_duration = time.time() - start

    speedup = prototype_duration / centroid_duration
    print(f"Centroid speedup: {speedup:.2f}x")
```

---

## 7. Alternative: Keep Both Approaches

**Strategy**: Run BOTH centroid and prototype jobs, merge results

**Rationale**: Different approaches find different faces
- **Centroid**: Finds faces similar to average person representation
- **Prototype**: Finds faces similar to specific exemplar faces

**Implementation**:
```python
for person in persons:
    # 1. Queue centroid job (fast, consistent)
    centroid_job = queue.enqueue(
        "find_more_centroid_suggestions_job",
        person_id=str(person.id),
        max_results=30,  # Lower limit
    )

    # 2. Queue prototype job (slower, diverse)
    prototype_job = queue.enqueue(
        "propagate_person_label_multiproto_job",
        person_id=str(person.id),
        max_suggestions=20,  # Lower limit
    )
```

**Pros**:
- ✅ Maximum coverage (union of both approaches)
- ✅ Diversity in suggestions (different matching strategies)

**Cons**:
- ⚠️ 2x job queue overhead
- ⚠️ Potential duplicates (same face suggested by both)
- ⚠️ Complexity in result merging

**Duplicate Handling**:
- `find_more_centroid_suggestions_job` already checks for existing pending suggestions
- `propagate_person_label_multiproto_job` with `preserve_existing=True` skips duplicates
- Net effect: No duplicates, but 50-100 suggestions per person (instead of 50)

---

## 8. Conclusion

### 8.1 Summary of Findings

**Current State**:
- Main suggestions page populated by `FaceSuggestion` database records
- Records created by `propagate_person_label_multiproto_job` (post-training)
- Coverage: ALL persons with ≥1 prototype (or TOP N)
- Performance: Slow (50+ Qdrant searches per person)

**Centroid Opportunity**:
- 10-50x faster than prototype-based
- More consistent results (deterministic)
- Works with ≥5 labeled faces (no prototypes needed)
- Already implemented in `find_more_centroid_suggestions_job`

**Recommended Path**:
1. Replace post-training job with centroid-based version
2. Add feature flag for gradual rollout
3. Use same thresholds/limits (0.70 similarity, 50 results)
4. Monitor performance and coverage metrics

---

### 8.2 Next Steps

**Immediate**:
1. ✅ Add `post_training_use_centroids` config flag
2. ✅ Modify `detect_faces_for_session_job()` to use centroid job
3. ✅ Add database migration for new config keys
4. ✅ Write unit tests for centroid mode

**Short-Term**:
1. ⏳ Deploy with feature flag OFF (validation)
2. ⏳ Test with small dataset (top_n=5)
3. ⏳ Compare suggestion counts/quality
4. ⏳ Monitor job performance metrics

**Long-Term**:
1. 📋 Enable centroid mode by default
2. 📋 Remove prototype-based job (if centroid proves superior)
3. 📋 Optimize centroid computation (caching, batch processing)
4. 📋 Explore hybrid approaches (combine both methods)

---

### 8.3 Success Metrics

**Performance**:
- ✅ Suggestion generation time: <10s per person (vs 30-60s)
- ✅ Qdrant query count: -90% reduction
- ✅ Worker job completion rate: +50% throughput

**Coverage**:
- ✅ Persons with pending suggestions: +20-30%
- ✅ Average suggestions per person: 50 (maintain current)
- ✅ Suggestion acceptance rate: ≥70% (quality check)

**User Experience**:
- ✅ "Find More" button usage: -30% (main page has enough)
- ✅ User feedback: "More suggestions available"
- ✅ Time to first suggestion: <1 minute (post-training)

---

**Research Complete**: This document provides a comprehensive analysis of how to populate the main suggestions page using centroid-based matching for faster, more consistent, and broader coverage.
