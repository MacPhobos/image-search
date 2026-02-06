# Plan 8: Add Optimistic Locking to Face Assignment

**Issue ID**: H5 (High Severity)
**Priority**: High
**Estimated Effort**: 4-6 hours
**Risk Level**: Medium (database migration + multiple code paths)
**Date**: 2026-02-06

---

## Problem Statement

All face-to-person assignment operations use a read-modify-write pattern without any concurrency protection. When two users (or a user and an automated job) attempt to assign the same face simultaneously, the last write wins silently -- there is no detection of the conflict, no error returned to the caller, and no audit trail of the overwritten assignment. This is particularly dangerous because the suggestion acceptance path (`face_suggestions.py`) does not update Qdrant at all after assigning, which means the PostgreSQL `person_id` and the Qdrant `person_id` payload can diverge permanently.

## Root Cause Analysis

The `FaceInstance` model (`db/models.py` line 459) has an `updated_at` column with `onupdate=func.now()` but no explicit version column. None of the assignment code paths use SQLAlchemy's `with_for_update()` (SELECT FOR UPDATE) or check `updated_at` before writing. The code follows a simple pattern:

1. `await db.get(FaceInstance, face_id)` -- reads the row
2. `face.person_id = person.id` -- modifies in memory
3. `await db.commit()` -- writes back

Between steps 1 and 3, another request can read the same row and write a different `person_id`. The second commit overwrites the first without any error.

## Affected Code Paths

There are **eight distinct code paths** that modify `face.person_id`:

| # | File | Line | Function | Context |
|---|------|------|----------|---------|
| 1 | `api/routes/faces.py` | 1644 | `assign_face_to_person()` | Single face assignment (UI "Assign" button) |
| 2 | `api/routes/faces.py` | 379 | Bulk cluster assignment | Assigns all faces in a cluster to a person |
| 3 | `api/routes/faces.py` | 1082 | Merge persons | Reassigns faces from source to target person |
| 4 | `api/routes/faces.py` | 1183 | Unassign faces | Sets `person_id = None` for batch of faces |
| 5 | `api/routes/faces.py` | 1387 | Move faces between persons | Reassigns faces to target person |
| 6 | `api/routes/faces.py` | 1789 | `unassign_face()` | Single face unassignment |
| 7 | `api/routes/face_suggestions.py` | 527 | `accept_suggestion()` | Accept single suggestion |
| 8 | `api/routes/face_suggestions.py` | 681 | `bulk_suggestion_action()` | Accept multiple suggestions in loop |

**Additional path** (admin import, lower risk):
| 9 | `services/admin_service.py` | 780 | Person metadata import | Assigns face during data import |

### Critical Secondary Bug in Path 7

The `accept_suggestion()` endpoint (path 7) is missing the Qdrant update entirely. After `face.person_id = suggestion.suggested_person_id` at line 527, the code commits to PostgreSQL at line 533 but never calls `qdrant.update_person_ids()`. This means:

- PostgreSQL says face belongs to Person A
- Qdrant payload still says face has no person (or belongs to someone else)
- Future similarity searches return incorrect results
- Future clustering may re-cluster this face as "unknown"

This is documented in the research as "Bomb #1" and should be fixed as part of this plan.

## Race Condition Scenario

```
Time    User A (UI)                      Background Job (RQ Worker)
----    -----------                      --------------------------
T1      GET face #123                    GET face #123
        person_id = NULL                 person_id = NULL

T2      face.person_id = Alice           face.person_id = Bob
        (via assign_face_to_person)      (via accept_suggestion)

T3      UPDATE Qdrant: Alice             [NO Qdrant update - bug!]

T4      COMMIT -> face.person_id=Alice   COMMIT -> face.person_id=Bob
                                         (silently overwrites Alice)

T5      Result: PostgreSQL says Bob, Qdrant says Alice
        User A sees "assigned to Alice" but DB says Bob
```

---

## Solution Options

### Option A: Optimistic Locking with Version Column (RECOMMENDED)

Add a `version` integer column to `FaceInstance`. Every assignment operation reads the current version, then uses `WHERE version = :expected_version` in the UPDATE. If another process modified the row, the version will have incremented and the UPDATE will affect 0 rows, allowing the application to detect the conflict and return an appropriate error.

**Pros**: No lock contention, works well with async SQLAlchemy, standard pattern
**Cons**: Requires migration, requires updating all 8+ code paths

### Option B: Pessimistic Locking with SELECT FOR UPDATE

Use `await db.get(FaceInstance, face_id, with_for_update=True)` to acquire a row-level lock at read time. This prevents concurrent reads from proceeding until the first transaction commits.

**Pros**: No migration needed, simpler to reason about
**Cons**: Blocks concurrent requests (reduces throughput), can cause deadlocks with bulk operations, does not work well with the Qdrant update between read and commit

### Option C: Database Constraint with Updated-At Check

Use the existing `updated_at` column as a poor-man's version: include `WHERE updated_at = :expected_updated_at` in the UPDATE. This is similar to Option A but uses timestamp precision instead of an integer counter.

**Pros**: No migration needed
**Cons**: Timestamp precision issues (two writes in same millisecond), `updated_at` has `server_default=func.now()` with `onupdate=func.now()` which is handled by SQLAlchemy ORM, making manual WHERE checks awkward

### Recommendation: Option A (Optimistic Locking)

Option A is the industry standard for this pattern. It is non-blocking, reliable, and integrates cleanly with SQLAlchemy's `version_id_col` feature which handles the WHERE clause automatically.

---

## Implementation Steps

### Step 1: Add `version` column to FaceInstance model (15 minutes)

**File**: `image-search-service/src/image_search_service/db/models.py`

```python
# CURRENT (lines 459-519):
class FaceInstance(Base):
    """Face instance detected in an image asset."""
    __tablename__ = "face_instances"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    # ... existing columns ...
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )
```

```python
# AFTER:
class FaceInstance(Base):
    """Face instance detected in an image asset."""
    __tablename__ = "face_instances"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    # ... existing columns ...
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )

    # Optimistic locking version counter
    version: Mapped[int] = mapped_column(
        Integer, nullable=False, server_default="1", default=1
    )

    __mapper_args__ = {
        "version_id_col": version,
    }
```

The `version_id_col` tells SQLAlchemy to automatically:
- Include `WHERE version = :current_version` on every UPDATE
- Increment `version` to `version + 1` on every UPDATE
- Raise `StaleDataError` if the UPDATE affects 0 rows

### Step 2: Create database migration (10 minutes)

```bash
cd image-search-service

# Verify single Alembic head
uv run alembic heads

# Generate migration
uv run alembic revision --autogenerate -m "add version column to face_instances for optimistic locking"

# Review the generated migration
```

The generated migration should look like:

```python
"""add version column to face_instances for optimistic locking

This adds an integer version column to face_instances to enable optimistic
locking. All face assignment operations will check this version to detect
concurrent modifications and prevent silent overwrites.
"""

from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    op.add_column(
        "face_instances",
        sa.Column("version", sa.Integer(), nullable=False, server_default="1"),
    )

def downgrade() -> None:
    op.drop_column("face_instances", "version")
```

Run and verify:

```bash
make migrate
make test
```

### Step 3: Add conflict handling to `assign_face_to_person()` (20 minutes)

**File**: `image-search-service/src/image_search_service/api/routes/faces.py`
**Function**: `assign_face_to_person()` starting at line 1617

```python
# CURRENT (lines 1631-1676, simplified):
face = await db.get(FaceInstance, face_id)
if not face:
    raise HTTPException(status_code=404, detail=f"Face {face_id} not found")

person = await db.get(Person, request.person_id)
if not person:
    raise HTTPException(status_code=404, detail=f"Person {request.person_id} not found")

previous_person_id = face.person_id
face.person_id = person.id

# ... Qdrant update ...
# ... Audit event ...
await db.commit()
```

```python
# AFTER:
from sqlalchemy.orm.exc import StaleDataError

face = await db.get(FaceInstance, face_id)
if not face:
    raise HTTPException(status_code=404, detail=f"Face {face_id} not found")

person = await db.get(Person, request.person_id)
if not person:
    raise HTTPException(status_code=404, detail=f"Person {request.person_id} not found")

previous_person_id = face.person_id
face.person_id = person.id

try:
    # ... Qdrant update (existing code) ...
    # ... Audit event (existing code) ...
    await db.commit()
except StaleDataError:
    await db.rollback()
    raise HTTPException(
        status_code=409,
        detail=(
            f"Face {face_id} was modified by another operation. "
            "Please refresh and try again."
        ),
    )
```

### Step 4: Add conflict handling to `accept_suggestion()` AND fix missing Qdrant update (30 minutes)

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`
**Function**: `accept_suggestion()` starting at line 503

```python
# CURRENT (lines 521-534):
face = await db.get(FaceInstance, suggestion.face_instance_id)
if not face:
    raise HTTPException(status_code=404, detail="Face instance not found")

# Assign face to person
face.person_id = suggestion.suggested_person_id

# Update suggestion status
suggestion.status = FaceSuggestionStatus.ACCEPTED.value
suggestion.reviewed_at = datetime.now(UTC)

await db.commit()
await db.refresh(suggestion)
```

```python
# AFTER:
from sqlalchemy.orm.exc import StaleDataError
from image_search_service.vector.face_qdrant import get_face_qdrant_client

face = await db.get(FaceInstance, suggestion.face_instance_id)
if not face:
    raise HTTPException(status_code=404, detail="Face instance not found")

# Assign face to person
face.person_id = suggestion.suggested_person_id

# Update suggestion status
suggestion.status = FaceSuggestionStatus.ACCEPTED.value
suggestion.reviewed_at = datetime.now(UTC)

try:
    # Update Qdrant payload to match PostgreSQL (BUG FIX: was missing)
    qdrant = get_face_qdrant_client()
    if face.qdrant_point_id and qdrant.point_exists(face.qdrant_point_id):
        qdrant.update_person_ids(
            [face.qdrant_point_id], suggestion.suggested_person_id
        )

    await db.commit()
    await db.refresh(suggestion)
except StaleDataError:
    await db.rollback()
    raise HTTPException(
        status_code=409,
        detail=(
            f"Face {suggestion.face_instance_id} was modified by another operation. "
            "Please refresh and try again."
        ),
    )
```

### Step 5: Add conflict handling to `bulk_suggestion_action()` (20 minutes)

**File**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`
**Function**: `bulk_suggestion_action()` around line 676

```python
# CURRENT (lines 676-695):
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

```python
# AFTER:
from sqlalchemy.orm.exc import StaleDataError

try:
    if request.action == "accept":
        face = await db.get(FaceInstance, suggestion.face_instance_id)
        if face:
            face.person_id = suggestion.suggested_person_id
            # Update Qdrant payload (BUG FIX: was missing)
            try:
                qdrant = get_face_qdrant_client()
                if face.qdrant_point_id and qdrant.point_exists(face.qdrant_point_id):
                    qdrant.update_person_ids(
                        [face.qdrant_point_id], suggestion.suggested_person_id
                    )
            except Exception as qe:
                logger.warning(
                    f"Failed to update Qdrant for face {suggestion.face_instance_id}: {qe}"
                )
        suggestion.status = FaceSuggestionStatus.ACCEPTED.value
        affected_person_ids.add(suggestion.suggested_person_id)
    else:  # reject
        suggestion.status = FaceSuggestionStatus.REJECTED.value

    suggestion.reviewed_at = datetime.now(UTC)
    processed += 1

except Exception as e:
    failed += 1
    errors.append(f"Error processing suggestion {suggestion_id}: {str(e)}")

try:
    await db.commit()
except StaleDataError:
    await db.rollback()
    # Re-raise as HTTP error since bulk operations should retry
    raise HTTPException(
        status_code=409,
        detail=(
            f"One or more faces were modified by concurrent operations. "
            f"Processed {processed} of {len(request.suggestion_ids)} suggestions. "
            "Please refresh and retry the remaining items."
        ),
    )
```

### Step 6: Add conflict handling to bulk assignment paths in faces.py (20 minutes)

The four bulk assignment paths in `faces.py` (cluster assign, merge, unassign, move) all follow a similar loop pattern:

```python
for face in faces:
    face.person_id = target.id  # or None
```

For these, wrap the `await db.commit()` that follows each loop:

**Pattern for each bulk path** (lines 379, 1082, 1183, 1387):

```python
# AFTER each bulk loop's commit:
try:
    await db.commit()
except StaleDataError:
    await db.rollback()
    raise HTTPException(
        status_code=409,
        detail=(
            "One or more faces were modified by a concurrent operation. "
            "Please refresh the page and retry."
        ),
    )
```

### Step 7: Add conflict handling to single unassign (10 minutes)

**File**: `image-search-service/src/image_search_service/api/routes/faces.py`
**Function**: Around line 1789

```python
# CURRENT:
face.person_id = None
# ... Qdrant update ...
await db.commit()
```

```python
# AFTER:
face.person_id = None
try:
    # ... Qdrant update (existing code) ...
    await db.commit()
except StaleDataError:
    await db.rollback()
    raise HTTPException(
        status_code=409,
        detail=(
            f"Face was modified by another operation. "
            "Please refresh and try again."
        ),
    )
```

### Step 8: Handle 409 responses in the UI (30 minutes)

**File**: `image-search-ui/src/lib/api/faces.ts`

Add handling for HTTP 409 status codes in the relevant API functions. The UI should display a toast notification and prompt the user to refresh.

```typescript
// In acceptSuggestion(), rejectSuggestion(), assignFace(), etc.:
if (response.status === 409) {
    throw new ApiError(
        'This face was modified by another operation. Please refresh and try again.',
        409
    );
}
```

**File**: `image-search-ui/src/routes/faces/suggestions/+page.svelte`

In the error handling sections, display a clear message:

```typescript
} catch (e) {
    if (e instanceof ApiError && e.status === 409) {
        toast.error('Conflict: Face was modified elsewhere. Refreshing...');
        await loadSuggestions(); // Auto-refresh on conflict
    } else {
        error = e instanceof Error ? e.message : 'Failed to accept suggestion';
    }
}
```

---

## Verification

### Unit Tests

Add tests for the optimistic locking behavior:

```python
# tests/api/test_face_assignment_locking.py

import pytest
from sqlalchemy.orm.exc import StaleDataError


async def test_concurrent_assignment_detected(db_session, face_instance, person_a, person_b):
    """Verify that concurrent face assignment raises StaleDataError."""
    # Simulate two sessions reading the same face
    face_1 = await db_session.get(FaceInstance, face_instance.id)
    face_2 = await db_session.get(FaceInstance, face_instance.id)

    # First assignment succeeds
    face_1.person_id = person_a.id
    await db_session.commit()

    # Second assignment should detect the conflict
    face_2.person_id = person_b.id
    with pytest.raises(StaleDataError):
        await db_session.commit()


async def test_assignment_returns_409_on_conflict(client, face_instance, person_a, person_b):
    """Verify that the API returns 409 on concurrent modification."""
    # First: assign via API
    response1 = await client.post(
        f"/api/v1/faces/faces/{face_instance.id}/assign",
        json={"person_id": str(person_a.id)},
    )
    assert response1.status_code == 200

    # Simulate stale read by manually decrementing version
    # (In real concurrency, this happens naturally)
    # Test the HTTP 409 response path
```

### Manual Testing

1. Open two browser tabs on the Suggestions page
2. In tab 1, click "Accept" on a suggestion
3. In tab 2, click "Accept" on the same suggestion before tab 1's request completes
4. Tab 2 should display a conflict toast and auto-refresh

### Integration Verification

```bash
cd image-search-service
make test
make typecheck
```

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Migration fails on large `face_instances` table | Low | High | `server_default="1"` avoids needing to update existing rows; the ADD COLUMN with DEFAULT is fast in PostgreSQL 11+ |
| `StaleDataError` propagates unexpectedly | Medium | Medium | Wrap every `db.commit()` that touches `FaceInstance` with StaleDataError handling |
| Qdrant and PostgreSQL get out of sync during conflict | Medium | High | Always update Qdrant BEFORE `db.commit()`; on StaleDataError, Qdrant may have stale data but the rollback prevents PostgreSQL inconsistency. Consider: Qdrant is eventually consistent anyway |
| Bulk operations become slower | Low | Low | Version checking adds negligible overhead; PostgreSQL increments an integer column |
| UI does not handle 409 gracefully | Medium | Low | Auto-refresh on 409 is the safest UX pattern; user sees fresh state |
| Breaking change for API clients | Low | Medium | 409 is a new response code; existing clients that do not handle it will see an error rather than silent data corruption (which is an improvement) |

## Dependencies

- **Database migration**: Requires running `make migrate` before deploying the code changes
- **Sequential deployment**: Deploy migration first, then code changes. The `server_default="1"` ensures the column exists with a valid value before the new code reads it.
- **No UI deployment dependency**: The UI changes (409 handling) can be deployed independently after the backend.

## Estimated Effort Breakdown

| Task | Time |
|------|------|
| Model change + migration | 25 min |
| `assign_face_to_person()` conflict handling | 20 min |
| `accept_suggestion()` conflict handling + Qdrant fix | 30 min |
| `bulk_suggestion_action()` conflict handling + Qdrant fix | 20 min |
| Bulk paths in `faces.py` (4 locations) | 30 min |
| Single `unassign_face()` conflict handling | 10 min |
| UI 409 handling | 30 min |
| Tests | 45 min |
| Verification and review | 15 min |
| **Total** | **~4 hours** |

## Secondary Bug Fix Included

This plan includes fixing the missing Qdrant update in `accept_suggestion()` (line 527-533) and `bulk_suggestion_action()` (line 681). This was identified in the research as "Bomb #1" -- the highest-risk data integrity issue. Without this fix:

- Accepting a suggestion updates PostgreSQL `person_id` but leaves Qdrant unchanged
- Future similarity searches return wrong results
- Future clustering may re-assign the face to a different person
- The optimistic locking alone would not fix this Qdrant divergence

The Qdrant update is added in Steps 4 and 5 above, using the same pattern already established in `assign_face_to_person()` (line 1660).
