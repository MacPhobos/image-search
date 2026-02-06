# C2: Fix Suggestion Accept Qdrant Sync

## References
- **Research**: `docs/research/clustering_and_suggestions/04-devils-advocate-analysis.md` -- Section "Suggestion Acceptance Silently Desynchronizes Qdrant" (Bomb #1)
- **Research**: `docs/research/clustering_and_suggestions/02-backend-analysis.md` -- Qdrant sync patterns in assignment flow
- **Source Files**:
  - `image-search-service/src/image_search_service/api/routes/face_suggestions.py` (accept_suggestion + bulk_suggestion_action)
  - `image-search-service/src/image_search_service/faces/assigner.py` (working sync pattern)
  - `image-search-service/src/image_search_service/vector/face_qdrant.py` (update_person_ids method)

## Problem Statement

When a user accepts a face suggestion, the `accept_suggestion` endpoint in `face_suggestions.py` (line 502) updates `face.person_id` in PostgreSQL and commits, but **never updates the `person_id` payload in the Qdrant `faces` collection**. The same bug exists in the `bulk_suggestion_action` endpoint (line 651). This causes permanent drift between PostgreSQL and Qdrant.

**Evidence**: The `accept_suggestion` function at `face_suggestions.py` lines 526-533:

```python
# Line 527: DB updated
face.person_id = suggestion.suggested_person_id

# Line 530-531: Suggestion marked accepted
suggestion.status = FaceSuggestionStatus.ACCEPTED.value
suggestion.reviewed_at = datetime.now(UTC)

# Line 533: Committed to DB
await db.commit()

# MISSING: No call to qdrant.update_person_ids()
```

In contrast, the `FaceAssigner.assign_new_faces()` method at `assigner.py` lines 236-237 correctly syncs both stores:

```python
# Update Qdrant
qdrant.update_person_ids(qdrant_point_ids, person_id)
```

The Qdrant sync call exists in the assignment path but was never added to the suggestion acceptance path.

## Root Cause Analysis

### The suggestion acceptance code path (BROKEN)

In `face_suggestions.py`, the `accept_suggestion` endpoint (line 502-571):

1. Looks up the `FaceSuggestion` by ID (line 509)
2. Looks up the `FaceInstance` by `suggestion.face_instance_id` (line 522)
3. Sets `face.person_id = suggestion.suggested_person_id` (line 527) -- **PostgreSQL only**
4. Updates suggestion status to ACCEPTED (line 530)
5. Commits to DB (line 533)
6. Returns response

**Missing step**: After step 3, there should be a call to `FaceQdrantClient.update_person_ids()` to update the `person_id` payload on the face's Qdrant point.

### The bulk acceptance code path (ALSO BROKEN)

In `face_suggestions.py`, `bulk_suggestion_action` (line 651-751):

1. Iterates over each suggestion ID (line 662)
2. For accepted suggestions, sets `face.person_id = suggestion.suggested_person_id` (line 681) -- **PostgreSQL only**
3. Commits all changes (line 695)

**Missing step**: No Qdrant sync for any of the accepted faces.

### The face assignment code path (WORKING -- reference pattern)

In `assigner.py`, `assign_new_faces()` (lines 223-239):

```python
# Batch update database and Qdrant for auto-assignments
for person_id, face_data in assignments.items():
    face_ids = [f[0] for f in face_data]
    qdrant_point_ids = [f[1] for f in face_data]

    # Update database
    stmt = (
        update(FaceInstance)
        .where(FaceInstance.id.in_(face_ids))
        .values(person_id=person_id)
    )
    self.db.execute(stmt)

    # Update Qdrant  <-- THIS IS THE PATTERN WE NEED
    qdrant.update_person_ids(qdrant_point_ids, person_id)

self.db.commit()
```

This correctly updates both PostgreSQL and Qdrant in the same flow.

### The Qdrant update method

`FaceQdrantClient.update_person_ids()` at `face_qdrant.py` lines 357-390:

```python
def update_person_ids(
    self,
    point_ids: list[uuid.UUID],
    person_id: uuid.UUID | None,
) -> None:
    if not point_ids:
        return

    try:
        if person_id is None:
            self.client.delete_payload(
                collection_name=_get_face_collection_name(),
                keys=["person_id"],
                points=[str(point_id) for point_id in point_ids],
            )
        else:
            self.client.set_payload(
                collection_name=_get_face_collection_name(),
                payload={"person_id": str(person_id)},
                points=[str(point_id) for point_id in point_ids],
            )
    except Exception as e:
        logger.error(f"Failed to update person_ids: {e}")
        raise
```

This method already handles both setting and clearing person IDs. It is synchronous (uses the sync Qdrant client). For the async API routes, we need to call it in a way that works within the async context.

## Current Behavior (Before Fix)

1. User accepts a face suggestion in the UI.
2. PostgreSQL is updated: `face.person_id = suggested_person_id`.
3. Qdrant is NOT updated: the `person_id` field in the face's Qdrant payload remains `None` (or whatever it was before).
4. Future Qdrant searches that filter by `person_id` (e.g., `exclude_person_id` in `search_faces_with_centroid`) will return stale results.
5. The same face may appear as a suggestion for the same person again (since Qdrant still thinks it is unassigned).
6. Face counts per person in Qdrant-based queries will be wrong.
7. This desync accumulates over time with every accepted suggestion.

## Expected Behavior (After Fix)

1. User accepts a face suggestion.
2. PostgreSQL is updated: `face.person_id = suggested_person_id`.
3. Qdrant is also updated: `person_id` payload set to `suggested_person_id` for the face's `qdrant_point_id`.
4. Future searches correctly exclude this face when filtering by person.
5. The face will not reappear as a suggestion for the same person.
6. Both data stores remain in sync.

## Implementation Plan

### Step 1: Add Qdrant sync to `accept_suggestion`

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

**Current code** (lines 526-533):
```python
    # Assign face to person
    face.person_id = suggestion.suggested_person_id

    # Update suggestion status
    suggestion.status = FaceSuggestionStatus.ACCEPTED.value
    suggestion.reviewed_at = datetime.now(UTC)

    await db.commit()
    await db.refresh(suggestion)
```

**Change to**:
```python
    # Assign face to person
    face.person_id = suggestion.suggested_person_id

    # Update suggestion status
    suggestion.status = FaceSuggestionStatus.ACCEPTED.value
    suggestion.reviewed_at = datetime.now(UTC)

    await db.commit()
    await db.refresh(suggestion)

    # Sync person_id to Qdrant (must happen after DB commit succeeds)
    try:
        qdrant = get_face_qdrant_client()
        qdrant.update_person_ids(
            [face.qdrant_point_id],
            suggestion.suggested_person_id,
        )
    except Exception as e:
        logger.error(
            f"Failed to sync person_id to Qdrant for face {face.id}: {e}. "
            f"DB is updated but Qdrant is out of sync."
        )
        # Do not re-raise -- DB commit succeeded, Qdrant can be repaired later
```

**Why**: This adds the Qdrant sync after the DB commit. We place the Qdrant update AFTER `db.commit()` so that if the DB commit fails, we do not update Qdrant. If the Qdrant update fails, we log the error but do not roll back the DB commit (since the DB is the source of truth and Qdrant can be repaired). The `get_face_qdrant_client()` function returns the singleton FaceQdrantClient which uses a sync Qdrant client -- calling sync methods from async context is safe here because `update_person_ids` uses the underlying `qdrant_client` which handles this correctly.

### Step 2: Add import for `get_face_qdrant_client`

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

Check if `get_face_qdrant_client` is already imported. If not, add to the imports at the top of the file:

```python
from image_search_service.vector.face_qdrant import get_face_qdrant_client
```

Verify by running:
```bash
grep -n "get_face_qdrant_client" image-search-service/src/image_search_service/api/routes/face_suggestions.py
```

### Step 3: Add Qdrant sync to `bulk_suggestion_action`

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

**Current code** (lines 676-695):
```python
        try:
            if request.action == "accept":
                # Get and update face
                face = await db.get(FaceInstance, suggestion.face_instance_id)
                if face:
                    face.person_id = suggestion.suggested_person_id
                suggestion.status = FaceSuggestionStatus.ACCEPTED.value
                # Track affected person IDs for auto-find-more
                affected_person_ids.add(suggestion.suggested_person_id)
            else:  # reject
                suggestion.status = FaceSuggestionStatus.REJECTED.value

            suggestion.reviewed_at = datetime.now(UTC)
            processed += 1

        except Exception as e:
            failed += 1
            errors.append(f"Error processing suggestion {suggestion_id}: {str(e)}")

    await db.commit()
```

**Change to** (collect faces for Qdrant sync, then batch update after commit):
```python
        try:
            if request.action == "accept":
                # Get and update face
                face = await db.get(FaceInstance, suggestion.face_instance_id)
                if face:
                    face.person_id = suggestion.suggested_person_id
                    # Collect for Qdrant batch sync
                    if suggestion.suggested_person_id not in qdrant_sync_batch:
                        qdrant_sync_batch[suggestion.suggested_person_id] = []
                    qdrant_sync_batch[suggestion.suggested_person_id].append(
                        face.qdrant_point_id
                    )
                suggestion.status = FaceSuggestionStatus.ACCEPTED.value
                # Track affected person IDs for auto-find-more
                affected_person_ids.add(suggestion.suggested_person_id)
            else:  # reject
                suggestion.status = FaceSuggestionStatus.REJECTED.value

            suggestion.reviewed_at = datetime.now(UTC)
            processed += 1

        except Exception as e:
            failed += 1
            errors.append(f"Error processing suggestion {suggestion_id}: {str(e)}")

    await db.commit()

    # Batch sync person_ids to Qdrant (after DB commit succeeds)
    if qdrant_sync_batch:
        try:
            qdrant = get_face_qdrant_client()
            for person_id, point_ids in qdrant_sync_batch.items():
                qdrant.update_person_ids(point_ids, person_id)
            logger.info(
                f"Synced {sum(len(v) for v in qdrant_sync_batch.values())} "
                f"face person_ids to Qdrant for {len(qdrant_sync_batch)} persons"
            )
        except Exception as e:
            logger.error(
                f"Failed to sync person_ids to Qdrant during bulk action: {e}. "
                f"DB is updated but Qdrant is out of sync."
            )
```

Also add the `qdrant_sync_batch` initialization near the top of the function, before the loop. Add after line 660 (`affected_person_ids: set[UUID] = set()`):

```python
    qdrant_sync_batch: dict[UUID, list[UUID]] = {}  # person_id -> [qdrant_point_ids]
```

**Why**: The bulk action processes multiple suggestions in a single request. We collect all Qdrant updates and apply them in a batch after the DB commit succeeds. This is more efficient than updating Qdrant inside the loop and matches the batching pattern used in `assigner.py`.

## Data Repair

### Detect out-of-sync records

All faces that were assigned via suggestion acceptance (rather than auto-assignment) will have `person_id` set in PostgreSQL but NOT in Qdrant. Run the following repair script to detect and fix them.

### Repair script

Create a one-time repair job or management command:

```python
"""Repair script: Sync person_ids from PostgreSQL to Qdrant for all assigned faces.

Run this once after deploying the C2 fix to repair historical desync.
Can be run as a management command or one-time RQ job.
"""
import logging
from collections import defaultdict

from sqlalchemy import select
from sqlalchemy.orm import Session

from image_search_service.db.models import FaceInstance
from image_search_service.db.session import get_sync_session
from image_search_service.vector.face_qdrant import get_face_qdrant_client

logger = logging.getLogger(__name__)

def repair_qdrant_person_ids():
    """Sync all person_id values from PostgreSQL to Qdrant."""
    db_session = get_sync_session()
    qdrant = get_face_qdrant_client()

    try:
        # Get all faces with person_id assigned
        query = select(FaceInstance).where(FaceInstance.person_id.isnot(None))
        result = db_session.execute(query)
        faces = result.scalars().all()

        logger.info(f"Found {len(faces)} faces with person_id assigned in DB")

        # Group by person_id for batch Qdrant updates
        person_faces: dict[str, list] = defaultdict(list)
        for face in faces:
            person_faces[face.person_id].append(face.qdrant_point_id)

        # Batch update Qdrant
        total_synced = 0
        for person_id, point_ids in person_faces.items():
            try:
                qdrant.update_person_ids(point_ids, person_id)
                total_synced += len(point_ids)
                logger.debug(
                    f"Synced {len(point_ids)} faces for person {person_id}"
                )
            except Exception as e:
                logger.error(
                    f"Failed to sync faces for person {person_id}: {e}"
                )

        logger.info(
            f"Repair complete: synced {total_synced} faces "
            f"across {len(person_faces)} persons"
        )

    finally:
        db_session.close()
```

### How to detect the extent of desync

Before running the repair, you can estimate how many records are out of sync:

```python
# Count faces with person_id in DB
SELECT COUNT(*) FROM face_instances WHERE person_id IS NOT NULL;

# Compare with Qdrant: scroll the faces collection and count
# faces where person_id payload is set. The difference indicates
# how many faces are out of sync.
```

Alternatively, add a verification step to the repair script that checks each face's Qdrant payload before updating, to count how many were actually out of sync.

## Verification

### Manual Verification

1. **Before deploying** (confirm the bug):
   - Accept a suggestion via the UI or API.
   - Query Qdrant directly for the accepted face's point ID.
   - Verify that `person_id` is NOT set in the Qdrant payload (confirming the bug).

2. **After deploying** (confirm the fix):
   - Accept a new suggestion.
   - Query Qdrant for the face's point ID.
   - Verify that `person_id` IS set in the Qdrant payload to the correct person UUID.

3. **After running repair script**:
   - Spot-check 5-10 previously accepted suggestions.
   - Verify their Qdrant payloads now have the correct `person_id`.

4. **Verify centroid exclusion works**:
   - For a person with many accepted suggestions, trigger "Find More".
   - Verify that already-assigned faces are NOT returned as new suggestions (confirming the Qdrant `exclude_person_id` filter now works correctly).

### Automated Tests

```python
# tests/unit/test_suggestion_acceptance_qdrant_sync.py

import uuid
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from image_search_service.db.models import (
    FaceInstance,
    FaceSuggestion,
    FaceSuggestionStatus,
)


@pytest.mark.asyncio
async def test_accept_suggestion_syncs_qdrant():
    """Verify that accepting a suggestion updates Qdrant person_id."""
    face_id = uuid.uuid4()
    person_id = uuid.uuid4()
    qdrant_point_id = uuid.uuid4()

    # Mock face instance
    mock_face = MagicMock(spec=FaceInstance)
    mock_face.id = face_id
    mock_face.person_id = None
    mock_face.qdrant_point_id = qdrant_point_id

    # Mock suggestion
    mock_suggestion = MagicMock(spec=FaceSuggestion)
    mock_suggestion.suggested_person_id = person_id
    mock_suggestion.face_instance_id = face_id
    mock_suggestion.status = FaceSuggestionStatus.PENDING.value

    # Mock Qdrant client
    mock_qdrant = MagicMock()

    with patch(
        "image_search_service.api.routes.face_suggestions.get_face_qdrant_client",
        return_value=mock_qdrant,
    ):
        # Simulate the acceptance flow
        mock_face.person_id = mock_suggestion.suggested_person_id

        # This is what the fix should do:
        mock_qdrant.update_person_ids(
            [mock_face.qdrant_point_id],
            mock_suggestion.suggested_person_id,
        )

        # Verify Qdrant was called with correct args
        mock_qdrant.update_person_ids.assert_called_once_with(
            [qdrant_point_id],
            person_id,
        )


@pytest.mark.asyncio
async def test_bulk_accept_syncs_qdrant_batch():
    """Verify that bulk accept updates Qdrant for all accepted faces."""
    faces = []
    for _ in range(5):
        face = MagicMock(spec=FaceInstance)
        face.id = uuid.uuid4()
        face.qdrant_point_id = uuid.uuid4()
        face.person_id = None
        faces.append(face)

    person_id = uuid.uuid4()
    mock_qdrant = MagicMock()

    with patch(
        "image_search_service.api.routes.face_suggestions.get_face_qdrant_client",
        return_value=mock_qdrant,
    ):
        # Simulate bulk acceptance
        point_ids = [f.qdrant_point_id for f in faces]
        mock_qdrant.update_person_ids(point_ids, person_id)

        # Verify batch call
        mock_qdrant.update_person_ids.assert_called_once_with(
            point_ids, person_id
        )
```

## Risk Assessment

- **Risk**: If the Qdrant update fails after DB commit, the face will be assigned in PostgreSQL but not in Qdrant (same as current behavior for one face, but now detectable via error log).
- **Mitigation**: The fix includes error handling that logs the failure without rolling back the DB commit. The repair script can fix any remaining desync. Future improvement: add a periodic reconciliation job (see recommendation below).

- **Risk**: The `update_person_ids` call is synchronous but called from an async route handler.
- **Mitigation**: The `qdrant_client` library already handles sync/async correctly. The `update_person_ids` call uses `self.client.set_payload()` which is a lightweight HTTP request (payload update, not a search). Latency impact is minimal (<10ms per call for single-face updates).

- **Risk**: The bulk action with many suggestions could be slow due to per-person Qdrant calls.
- **Mitigation**: The batch is grouped by person_id, and `update_person_ids` already accepts a list of point IDs. For N suggestions across P persons, this makes P Qdrant calls (not N). Typical bulk actions have 10-50 suggestions for 1-3 persons.

- **Risk**: Running the repair script against a large database could be slow.
- **Mitigation**: The repair script groups by person_id and uses batch Qdrant updates. For 10,000 faces across 100 persons, this is ~100 Qdrant API calls. Add progress logging and optionally run during off-peak hours.

## Future Improvement: Reconciliation Job

After this fix, consider adding a periodic reconciliation job that:
1. Samples random faces from PostgreSQL with `person_id` set.
2. Checks their Qdrant payload `person_id` matches.
3. Logs/alerts on any mismatches.
4. Optionally auto-repairs detected mismatches.

This would catch any future code paths that modify `person_id` without syncing Qdrant.

## Estimated Effort

**Time**: 1-2 hours (code changes + repair script + testing)

**Complexity**: Low-medium -- the fix follows an established pattern already used in `assigner.py`. The main work is writing the repair script and verifying it works.

**Risk level**: Low -- adds a new Qdrant call after an existing DB commit. Failure of the Qdrant call does not affect the DB transaction.
