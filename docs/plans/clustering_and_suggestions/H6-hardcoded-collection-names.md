# Plan 9: Fix Hardcoded Qdrant Collection Names

**Issue ID**: H6 (High Severity)
**Priority**: High
**Estimated Effort**: 2-3 hours
**Risk Level**: Low (mechanical refactor with existing pattern)
**Date**: 2026-02-06

---

## Problem Statement

Nine locations across six Python files bypass the configurable `_get_face_collection_name()` function and instead hardcode the Qdrant collection name as the string literal `"faces"`. This defeats the environment-based configuration system that already exists, meaning tests that override `QDRANT_FACE_COLLECTION` to use a test collection (e.g., `"test_faces"`) will still have these code paths hitting the production `"faces"` collection. In a shared Qdrant instance, this creates a real risk of production data corruption during testing.

## Root Cause Analysis

The `_get_face_collection_name()` function was introduced in `face_qdrant.py` as the canonical way to resolve the collection name from settings. The `FaceQdrantClient` class uses it consistently across 20+ call sites. However, the clustering/training services (`clusterer.py`, `dual_clusterer.py`, `trainer.py`, `assigner.py`) and the queue jobs (`face_jobs.py`) bypass `FaceQdrantClient` entirely and use the raw `qdrant_client` directly with hardcoded `collection_name="faces"`. The bootstrap script (`bootstrap_qdrant.py`) also hardcodes the name, though its role as infrastructure setup makes this somewhat more justifiable.

The pattern appears to have emerged because the clustering and training services were written to use the low-level `qdrant.client` (the raw `QdrantClient`) directly rather than going through `FaceQdrantClient` methods. Each service has its own `_get_face_embedding()` helper that duplicates the same retrieval logic but hardcodes the collection name.

## Current Behavior

When `QDRANT_FACE_COLLECTION=test_faces` is set in the environment:

- `FaceQdrantClient.upsert_face()` writes to `"test_faces"` (correct)
- `FaceQdrantClient.search_similar()` reads from `"test_faces"` (correct)
- `clusterer.py` scrolls from `"faces"` (WRONG -- reads production data)
- `trainer.py` retrieves from `"faces"` (WRONG -- reads production data)
- `assigner.py` retrieves and sets payload on `"faces"` (WRONG -- mutates production data)
- `dual_clusterer.py` retrieves from `"faces"` (WRONG -- reads production data)
- `face_jobs.py` retrieves and queries `"faces"` (WRONG -- reads production data)
- `bootstrap_qdrant.py` creates/verifies `"faces"` (partially wrong)

## Expected Behavior

All Qdrant operations should resolve the collection name through the same configuration path: `_get_face_collection_name()` from `image_search_service.vector.face_qdrant`, which reads `get_settings().qdrant_face_collection`. When `QDRANT_FACE_COLLECTION=test_faces`, every code path should use `"test_faces"`.

---

## Complete Inventory of Hardcoded Occurrences

### Occurrence 1: `faces/clusterer.py` line 88

**File**: `image-search-service/src/image_search_service/faces/clusterer.py`
**Function**: `_fetch_face_embeddings()` (called during HDBSCAN clustering)
**Operation**: `qdrant.client.scroll()`

```python
# CURRENT (line 87-88):
records, next_offset = qdrant.client.scroll(
    collection_name="faces",
    scroll_filter=scroll_filter,
    limit=min(1000, max_faces - len(face_ids)),
    offset=offset,
    with_vectors=True,
    with_payload=True,
)
```

**Fix**:
```python
# AFTER:
from image_search_service.vector.face_qdrant import _get_face_collection_name

records, next_offset = qdrant.client.scroll(
    collection_name=_get_face_collection_name(),
    scroll_filter=scroll_filter,
    limit=min(1000, max_faces - len(face_ids)),
    offset=offset,
    with_vectors=True,
    with_payload=True,
)
```

---

### Occurrence 2: `faces/assigner.py` line 274

**File**: `image-search-service/src/image_search_service/faces/assigner.py`
**Function**: `_get_face_embedding()` (retrieves embedding for centroid computation)
**Operation**: `qdrant.client.retrieve()`

```python
# CURRENT (line 273-274):
points = qdrant.client.retrieve(
    collection_name="faces",
    ids=[str(qdrant_point_id)],
    with_vectors=True,
)
```

**Fix**:
```python
# AFTER:
from image_search_service.vector.face_qdrant import _get_face_collection_name

points = qdrant.client.retrieve(
    collection_name=_get_face_collection_name(),
    ids=[str(qdrant_point_id)],
    with_vectors=True,
)
```

---

### Occurrence 3: `faces/assigner.py` line 344

**File**: `image-search-service/src/image_search_service/faces/assigner.py`
**Function**: `compute_person_centroids()` (updates centroid payload in Qdrant)
**Operation**: `qdrant.client.set_payload()`

```python
# CURRENT (line 343-344):
qdrant.client.set_payload(
    collection_name="faces",
    payload={"is_centroid": True},
    points=[str(existing_centroid.qdrant_point_id)],
)
```

**Fix**:
```python
# AFTER:
qdrant.client.set_payload(
    collection_name=_get_face_collection_name(),
    payload={"is_centroid": True},
    points=[str(existing_centroid.qdrant_point_id)],
)
```

Note: The import for `_get_face_collection_name` is already present for Occurrence 2 if done in the same file. Only one import is needed at the top of `_get_face_embedding()` or at the module level.

---

### Occurrence 4: `faces/dual_clusterer.py` line 352

**File**: `image-search-service/src/image_search_service/faces/dual_clusterer.py`
**Function**: `_get_face_embedding()` (retrieves embedding for dual-mode clustering)
**Operation**: `qdrant.client.retrieve()`

```python
# CURRENT (line 351-352):
points = qdrant.client.retrieve(
    collection_name="faces",
    ids=[str(qdrant_point_id)],
    with_vectors=True,
)
```

**Fix**:
```python
# AFTER:
from image_search_service.vector.face_qdrant import _get_face_collection_name

points = qdrant.client.retrieve(
    collection_name=_get_face_collection_name(),
    ids=[str(qdrant_point_id)],
    with_vectors=True,
)
```

---

### Occurrence 5: `faces/trainer.py` line 416

**File**: `image-search-service/src/image_search_service/faces/trainer.py`
**Function**: `_get_face_embedding()` (retrieves embedding for training)
**Operation**: `qdrant.client.retrieve()`

```python
# CURRENT (line 415-416):
points = qdrant.client.retrieve(
    collection_name="faces",
    ids=[str(qdrant_point_id)],
    with_vectors=True,
)
```

**Fix**:
```python
# AFTER:
from image_search_service.vector.face_qdrant import _get_face_collection_name

points = qdrant.client.retrieve(
    collection_name=_get_face_collection_name(),
    ids=[str(qdrant_point_id)],
    with_vectors=True,
)
```

---

### Occurrence 6: `queue/face_jobs.py` line 1027

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Function**: `find_more_suggestions_job()` (retrieves source face vector for suggestion generation)
**Operation**: `face_client.client.retrieve()`

```python
# CURRENT (line 1026-1027):
points = face_client.client.retrieve(
    collection_name="faces",
    ids=[str(source_face.qdrant_point_id)],
    with_vectors=True,
)
```

**Fix**:
```python
# AFTER:
from image_search_service.vector.face_qdrant import _get_face_collection_name

points = face_client.client.retrieve(
    collection_name=_get_face_collection_name(),
    ids=[str(source_face.qdrant_point_id)],
    with_vectors=True,
)
```

---

### Occurrence 7: `queue/face_jobs.py` line 1048

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Function**: `find_more_suggestions_job()` (searches for similar unassigned faces)
**Operation**: `face_client.client.query_points()`

```python
# CURRENT (line 1047-1048):
search_results = face_client.client.query_points(
    collection_name="faces",
    query=source_vector,
    limit=max_suggestions + 10,
    score_threshold=min_confidence,
    with_payload=True,
)
```

**Fix**:
```python
# AFTER (import already present from Occurrence 6):
search_results = face_client.client.query_points(
    collection_name=_get_face_collection_name(),
    query=source_vector,
    limit=max_suggestions + 10,
    score_threshold=min_confidence,
    with_payload=True,
)
```

---

### Occurrence 8: `scripts/bootstrap_qdrant.py` line 68

**File**: `image-search-service/src/image_search_service/scripts/bootstrap_qdrant.py`
**Function**: `ensure_faces_collection()` (creates the faces collection if it does not exist)
**Operation**: Variable assignment used in `client.create_collection()` and index creation

```python
# CURRENT (line 68-69):
collection_name = "faces"
face_vector_dim = 512  # Hardcoded from face_qdrant.py
```

**Fix**:
```python
# AFTER:
from image_search_service.vector.face_qdrant import _get_face_collection_name

collection_name = _get_face_collection_name()
face_vector_dim = 512  # ArcFace/InsightFace embedding dimension
```

---

### Occurrence 9: `scripts/bootstrap_qdrant.py` line 224

**File**: `image-search-service/src/image_search_service/scripts/bootstrap_qdrant.py`
**Function**: `verify()` (CLI command that validates collection configuration)
**Operation**: Variable assignment used to verify collection exists and has correct parameters

```python
# CURRENT (line 224-225):
faces_collection_name = "faces"
expected_face_dim = 512
```

**Fix**:
```python
# AFTER (import already present from Occurrence 8):
faces_collection_name = _get_face_collection_name()
expected_face_dim = 512
```

---

## Implementation Steps

### Step 1: Fix the six service/job files (30 minutes)

Apply the changes documented above to each file. The import pattern is consistent:

```python
from image_search_service.vector.face_qdrant import _get_face_collection_name
```

**Files to modify** (in order):
1. `faces/clusterer.py` -- 1 occurrence (line 88)
2. `faces/assigner.py` -- 2 occurrences (lines 274, 344)
3. `faces/dual_clusterer.py` -- 1 occurrence (line 352)
4. `faces/trainer.py` -- 1 occurrence (line 416)
5. `queue/face_jobs.py` -- 2 occurrences (lines 1027, 1048)
6. `scripts/bootstrap_qdrant.py` -- 2 occurrences (lines 68, 224)

### Step 2: Consider refactoring duplicate `_get_face_embedding()` methods (optional, 30 minutes)

Three files contain nearly identical `_get_face_embedding()` methods:
- `faces/assigner.py` lines 260-284
- `faces/dual_clusterer.py` lines 344-366
- `faces/trainer.py` lines 401-429

All follow the same pattern: get `FaceQdrantClient`, call `qdrant.client.retrieve()` with hardcoded collection name, handle dict/list vector format. Consider extracting this into a shared utility on `FaceQdrantClient`:

```python
# In face_qdrant.py, add method to FaceQdrantClient:
def get_embedding(self, qdrant_point_id: uuid.UUID) -> list[float] | None:
    """Get face embedding vector by Qdrant point ID."""
    try:
        points = self.client.retrieve(
            collection_name=_get_face_collection_name(),
            ids=[str(qdrant_point_id)],
            with_vectors=True,
        )
        if points and points[0].vector:
            vector = points[0].vector
            if isinstance(vector, dict):
                return list(vector.values())[0]
            return vector
        return None
    except Exception as e:
        logger.error(f"Error retrieving embedding for {qdrant_point_id}: {e}")
        return None
```

This is optional but would eliminate the root cause of future hardcoding -- developers copy-pasting code that bypasses the configuration.

### Step 3: Add a ruff linting rule (optional, 15 minutes)

Create a custom ruff rule or a grep-based CI check to catch future regressions:

```bash
# Add to CI pipeline or Makefile:
check-hardcoded-collections:
    @echo "Checking for hardcoded Qdrant collection names..."
    @if grep -rn 'collection_name="faces"' \
        --include="*.py" \
        src/image_search_service/ \
        | grep -v 'face_qdrant.py' \
        | grep -v '_get_face_collection_name'; then \
        echo "ERROR: Found hardcoded collection_name='faces' outside face_qdrant.py"; \
        exit 1; \
    fi
    @echo "OK: No hardcoded collection names found"
```

Add this as a Makefile target and call it from `make check`.

### Step 4: Verify with grep (5 minutes)

After applying all fixes, run:

```bash
grep -rn 'collection_name="faces"' \
    --include="*.py" \
    src/image_search_service/

# Expected output: ZERO results
# (face_qdrant.py uses _get_face_collection_name() already)

grep -rn 'collection_name = "faces"' \
    --include="*.py" \
    src/image_search_service/

# Expected output: ZERO results
```

### Step 5: Run existing tests (10 minutes)

```bash
cd image-search-service
make test
make typecheck
make format
```

All existing tests should pass without modification since `_get_face_collection_name()` returns `"faces"` by default when `QDRANT_FACE_COLLECTION` is not set.

---

## Verification

### Manual Verification

1. Set `QDRANT_FACE_COLLECTION=test_faces` in your environment
2. Run the face clustering pipeline
3. Verify all Qdrant operations target `test_faces` collection (check Qdrant dashboard or logs)
4. Confirm no operations hit the `faces` collection

### Automated Verification

```bash
# Grep verification (should return empty):
grep -rn 'collection_name="faces"\|collection_name = "faces"\|faces_collection_name = "faces"' \
    --include="*.py" \
    src/image_search_service/

# Test suite:
make test

# Type checking (ensures imports are valid):
make typecheck
```

### Test Enhancement (optional)

Add a test that validates all collection name usage goes through the config:

```python
# tests/unit/test_collection_name_config.py
import ast
import pathlib

def test_no_hardcoded_faces_collection_name():
    """Ensure no Python file hardcodes collection_name='faces'."""
    src_dir = pathlib.Path("src/image_search_service")
    violations = []

    for py_file in src_dir.rglob("*.py"):
        if py_file.name == "face_qdrant.py":
            continue  # This file defines the canonical function

        content = py_file.read_text()
        if 'collection_name="faces"' in content or "collection_name='faces'" in content:
            violations.append(str(py_file))

    assert not violations, (
        f"Hardcoded collection_name='faces' found in: {violations}. "
        "Use _get_face_collection_name() from face_qdrant.py instead."
    )
```

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Import cycle from `_get_face_collection_name` | Low | Medium | The import is already used by `centroid_qdrant.py` without issues; lazy imports inside functions can be used if needed |
| `get_settings()` fails at import time | Low | Medium | `_get_face_collection_name()` is called at runtime, not import time; settings use lazy initialization |
| Breaks existing behavior | Very Low | Low | Default value is `"faces"` so behavior is unchanged unless env var is explicitly set |
| Missed occurrence | Low | Medium | Grep verification step catches any missed sites |

## Dependencies

- None. This is a self-contained refactor that does not change any API contracts, database schemas, or UI code.

## Files Modified (Summary)

| File | Line(s) | Change |
|------|---------|--------|
| `faces/clusterer.py` | 88 | Replace `"faces"` with `_get_face_collection_name()` |
| `faces/assigner.py` | 274, 344 | Replace `"faces"` with `_get_face_collection_name()` |
| `faces/dual_clusterer.py` | 352 | Replace `"faces"` with `_get_face_collection_name()` |
| `faces/trainer.py` | 416 | Replace `"faces"` with `_get_face_collection_name()` |
| `queue/face_jobs.py` | 1027, 1048 | Replace `"faces"` with `_get_face_collection_name()` |
| `scripts/bootstrap_qdrant.py` | 68, 224 | Replace `"faces"` with `_get_face_collection_name()` |

**Total**: 6 files, 9 string literal replacements, 6 new import lines.

## Reference: Correct Pattern (Already in Use)

The correct pattern already exists in `face_qdrant.py` (used 20+ times):

```python
# image-search-service/src/image_search_service/vector/face_qdrant.py

def _get_face_collection_name() -> str:
    """Get face collection name from settings (allows test override)."""
    return get_settings().qdrant_face_collection

# Usage throughout FaceQdrantClient:
collection_name = _get_face_collection_name()
```

The configuration lives in `core/config.py` line 29:

```python
qdrant_face_collection: str = Field(default="faces", alias="QDRANT_FACE_COLLECTION")
```

One additional correct usage exists in `face_detection_restart_service.py` line 413:

```python
collection_name = get_settings().qdrant_face_collection
```

This file accesses the setting directly rather than through the helper function, which is also acceptable but less consistent.
