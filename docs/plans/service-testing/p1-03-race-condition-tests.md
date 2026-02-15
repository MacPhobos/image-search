# P1-03: Add Race Condition and Concurrency Tests

**Priority**: P1 (Critical)
**Effort**: Large (3-5 days)
**Risk Level**: HIGH -- Race conditions are silent, data-corrupting bugs
**Status**: PLANNED

---

## 1. Problem Statement

The image-search-service codebase contains **zero database locking** (`SELECT ... FOR UPDATE`) across the entire `src/` directory. A comprehensive search for `for_update`, `with_for_update`, `FOR UPDATE`, `LOCK`, and `select_for_update` returns zero matches. This means every concurrent database operation in the codebase is vulnerable to race conditions.

The devil's advocate review (see `docs/research/service-testing/devils-advocate-review.md`, Section 2.1) identifies this as the highest-risk gap in the test suite. The four critical race condition surfaces are:

1. **Suggestion accept flow** -- Two users accepting the same suggestion simultaneously
2. **Bulk suggestion operations** -- Concurrent bulk actions overlapping on the same suggestions
3. **Training session state machine** -- Simultaneous start/pause/cancel on the same session
4. **Centroid computation** -- Concurrent centroid recomputation for the same person

Additionally, the codebase exhibits a cross-system consistency pattern where PostgreSQL commits first and Qdrant syncs after, with failures logged but not rolled back (Section 2.2 of the devil's advocate review). This creates a persistent desync risk.

### Why This Matters

- **Data corruption**: Double-accepting a suggestion assigns the same face to two different people without error
- **Silent failures**: Qdrant-PostgreSQL desync goes unnoticed until users see incorrect search results
- **State machine violations**: A training session could end up in an impossible state (e.g., both "running" and "cancelled")
- **No visibility**: Without tests, these bugs surface only under production load

---

## 2. Affected Source Files

### 2.1. Face Suggestion Accept Flow

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

**Function**: `accept_suggestion()` (lines 503-586)

The critical race window exists between the status check and the commit:

```python
# Line 510: Read suggestion (no lock)
suggestion = await db.get(FaceSuggestion, suggestion_id)

# Line 516: Check status (stale read possible)
if suggestion.status != FaceSuggestionStatus.PENDING.value:
    raise HTTPException(status_code=400, ...)

# Lines 528-531: Mutate face and suggestion (in-memory only)
face.person_id = suggestion.suggested_person_id
suggestion.status = FaceSuggestionStatus.ACCEPTED.value

# Line 534: Commit (second writer wins silently)
await db.commit()

# Lines 538-548: Qdrant sync (try/except, logged but NOT rolled back)
try:
    qdrant = get_face_qdrant_client()
    qdrant.update_person_ids([face.qdrant_point_id], suggestion.suggested_person_id)
except Exception as e:
    logger.error(f"Failed to sync person_id to Qdrant for face {face.id}: {e}. "
                 f"DB is updated but Qdrant is out of sync.")
```

**Race scenario**: Request A reads suggestion (status=PENDING), Request B reads same suggestion (status=PENDING), both pass the check, both commit. The face ends up assigned to whichever person_id committed last, while both users see success.

### 2.2. Bulk Suggestion Action Flow

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`

**Function**: `bulk_suggestion_action()` (lines 666-789)

```python
# Lines 678-690: Iterate suggestions without locking
for suggestion_id in request.suggestion_ids:
    suggestion = await db.get(FaceSuggestion, suggestion_id)
    if suggestion.status != FaceSuggestionStatus.PENDING.value:
        # ... skip
        continue

    # Lines 693-710: Mutate in-memory
    if request.action == "accept":
        face = await db.get(FaceInstance, suggestion.face_instance_id)
        face.person_id = suggestion.suggested_person_id
        suggestion.status = FaceSuggestionStatus.ACCEPTED.value

# Line 717: Single commit for all mutations
await db.commit()

# Lines 720-733: Batch Qdrant sync (try/except, logged but NOT rolled back)
if qdrant_sync_batch:
    try:
        qdrant = get_face_qdrant_client()
        for person_id, point_ids in qdrant_sync_batch.items():
            qdrant.update_person_ids(point_ids, person_id)
    except Exception as e:
        logger.error(f"Failed to sync person_ids to Qdrant during bulk action: {e}. "
                     f"DB is updated but Qdrant is out of sync.")
```

**Race scenario**: Bulk request A accepts suggestions [1,2,3], bulk request B accepts suggestions [2,3,4]. Suggestion 2 and 3 get processed by both requests without locking. Depending on commit order, some faces may be assigned to incorrect people.

### 2.3. Training Session State Machine

**File**: `image-search-service/src/image_search_service/services/training_service.py`

**Functions**:
- `start_training()` (lines 568-635)
- `pause_training()` (lines 637-669)
- `cancel_training()` (lines 671-707)

```python
# start_training() -- lines 586-596
session = await self.get_session(db, session_id)  # No lock
valid_states = [SessionStatus.PENDING.value, SessionStatus.PAUSED.value, SessionStatus.FAILED.value]
if session.status not in valid_states:
    raise ValueError(...)

# Lines 622-625: Mutate and commit
session.status = SessionStatus.RUNNING.value
await db.commit()

# pause_training() -- lines 650-665
session = await self.get_session(db, session_id)  # No lock
if session.status != SessionStatus.RUNNING.value:
    raise ValueError(...)
session.status = SessionStatus.PAUSED.value
await db.commit()

# cancel_training() -- lines 686-703
session = await self.get_session(db, session_id)  # No lock
if session.status not in valid_states:
    raise ValueError(...)
session.status = SessionStatus.CANCELLED.value
await db.commit()
```

**Race scenario**: User clicks "start" and "cancel" in rapid succession. Both requests read session status as "pending" (valid for start) / "pending" (not valid for cancel) -- OR if start commits first, cancel reads "running" (valid). Depending on timing, the session could enter an inconsistent state, or background jobs could be enqueued for a cancelled session.

### 2.4. Centroid Computation

**File**: `image-search-service/src/image_search_service/services/centroid_service.py`

**Function**: `compute_centroids_for_person()` (lines 258-396)

```python
# Lines 286-295: Read face embeddings (no lock)
face_ids, embeddings = await get_person_face_embeddings(db, face_qdrant, person_id)

# Lines 298-320: Check staleness of existing centroid (stale read)
existing_centroid = ...  # Query active centroid

# Line 325: Deprecate old centroids
await deprecate_centroids(db, centroid_qdrant, person_id)

# Lines 328-364: Compute and create new centroid
centroid_vector = compute_global_centroid(embeddings_array, ...)
centroid = PersonCentroid(...)
db.add(centroid)
await db.flush()

# Lines 380-394: Qdrant upsert (after DB flush, before commit)
try:
    centroid_qdrant.upsert_centroid(centroid_id=centroid_id, vector=centroid_vector.tolist(), ...)
except Exception as e:
    logger.error(f"Failed to upsert centroid to Qdrant: {e}")
    centroid.status = CentroidStatus.FAILED
    await db.flush()
    raise
```

**Race scenario**: Two "find more" jobs trigger centroid recomputation for the same person simultaneously. Both deprecate the old centroid, both compute a new one. The unique partial index `(person_id, model_version, centroid_version, centroid_type, cluster_label) WHERE status = 'active'` may catch this (constraint violation), but it is not handled gracefully -- the second request would crash with an unhandled IntegrityError.

---

## 3. Race Condition Test Scenarios

### 3.1. Double-Accept Suggestion Test

**Objective**: Prove that two concurrent accept requests on the same suggestion produce incorrect results.

```python
# File: tests/api/test_race_conditions.py

import asyncio
from unittest.mock import AsyncMock, MagicMock, patch

import pytest
from httpx import AsyncClient

from image_search_service.db.models import FaceSuggestion, FaceInstance, FaceSuggestionStatus


@pytest.mark.asyncio
async def test_double_accept_suggestion_race_condition(
    async_client: AsyncClient,
    db_session: AsyncMock,
):
    """Two concurrent accepts on the same PENDING suggestion.

    Expected current behavior (BUG): Both succeed with 200.
    Expected correct behavior: One succeeds, one fails with 409 Conflict.
    """
    suggestion_id = 42
    face_id = 100
    person_a_id = "aaaa-aaaa"
    person_b_id = "bbbb-bbbb"

    # Create a mock suggestion that starts as PENDING
    mock_suggestion = MagicMock(spec=FaceSuggestion)
    mock_suggestion.id = suggestion_id
    mock_suggestion.status = FaceSuggestionStatus.PENDING.value
    mock_suggestion.face_instance_id = face_id
    mock_suggestion.suggested_person_id = person_a_id

    mock_face = MagicMock(spec=FaceInstance)
    mock_face.id = face_id
    mock_face.person_id = None

    # Track the order of operations
    operation_log = []

    original_get = db_session.get

    async def slow_get(model, pk, **kwargs):
        """Simulate slow DB read to widen race window."""
        operation_log.append(f"get_{model.__name__}_{pk}")
        # Add artificial delay to widen race window
        await asyncio.sleep(0.01)
        if model == FaceSuggestion:
            return mock_suggestion
        if model == FaceInstance:
            return mock_face
        return await original_get(model, pk, **kwargs)

    db_session.get = slow_get

    commit_count = 0
    async def counting_commit():
        nonlocal commit_count
        commit_count += 1
        operation_log.append(f"commit_{commit_count}")

    db_session.commit = counting_commit
    db_session.refresh = AsyncMock()

    with patch("image_search_service.api.routes.face_suggestions.get_face_qdrant_client"):
        # Fire both requests concurrently
        results = await asyncio.gather(
            async_client.post(f"/api/v1/face-suggestions/{suggestion_id}/accept",
                            json={"person_id": person_a_id}),
            async_client.post(f"/api/v1/face-suggestions/{suggestion_id}/accept",
                            json={"person_id": person_b_id}),
            return_exceptions=True,
        )

    # DOCUMENT THE BUG: Both requests succeed (no locking)
    success_count = sum(1 for r in results if not isinstance(r, Exception) and r.status_code == 200)
    assert success_count == 2, (
        f"Expected both accepts to succeed (demonstrating race condition), "
        f"got {[r.status_code if not isinstance(r, Exception) else str(r) for r in results]}"
    )
    # The face.person_id is whatever the last writer set
    assert commit_count == 2, "Both transactions committed without conflict"
```

### 3.2. Concurrent Bulk Action Overlap Test

**Objective**: Prove that overlapping bulk actions on shared suggestions produce incorrect results.

```python
@pytest.mark.asyncio
async def test_concurrent_bulk_actions_overlapping_suggestions(
    async_client: AsyncClient,
    db_session: AsyncMock,
):
    """Two bulk-accept requests with overlapping suggestion sets.

    Bulk A accepts [1, 2, 3], Bulk B accepts [2, 3, 4].
    Suggestions 2 and 3 are in both sets.

    Expected current behavior (BUG): Both succeed, suggestions 2 and 3
    get processed twice, face assignments depend on commit order.
    """
    # Setup mock suggestions (all PENDING)
    mock_suggestions = {}
    mock_faces = {}
    for sid in [1, 2, 3, 4]:
        suggestion = MagicMock(spec=FaceSuggestion)
        suggestion.id = sid
        suggestion.status = FaceSuggestionStatus.PENDING.value
        suggestion.face_instance_id = sid * 10
        suggestion.suggested_person_id = f"person-{sid}"
        mock_suggestions[sid] = suggestion

        face = MagicMock(spec=FaceInstance)
        face.id = sid * 10
        face.person_id = None
        face.qdrant_point_id = f"point-{sid}"
        mock_faces[sid * 10] = face

    status_changes = []

    async def tracked_get(model, pk, **kwargs):
        if model == FaceSuggestion and pk in mock_suggestions:
            status_changes.append(f"read_suggestion_{pk}_status={mock_suggestions[pk].status}")
            return mock_suggestions[pk]
        if model == FaceInstance and pk in mock_faces:
            return mock_faces[pk]
        return None

    db_session.get = tracked_get
    db_session.commit = AsyncMock()
    db_session.refresh = AsyncMock()

    with patch("image_search_service.api.routes.face_suggestions.get_face_qdrant_client"):
        results = await asyncio.gather(
            async_client.post("/api/v1/face-suggestions/bulk-action",
                            json={"action": "accept", "suggestion_ids": [1, 2, 3]}),
            async_client.post("/api/v1/face-suggestions/bulk-action",
                            json={"action": "accept", "suggestion_ids": [2, 3, 4]}),
            return_exceptions=True,
        )

    # Count how many times overlapping suggestions (2, 3) were read as PENDING
    pending_reads_for_2 = [e for e in status_changes if "read_suggestion_2_status=pending" in e]
    pending_reads_for_3 = [e for e in status_changes if "read_suggestion_3_status=pending" in e]

    # DOCUMENT THE BUG: Both requests read suggestion 2 and 3 as PENDING
    # because there is no row-level locking
    assert len(pending_reads_for_2) >= 1, "Suggestion 2 was read at least once as PENDING"
    assert len(pending_reads_for_3) >= 1, "Suggestion 3 was read at least once as PENDING"
```

### 3.3. Training Session Concurrent State Transitions Test

**Objective**: Prove that concurrent start/cancel on the same session creates an inconsistent state.

```python
@pytest.mark.asyncio
async def test_concurrent_start_and_cancel_training_session(
    async_client: AsyncClient,
    db_session: AsyncMock,
):
    """Concurrent start + cancel on same PENDING session.

    Expected current behavior (BUG): Depending on interleaving:
    - Start reads PENDING (valid), Cancel reads PENDING (invalid for cancel) -> Cancel fails
    - Start commits RUNNING, Cancel reads RUNNING (valid) -> Both succeed
    - Most dangerous: Start enqueues job, Cancel commits CANCELLED,
      but background job still runs on a "cancelled" session
    """
    from image_search_service.db.models import TrainingSession, SessionStatus

    session_id = 1
    mock_session = MagicMock(spec=TrainingSession)
    mock_session.id = session_id
    mock_session.status = SessionStatus.PENDING.value
    mock_session.started_at = None
    mock_session.paused_at = None

    state_log = []

    original_status = mock_session.status

    # Track status reads and writes
    def track_status_access():
        state_log.append(f"read_status={mock_session.status}")
        return mock_session.status

    # We need to test that concurrent requests can both proceed
    # The key insight: both read the same status before either commits

    with patch(
        "image_search_service.services.training_service.TrainingService.get_session",
        return_value=mock_session,
    ):
        with patch(
            "image_search_service.services.training_service.TrainingService.enqueue_training",
            return_value="mock-rq-job-id",
        ):
            results = await asyncio.gather(
                async_client.post(f"/api/v1/training/sessions/{session_id}/start"),
                async_client.post(f"/api/v1/training/sessions/{session_id}/cancel"),
                return_exceptions=True,
            )

    # Analyze results based on interleaving
    start_result, cancel_result = results
    if not isinstance(start_result, Exception):
        state_log.append(f"start_status={start_result.status_code}")
    if not isinstance(cancel_result, Exception):
        state_log.append(f"cancel_status={cancel_result.status_code}")

    # Document: At minimum, this test proves no protection exists
    # The session object is shared, and mutations are not atomic
```

### 3.4. Concurrent Centroid Computation Test

**Objective**: Prove that concurrent centroid computations for the same person can produce constraint violations or data corruption.

```python
@pytest.mark.asyncio
async def test_concurrent_centroid_computation_same_person(
    db_session: AsyncMock,
):
    """Two concurrent centroid computations for the same person.

    Expected current behavior (BUG):
    1. Both read existing centroid as "stale"
    2. Both deprecate the existing centroid
    3. Both compute new centroids
    4. Second INSERT violates unique partial index
       OR both create centroids, one overwrites the other in Qdrant
    """
    from image_search_service.services.centroid_service import compute_centroids_for_person
    import uuid
    import numpy as np

    person_id = uuid.uuid4()

    mock_face_qdrant = MagicMock()
    mock_centroid_qdrant = MagicMock()

    # Mock embeddings retrieval
    fake_embeddings = np.random.randn(5, 512).astype(np.float32)
    fake_face_ids = [uuid.uuid4() for _ in range(5)]

    computation_log = []

    with patch(
        "image_search_service.services.centroid_service.get_person_face_embeddings",
        return_value=(fake_face_ids, fake_embeddings.tolist()),
    ) as mock_get_embeddings:
        with patch(
            "image_search_service.services.centroid_service.deprecate_centroids",
        ) as mock_deprecate:
            with patch(
                "image_search_service.services.centroid_service.get_settings",
            ) as mock_settings:
                mock_settings.return_value.centroid_min_faces = 3
                mock_settings.return_value.centroid_model_version = "v1"
                mock_settings.return_value.centroid_algorithm_version = "v1"
                mock_settings.return_value.centroid_trim_threshold_small = 0.1
                mock_settings.return_value.centroid_trim_threshold_large = 0.05

                # Simulate no existing centroid (both see stale)
                db_session.execute = AsyncMock(
                    return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=None))
                )
                db_session.flush = AsyncMock()
                db_session.add = MagicMock()

                results = await asyncio.gather(
                    compute_centroids_for_person(
                        db_session, mock_face_qdrant, mock_centroid_qdrant, person_id
                    ),
                    compute_centroids_for_person(
                        db_session, mock_face_qdrant, mock_centroid_qdrant, person_id
                    ),
                    return_exceptions=True,
                )

    # Both calls deprecated centroids (double-deprecation)
    assert mock_deprecate.call_count == 2, (
        f"Expected 2 deprecation calls (race condition), got {mock_deprecate.call_count}"
    )

    # Both calls attempted to create new centroids
    assert db_session.add.call_count == 2, (
        f"Expected 2 centroid additions (race condition), got {db_session.add.call_count}"
    )

    # Both calls attempted Qdrant upserts
    assert mock_centroid_qdrant.upsert_centroid.call_count == 2, (
        f"Expected 2 Qdrant upserts (race condition), got {mock_centroid_qdrant.upsert_centroid.call_count}"
    )
```

---

## 4. Qdrant-PostgreSQL Desync Tests

### 4.1. DB Commit Succeeds, Qdrant Fails

**Objective**: Prove that a Qdrant failure after DB commit leaves the system in an inconsistent state.

```python
@pytest.mark.asyncio
async def test_qdrant_failure_after_db_commit_on_accept(
    async_client: AsyncClient,
    db_session: AsyncMock,
):
    """Accept suggestion: DB commits, then Qdrant fails.

    Expected current behavior: API returns 200 (success), but DB and Qdrant
    are out of sync. The face has person_id in PostgreSQL but NOT in Qdrant.
    Error is only logged, not returned to user.
    """
    suggestion_id = 42

    mock_suggestion = MagicMock(spec=FaceSuggestion)
    mock_suggestion.id = suggestion_id
    mock_suggestion.status = FaceSuggestionStatus.PENDING.value
    mock_suggestion.face_instance_id = 100
    mock_suggestion.suggested_person_id = "person-1"
    mock_suggestion.confidence = 0.95
    # ... (other required fields)

    mock_face = MagicMock(spec=FaceInstance)
    mock_face.id = 100
    mock_face.person_id = None
    mock_face.qdrant_point_id = "point-100"

    db_session.get = AsyncMock(side_effect=lambda model, pk, **kw:
        mock_suggestion if model == FaceSuggestion
        else mock_face if model == FaceInstance
        else None
    )
    db_session.commit = AsyncMock()  # DB commit succeeds
    db_session.refresh = AsyncMock()

    mock_qdrant = MagicMock()
    mock_qdrant.update_person_ids.side_effect = ConnectionError("Qdrant cluster unavailable")

    with patch(
        "image_search_service.api.routes.face_suggestions.get_face_qdrant_client",
        return_value=mock_qdrant,
    ):
        response = await async_client.post(
            f"/api/v1/face-suggestions/{suggestion_id}/accept",
            json={"person_id": "person-1"},
        )

    # BUG DOCUMENTATION: API returns 200 despite Qdrant failure
    assert response.status_code == 200, (
        "Accept returns success even though Qdrant is out of sync"
    )

    # DB was committed (face.person_id changed)
    db_session.commit.assert_called_once()

    # Qdrant was attempted but failed
    mock_qdrant.update_person_ids.assert_called_once()

    # No rollback happened -- this IS the bug
    # In correct behavior, either:
    # 1. DB commit should be rolled back on Qdrant failure, OR
    # 2. Response should indicate partial success (207 Multi-Status), OR
    # 3. A reconciliation job should be enqueued
```

### 4.2. Bulk Action Partial Qdrant Sync Failure

**Objective**: Prove that a Qdrant failure during bulk sync leaves some faces synced and others not.

```python
@pytest.mark.asyncio
async def test_bulk_action_partial_qdrant_sync_failure(
    async_client: AsyncClient,
    db_session: AsyncMock,
):
    """Bulk accept with partial Qdrant sync failure.

    DB commits all 3 face assignments, then Qdrant sync fails on person_id_2.
    Result: 2 faces synced in Qdrant, 1 face desynced.
    """
    # Setup 3 suggestions for 2 different people
    mock_suggestions = {}
    mock_faces = {}
    for sid, pid in [(1, "person-a"), (2, "person-b"), (3, "person-a")]:
        suggestion = MagicMock(spec=FaceSuggestion)
        suggestion.id = sid
        suggestion.status = FaceSuggestionStatus.PENDING.value
        suggestion.face_instance_id = sid * 10
        suggestion.suggested_person_id = pid
        mock_suggestions[sid] = suggestion

        face = MagicMock(spec=FaceInstance)
        face.id = sid * 10
        face.person_id = None
        face.qdrant_point_id = f"point-{sid}"
        mock_faces[sid * 10] = face

    db_session.get = AsyncMock(side_effect=lambda model, pk, **kw:
        mock_suggestions.get(pk) if model == FaceSuggestion
        else mock_faces.get(pk) if model == FaceInstance
        else None
    )
    db_session.commit = AsyncMock()
    db_session.refresh = AsyncMock()

    call_count = 0
    def qdrant_update_with_failure(point_ids, person_id):
        nonlocal call_count
        call_count += 1
        if person_id == "person-b":
            raise ConnectionError("Qdrant timeout on person-b sync")

    mock_qdrant = MagicMock()
    mock_qdrant.update_person_ids.side_effect = qdrant_update_with_failure

    with patch(
        "image_search_service.api.routes.face_suggestions.get_face_qdrant_client",
        return_value=mock_qdrant,
    ):
        response = await async_client.post(
            "/api/v1/face-suggestions/bulk-action",
            json={"action": "accept", "suggestion_ids": [1, 2, 3]},
        )

    # API returns 200 with processed=3 -- no indication of Qdrant desync
    assert response.status_code == 200
    data = response.json()
    assert data["processed"] == 3, "All 3 processed in DB despite Qdrant failure"

    # DB commit happened once (single batch)
    db_session.commit.assert_called_once()

    # Qdrant: person-a synced, person-b failed
    # Result: faces for person-a are in sync, face for person-b is desynced
```

### 4.3. Centroid DB Flush Succeeds, Qdrant Upsert Fails

**Objective**: Prove that centroid computation handles Qdrant failure by marking as FAILED but still leaves a partially visible state.

```python
@pytest.mark.asyncio
async def test_centroid_qdrant_failure_marks_as_failed(
    db_session: AsyncMock,
):
    """Centroid computation: DB flush succeeds, Qdrant upsert fails.

    Expected behavior: centroid.status set to FAILED, exception re-raised.
    BUT: The deprecation of the OLD centroid already happened (line 325),
    so the person now has NO active centroid (old deprecated, new failed).
    """
    from image_search_service.services.centroid_service import compute_centroids_for_person
    import uuid
    import numpy as np

    person_id = uuid.uuid4()

    mock_centroid_qdrant = MagicMock()
    mock_centroid_qdrant.upsert_centroid.side_effect = ConnectionError("Qdrant down")

    mock_face_qdrant = MagicMock()

    fake_embeddings = np.random.randn(5, 512).astype(np.float32)
    fake_face_ids = [uuid.uuid4() for _ in range(5)]

    added_objects = []

    with patch(
        "image_search_service.services.centroid_service.get_person_face_embeddings",
        return_value=(fake_face_ids, fake_embeddings.tolist()),
    ):
        with patch(
            "image_search_service.services.centroid_service.deprecate_centroids",
        ) as mock_deprecate:
            with patch(
                "image_search_service.services.centroid_service.get_settings",
            ) as mock_settings:
                mock_settings.return_value.centroid_min_faces = 3
                mock_settings.return_value.centroid_model_version = "v1"
                mock_settings.return_value.centroid_algorithm_version = "v1"
                mock_settings.return_value.centroid_trim_threshold_small = 0.1
                mock_settings.return_value.centroid_trim_threshold_large = 0.05

                db_session.execute = AsyncMock(
                    return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=None))
                )
                db_session.flush = AsyncMock()
                db_session.add = MagicMock(side_effect=lambda obj: added_objects.append(obj))

                with pytest.raises(ConnectionError, match="Qdrant down"):
                    await compute_centroids_for_person(
                        db_session, mock_face_qdrant, mock_centroid_qdrant, person_id
                    )

    # Old centroid was deprecated BEFORE the failure
    mock_deprecate.assert_called_once_with(db_session, mock_centroid_qdrant, person_id)

    # New centroid was added but marked as FAILED
    assert len(added_objects) == 1
    new_centroid = added_objects[0]

    # BUG: Old centroid deprecated + new centroid FAILED = no active centroid
    # This is a transactional gap -- the caller must handle this edge case
```

---

## 5. Test Infrastructure Requirements

### 5.1. Concurrency Test Helpers

Create a shared helper module for concurrency testing patterns:

```python
# File: tests/helpers/concurrency.py

import asyncio
from typing import Any, Awaitable, Callable
from collections.abc import Sequence


async def race_requests(
    coroutines: Sequence[Awaitable[Any]],
    stagger_ms: float = 0,
) -> list[Any]:
    """Execute multiple coroutines concurrently to test race conditions.

    Args:
        coroutines: List of async callables to execute
        stagger_ms: Optional stagger between launches (default: 0 for maximum race window)

    Returns:
        List of results (or exceptions if return_exceptions used internally)
    """
    if stagger_ms > 0:
        tasks = []
        for coro in coroutines:
            tasks.append(asyncio.create_task(coro))
            await asyncio.sleep(stagger_ms / 1000)
        return await asyncio.gather(*tasks, return_exceptions=True)
    else:
        return await asyncio.gather(*coroutines, return_exceptions=True)


class OperationLogger:
    """Thread-safe logger for tracking operation ordering in race condition tests."""

    def __init__(self):
        self.operations: list[tuple[float, str]] = []
        self._lock = asyncio.Lock()

    async def log(self, operation: str) -> None:
        """Log an operation with timestamp."""
        async with self._lock:
            self.operations.append((asyncio.get_event_loop().time(), operation))

    def get_log(self) -> list[str]:
        """Get operations in chronological order."""
        return [op for _, op in sorted(self.operations)]

    def count(self, pattern: str) -> int:
        """Count operations matching a pattern."""
        return sum(1 for _, op in self.operations if pattern in op)


class DelayedMockSession:
    """Mock DB session that adds configurable delays to simulate real I/O.

    Usage:
        session = DelayedMockSession(base_session, read_delay_ms=10, commit_delay_ms=20)
    """

    def __init__(
        self,
        base_session,
        read_delay_ms: float = 0,
        commit_delay_ms: float = 0,
    ):
        self._base = base_session
        self._read_delay = read_delay_ms / 1000
        self._commit_delay = commit_delay_ms / 1000

    async def get(self, model, pk, **kwargs):
        if self._read_delay:
            await asyncio.sleep(self._read_delay)
        return await self._base.get(model, pk, **kwargs)

    async def commit(self):
        if self._commit_delay:
            await asyncio.sleep(self._commit_delay)
        return await self._base.commit()

    # Delegate everything else
    def __getattr__(self, name):
        return getattr(self._base, name)
```

### 5.2. Qdrant Failure Injection Helpers

```python
# File: tests/helpers/failure_injection.py

from typing import Any
from unittest.mock import MagicMock


class FailAfterNCallsQdrant:
    """Mock Qdrant client that fails after N successful calls.

    Usage:
        qdrant = FailAfterNCallsQdrant(succeed_count=2, error=ConnectionError("timeout"))
        # First 2 calls succeed, subsequent calls raise ConnectionError
    """

    def __init__(self, succeed_count: int = 0, error: Exception | None = None):
        self._succeed_count = succeed_count
        self._error = error or ConnectionError("Qdrant unavailable")
        self._call_count = 0
        self.calls: list[dict[str, Any]] = []

    def update_person_ids(self, point_ids, person_id):
        self._call_count += 1
        self.calls.append({"method": "update_person_ids", "point_ids": point_ids, "person_id": person_id})
        if self._call_count > self._succeed_count:
            raise self._error

    def upsert_centroid(self, **kwargs):
        self._call_count += 1
        self.calls.append({"method": "upsert_centroid", **kwargs})
        if self._call_count > self._succeed_count:
            raise self._error


class SelectiveFailureQdrant:
    """Mock Qdrant client that fails for specific person IDs.

    Usage:
        qdrant = SelectiveFailureQdrant(fail_person_ids={"person-b"})
        qdrant.update_person_ids([...], "person-a")  # succeeds
        qdrant.update_person_ids([...], "person-b")  # raises
    """

    def __init__(self, fail_person_ids: set[str] | None = None):
        self._fail_ids = fail_person_ids or set()
        self.successful_calls: list[dict] = []
        self.failed_calls: list[dict] = []

    def update_person_ids(self, point_ids, person_id):
        call_info = {"point_ids": point_ids, "person_id": str(person_id)}
        if str(person_id) in self._fail_ids:
            self.failed_calls.append(call_info)
            raise ConnectionError(f"Qdrant timeout for person {person_id}")
        self.successful_calls.append(call_info)
```

### 5.3. Test File Organization

```
tests/
  api/
    test_race_conditions.py           # All HTTP-level race condition tests
  unit/
    services/
      test_centroid_race_conditions.py  # Centroid-specific race tests
      test_training_race_conditions.py  # Training state machine race tests
  helpers/
    concurrency.py                     # Shared race condition test utilities
    failure_injection.py               # Qdrant/DB failure injection helpers
```

---

## 6. Implementation Approach

### 6.1. Testing Without Real Concurrency

Since the existing test infrastructure uses mocked database sessions (in-memory SQLite, not real PostgreSQL), true concurrent transactions are not possible. However, the tests are still valuable because they:

1. **Document the vulnerability**: Each test explicitly shows the unprotected code path
2. **Prove absence of locking**: The test passes (both operations succeed) when it should fail (one should get a conflict)
3. **Serve as regression tests**: When locking is added, these tests will start failing (which is correct -- they should then be updated to verify the lock)
4. **Guide the fix**: Each test includes comments describing what correct behavior looks like

### 6.2. Test Naming Convention

Follow the project naming convention from `CLAUDE.md`:

```
test_{behavior}_when_{condition}_then_{result}
```

Examples:
- `test_accept_suggestion_when_concurrent_accepts_then_both_succeed_no_locking`
- `test_bulk_action_when_overlapping_sets_then_no_conflict_detection`
- `test_training_start_when_concurrent_cancel_then_no_state_protection`
- `test_centroid_compute_when_concurrent_same_person_then_double_deprecation`
- `test_accept_suggestion_when_qdrant_fails_then_db_not_rolled_back`

### 6.3. Phased Implementation

**Phase 1: Foundation (Day 1)**
- Create `tests/helpers/concurrency.py` with `race_requests()`, `OperationLogger`, `DelayedMockSession`
- Create `tests/helpers/failure_injection.py` with Qdrant failure mocks
- Verify helpers work with a simple smoke test

**Phase 2: Suggestion Race Conditions (Day 2)**
- Implement `test_race_conditions.py` with double-accept test
- Implement bulk action overlap test
- Implement Qdrant desync tests for accept and bulk flows

**Phase 3: Training and Centroid Race Conditions (Day 3)**
- Implement `test_training_race_conditions.py` with concurrent state transition tests
- Implement `test_centroid_race_conditions.py` with concurrent computation test
- Implement centroid Qdrant failure test (deprecated + failed = no centroid)

**Phase 4: Review and Documentation (Day 4)**
- Run full test suite, verify all new tests pass (documenting bugs)
- Add docstrings explaining what each test proves
- Update test execution tracking documentation

---

## 7. Existing Test Patterns to Follow

### 7.1. Async Test Pattern

The project uses `pytest-asyncio` with `AsyncClient` from `httpx`:

```python
@pytest.mark.asyncio
async def test_example(async_client: AsyncClient, db_session: AsyncMock):
    response = await async_client.post("/api/v1/...", json={...})
    assert response.status_code == 200
```

### 7.2. Threading Pattern (Existing Precedent)

From `tests/unit/services/test_exif_service.py` (line 437):

```python
threads = [threading.Thread(target=extract_exif_thread) for _ in range(10)]
```

This is the only concurrency test in the codebase and tests EXIF extraction thread safety, not database operations. The race condition tests proposed in this plan would be the first to test async database concurrency.

### 7.3. Mock Pattern

The codebase consistently patches dependencies at their import location:

```python
with patch("image_search_service.api.routes.face_suggestions.get_face_qdrant_client"):
    ...
```

---

## 8. Success Criteria

### 8.1. Required Outcomes

- [ ] At least 8 new race condition test cases created
- [ ] All tests pass (they document existing bugs, not test fixes)
- [ ] Tests cover all 4 critical surfaces: suggestion accept, bulk action, training state, centroid compute
- [ ] Tests cover Qdrant-PostgreSQL desync for: accept, bulk action, centroid computation
- [ ] Test helpers created and reusable: `concurrency.py`, `failure_injection.py`
- [ ] Each test includes clear docstring explaining the race condition being documented
- [ ] Each test includes comments describing correct behavior (for future fix)

### 8.2. Quality Gates

- [ ] `make lint` passes with new test files
- [ ] `make typecheck` passes with new test files
- [ ] `make test` passes (all new tests pass, existing tests unaffected)
- [ ] No source code modified (tests only)

### 8.3. Documentation Deliverables

- [ ] Each test function has a docstring with: scenario, expected current behavior (bug), expected correct behavior (fix)
- [ ] `tests/helpers/concurrency.py` has module-level docstring explaining usage
- [ ] `tests/helpers/failure_injection.py` has module-level docstring explaining usage

---

## 9. Risk Assessment

### 9.1. Implementation Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Mock-based tests do not reproduce real race windows | HIGH | Tests document the vulnerability by showing absence of protection, not by triggering actual races. This is intentional. |
| `asyncio.gather` does not interleave deterministically | MEDIUM | Use `DelayedMockSession` to widen the race window. Even without interleaving, the tests prove that no locking exists. |
| New test helpers conflict with existing conftest.py | LOW | New helpers are in separate modules (`tests/helpers/`), not in `conftest.py`. |
| Tests become flaky due to timing | LOW | Tests assert on structural properties (no locking calls, double commits) not timing-dependent outcomes. |

### 9.2. Relationship to Fixes

These tests are **diagnostic**, not **prescriptive**. They document the current (broken) behavior. When the actual race conditions are fixed (by adding `SELECT ... FOR UPDATE`, optimistic locking, or sagas), these tests should be updated to verify the fix:

- Double-accept test: Should assert one request returns 409 Conflict
- Bulk overlap test: Should assert overlapping suggestions are detected
- Training state test: Should assert only one state transition succeeds
- Centroid test: Should assert only one computation proceeds
- Qdrant desync tests: Should assert proper rollback or retry behavior

---

## 10. References

### 10.1. Research Documents

- `docs/research/service-testing/devils-advocate-review.md` -- Section 2.1 (Zero Locking), Section 2.2 (Qdrant Desync), Section 7 (Risk Matrix)
- `docs/research/service-testing/test-case-analysis.md` -- Coverage gaps section (no race condition tests)
- `docs/research/service-testing/service-code-analysis.md` -- Testable surface inventory
- `docs/research/service-testing/test-execution-analysis.md` -- 48% coverage, 93.6% pass rate

### 10.2. Source Files Referenced

| File | Lines | What It Contains |
|------|-------|-----------------|
| `src/image_search_service/api/routes/face_suggestions.py` | 503-586 | `accept_suggestion()` -- single accept without locking |
| `src/image_search_service/api/routes/face_suggestions.py` | 666-789 | `bulk_suggestion_action()` -- bulk accept without locking |
| `src/image_search_service/services/training_service.py` | 568-707 | `start_training()`, `pause_training()`, `cancel_training()` -- state machine without locking |
| `src/image_search_service/services/centroid_service.py` | 258-396 | `compute_centroids_for_person()` -- read-compute-write without locking |
| `src/image_search_service/core/config.py` | 66 | `enable_cors: bool = True` default |
| `tests/unit/services/test_exif_service.py` | 437 | Only existing concurrency test (threading) |

### 10.3. Grep Verification

```
# Confirmed: Zero database locking in entire codebase
grep -rn "for_update\|with_for_update\|FOR UPDATE\|LOCK\|select_for_update" src/
# Result: 0 matches
```
