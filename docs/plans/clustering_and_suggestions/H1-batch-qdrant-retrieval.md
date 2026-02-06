# H1: Batch Qdrant Embedding Retrieval (N+1 Fix)

**Priority**: High
**Severity**: High (Performance Degradation)
**Estimated Effort**: 3-4 days
**Date**: 2026-02-06

---

## References

| Artifact | Path |
|----------|------|
| Research: Executive Synthesis | `docs/research/clustering_and_suggestions/00-executive-synthesis.md` |
| Research: Backend Analysis | `docs/research/clustering_and_suggestions/02-backend-analysis.md` |
| Research: Devil's Advocate | `docs/research/clustering_and_suggestions/04-devils-advocate-analysis.md` |
| Face Qdrant Client | `image-search-service/src/image_search_service/vector/face_qdrant.py` |
| Centroid Service | `image-search-service/src/image_search_service/services/centroid_service.py` |
| Face Clustering Service | `image-search-service/src/image_search_service/services/face_clustering_service.py` |
| Face Jobs | `image-search-service/src/image_search_service/queue/face_jobs.py` |
| Assigner | `image-search-service/src/image_search_service/faces/assigner.py` |
| Dual Clusterer | `image-search-service/src/image_search_service/faces/dual_clusterer.py` |
| Faces API Routes | `image-search-service/src/image_search_service/api/routes/faces.py` |

---

## 1. Problem Statement

The `FaceQdrantClient.get_embedding_by_point_id()` method retrieves a single embedding from Qdrant per call. It is called inside loops at **7 distinct call sites** across the codebase, creating an N+1 query pattern. For a person with 50 faces, this means 50 separate HTTP round-trips to Qdrant instead of 1 batch request.

Qdrant's `retrieve()` API natively supports batch ID retrieval -- passing multiple IDs in a single call -- but this capability is never used. The current pattern causes:

1. **Linear performance degradation**: Each face adds ~5-15ms of network latency for Qdrant round-trips.
2. **Clustering bottlenecks**: Dual-mode clustering of 500 faces requires 500+ Qdrant calls (5-8 seconds of pure network overhead).
3. **Centroid computation slowdown**: Computing a centroid for a person with 100 faces requires 100 Qdrant calls instead of 1.
4. **Background job timeout risk**: Large clustering or suggestion jobs may timeout due to accumulated network latency.

---

## 2. Root Cause Analysis

### Current Single-Retrieval Method

**File**: `image-search-service/src/image_search_service/vector/face_qdrant.py`
**Lines**: 475-519

```python
def get_embedding_by_point_id(self, point_id: uuid.UUID) -> list[float] | None:
    """Retrieve the embedding vector for a specific face point."""
    try:
        points = self.client.retrieve(
            collection_name=_get_face_collection_name(),
            ids=[str(point_id)],       # <-- Single ID in list
            with_payload=False,
            with_vectors=True,
        )

        if not points:
            logger.warning(f"Face point {point_id} not found in Qdrant")
            return None

        # Handle both dict and list vector formats
        vector = points[0].vector
        # ... vector format handling ...
        return vector
    except Exception as e:
        logger.error(f"Failed to retrieve embedding for point {point_id}: {e}")
        return None
```

Note that the underlying `self.client.retrieve()` call already accepts a **list** of IDs -- it just always receives a list with a single element. The batch capability exists in the Qdrant client but is never leveraged.

### All 7 N+1 Call Sites

Each call site follows the same anti-pattern: loop over a list of faces, calling `get_embedding_by_point_id()` for each one.

---

**Call Site 1: centroid_service.py -- get_person_face_embeddings()**

**File**: `image-search-service/src/image_search_service/services/centroid_service.py`
**Lines**: 205-206
**Context**: Retrieving all face embeddings for a person during centroid computation.

```python
for face in faces:
    embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
```

**Typical N**: 10-200 faces per person. **Impact**: 50-1000ms latency for centroid computation.

---

**Call Site 2: face_clustering_service.py -- calculate_cluster_confidence()**

**File**: `image-search-service/src/image_search_service/services/face_clustering_service.py`
**Lines**: 88-91
**Context**: Calculating pairwise similarity confidence for a cluster.

```python
for point_id in qdrant_point_ids:
    try:
        embedding = self.qdrant.get_embedding_by_point_id(point_id)
        if embedding is not None:
            embeddings.append(np.array(embedding))
```

**Typical N**: 3-20 faces per cluster (capped by `max_faces_for_calculation=20`). **Impact**: 15-100ms per cluster confidence calculation.

---

**Call Site 3: face_jobs.py -- find_more_suggestions_job() (prototype search)**

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Line**: 1423
**Context**: Getting embeddings for selected prototype faces to use as search queries.

```python
for idx, face in enumerate(selected_faces):
    if not face.qdrant_point_id:
        continue
    embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
```

**Typical N**: 5-20 prototype faces. **Impact**: 25-100ms for prototype embedding retrieval.

---

**Call Site 4: face_jobs.py -- find_more_multi_prototype_suggestions_job()**

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Line**: 1629
**Context**: Getting embeddings for all prototypes of a person for multi-prototype search.

```python
for proto in prototypes:
    face = db_session.get(FaceInstance, proto.face_instance_id)
    if face and face.qdrant_point_id:
        embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
```

**Typical N**: 3-10 prototypes per person. **Impact**: 15-50ms for prototype retrieval.

---

**Call Site 5: face_jobs.py -- find_more_centroid_suggestions_job()**

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Line**: 1929
**Context**: Computing on-demand centroid when no active centroid exists.

```python
for face in faces:
    if face.qdrant_point_id:
        embedding = face_qdrant.get_embedding_by_point_id(face.qdrant_point_id)
```

**Typical N**: 5-200 faces (minimum 5 required). **Impact**: 25-1000ms for on-demand centroid computation.

---

**Call Site 6: faces.py API route -- get_face_suggestions()**

**File**: `image-search-service/src/image_search_service/api/routes/faces.py`
**Line**: 1902
**Context**: Getting a single face's embedding to search for similar prototypes. This is technically N=1 (single face), not an N+1 issue.

```python
embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
```

**N**: Always 1. **Impact**: Minimal (~5ms). This call site does NOT need batch optimization but can still use the batch method for API consistency.

---

**Call Site 7: assigner.py -- _get_face_embedding() (called in loops)**

**File**: `image-search-service/src/image_search_service/faces/assigner.py`
**Lines**: 258-284 (method definition), called at lines 128, 173, 230, 322
**Context**: The `_get_face_embedding()` private method wraps a direct `qdrant.client.retrieve()` call (not using `get_embedding_by_point_id`). It is called in multiple loops.

```python
def _get_face_embedding(self, qdrant_point_id: uuid.UUID) -> list[float] | None:
    qdrant = get_face_qdrant_client()
    try:
        points = qdrant.client.retrieve(
            collection_name="faces",
            ids=[str(qdrant_point_id)],
            with_vectors=True,
        )
        if points and points[0].vector:
            return points[0].vector
        return None
```

Called in loops at:
- Line 128: `assign_new_faces()` -- loop over unassigned faces
- Line 150: `assign_to_known_people()` -- loop over labeled faces (inside `DualModeClusterer` which also calls its own `_get_face_embedding`)
- Line 173: `assign_new_faces()` -- loop over unlabeled faces
- Line 322: `compute_person_centroids()` -- loop over all faces for a person

**Typical N**: 10-1000 faces per batch. **Impact**: 50-5000ms for face assignment.

---

**Call Site 8 (Bonus): dual_clusterer.py -- _get_face_embedding()**

**File**: `image-search-service/src/image_search_service/faces/dual_clusterer.py`
**Lines**: 344-366 (method definition), called at lines 150, 173, 230
**Context**: Identical anti-pattern as assigner.py. Private method wraps direct `retrieve()` call, called in loops during clustering.

```python
def _get_face_embedding(self, qdrant_point_id: uuid.UUID) -> npt.NDArray[np.float64] | None:
    qdrant = get_face_qdrant_client()
    try:
        points = qdrant.client.retrieve(
            collection_name="faces",
            ids=[str(qdrant_point_id)],
            with_vectors=True,
        )
        if points and points[0].vector:
            vector = points[0].vector
            if isinstance(vector, dict):
                return np.array(list(vector.values())[0])
            return np.array(vector)
```

Called in loops at:
- Line 150: Loop over labeled faces for building person centroids
- Line 173: Loop over unlabeled faces for assignment to known people
- Line 230: Loop over unknown faces for unsupervised clustering

**Typical N**: 50-5000 faces across all calls. **Impact**: 250-25000ms for full dual-mode clustering.

---

## 3. Current Behavior vs Expected Behavior

### Current Behavior

For a person with 100 faces:
- Centroid computation: 100 HTTP requests to Qdrant (~500-1500ms)
- Dual-mode clustering with 1000 faces: 1000+ HTTP requests (~5000-15000ms)
- Each request: ~5-15ms network round-trip (local) or ~20-50ms (remote Qdrant)

### Expected Behavior

For a person with 100 faces:
- Centroid computation: 1 HTTP request with 100 IDs (~10-30ms)
- Dual-mode clustering with 1000 faces: 10 HTTP requests of 100 IDs each (~100-300ms)
- **10-50x improvement** in Qdrant retrieval time

---

## 4. Implementation Plan

### Step 1: Add Batch Retrieval Method to FaceQdrantClient

**File**: `image-search-service/src/image_search_service/vector/face_qdrant.py`
**Insert after**: `get_embedding_by_point_id()` method (after line 519)

```python
def get_embeddings_by_point_ids(
    self,
    point_ids: list[uuid.UUID],
    batch_size: int = 100,
) -> dict[uuid.UUID, list[float]]:
    """Batch retrieve embedding vectors for multiple face points.

    Uses Qdrant's native batch retrieve capability to fetch multiple
    embeddings in a single request, avoiding the N+1 query pattern.

    Args:
        point_ids: List of face point IDs to retrieve
        batch_size: Maximum IDs per request (Qdrant recommendation: 100)

    Returns:
        Dict mapping point_id -> embedding vector.
        Missing points are silently excluded from the result.
    """
    if not point_ids:
        return {}

    result: dict[uuid.UUID, list[float]] = {}

    # Process in batches to avoid overwhelming Qdrant
    for i in range(0, len(point_ids), batch_size):
        batch = point_ids[i : i + batch_size]
        batch_str_ids = [str(pid) for pid in batch]

        try:
            points = self.client.retrieve(
                collection_name=_get_face_collection_name(),
                ids=batch_str_ids,
                with_payload=False,
                with_vectors=True,
            )

            for point in points:
                # Parse the point ID back to UUID
                try:
                    point_uuid = uuid.UUID(str(point.id))
                except ValueError:
                    continue

                # Extract vector (handle both dict and list formats)
                vector = point.vector
                if vector is None:
                    continue

                if isinstance(vector, dict):
                    # Named vector case
                    first_value = next(iter(vector.values()))
                    if isinstance(first_value, list) and all(
                        isinstance(x, float) for x in first_value
                    ):
                        result[point_uuid] = first_value
                elif isinstance(vector, list):
                    if all(isinstance(x, float) for x in vector):
                        result[point_uuid] = vector

        except Exception as e:
            logger.error(
                f"Failed to batch retrieve embeddings for {len(batch)} points: {e}"
            )
            # Fall back to individual retrieval for this batch
            for pid in batch:
                embedding = self.get_embedding_by_point_id(pid)
                if embedding is not None:
                    result[pid] = embedding

    if len(result) < len(point_ids):
        missing = len(point_ids) - len(result)
        logger.warning(
            f"Batch retrieve: {missing} of {len(point_ids)} points not found"
        )

    return result
```

Key design decisions:
- **Batch size of 100**: Prevents oversized HTTP requests while dramatically reducing call count.
- **Graceful fallback**: If batch retrieval fails, falls back to individual retrieval per point (preserves existing behavior).
- **Silent exclusion**: Missing points are excluded from the result dict rather than raising an exception. Callers check for missing keys.
- **Same vector format handling**: Reuses the same dict/list vector format detection as the single-point method.

### Step 2: Update centroid_service.py -- get_person_face_embeddings()

**File**: `image-search-service/src/image_search_service/services/centroid_service.py`
**Lines**: 201-217

**Before**:
```python
# Retrieve embeddings from Qdrant
face_ids = []
embeddings = []

for face in faces:
    embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
    if embedding is not None:
        face_ids.append(face.id)
        embeddings.append(embedding)
    else:
        logger.warning(
            f"Embedding not found in Qdrant for face {face.id} "
            f"(point_id={face.qdrant_point_id})"
        )
```

**After**:
```python
# Retrieve embeddings from Qdrant (batch)
point_ids = [face.qdrant_point_id for face in faces if face.qdrant_point_id]
embeddings_map = qdrant.get_embeddings_by_point_ids(point_ids)

face_ids = []
embeddings = []

for face in faces:
    if face.qdrant_point_id and face.qdrant_point_id in embeddings_map:
        face_ids.append(face.id)
        embeddings.append(embeddings_map[face.qdrant_point_id])
    elif face.qdrant_point_id:
        logger.warning(
            f"Embedding not found in Qdrant for face {face.id} "
            f"(point_id={face.qdrant_point_id})"
        )
```

### Step 3: Update face_clustering_service.py -- calculate_cluster_confidence()

**File**: `image-search-service/src/image_search_service/services/face_clustering_service.py`
**Lines**: 87-97

**Before**:
```python
# Retrieve embeddings from Qdrant
embeddings = []
for point_id in qdrant_point_ids:
    try:
        embedding = self.qdrant.get_embedding_by_point_id(point_id)
        if embedding is not None:
            embeddings.append(np.array(embedding))
    except Exception as e:
        logger.warning(
            f"Failed to retrieve embedding for point {point_id} in cluster {cluster_id}: {e}"
        )
        continue
```

**After**:
```python
# Retrieve embeddings from Qdrant (batch)
embeddings_map = self.qdrant.get_embeddings_by_point_ids(qdrant_point_ids)

embeddings = []
for point_id in qdrant_point_ids:
    if point_id in embeddings_map:
        embeddings.append(np.array(embeddings_map[point_id]))
    else:
        logger.warning(
            f"Embedding not found for point {point_id} in cluster {cluster_id}"
        )
```

### Step 4: Update face_jobs.py -- find_more_suggestions_job() (prototype search)

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Around line**: 1418-1426

**Before**:
```python
for idx, face in enumerate(selected_faces):
    if not face.qdrant_point_id:
        continue
    # Get embedding
    embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
    if not embedding:
        continue
    # Search for similar faces...
```

**After**:
```python
# Batch retrieve all prototype embeddings
prototype_point_ids = [
    face.qdrant_point_id for face in selected_faces if face.qdrant_point_id
]
prototype_embeddings_map = qdrant.get_embeddings_by_point_ids(prototype_point_ids)

for idx, face in enumerate(selected_faces):
    if not face.qdrant_point_id:
        continue
    embedding = prototype_embeddings_map.get(face.qdrant_point_id)
    if not embedding:
        continue
    # Search for similar faces...
```

### Step 5: Update face_jobs.py -- find_more_multi_prototype_suggestions_job()

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Around line**: 1624-1635

**Before**:
```python
for proto in prototypes:
    face = db_session.get(FaceInstance, proto.face_instance_id)
    if face and face.qdrant_point_id:
        embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
        if embedding:
            prototype_embeddings[str(proto.face_instance_id)] = {
                "embedding": embedding,
                ...
            }
```

**After**:
```python
# Collect all prototype face instances first
proto_faces = {}
for proto in prototypes:
    face = db_session.get(FaceInstance, proto.face_instance_id)
    if face and face.qdrant_point_id:
        proto_faces[proto] = face

# Batch retrieve all embeddings
point_ids = [face.qdrant_point_id for face in proto_faces.values()]
embeddings_map = qdrant.get_embeddings_by_point_ids(point_ids)

# Build prototype embeddings dict
for proto, face in proto_faces.items():
    embedding = embeddings_map.get(face.qdrant_point_id)
    if embedding:
        prototype_embeddings[str(proto.face_instance_id)] = {
            "embedding": embedding,
            "face_id": str(face.id),
            "quality": face.quality_score or 0.0,
        }
```

### Step 6: Update face_jobs.py -- find_more_centroid_suggestions_job()

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Around line**: 1927-1932

**Before**:
```python
for face in faces:
    if face.qdrant_point_id:
        embedding = face_qdrant.get_embedding_by_point_id(face.qdrant_point_id)
        if embedding:
            embeddings.append(embedding)
            face_ids.append(face.id)
```

**After**:
```python
# Batch retrieve all face embeddings
point_ids = [face.qdrant_point_id for face in faces if face.qdrant_point_id]
embeddings_map = face_qdrant.get_embeddings_by_point_ids(point_ids)

for face in faces:
    if face.qdrant_point_id and face.qdrant_point_id in embeddings_map:
        embeddings.append(embeddings_map[face.qdrant_point_id])
        face_ids.append(face.id)
```

Note: If C5 is implemented first, this call site will be replaced by `compute_centroids_for_person_sync()` which itself calls `get_person_face_embeddings()`. In that case, only Step 2 needs this fix. However, applying the fix here ensures it works regardless of C5 implementation order.

### Step 7: Update assigner.py -- Replace _get_face_embedding() with Batch

**File**: `image-search-service/src/image_search_service/faces/assigner.py`

The `_get_face_embedding()` method (lines 258-284) is called in multiple loops. Rather than modifying each loop individually, refactor `assign_new_faces()` to batch-retrieve all embeddings upfront:

**Before** (inside `assign_new_faces()`, around line 126-128):
```python
for face in faces:
    # Get face embedding from Qdrant
    embedding = self._get_face_embedding(face.qdrant_point_id)
    if embedding is None:
        ...
```

**After**:
```python
from image_search_service.vector.face_qdrant import get_face_qdrant_client

qdrant = get_face_qdrant_client()

# Batch retrieve all face embeddings upfront
face_point_ids = [face.qdrant_point_id for face in faces if face.qdrant_point_id]
face_embeddings_map = qdrant.get_embeddings_by_point_ids(face_point_ids)

for face in faces:
    embedding = face_embeddings_map.get(face.qdrant_point_id)
    if embedding is None:
        ...
```

Similarly, refactor `compute_person_centroids()` (line 321-323) to batch-retrieve:

```python
# Before: loop with individual retrieval
for face in faces:
    embedding = self._get_face_embedding(face.qdrant_point_id)

# After: batch retrieval
point_ids = [face.qdrant_point_id for face in faces if face.qdrant_point_id]
embeddings_map = qdrant.get_embeddings_by_point_ids(point_ids)
```

### Step 8: Update dual_clusterer.py -- Replace _get_face_embedding() with Batch

**File**: `image-search-service/src/image_search_service/faces/dual_clusterer.py`

Refactor `assign_to_known_people()` and `cluster_unknown_faces()` to batch-retrieve embeddings:

**Before** (`assign_to_known_people()`, lines 149-152):
```python
for face in labeled_faces:
    embedding = self._get_face_embedding(face["qdrant_point_id"])
    if embedding is not None:
        person_embeddings[face["person_id"]].append(embedding)
```

**After**:
```python
from image_search_service.vector.face_qdrant import get_face_qdrant_client

qdrant = get_face_qdrant_client()

# Batch retrieve all labeled face embeddings
labeled_point_ids = [f["qdrant_point_id"] for f in labeled_faces]
labeled_embeddings_map = qdrant.get_embeddings_by_point_ids(labeled_point_ids)

for face in labeled_faces:
    embedding_list = labeled_embeddings_map.get(face["qdrant_point_id"])
    if embedding_list is not None:
        embedding = np.array(embedding_list)
        person_embeddings[face["person_id"]].append(embedding)
```

Apply the same pattern to:
- The unlabeled face loop in `assign_to_known_people()` (line 172-174)
- The `cluster_unknown_faces()` method (lines 229-233)

After batch conversion, the `_get_face_embedding()` private method (lines 344-366) can be marked as deprecated or removed.

### Step 9: Keep get_embedding_by_point_id() for Backward Compatibility

**File**: `image-search-service/src/image_search_service/vector/face_qdrant.py`

The existing `get_embedding_by_point_id()` method should be kept for cases where only a single embedding is needed (e.g., the API route at `faces.py:1902`). Add a docstring note pointing to the batch method:

```python
def get_embedding_by_point_id(self, point_id: uuid.UUID) -> list[float] | None:
    """Retrieve the embedding vector for a specific face point.

    For retrieving multiple embeddings, use get_embeddings_by_point_ids()
    which batches the request for better performance.

    Args:
        point_id: Face point ID (qdrant_point_id from FaceInstance)

    Returns:
        512-dim embedding vector, or None if not found
    """
    # ... existing implementation unchanged ...
```

---

## 5. Data Repair

No data repair is needed. This is a pure performance optimization that does not change any stored data or computation results. The batch retrieval returns the same embeddings as individual retrieval.

---

## 6. Verification Checklist

### Performance Tests

- [ ] Benchmark: Retrieve 100 embeddings via individual calls vs batch call. Expect >10x speedup.
- [ ] Benchmark: Centroid computation for person with 50 faces. Measure before/after.
- [ ] Benchmark: Dual-mode clustering of 500 faces. Measure before/after.
- [ ] Benchmark: find_more_centroid_suggestions_job for person with 100 faces. Measure before/after.

### Correctness Tests

- [ ] Test `get_embeddings_by_point_ids()` with empty list returns empty dict.
- [ ] Test `get_embeddings_by_point_ids()` with 1 ID returns same result as `get_embedding_by_point_id()`.
- [ ] Test `get_embeddings_by_point_ids()` with 200 IDs (exceeds batch_size=100) processes correctly.
- [ ] Test `get_embeddings_by_point_ids()` with mix of valid and invalid IDs returns only valid ones.
- [ ] Test `get_embeddings_by_point_ids()` graceful fallback when batch retrieve fails.
- [ ] Test that centroid computation produces identical results before and after batch conversion.
- [ ] Test that cluster confidence calculation produces identical results before and after.

### Integration Tests

- [ ] Run `cluster_faces_job` with batch retrieval enabled. Verify clustering results are identical.
- [ ] Run `find_more_centroid_suggestions_job` with batch retrieval. Verify suggestion results are identical.
- [ ] Run `assign_faces_job` with batch retrieval. Verify assignment results are identical.

### Existing Test Compatibility

- [ ] All existing tests in `tests/unit/services/test_face_clustering_service.py` pass (these mock `get_embedding_by_point_id` -- verify mocks are updated for batch method).
- [ ] All existing tests in `tests/api/test_face_suggestions.py` pass.
- [ ] All existing tests in `tests/api/test_clusters_filtering.py` pass.

**Mock Update Required**: Tests that mock `get_embedding_by_point_id` need parallel mocks for `get_embeddings_by_point_ids`. Example:

```python
# Before:
mock_qdrant.get_embedding_by_point_id.side_effect = [embedding1, embedding2]

# After (add both for compatibility):
mock_qdrant.get_embedding_by_point_id.side_effect = [embedding1, embedding2]
mock_qdrant.get_embeddings_by_point_ids.return_value = {
    point_id_1: embedding1,
    point_id_2: embedding2,
}
```

---

## 7. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Batch retrieval returns different order than expected | Low | Low | Method returns a dict, not a list; callers look up by key, not position |
| Qdrant batch request size limits | Low | Low | Batch size of 100 is well within Qdrant's limits (tested up to 10K) |
| Network timeout for large batches | Low | Medium | batch_size=100 limits individual request size; graceful fallback to individual |
| Mock incompatibility in existing tests | Medium | Low | Update mocks incrementally; keep old method working |
| Race condition: embedding deleted between batch request and use | Very Low | Low | Already possible with single retrieval; dict lookup returns None |

### Rollback Plan

1. The batch method is additive -- `get_embedding_by_point_id()` remains unchanged.
2. Revert individual call sites back to the loop pattern by replacing `get_embeddings_by_point_ids()` calls with the original loop.
3. No data changes, so no data rollback needed.

---

## 8. Performance Impact Estimates

| Call Site | Typical N | Before (N calls) | After (N/100 calls) | Speedup |
|-----------|----------|-------------------|---------------------|---------|
| centroid_service.py | 50 faces | 250-750ms | 10-30ms | ~25x |
| face_clustering_service.py | 20 faces | 100-300ms | 10-30ms | ~10x |
| face_jobs.py (prototype) | 10 protos | 50-150ms | 10-30ms | ~5x |
| face_jobs.py (multi-proto) | 5 protos | 25-75ms | 10-30ms | ~3x |
| face_jobs.py (centroid) | 100 faces | 500-1500ms | 10-30ms | ~50x |
| assigner.py | 200 faces | 1000-3000ms | 20-60ms | ~50x |
| dual_clusterer.py | 500 faces | 2500-7500ms | 50-150ms | ~50x |

**Total savings for a full dual-mode clustering of 1000 faces**: ~4-12 seconds of network round-trip time eliminated.

---

## 9. Estimated Effort

| Task | Effort |
|------|--------|
| Step 1: Add `get_embeddings_by_point_ids()` | 0.5 day |
| Steps 2-3: Update centroid_service + clustering_service | 0.5 day |
| Steps 4-6: Update face_jobs.py (3 call sites) | 0.5 day |
| Steps 7-8: Update assigner.py + dual_clusterer.py | 0.5 day |
| Step 9: Documentation + deprecation notes | 0.25 day |
| Testing (unit + integration + performance) | 1 day |
| Code review + QA | 0.5 day |
| **Total** | **~3.75 days** |

---

## 10. Implementation Order Recommendation

These steps can be implemented incrementally with independent PRs:

1. **PR 1 (Foundation)**: Step 1 -- Add `get_embeddings_by_point_ids()` to FaceQdrantClient + unit tests. This is zero-risk since it adds a new method without changing existing code.

2. **PR 2 (Core Services)**: Steps 2-3 -- Update `centroid_service.py` and `face_clustering_service.py`. These are the most impactful and well-tested services.

3. **PR 3 (Background Jobs)**: Steps 4-6 -- Update `face_jobs.py` call sites. These run in the background and can be tested by running jobs.

4. **PR 4 (Clustering)**: Steps 7-8 -- Update `assigner.py` and `dual_clusterer.py`. These are the highest-N call sites but also the least frequently run.

Each PR is independently shippable and provides incremental performance improvement.

---

## 11. File Change Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `image-search-service/.../vector/face_qdrant.py` | Modify | Add `get_embeddings_by_point_ids()` batch method |
| `image-search-service/.../services/centroid_service.py` | Modify | Replace loop in `get_person_face_embeddings()` with batch call |
| `image-search-service/.../services/face_clustering_service.py` | Modify | Replace loop in `calculate_cluster_confidence()` with batch call |
| `image-search-service/.../queue/face_jobs.py` | Modify | Replace 3 loop-based retrieval sites with batch calls |
| `image-search-service/.../faces/assigner.py` | Modify | Replace `_get_face_embedding()` loops with batch retrieval |
| `image-search-service/.../faces/dual_clusterer.py` | Modify | Replace `_get_face_embedding()` loops with batch retrieval |
| `tests/unit/vector/test_face_qdrant.py` | Add/Modify | Unit tests for batch retrieval method |
| `tests/unit/services/test_face_clustering_service.py` | Modify | Update mocks for batch retrieval |
| `tests/api/test_face_suggestions.py` | Modify | Update mocks for batch retrieval |
