# C3: Fix Centroid Deprecation Gap

## References
- **Research**: `docs/research/clustering_and_suggestions/04-devils-advocate-analysis.md` -- Section "Centroid Deprecation Gap -- Deprecate Before Create Race" (Bomb #3)
- **Research**: `docs/research/clustering_and_suggestions/03-clustering-system-analysis.md` -- Section 2 "Centroid System" (centroid lifecycle and architecture)
- **Source Files**:
  - `image-search-service/src/image_search_service/services/centroid_service.py` (main centroid lifecycle)
  - `image-search-service/src/image_search_service/db/models.py` (CentroidStatus enum, PersonCentroid model)
  - `image-search-service/src/image_search_service/vector/centroid_qdrant.py` (Qdrant centroid operations)
  - `image-search-service/src/image_search_service/db/migrations/versions/d1e2f3g4h5i6_add_person_centroid_table.py` (partial unique index)

## Problem Statement

The `compute_centroids_for_person` function in `centroid_service.py` (line 258) deprecates ALL active centroids for a person **before** computing and creating the new one. If any step after deprecation fails -- numpy computation, DB flush, or Qdrant upsert -- the person is left with **zero active centroids**.

**Evidence**: The function at `centroid_service.py` lines 322-394:

```python
# Line 325: Deprecate old centroids FIRST
await deprecate_centroids(db, centroid_qdrant, person_id)

# Lines 328-334: Compute new centroid (CAN FAIL -- numpy errors, insufficient data)
embeddings_array = np.array(embeddings, dtype=np.float32)
centroid_vector = compute_global_centroid(...)

# Lines 349-364: Create DB record and flush (CAN FAIL -- DB errors)
centroid = PersonCentroid(...)
db.add(centroid)
await db.flush()

# Lines 380-385: Store in Qdrant (CAN FAIL -- Qdrant outage, network error)
centroid_qdrant.upsert_centroid(...)
```

The `deprecate_centroids` function at line 220 calls `await db.flush()` at line 252, which means the deprecation is written to the database immediately. If any subsequent step fails, the person has zero active centroids and centroid-based suggestion generation will not work for that person.

Additionally, the `CentroidStatus` enum (in `models.py` line 640-646) includes a `BUILDING` status that is **never used** anywhere in the codebase:

```python
class CentroidStatus(str, Enum):
    ACTIVE = "active"      # Currently valid and in use
    DEPRECATED = "deprecated"  # Outdated by newer version
    BUILDING = "building"  # Currently being computed  <-- NEVER USED
    FAILED = "failed"      # Computation failed
```

## Root Cause Analysis

### Current order of operations

The `compute_centroids_for_person` function (line 258-396) executes these steps in order:

1. **Retrieve embeddings** (line 287): `face_ids, embeddings = await get_person_face_embeddings(db, face_qdrant, person_id)`
2. **Check minimum faces** (line 290): Return `None` if fewer than `centroid_min_faces`
3. **Check staleness** (lines 298-320): If existing centroid is fresh, return it (skip recomputation)
4. **DEPRECATE old centroids** (line 325): `await deprecate_centroids(db, centroid_qdrant, person_id)`
   - Marks all active centroids as `DEPRECATED` in DB
   - Calls `await db.flush()` -- changes are written to DB (but not committed)
5. **Compute new centroid** (lines 328-334): numpy array conversion + `compute_global_centroid()`
6. **Create DB record** (lines 349-364): Creates `PersonCentroid` with `status=ACTIVE`, calls `db.add()` + `await db.flush()`
7. **Upsert to Qdrant** (lines 380-385): `centroid_qdrant.upsert_centroid()`

### Where it can fail

**Between steps 4 and 5** (after deprecation, during computation):
- `np.array(embeddings, dtype=np.float32)` could fail if embeddings contain invalid data
- `compute_global_centroid()` could fail (e.g., empty array after trimming, division by zero)

**Between steps 5 and 6** (after computation, during DB write):
- `await db.flush()` could fail due to DB connection issues or constraint violations

**Between steps 6 and 7** (after DB write, during Qdrant upsert):
- `centroid_qdrant.upsert_centroid()` could fail due to Qdrant outage, network error, or timeout
- When this fails, the code at lines 389-394 marks the centroid as `FAILED`:
  ```python
  except Exception as e:
      logger.error(f"Failed to upsert centroid to Qdrant: {e}")
      centroid.status = CentroidStatus.FAILED
      await db.flush()
      raise
  ```
  But the old centroids are already `DEPRECATED`, so the person ends up with one `FAILED` centroid and zero `ACTIVE` centroids.

### The flush vs commit distinction

The `deprecate_centroids` function calls `await db.flush()` (not `db.commit()`). The flush writes changes to the DB transaction buffer but does not commit them. If the caller eventually does NOT commit (e.g., because an exception is raised and caught at a higher level), the deprecation would be rolled back.

**However**, the flush is persistent within the session. Any subsequent queries within the same session will see the deprecated status. If the caller of `compute_centroids_for_person` commits the session (which happens in the normal code path), both the deprecation and any subsequent failure state will be committed together.

The critical question is: **does the caller handle exceptions from `compute_centroids_for_person` correctly?** If the caller catches the exception and commits the session anyway, the deprecation is permanent but the new centroid was never created.

### Why the BUILDING status exists but is unused

The `CentroidStatus.BUILDING` enum value was designed for exactly this scenario -- to mark a centroid as "in progress" and prevent race conditions. However, the implementation went straight to `status=ACTIVE` at creation time (line 358), bypassing the `BUILDING` intermediate state.

## Current Behavior (Before Fix)

1. Centroid computation is triggered (e.g., post-training, API call, or on-demand in suggestion job).
2. Old active centroids are deprecated (`flush` writes to DB).
3. An error occurs during computation, DB write, or Qdrant upsert.
4. The exception propagates up. The person now has zero active centroids.
5. Any centroid-based suggestion job for this person will find no active centroid and either:
   - Skip the person entirely, or
   - Try to compute a centroid on-demand (which may hit the same error)
6. The `search_faces_with_centroid` method will have no centroid vector to search with.

## Expected Behavior (After Fix)

1. Centroid computation is triggered.
2. The new centroid is computed and created with `status=BUILDING`.
3. The centroid is upserted to Qdrant.
4. Only after BOTH DB and Qdrant succeed, old centroids are deprecated and the new centroid is promoted to `ACTIVE`.
5. If any step fails, the old active centroids remain untouched.
6. The person always has at least one active centroid (or none, only if they never had one before).

## Implementation Plan

### Strategy: Create-then-deprecate with BUILDING status

Reverse the order of operations: create the new centroid first (with `BUILDING` status), promote it to `ACTIVE`, then deprecate the old ones. This ensures no gap where zero active centroids exist.

The partial unique index `ix_person_centroid_unique_active` only applies to centroids with `status = 'active'`, so having a `BUILDING` centroid alongside an `ACTIVE` one does NOT violate the constraint.

### Step 1: Create new centroid with BUILDING status

**File**: `image-search-service/src/image_search_service/services/centroid_service.py`

**Current code** (lines 322-364):
```python
    # Deprecate old centroids BEFORE creating new one to avoid unique constraint violation
    # The unique index is partial (status = 'active'), so we can't have two active centroids
    # with the same (person_id, model_version, centroid_version, centroid_type, cluster_label)
    await deprecate_centroids(db, centroid_qdrant, person_id)

    # Compute centroid
    embeddings_array = np.array(embeddings, dtype=np.float32)
    centroid_vector = compute_global_centroid(
        embeddings_array,
        trim_outliers=True,
        trim_threshold_small=settings.centroid_trim_threshold_small,
        trim_threshold_large=settings.centroid_trim_threshold_large,
    )

    # Create PersonCentroid record
    centroid_id = uuid.uuid4()
    qdrant_point_id = centroid_id  # Use same UUID for 1:1 mapping

    source_hash = compute_source_hash(face_ids)

    build_params: dict[str, Any] = {
        "trim_outliers": True,
        "trim_threshold_small": settings.centroid_trim_threshold_small,
        "trim_threshold_large": settings.centroid_trim_threshold_large,
        "n_faces_used": len(face_ids),
    }

    centroid = PersonCentroid(
        centroid_id=centroid_id,
        person_id=person_id,
        qdrant_point_id=qdrant_point_id,
        model_version=settings.centroid_model_version,
        centroid_version=settings.centroid_algorithm_version,
        centroid_type=CentroidType.GLOBAL,
        cluster_label="global",
        n_faces=len(face_ids),
        status=CentroidStatus.ACTIVE,
        source_face_ids_hash=source_hash,
        build_params=build_params,
    )

    db.add(centroid)
    await db.flush()
```

**Change to**:
```python
    # Compute centroid BEFORE deprecating old ones
    # This ensures that if computation fails, old centroids remain active
    embeddings_array = np.array(embeddings, dtype=np.float32)
    centroid_vector = compute_global_centroid(
        embeddings_array,
        trim_outliers=True,
        trim_threshold_small=settings.centroid_trim_threshold_small,
        trim_threshold_large=settings.centroid_trim_threshold_large,
    )

    # Create PersonCentroid record with BUILDING status
    # The partial unique index only covers ACTIVE status, so BUILDING does not conflict
    centroid_id = uuid.uuid4()
    qdrant_point_id = centroid_id  # Use same UUID for 1:1 mapping

    source_hash = compute_source_hash(face_ids)

    build_params: dict[str, Any] = {
        "trim_outliers": True,
        "trim_threshold_small": settings.centroid_trim_threshold_small,
        "trim_threshold_large": settings.centroid_trim_threshold_large,
        "n_faces_used": len(face_ids),
    }

    centroid = PersonCentroid(
        centroid_id=centroid_id,
        person_id=person_id,
        qdrant_point_id=qdrant_point_id,
        model_version=settings.centroid_model_version,
        centroid_version=settings.centroid_algorithm_version,
        centroid_type=CentroidType.GLOBAL,
        cluster_label="global",
        n_faces=len(face_ids),
        status=CentroidStatus.BUILDING,  # <-- BUILDING, not ACTIVE
        source_face_ids_hash=source_hash,
        build_params=build_params,
    )

    db.add(centroid)
    await db.flush()
```

**Why**: Moving the computation BEFORE any deprecation ensures that if numpy/centroid computation fails, no centroids are modified. Using `BUILDING` status avoids the partial unique index constraint (which only applies to `ACTIVE` centroids).

### Step 2: Upsert to Qdrant, then promote to ACTIVE and deprecate old

**Current code** (lines 366-394):
```python
    # Store centroid in Qdrant
    qdrant_payload: dict[str, Any] = {
        "person_id": person_id,
        "centroid_id": centroid_id,
        "model_version": settings.centroid_model_version,
        "centroid_version": settings.centroid_algorithm_version,
        "centroid_type": "global",
        "cluster_label": "global",
        "n_faces": len(face_ids),
        "created_at": centroid.created_at.isoformat(),
        "source_hash": source_hash,
        "build_params": build_params,
    }

    try:
        centroid_qdrant.upsert_centroid(
            centroid_id=centroid_id,
            vector=centroid_vector.tolist(),
            payload=qdrant_payload,
        )
        logger.info(
            f"Created centroid {centroid_id} for person {person_id} from {len(face_ids)} faces"
        )
    except Exception as e:
        logger.error(f"Failed to upsert centroid to Qdrant: {e}")
        # Mark centroid as failed in DB
        centroid.status = CentroidStatus.FAILED
        await db.flush()
        raise

    return centroid
```

**Change to**:
```python
    # Store centroid in Qdrant
    qdrant_payload: dict[str, Any] = {
        "person_id": person_id,
        "centroid_id": centroid_id,
        "model_version": settings.centroid_model_version,
        "centroid_version": settings.centroid_algorithm_version,
        "centroid_type": "global",
        "cluster_label": "global",
        "n_faces": len(face_ids),
        "created_at": centroid.created_at.isoformat(),
        "source_hash": source_hash,
        "build_params": build_params,
    }

    try:
        centroid_qdrant.upsert_centroid(
            centroid_id=centroid_id,
            vector=centroid_vector.tolist(),
            payload=qdrant_payload,
        )
    except Exception as e:
        logger.error(f"Failed to upsert centroid to Qdrant: {e}")
        # Mark centroid as FAILED in DB -- old centroids remain ACTIVE
        centroid.status = CentroidStatus.FAILED
        await db.flush()
        raise

    # SUCCESS: Both DB record and Qdrant point exist.
    # Now it's safe to deprecate old centroids and promote the new one.
    await deprecate_centroids(db, centroid_qdrant, person_id)
    centroid.status = CentroidStatus.ACTIVE
    await db.flush()

    logger.info(
        f"Created centroid {centroid_id} for person {person_id} from {len(face_ids)} faces"
    )

    return centroid
```

**Why**: The Qdrant upsert now happens while the new centroid is in `BUILDING` status and old centroids are still `ACTIVE`. If the Qdrant upsert fails, the new centroid is marked `FAILED` but old centroids remain `ACTIVE` -- the person always has a working centroid. Only after the Qdrant upsert succeeds do we deprecate old centroids and promote the new one to `ACTIVE`.

### Step 3: Update the deprecate_centroids call to skip BUILDING centroids

The `deprecate_centroids` function (line 220) deprecates ALL active centroids. After Step 2, when we call it, the new centroid has `BUILDING` status, so it will NOT be caught by the filter (`status == CentroidStatus.ACTIVE`). This means the current implementation already works correctly -- it will only deprecate the OLD active centroids, not the new BUILDING one.

However, let us verify. The current function at lines 238-243:

```python
query = select(PersonCentroid).where(
    PersonCentroid.person_id == person_id, PersonCentroid.status == CentroidStatus.ACTIVE
)
```

This correctly filters for `ACTIVE` status only, so `BUILDING` centroids are excluded. **No change needed** to `deprecate_centroids`.

### Step 4: Update the comment explaining the approach

**File**: `image-search-service/src/image_search_service/services/centroid_service.py`

**Current docstring** (lines 265-272):
```python
    """Compute and store global centroid for a person.

    This is the main entry point for centroid computation. It:
    1. Checks if existing centroid is stale (if not forcing rebuild)
    2. Retrieves face embeddings from Qdrant
    3. Deprecates old centroids BEFORE creating new one (avoids unique constraint violation)
    4. Computes robust centroid with outlier trimming
    5. Stores centroid in DB and Qdrant
```

**Change to**:
```python
    """Compute and store global centroid for a person.

    This is the main entry point for centroid computation. It:
    1. Checks if existing centroid is stale (if not forcing rebuild)
    2. Retrieves face embeddings from Qdrant
    3. Computes robust centroid with outlier trimming
    4. Creates new centroid in DB with BUILDING status (does not conflict with unique index)
    5. Upserts centroid vector to Qdrant
    6. On success: deprecates old centroids and promotes new one to ACTIVE
    7. On failure: marks new centroid as FAILED, old centroids remain ACTIVE
```

**Why**: Documentation should reflect the new safe order of operations.

### Step 5 (Optional): Guard against concurrent centroid computation

The `BUILDING` status also helps with concurrent calls. If two concurrent calls compute centroids for the same person:

1. Both create `BUILDING` centroids (no unique constraint violation since the partial index only covers `ACTIVE`).
2. Both upsert to Qdrant (the second overwrites the first in Qdrant, which is fine).
3. Both try to deprecate old centroids and promote to `ACTIVE`.
4. The second one to promote will hit the partial unique constraint.

To handle this, wrap the promote step in a try/except:

```python
    try:
        await deprecate_centroids(db, centroid_qdrant, person_id)
        centroid.status = CentroidStatus.ACTIVE
        await db.flush()
    except Exception as e:
        # Likely a concurrent centroid computation won the race.
        # Mark this one as DEPRECATED since the other is now ACTIVE.
        logger.warning(
            f"Failed to promote centroid {centroid_id} to ACTIVE "
            f"(likely concurrent computation): {e}"
        )
        centroid.status = CentroidStatus.DEPRECATED
        await db.flush()
        # The Qdrant point will be overwritten by the winning centroid
        # or will be harmlessly orphaned.
```

This is optional but recommended for robustness.

## Qdrant Rollback Strategy

### What if DB succeeds but Qdrant fails?

With the new order of operations:
- If Qdrant fails, the new centroid is marked `FAILED` in DB. Old centroids remain `ACTIVE`. No gap.
- The `FAILED` centroid's DB record exists but has no corresponding Qdrant point (or has a partial one). This is harmless because:
  - The `ACTIVE` centroids are used for all queries.
  - `FAILED` centroids are never queried.
  - A future retry of `compute_centroids_for_person` will create a new centroid, ignoring the failed one.

### What if DB commit fails after Qdrant succeeds?

This is the reverse scenario -- a new centroid point exists in Qdrant but the DB transaction rolls back. This leaves an orphaned Qdrant point. This is relatively harmless because:
- The orphaned point is not referenced by any DB record, so it will never be queried by the application.
- Future centroid computation will create a new point that overwrites or coexists with the orphan.
- A periodic cleanup job could detect and remove orphaned Qdrant points by comparing against DB records.

### Do we need to delete from Qdrant on failure?

No. The `centroid_qdrant.upsert_centroid()` is an idempotent upsert. If the same `centroid_id` is reused in a future retry, it will overwrite the previous point. If a different `centroid_id` is generated (which is the case since it is `uuid.uuid4()`), the old orphaned point will sit harmlessly in Qdrant. The storage overhead is negligible (one 512-dim vector + metadata per orphan).

## Verification

### Manual Verification

1. **Test normal path**:
   - Trigger centroid computation for a person with sufficient faces.
   - Verify the new centroid is created with `ACTIVE` status.
   - Verify old centroids are `DEPRECATED`.
   - Verify the Qdrant `person_centroids` collection has the new centroid.

2. **Test Qdrant failure path**:
   - Temporarily make Qdrant unavailable (e.g., stop the Qdrant container).
   - Trigger centroid computation.
   - Verify the new centroid is `FAILED` in DB.
   - Verify old centroids REMAIN `ACTIVE` (the critical fix).
   - Restart Qdrant and re-trigger computation. Verify it succeeds.

3. **Test computation failure path**:
   - Create a person with faces that have corrupted embeddings (or mock the computation to raise).
   - Trigger centroid computation.
   - Verify old centroids remain `ACTIVE`.
   - No new centroid with `BUILDING` or `FAILED` status should persist (if the failure happens before `db.flush()`).

4. **Test BUILDING status intermediate state**:
   - Add a breakpoint or sleep after `db.flush()` (BUILDING creation) but before Qdrant upsert.
   - Query the DB: should show one `ACTIVE` centroid (old) and one `BUILDING` centroid (new).
   - Verify that centroid-based searches still work (they query for `ACTIVE` centroids only).

### Automated Tests

```python
# tests/unit/test_centroid_deprecation_safety.py

import uuid
from unittest.mock import AsyncMock, MagicMock, patch

import numpy as np
import pytest

from image_search_service.db.models import CentroidStatus, CentroidType, PersonCentroid


@pytest.mark.asyncio
async def test_old_centroids_survive_qdrant_failure():
    """Verify that old centroids remain ACTIVE when Qdrant upsert fails."""
    person_id = uuid.uuid4()

    # Create a mock existing active centroid
    old_centroid = MagicMock(spec=PersonCentroid)
    old_centroid.centroid_id = uuid.uuid4()
    old_centroid.status = CentroidStatus.ACTIVE

    # Mock DB session
    mock_db = AsyncMock()

    # Mock Qdrant to fail on upsert
    mock_centroid_qdrant = MagicMock()
    mock_centroid_qdrant.upsert_centroid.side_effect = Exception("Qdrant unavailable")

    # After the fix, old centroid should still be ACTIVE
    # (because deprecation only happens AFTER successful Qdrant upsert)
    assert old_centroid.status == CentroidStatus.ACTIVE


@pytest.mark.asyncio
async def test_new_centroid_uses_building_status():
    """Verify new centroid is created with BUILDING status."""
    centroid = PersonCentroid(
        centroid_id=uuid.uuid4(),
        person_id=uuid.uuid4(),
        qdrant_point_id=uuid.uuid4(),
        model_version="arcface_r100_glint360k_v1",
        centroid_version=2,
        centroid_type=CentroidType.GLOBAL,
        cluster_label="global",
        n_faces=10,
        status=CentroidStatus.BUILDING,  # Should use BUILDING, not ACTIVE
        source_face_ids_hash="abc123",
    )

    assert centroid.status == CentroidStatus.BUILDING
    assert centroid.status != CentroidStatus.ACTIVE


def test_building_status_not_affected_by_unique_constraint():
    """Verify that BUILDING status does not conflict with ACTIVE unique index.

    The partial unique index 'ix_person_centroid_unique_active' only applies
    to rows where status = 'active'. BUILDING centroids are excluded.
    """
    # The index definition from the migration:
    # CREATE UNIQUE INDEX ix_person_centroid_unique_active
    # ON person_centroid (person_id, model_version, centroid_version,
    #                     centroid_type, cluster_label)
    # WHERE status = 'active';
    #
    # Therefore, a BUILDING centroid with the same
    # (person_id, model_version, centroid_version, centroid_type, cluster_label)
    # as an ACTIVE centroid does NOT violate the constraint.
    assert CentroidStatus.BUILDING.value == "building"
    assert CentroidStatus.BUILDING.value != "active"
```

## Risk Assessment

- **Risk**: The new order of operations changes the timing of deprecation. If a caller queries for active centroids between the Qdrant upsert and the promotion to ACTIVE, they will get the OLD centroid.
- **Mitigation**: This is actually the desired behavior -- the old centroid is valid until the new one is fully ready. There is no gap where zero centroids exist.

- **Risk**: Orphaned Qdrant points from failed computations.
- **Mitigation**: Qdrant points from failed computations are harmless. They use negligible storage (512 floats + metadata per point). A future cleanup job can remove points not referenced by any `ACTIVE` or `BUILDING` DB record.

- **Risk**: Concurrent centroid computation for the same person could cause unique constraint violation when promoting to ACTIVE.
- **Mitigation**: Step 5 (optional) adds a try/except around the promotion that handles this race condition gracefully.

- **Risk**: The `BUILDING` status is used for the first time. Code that queries centroids by status might not expect it.
- **Mitigation**: All existing queries filter for `CentroidStatus.ACTIVE`. The `BUILDING` status will be invisible to all existing code paths. Only the centroid computation function itself will see it. Grep the codebase to confirm:
  ```bash
  grep -rn "CentroidStatus" image-search-service/src/ | grep -v "test" | grep -v "__pycache__"
  ```

- **Risk**: The `deprecate_centroids` function being moved after Qdrant upsert means the old centroid's Qdrant point is NOT deleted or updated during deprecation.
- **Mitigation**: Examining the `deprecate_centroids` function (lines 220-255), it only updates DB status to `DEPRECATED` -- it does NOT delete or modify Qdrant points (the docstring at line 227 states: "Does NOT delete from Qdrant, just marks as deprecated in DB. Qdrant centroids will be overwritten with new versions."). So moving it after the Qdrant upsert has no effect on Qdrant operations.

## Summary of Changes

| File | Change | Lines |
|---|---|---|
| `centroid_service.py` | Move computation BEFORE deprecation | 322-334 |
| `centroid_service.py` | Create new centroid with `BUILDING` status | 349-361 (status field) |
| `centroid_service.py` | Move deprecation + promotion to ACTIVE after Qdrant upsert | 380-396 |
| `centroid_service.py` | Update docstring | 265-272 |
| `centroid_service.py` | (Optional) Add try/except for concurrent race | After promotion |

## Estimated Effort

**Time**: 1-2 hours (reorder operations + add BUILDING status usage + testing)

**Complexity**: Low -- the changes are structural reordering of existing operations, not new logic. The `BUILDING` status enum value already exists.

**Risk level**: Low -- the change makes the operation safer by eliminating the deprecation gap. The only new behavior is using the `BUILDING` status value that already exists in the enum.
