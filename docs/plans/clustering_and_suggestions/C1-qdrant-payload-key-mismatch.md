# C1: Fix Qdrant Payload Key Mismatch (`face_id` vs `face_instance_id`)

## References
- **Research**: `docs/research/clustering_and_suggestions/04-devils-advocate-analysis.md` -- Section "Phantom Face IDs -- Qdrant Payload Key Mismatch" (Bomb #2)
- **Research**: `docs/research/clustering_and_suggestions/02-backend-analysis.md` -- Qdrant payload schema documentation
- **Source Files**:
  - `image-search-service/src/image_search_service/vector/face_qdrant.py` (upsert payload definition)
  - `image-search-service/src/image_search_service/queue/face_jobs.py` (three broken extraction sites)

## Problem Statement

Three background job functions extract face IDs from Qdrant search results using `result.payload.get("face_id")`, but the Qdrant `faces` collection stores the field as `face_instance_id`. This key mismatch causes every call to silently return zero results -- no error is thrown, but no suggestions are ever created.

**Evidence**: In `face_qdrant.py`, the `upsert_face` method (line 165) builds the payload at lines 194-196:

```python
payload: dict[str, Any] = {
    "asset_id": str(asset_id),
    "face_instance_id": str(face_instance_id),  # <-- canonical key name
    "detection_confidence": detection_confidence,
    "is_prototype": is_prototype,
}
```

The key stored in Qdrant is `face_instance_id`. A payload index is also created on this key at `face_qdrant.py` line 151-155:

```python
self.client.create_payload_index(
    collection_name=collection_name,
    field_name="face_instance_id",
    field_schema=PayloadSchemaType.KEYWORD,
)
```

However, three job functions read `face_id` (a key that does not exist in any payload):

1. `find_more_suggestions_job` at `face_jobs.py` line 1438
2. `propagate_person_label_multiproto_job` at `face_jobs.py` line 1665
3. `find_more_centroid_suggestions_job` at `face_jobs.py` line 2062

## Root Cause Analysis

### How upsert stores the key

`FaceQdrantClient.upsert_face()` (`face_qdrant.py`, lines 194-196) creates a payload dict with the key `face_instance_id`:

```python
payload: dict[str, Any] = {
    "asset_id": str(asset_id),
    "face_instance_id": str(face_instance_id),  # stored as "face_instance_id"
    ...
}
```

This is consistent with the `FaceInstance.id` field in the PostgreSQL model and with the Qdrant payload index on `face_instance_id`.

### How the jobs read the key (incorrectly)

All three broken jobs extract from the search result payload using `"face_id"`:

```python
face_id_str = result.payload.get("face_id")  # returns None -- key does not exist
if not face_id_str:
    continue  # silently skips every result
```

Since `"face_id"` is never set in the payload, `get("face_id")` always returns `None`, the `if not face_id_str` check is always True, and the loop body (creating suggestions) is never reached.

### Why it appears to work

No exceptions are raised. The jobs complete successfully with `suggestions_created = 0`. The log output reports "Found N candidate faces from M prototypes" (showing that the Qdrant search works), followed by "Created 0 suggestions" (because every result is skipped). Without examining the zero-count closely, this appears like "no matches found" rather than a bug.

### Working reference

The single-prototype propagation job `propagate_person_label_job` at `face_jobs.py` line 1060 does NOT use payload extraction at all. Instead, it uses the `qdrant_point_id` directly to look up the `FaceInstance` in PostgreSQL:

```python
qdrant_point_id = uuid_lib.UUID(str(result.id))
face = db_session.execute(
    select(FaceInstance).where(FaceInstance.qdrant_point_id == qdrant_point_id)
).scalar_one_or_none()
```

This approach is correct and does not depend on payload key names.

## Current Behavior (Before Fix)

1. User triggers "Find More" for a person, or post-training queues centroid/multi-proto suggestion jobs.
2. The job retrieves prototype/centroid embeddings and searches Qdrant successfully -- results are found.
3. For each search result, the job calls `result.payload.get("face_id")` -- returns `None`.
4. The `if not face_id_str: continue` skips every result.
5. The job reports 0 suggestions created.
6. The user sees no new suggestions despite many unlabeled faces being similar to their person.

## Expected Behavior (After Fix)

1. Same trigger as above.
2. Qdrant search returns results.
3. For each result, the job calls `result.payload.get("face_instance_id")` -- returns the correct UUID string.
4. The UUID is parsed and used to look up the `FaceInstance` in PostgreSQL.
5. Suggestions are created for qualifying candidates.
6. The user sees new suggestions in the UI.

## Implementation Plan

### Canonical key: `face_instance_id`

The canonical key is `face_instance_id` because:
- It matches the Qdrant payload defined in `upsert_face()` (line 196)
- It matches the Qdrant payload index (line 151-155)
- It matches the PostgreSQL column name `FaceInstance.id` (which is the face instance ID)
- It is used correctly in other parts of the codebase (`face_qdrant.py:621`, `clusterer.py:101`, `clusterer.py:255`, `face_centroids.py:371`)

We should change the **reads** (job functions), NOT the write (upsert payload).

### Step 1: Fix `find_more_suggestions_job`

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`

**Current code** (line 1438):
```python
face_id_str = result.payload.get("face_id")
```

**Change to**:
```python
face_id_str = result.payload.get("face_instance_id")
```

**Why**: This is the first of three identical mismatches. The Qdrant payload stores this field as `face_instance_id`.

### Step 2: Fix `propagate_person_label_multiproto_job`

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`

**Current code** (line 1665):
```python
face_id_str = result.payload.get("face_id")
```

**Change to**:
```python
face_id_str = result.payload.get("face_instance_id")
```

**Why**: Same mismatch in the multi-prototype propagation path.

### Step 3: Fix `find_more_centroid_suggestions_job`

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`

**Current code** (line 2062):
```python
face_id_str = result.payload.get("face_id")
```

**Change to**:
```python
face_id_str = result.payload.get("face_instance_id")
```

**Why**: Same mismatch in the centroid-based suggestion path.

### Step 4: Audit for additional occurrences

Run the following commands to find ALL references to the wrong key and confirm no other occurrences exist:

```bash
# Find all uses of the incorrect key "face_id" in payload access patterns
grep -rn 'payload.get("face_id")' image-search-service/src/
grep -rn "payload.get('face_id')" image-search-service/src/
grep -rn 'payload\[.face_id.\]' image-search-service/src/

# Confirm all uses of the correct key
grep -rn 'payload.get("face_instance_id")' image-search-service/src/
grep -rn "payload.get('face_instance_id')" image-search-service/src/

# Verify no other files reference the wrong key
grep -rn '"face_id"' image-search-service/src/image_search_service/queue/face_jobs.py
```

Expected result after fix: Zero matches for `payload.get("face_id")` and three or more matches for `payload.get("face_instance_id")`.

## Data Repair

No data repair is needed. The Qdrant data is correct -- the payload has always contained `face_instance_id`. The bug is purely in the read path. Once the code is fixed, all existing Qdrant data will be read correctly.

However, users may have missed suggestions that should have been generated. After deploying the fix, consider:
- Re-running post-training suggestion generation for active persons
- Notifying users that "Find More" should now produce results

## Verification

### Manual Verification

1. **Before deploying**, verify the current broken state:
   - Trigger "Find More" for a person with known unlabeled similar faces.
   - Observe in the worker logs: "Created 0 suggestions" despite search results.

2. **After deploying**, verify the fix:
   - Trigger "Find More" for the same person.
   - Observe in the worker logs: "Created N suggestions" where N > 0.
   - Check the suggestions UI -- new suggestions should appear.

3. **Verify centroid path**:
   - Ensure `post_training_use_centroids` is enabled in system config.
   - Run a training session.
   - After completion, check worker logs for `find_more_centroid_suggestions_job` -- should report suggestions created > 0.

4. **Verify multi-prototype path**:
   - Trigger `propagate_person_label_multiproto_job` (either via post-training or directly).
   - Check worker logs for suggestions created > 0.

### Automated Tests

Add unit tests that verify payload key extraction:

```python
# tests/unit/test_face_jobs_payload.py

import uuid
from unittest.mock import MagicMock

def test_find_more_suggestions_extracts_face_instance_id():
    """Verify find_more_suggestions_job reads 'face_instance_id' from Qdrant payload."""
    face_id = uuid.uuid4()
    mock_result = MagicMock()
    mock_result.payload = {
        "face_instance_id": str(face_id),
        "asset_id": str(uuid.uuid4()),
        "detection_confidence": 0.95,
    }

    # The extraction should use "face_instance_id", not "face_id"
    face_id_str = mock_result.payload.get("face_instance_id")
    assert face_id_str == str(face_id)

    # The old incorrect key should return None
    wrong_key = mock_result.payload.get("face_id")
    assert wrong_key is None


def test_payload_key_consistency_with_upsert():
    """Verify that the key used in reads matches the key used in upsert_face."""
    # This test documents the canonical key name
    CANONICAL_KEY = "face_instance_id"

    # Simulate the payload created by FaceQdrantClient.upsert_face()
    face_instance_id = uuid.uuid4()
    upsert_payload = {
        "asset_id": str(uuid.uuid4()),
        "face_instance_id": str(face_instance_id),
        "detection_confidence": 0.95,
        "is_prototype": False,
    }

    # Reads must use the same key
    extracted = upsert_payload.get(CANONICAL_KEY)
    assert extracted == str(face_instance_id)
```

## Risk Assessment

- **Risk**: Extremely low. This is a simple string key rename in three locations, changing a non-functional path to a functional one.
- **Mitigation**: The change cannot break existing behavior because the current behavior already produces zero results. After the fix, it will produce correct results.
- **Risk**: Downstream code might not handle the now-functional suggestion creation correctly.
- **Mitigation**: The suggestion creation code (below the extraction line in each function) is well-tested via the single-prototype path (`propagate_person_label_job`) which already works correctly. The suggestion creation logic (checking for duplicates, creating `FaceSuggestion` records) is shared and known to work.
- **Risk**: Sudden surge of suggestions after deploying the fix (if post-training jobs are re-run).
- **Mitigation**: The `max_suggestions` parameter already limits the number of suggestions per job. Monitor suggestion counts after deployment.

## Estimated Effort

**Time**: 15-30 minutes (3 one-line changes + testing + audit grep)

**Complexity**: Trivial -- three identical one-word changes.

**Risk level**: Very low -- changes a broken code path to a working one. No existing functionality is altered.
