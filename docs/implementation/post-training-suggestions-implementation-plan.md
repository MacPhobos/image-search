# Post-Training Suggestions Implementation Plan

> **Date**: 2026-01-12
> **Feature**: Queue `propagate_person_label_multiproto_job` after training completion
> **Configurable**: Admin Panel setting (all persons vs. top N persons)
> **Status**: ✅ Implemented + Enhanced with centroid mode (2026-01-17)

> **📌 UPDATE**: This feature now supports **centroid-based suggestions** for faster performance (~50x speedup). See [post-training-suggestions-centroid.md](./post-training-suggestions-centroid.md) for details on the centroid enhancement.

---

## 🎯 Objective

After training session completes, automatically queue multi-prototype suggestion jobs for persons to generate face suggestions. This addresses the gap where manual assignment generates suggestions but training pipeline does not.

**User Requirements**:
- Default: Generate suggestions for **all persons**
- Configurable: Option to limit to **top N persons** (by labeled face count)
- Admin Panel: Settings UI for configuration
- Database-backed: Persist settings across sessions

---

## 📊 Current State vs. Desired State

### Current Behavior

```python
# detect_faces_for_session_job (face_jobs.py)
detect_faces() → assign_new_faces() → clustering()
# ❌ NO suggestion generation queued
```

### Desired Behavior

```python
# detect_faces_for_session_job (face_jobs.py)
detect_faces() → assign_new_faces() → clustering()
↓
# ✅ NEW: Queue multi-proto suggestions based on settings
if config.post_training_suggestions_mode == "all":
    queue multi-proto jobs for ALL persons
elif config.post_training_suggestions_mode == "top_n":
    queue multi-proto jobs for TOP N persons (by face count)
```

---

## 🗂️ Architecture Overview

### Component Stack

```
┌─────────────────────────────────────────────────────┐
│ Frontend: Admin Panel Settings UI                  │
│ (image-search-ui/src/routes/admin/settings/)       │
└────────────────┬────────────────────────────────────┘
                 │ PUT /api/v1/config/face-matching
                 ↓
┌─────────────────────────────────────────────────────┐
│ Backend: Settings API + ConfigService              │
│ (image-search-service/src/.../api/routes/config.py)│
└────────────────┬────────────────────────────────────┘
                 │ Validate & persist to DB
                 ↓
┌─────────────────────────────────────────────────────┐
│ Database: system_configs table                     │
│ - post_training_suggestions_mode (enum: all|top_n) │
│ - post_training_suggestions_top_n_count (int: 1-100)│
└────────────────┬────────────────────────────────────┘
                 │ Read during training
                 ↓
┌─────────────────────────────────────────────────────┐
│ Background Job: detect_faces_for_session_job       │
│ Reads settings → Queues multi-proto jobs           │
└─────────────────────────────────────────────────────┘
```

---

## 📋 Implementation Phases

### Phase 1: Database Schema (Backend)

**File**: `image-search-service/db/migrations/versions/{timestamp}_add_post_training_suggestion_settings.py`

**Migration**:
```python
"""Add post-training suggestion settings

Revision ID: {auto_generated}
Revises: {current_head}
Create Date: 2026-01-12
"""

from alembic import op

def upgrade() -> None:
    # Insert mode setting (enum: all | top_n)
    op.execute("""
        INSERT INTO system_configs (key, value, data_type, allowed_values, category, description)
        VALUES (
            'post_training_suggestions_mode',
            'all',
            'string',
            '["all", "top_n"]',
            'face_matching',
            'Mode for post-training suggestion generation: all persons or top N by face count'
        )
        ON CONFLICT (key) DO NOTHING;
    """)

    # Insert top N count setting (int: 1-100)
    op.execute("""
        INSERT INTO system_configs (key, value, data_type, min_value, max_value, category, description)
        VALUES (
            'post_training_suggestions_top_n_count',
            '10',
            'int',
            '1',
            '100',
            'face_matching',
            'Number of top persons to generate suggestions for (only used when mode is top_n)'
        )
        ON CONFLICT (key) DO NOTHING;
    """)

def downgrade() -> None:
    op.execute("DELETE FROM system_configs WHERE key = 'post_training_suggestions_mode'")
    op.execute("DELETE FROM system_configs WHERE key = 'post_training_suggestions_top_n_count'")
```

**Commands**:
```bash
cd image-search-service
make makemigrations  # Generate migration file
make migrate         # Apply migration
```

**Verification**:
```sql
SELECT key, value, data_type, allowed_values, min_value, max_value
FROM system_configs
WHERE key LIKE 'post_training_suggestions%';
```

---

### Phase 2: Backend Configuration Service (Backend)

**File**: `image-search-service/src/image_search_service/services/config_service.py`

**Changes**: Add default values for new settings

```python
# Around line 40 (DEFAULTS dictionary)
DEFAULTS = {
    # ... existing defaults ...

    # Post-training suggestions (NEW)
    "post_training_suggestions_mode": "all",
    "post_training_suggestions_top_n_count": 10,
}
```

**File**: `image-search-service/src/image_search_service/api/schemas.py`

**Changes**: Add Pydantic schemas for settings

```python
from typing import Literal

class PostTrainingSuggestionsConfig(BaseModel):
    """Post-training suggestion generation settings."""

    mode: Literal["all", "top_n"] = Field(
        default="all",
        description="Generate suggestions for all persons or top N by face count"
    )
    top_n_count: int = Field(
        default=10,
        ge=1,
        le=100,
        description="Number of top persons (only used when mode is 'top_n')"
    )

class FaceMatchingConfigUpdate(BaseModel):
    """Update request for face matching settings."""

    # ... existing fields ...

    # Post-training suggestions (NEW)
    post_training_suggestions_mode: Optional[Literal["all", "top_n"]] = None
    post_training_suggestions_top_n_count: Optional[int] = Field(None, ge=1, le=100)

class FaceMatchingConfigResponse(BaseModel):
    """Response model for face matching settings."""

    # ... existing fields ...

    # Post-training suggestions (NEW)
    post_training_suggestions_mode: str
    post_training_suggestions_top_n_count: int
```

---

### Phase 3: Backend API Endpoints (Backend)

**File**: `image-search-service/src/image_search_service/api/routes/config.py`

**Changes**: Update `/api/v1/config/face-matching` endpoints

```python
@router.get("/face-matching", response_model=FaceMatchingConfigResponse)
async def get_face_matching_config(
    config_service: ConfigService = Depends(get_config_service),
) -> FaceMatchingConfigResponse:
    """Get face matching configuration settings."""

    # ... existing field reads ...

    # Post-training suggestions (NEW)
    post_training_suggestions_mode = await config_service.get("post_training_suggestions_mode")
    post_training_suggestions_top_n_count = await config_service.get_int("post_training_suggestions_top_n_count")

    return FaceMatchingConfigResponse(
        # ... existing fields ...
        post_training_suggestions_mode=post_training_suggestions_mode,
        post_training_suggestions_top_n_count=post_training_suggestions_top_n_count,
    )


@router.put("/face-matching", response_model=FaceMatchingConfigResponse)
async def update_face_matching_config(
    updates: FaceMatchingConfigUpdate,
    config_service: ConfigService = Depends(get_config_service),
) -> FaceMatchingConfigResponse:
    """Update face matching configuration settings."""

    # ... existing field updates ...

    # Post-training suggestions (NEW)
    if updates.post_training_suggestions_mode is not None:
        await config_service.set(
            "post_training_suggestions_mode",
            updates.post_training_suggestions_mode
        )

    if updates.post_training_suggestions_top_n_count is not None:
        await config_service.set(
            "post_training_suggestions_top_n_count",
            str(updates.post_training_suggestions_top_n_count)
        )

    # Return updated config
    return await get_face_matching_config(config_service)
```

**Testing**:
```bash
# Get current settings
curl http://localhost:8000/api/v1/config/face-matching | jq

# Update to "all" mode
curl -X PUT http://localhost:8000/api/v1/config/face-matching \
  -H "Content-Type: application/json" \
  -d '{"post_training_suggestions_mode": "all"}'

# Update to "top_n" mode with count
curl -X PUT http://localhost:8000/api/v1/config/face-matching \
  -H "Content-Type: application/json" \
  -d '{
    "post_training_suggestions_mode": "top_n",
    "post_training_suggestions_top_n_count": 15
  }'
```

---

### Phase 4: Training Job Logic (Backend)

**File**: `image-search-service/src/image_search_service/jobs/face_jobs.py`

**Changes**: Update `detect_faces_for_session_job` to queue multi-proto jobs

```python
from image_search_service.services.sync_config_service import SyncConfigService

def detect_faces_for_session_job(
    session_id: str,
    limit: int | None = None,
    skip_clustering: bool = False,
) -> dict[str, Any]:
    """
    Detect faces for a training session.

    NEW: After clustering, queue multi-prototype suggestion jobs
    based on post_training_suggestions_mode setting.
    """

    # ... existing detection, assignment, clustering logic ...

    # ============================================================
    # NEW: Post-Training Suggestion Generation
    # ============================================================

    # Get configuration
    sync_config = SyncConfigService()
    suggestions_mode = sync_config.get("post_training_suggestions_mode", "all")
    top_n_count = sync_config.get_int("post_training_suggestions_top_n_count", 10)

    logger.info(
        "Queuing post-training suggestions",
        extra={
            "session_id": session_id,
            "mode": suggestions_mode,
            "top_n_count": top_n_count if suggestions_mode == "top_n" else None,
        }
    )

    # Get persons to generate suggestions for
    if suggestions_mode == "all":
        # Get ALL persons with at least 1 labeled face
        persons_query = (
            db.query(Person.id, func.count(Face.id).label("face_count"))
            .join(Face, Face.person_id == Person.id)
            .group_by(Person.id)
            .having(func.count(Face.id) > 0)
            .order_by(func.count(Face.id).desc())
        )
    else:  # top_n
        # Get TOP N persons by labeled face count
        persons_query = (
            db.query(Person.id, func.count(Face.id).label("face_count"))
            .join(Face, Face.person_id == Person.id)
            .group_by(Person.id)
            .having(func.count(Face.id) > 0)
            .order_by(func.count(Face.id).desc())
            .limit(top_n_count)
        )

    persons = persons_query.all()

    # Queue multi-prototype suggestion jobs
    from rq import Queue
    from redis import Redis
    from image_search_service.config import get_settings

    settings = get_settings()
    redis_client = Redis.from_url(settings.redis_url)
    queue = Queue("default", connection=redis_client)

    jobs_queued = 0
    for person in persons:
        job = queue.enqueue(
            "image_search_service.jobs.face_jobs.propagate_person_label_multiproto_job",
            person_id=str(person.id),
            max_suggestions=50,  # Standard limit
            min_confidence=0.7,  # Standard threshold
            job_timeout="10m",
        )
        jobs_queued += 1
        logger.info(
            "Queued multi-proto suggestion job",
            extra={
                "person_id": str(person.id),
                "face_count": person.face_count,
                "job_id": job.id,
            }
        )

    logger.info(
        "Post-training suggestion jobs queued",
        extra={
            "session_id": session_id,
            "jobs_queued": jobs_queued,
            "mode": suggestions_mode,
        }
    )

    # Return existing result with new stats
    return {
        # ... existing return values ...
        "suggestions_jobs_queued": jobs_queued,
        "suggestions_mode": suggestions_mode,
    }
```

**Testing**:
```bash
# Create training session and trigger job
curl -X POST http://localhost:8000/api/v1/faces/sessions \
  -H "Content-Type: application/json" \
  -d '{"description": "Test post-training suggestions"}'

# Monitor queue
redis-cli
> LLEN rq:queue:default  # Should show queued jobs

# Check logs for "Queuing post-training suggestions" messages
```

---

### Phase 5: Frontend API Client (Frontend)

**File**: `image-search-ui/src/lib/api/admin.ts`

**Changes**: Add TypeScript types and API functions

```typescript
// Add to existing types
export interface PostTrainingSuggestionsConfig {
  mode: 'all' | 'top_n';
  top_n_count: number;
}

export interface FaceMatchingConfig {
  // ... existing fields ...

  // Post-training suggestions
  post_training_suggestions_mode: 'all' | 'top_n';
  post_training_suggestions_top_n_count: number;
}

export interface FaceMatchingConfigUpdate {
  // ... existing fields ...

  // Post-training suggestions
  post_training_suggestions_mode?: 'all' | 'top_n';
  post_training_suggestions_top_n_count?: number;
}

// Existing functions already handle these new fields via generic update
```

---

### Phase 6: Frontend Admin Panel UI (Frontend)

**File**: `image-search-ui/src/routes/admin/settings/FaceMatchingSettings.svelte`

**Changes**: Add UI section for post-training suggestions

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import type { FaceMatchingConfig, FaceMatchingConfigUpdate } from '$lib/api/admin';
  import { getFaceMatchingConfig, updateFaceMatchingConfig } from '$lib/api/admin';

  // ... existing state ...

  let postTrainingSuggestionsMode = $state<'all' | 'top_n'>('all');
  let postTrainingSuggestionsTopNCount = $state<number>(10);

  // ... existing validation states ...

  let postTrainingTopNCountError = $derived(
    postTrainingSuggestionsMode === 'top_n' &&
    (postTrainingSuggestionsTopNCount < 1 || postTrainingSuggestionsTopNCount > 100)
      ? 'Must be between 1 and 100'
      : null
  );

  let hasErrors = $derived(
    // ... existing error checks ...
    postTrainingTopNCountError !== null
  );

  async function loadConfig() {
    try {
      loading = true;
      const config = await getFaceMatchingConfig();

      // ... existing field assignments ...

      // Post-training suggestions
      postTrainingSuggestionsMode = config.post_training_suggestions_mode;
      postTrainingSuggestionsTopNCount = config.post_training_suggestions_top_n_count;

      saveStatus = null;
    } catch (error) {
      console.error('Failed to load config:', error);
      saveStatus = 'error';
    } finally {
      loading = false;
    }
  }

  async function handleSave() {
    if (hasErrors) return;

    try {
      saving = true;

      const updates: FaceMatchingConfigUpdate = {
        // ... existing field updates ...

        // Post-training suggestions
        post_training_suggestions_mode: postTrainingSuggestionsMode,
        post_training_suggestions_top_n_count: postTrainingSuggestionsTopNCount,
      };

      await updateFaceMatchingConfig(updates);
      saveStatus = 'success';
      setTimeout(() => saveStatus = null, 3000);
    } catch (error) {
      console.error('Failed to save config:', error);
      saveStatus = 'error';
    } finally {
      saving = false;
    }
  }

  onMount(() => {
    loadConfig();
  });
</script>

<!-- ... existing sections ... -->

<!-- NEW: Post-Training Suggestions Section -->
<section class="config-section">
  <h3>Post-Training Suggestions</h3>
  <p class="section-description">
    Configure automatic suggestion generation after training sessions complete.
    This helps discover similar faces for persons with many labeled faces.
  </p>

  <div class="config-group">
    <label class="config-label">Suggestion Generation Mode</label>

    <div class="radio-group">
      <label class="radio-label">
        <input
          type="radio"
          value="all"
          bind:group={postTrainingSuggestionsMode}
          disabled={saving}
        />
        <span class="radio-text">
          <strong>All Persons</strong>
          <span class="radio-description">
            Generate suggestions for all persons with labeled faces (recommended for comprehensive matching)
          </span>
        </span>
      </label>

      <label class="radio-label">
        <input
          type="radio"
          value="top_n"
          bind:group={postTrainingSuggestionsMode}
          disabled={saving}
        />
        <span class="radio-text">
          <strong>Top N Persons</strong>
          <span class="radio-description">
            Generate suggestions only for top N persons by face count (faster for large databases)
          </span>
        </span>
      </label>
    </div>

    {#if postTrainingSuggestionsMode === 'top_n'}
      <div class="conditional-input">
        <label for="post-training-top-n-count" class="config-label">
          Number of Top Persons
          <span class="config-hint">
            How many persons to generate suggestions for (sorted by face count)
          </span>
        </label>

        <input
          id="post-training-top-n-count"
          type="number"
          min="1"
          max="100"
          bind:value={postTrainingSuggestionsTopNCount}
          disabled={saving}
          class:error={postTrainingTopNCountError}
        />

        {#if postTrainingTopNCountError}
          <span class="error-message">{postTrainingTopNCountError}</span>
        {/if}

        <div class="config-info">
          <strong>Example:</strong> Setting this to 10 will generate suggestions for the 10 persons
          with the most labeled faces after each training session completes.
        </div>
      </div>
    {/if}
  </div>

  <div class="config-info">
    <strong>Note:</strong> Suggestion generation runs as background jobs and may take several minutes
    depending on database size and number of persons. Check the queue status in the Admin Panel
    to monitor progress.
  </div>
</section>

<!-- ... existing save button ... -->

<style>
  /* ... existing styles ... */

  .radio-group {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-top: 0.5rem;
  }

  .radio-label {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    padding: 1rem;
    border: 1px solid var(--border-color);
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.2s;
  }

  .radio-label:hover {
    background-color: var(--hover-bg);
    border-color: var(--primary-color);
  }

  .radio-label input[type='radio'] {
    margin-top: 0.25rem;
  }

  .radio-text {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .radio-description {
    font-size: 0.875rem;
    color: var(--text-muted);
  }

  .conditional-input {
    margin-top: 1rem;
    padding-left: 2rem;
    border-left: 3px solid var(--primary-color);
  }

  .config-info {
    margin-top: 0.5rem;
    padding: 0.75rem;
    background-color: var(--info-bg);
    border-radius: 6px;
    font-size: 0.875rem;
    color: var(--text-muted);
  }

  .error-message {
    display: block;
    margin-top: 0.25rem;
    font-size: 0.875rem;
    color: var(--error-color);
  }

  input.error {
    border-color: var(--error-color);
  }
</style>
```

---

### Phase 7: Testing Strategy

#### Backend Tests

**File**: `image-search-service/tests/api/test_config.py`

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_get_post_training_suggestions_defaults(client: AsyncClient):
    """Test default post-training suggestion settings."""
    response = await client.get("/api/v1/config/face-matching")
    assert response.status_code == 200
    data = response.json()
    assert data["post_training_suggestions_mode"] == "all"
    assert data["post_training_suggestions_top_n_count"] == 10

@pytest.mark.asyncio
async def test_update_post_training_suggestions_to_top_n(client: AsyncClient):
    """Test updating to top_n mode."""
    response = await client.put(
        "/api/v1/config/face-matching",
        json={
            "post_training_suggestions_mode": "top_n",
            "post_training_suggestions_top_n_count": 15,
        }
    )
    assert response.status_code == 200
    data = response.json()
    assert data["post_training_suggestions_mode"] == "top_n"
    assert data["post_training_suggestions_top_n_count"] == 15

@pytest.mark.asyncio
async def test_invalid_top_n_count_rejected(client: AsyncClient):
    """Test validation rejects out-of-range values."""
    response = await client.put(
        "/api/v1/config/face-matching",
        json={"post_training_suggestions_top_n_count": 200}  # > max 100
    )
    assert response.status_code == 422  # Validation error
```

**File**: `image-search-service/tests/jobs/test_face_jobs_post_training_suggestions.py`

```python
import pytest
from unittest.mock import patch, MagicMock
from image_search_service.jobs.face_jobs import detect_faces_for_session_job

@pytest.mark.asyncio
async def test_post_training_suggestions_all_mode(db_session, mock_redis_queue):
    """Test that 'all' mode queues jobs for all persons."""
    # Setup: Create session, persons, faces
    session = create_test_session(db_session)
    person1 = create_test_person(db_session, name="Person 1")
    person2 = create_test_person(db_session, name="Person 2")
    create_test_faces(db_session, person1, count=10)
    create_test_faces(db_session, person2, count=5)

    # Mock config service to return 'all' mode
    with patch("image_search_service.jobs.face_jobs.SyncConfigService") as mock_config:
        mock_config.return_value.get.return_value = "all"

        # Run job
        result = detect_faces_for_session_job(session_id=str(session.id))

        # Verify 2 jobs queued (one per person)
        assert result["suggestions_jobs_queued"] == 2
        assert mock_redis_queue.enqueue.call_count == 2

@pytest.mark.asyncio
async def test_post_training_suggestions_top_n_mode(db_session, mock_redis_queue):
    """Test that 'top_n' mode queues jobs for top N persons only."""
    # Setup: Create 5 persons with varying face counts
    session = create_test_session(db_session)
    persons = []
    for i in range(5):
        person = create_test_person(db_session, name=f"Person {i}")
        create_test_faces(db_session, person, count=(5 - i) * 10)  # 50, 40, 30, 20, 10
        persons.append(person)

    # Mock config to return 'top_n' with count=3
    with patch("image_search_service.jobs.face_jobs.SyncConfigService") as mock_config:
        mock_config.return_value.get.return_value = "top_n"
        mock_config.return_value.get_int.return_value = 3

        # Run job
        result = detect_faces_for_session_job(session_id=str(session.id))

        # Verify only 3 jobs queued (top 3 persons)
        assert result["suggestions_jobs_queued"] == 3
        assert mock_redis_queue.enqueue.call_count == 3
```

#### Frontend Tests

**File**: `image-search-ui/src/tests/components/FaceMatchingSettings.test.ts`

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/svelte';
import { describe, it, expect, beforeEach, vi } from 'vitest';
import FaceMatchingSettings from '$routes/admin/settings/FaceMatchingSettings.svelte';
import { mockFaceMatchingConfig, mockResponse } from '$tests/helpers/mockFetch';

describe('FaceMatchingSettings - Post-Training Suggestions', () => {
  beforeEach(() => {
    mockFaceMatchingConfig({
      post_training_suggestions_mode: 'all',
      post_training_suggestions_top_n_count: 10,
    });
  });

  it('renders post-training suggestions section', async () => {
    render(FaceMatchingSettings);

    await waitFor(() => {
      expect(screen.getByText('Post-Training Suggestions')).toBeInTheDocument();
      expect(screen.getByLabelText(/All Persons/i)).toBeChecked();
    });
  });

  it('shows top N input when top_n mode selected', async () => {
    render(FaceMatchingSettings);

    await waitFor(() => screen.getByLabelText(/All Persons/i));

    const topNRadio = screen.getByLabelText(/Top N Persons/i);
    await fireEvent.click(topNRadio);

    // Should show conditional number input
    const topNInput = screen.getByLabelText(/Number of Top Persons/i);
    expect(topNInput).toBeInTheDocument();
    expect(topNInput).toHaveValue(10);
  });

  it('validates top N count range (1-100)', async () => {
    render(FaceMatchingSettings);

    await waitFor(() => screen.getByLabelText(/Top N Persons/i));

    const topNRadio = screen.getByLabelText(/Top N Persons/i);
    await fireEvent.click(topNRadio);

    const topNInput = screen.getByLabelText(/Number of Top Persons/i);

    // Test invalid value (> 100)
    await fireEvent.input(topNInput, { target: { value: '200' } });
    expect(screen.getByText(/Must be between 1 and 100/i)).toBeInTheDocument();

    // Save button should be disabled
    const saveButton = screen.getByRole('button', { name: /Save/i });
    expect(saveButton).toBeDisabled();

    // Test valid value
    await fireEvent.input(topNInput, { target: { value: '50' } });
    expect(screen.queryByText(/Must be between 1 and 100/i)).not.toBeInTheDocument();
    expect(saveButton).not.toBeDisabled();
  });

  it('saves post-training suggestions settings', async () => {
    const mockUpdate = vi.fn(() => mockResponse({ status: 200 }));
    global.fetch = mockUpdate;

    render(FaceMatchingSettings);

    await waitFor(() => screen.getByLabelText(/Top N Persons/i));

    // Switch to top_n mode
    const topNRadio = screen.getByLabelText(/Top N Persons/i);
    await fireEvent.click(topNRadio);

    // Set count to 25
    const topNInput = screen.getByLabelText(/Number of Top Persons/i);
    await fireEvent.input(topNInput, { target: { value: '25' } });

    // Save
    const saveButton = screen.getByRole('button', { name: /Save/i });
    await fireEvent.click(saveButton);

    // Verify API call
    await waitFor(() => {
      expect(mockUpdate).toHaveBeenCalledWith(
        expect.stringContaining('/api/v1/config/face-matching'),
        expect.objectContaining({
          method: 'PUT',
          body: expect.stringContaining('"post_training_suggestions_mode":"top_n"'),
        })
      );
    });
  });
});
```

#### Integration Testing

**Manual Test Checklist**:
```bash
# 1. Verify database migration
cd image-search-service
make migrate
psql -c "SELECT key, value FROM system_configs WHERE key LIKE 'post_training%';"

# 2. Verify API endpoints
curl http://localhost:8000/api/v1/config/face-matching | jq '.post_training_suggestions_mode'

curl -X PUT http://localhost:8000/api/v1/config/face-matching \
  -H "Content-Type: application/json" \
  -d '{"post_training_suggestions_mode": "top_n", "post_training_suggestions_top_n_count": 15}'

# 3. Verify frontend UI
# Open http://localhost:5173/admin/settings
# Check that Post-Training Suggestions section appears
# Toggle between "All Persons" and "Top N Persons"
# Verify conditional input appears/hides correctly
# Save and verify no errors

# 4. Verify training job
# Create training session with existing persons
curl -X POST http://localhost:8000/api/v1/faces/sessions \
  -H "Content-Type: application/json" \
  -d '{"description": "Test post-training suggestions"}'

# Monitor queue (should see multi-proto jobs queued after training completes)
redis-cli
> LLEN rq:queue:default
> LRANGE rq:queue:default 0 -1

# Check logs for "Queuing post-training suggestions" and "jobs_queued" count
docker logs image-search-service | grep "post-training"
```

---

## 📊 Success Criteria

### Phase 1-2 (Database + Config Service)
- [ ] Migration creates both settings with correct defaults
- [ ] ConfigService returns correct default values
- [ ] Settings persist after application restart

### Phase 3 (API Endpoints)
- [ ] GET `/api/v1/config/face-matching` includes new fields
- [ ] PUT `/api/v1/config/face-matching` validates and saves both fields
- [ ] Invalid values (e.g., top_n_count = 200) return 422 validation error
- [ ] OpenAPI spec includes new fields in schemas

### Phase 4 (Training Job)
- [ ] Training job reads settings from SyncConfigService
- [ ] "all" mode queues jobs for all persons with faces
- [ ] "top_n" mode queues jobs for top N persons (sorted by face count)
- [ ] Jobs queued count logged correctly
- [ ] Multi-proto jobs appear in Redis queue

### Phase 5-6 (Frontend)
- [ ] Admin Panel displays Post-Training Suggestions section
- [ ] "All Persons" radio button checked by default
- [ ] Switching to "Top N Persons" shows conditional number input
- [ ] Number input validates range (1-100)
- [ ] Save button disabled when validation errors present
- [ ] Settings persist after page refresh

### Phase 7 (Testing)
- [ ] Backend unit tests pass (config API, validation)
- [ ] Backend integration tests pass (training job logic)
- [ ] Frontend component tests pass (UI rendering, validation, save)
- [ ] Manual integration test checklist completed

---

## 🚨 Risks and Mitigations

### Risk 1: High Queue Load
**Risk**: Setting mode to "all" with 1000+ persons could flood queue.
**Mitigation**:
- Document in UI: "May take several minutes for large databases"
- Consider adding "estimated jobs" preview before training starts
- Future: Add queue capacity check before queuing

### Risk 2: Settings Not Loaded in Worker
**Risk**: Background worker might use stale settings if not properly initialized.
**Mitigation**:
- Use `SyncConfigService` (synchronous, worker-friendly)
- Add logging to verify settings loaded correctly
- Test with actual worker process (not just unit tests)

### Risk 3: UI Confusion (Top N Input)
**Risk**: Users might not understand "top N" vs "all" distinction.
**Mitigation**:
- Clear descriptions in radio button labels
- Example text: "Setting this to 10 will generate suggestions for the 10 persons with the most labeled faces"
- Info callout explaining background job timing

### Risk 4: Migration Conflicts
**Risk**: Migration might conflict if other migrations created in parallel.
**Mitigation**:
- Generate migration with `make makemigrations` (auto-resolves conflicts)
- Use `ON CONFLICT (key) DO NOTHING` in SQL inserts
- Review migration before applying

---

## 📝 Implementation Checklist

### Backend (image-search-service/)

- [ ] **Phase 1**: Create database migration
  - [ ] Generate migration file: `make makemigrations`
  - [ ] Verify SQL inserts (mode + top_n_count)
  - [ ] Apply migration: `make migrate`
  - [ ] Verify in database: `SELECT * FROM system_configs WHERE key LIKE 'post_training%'`

- [ ] **Phase 2**: Update ConfigService
  - [ ] Add defaults to `DEFAULTS` dict in `config_service.py`
  - [ ] Add Pydantic schemas to `schemas.py`:
    - [ ] `PostTrainingSuggestionsConfig`
    - [ ] Update `FaceMatchingConfigUpdate`
    - [ ] Update `FaceMatchingConfigResponse`

- [ ] **Phase 3**: Update API endpoints
  - [ ] Update GET `/api/v1/config/face-matching` to include new fields
  - [ ] Update PUT `/api/v1/config/face-matching` to handle new fields
  - [ ] Test with curl commands
  - [ ] Verify OpenAPI spec: `curl http://localhost:8000/openapi.json | jq '.components.schemas.FaceMatchingConfigResponse'`

- [ ] **Phase 4**: Update training job
  - [ ] Import `SyncConfigService` in `face_jobs.py`
  - [ ] Add settings read logic in `detect_faces_for_session_job`
  - [ ] Add person query logic (all vs. top_n)
  - [ ] Add job queuing loop
  - [ ] Add logging for jobs queued
  - [ ] Update return value with `suggestions_jobs_queued`

- [ ] **Phase 7a**: Backend tests
  - [ ] Add config API tests to `test_config.py`
  - [ ] Create `test_face_jobs_post_training_suggestions.py`
  - [ ] Run: `make test`

### Frontend (image-search-ui/)

- [ ] **Phase 5**: Update API client
  - [ ] Add `PostTrainingSuggestionsConfig` type to `admin.ts`
  - [ ] Update `FaceMatchingConfig` interface
  - [ ] Update `FaceMatchingConfigUpdate` interface
  - [ ] Verify TypeScript compilation: `npm run check`

- [ ] **Phase 6**: Update Admin Panel UI
  - [ ] Open `FaceMatchingSettings.svelte`
  - [ ] Add state variables (`postTrainingSuggestionsMode`, `postTrainingSuggestionsTopNCount`)
  - [ ] Add validation (`postTrainingTopNCountError`)
  - [ ] Update `loadConfig()` to read new fields
  - [ ] Update `handleSave()` to send new fields
  - [ ] Add UI section with radio buttons + conditional input
  - [ ] Add styles for radio group and conditional input
  - [ ] Test in browser: `npm run dev` → http://localhost:5173/admin/settings

- [ ] **Phase 7b**: Frontend tests
  - [ ] Update `FaceMatchingSettings.test.ts`
  - [ ] Add tests for rendering, validation, save
  - [ ] Run: `npm run test`

### Integration Testing

- [ ] Run full integration test checklist (see Phase 7)
- [ ] Verify settings persist across restarts
- [ ] Verify training job queues correct number of jobs
- [ ] Verify multi-proto jobs execute and create suggestions

---

## 🎯 Post-Implementation Verification

### Smoke Test (5 minutes)

```bash
# 1. Backend: Verify migration applied
psql -c "SELECT key, value FROM system_configs WHERE key LIKE 'post_training%';"
# Expected: 2 rows (mode=all, top_n_count=10)

# 2. Backend: Verify API returns new fields
curl http://localhost:8000/api/v1/config/face-matching | jq '.post_training_suggestions_mode'
# Expected: "all"

# 3. Frontend: Verify UI section exists
# Open http://localhost:5173/admin/settings
# Expected: "Post-Training Suggestions" section visible

# 4. Frontend: Verify save works
# Switch to "Top N Persons", set count to 20, click Save
# Expected: Success message, no errors

# 5. Backend: Verify settings saved
curl http://localhost:8000/api/v1/config/face-matching | jq '.post_training_suggestions_top_n_count'
# Expected: 20

# 6. Training: Verify job queuing (requires existing data)
# Trigger training session
# Check logs for "Queuing post-training suggestions" message
docker logs image-search-service 2>&1 | grep "post-training"
# Expected: Log entries with jobs_queued count

# 7. Queue: Verify jobs queued
redis-cli LLEN rq:queue:default
# Expected: Non-zero if jobs queued
```

---

## 📚 Documentation Updates

After implementation, update these docs:

1. **API Contract** (`docs/api-contract.md`)
   - Add new fields to `FaceMatchingConfig` schema
   - Add example requests/responses

2. **README** (`README.md`)
   - Add "Post-Training Suggestions" to features list
   - Mention Admin Panel configuration option

3. **Admin Guide** (create if not exists: `docs/admin-guide.md`)
   - Explain "All Persons" vs "Top N Persons" modes
   - Provide guidance on when to use each mode
   - Explain performance implications

4. **Backend CLAUDE.md** (`image-search-service/CLAUDE.md`)
   - Add section on post-training suggestion system
   - Document settings keys and defaults

---

## 🔮 Future Enhancements

### Phase 8 (Optional): Advanced Features

**Queue Capacity Check**:
```python
# Before queuing, check queue length
queue_length = queue.count
if queue_length > 1000:
    logger.warning("Queue capacity high, skipping post-training suggestions")
    return
```

**Progressive Queuing**:
```python
# Queue top N persons immediately, rest as low-priority
for i, person in enumerate(persons):
    priority = "high" if i < 10 else "low"
    queue = Queue(priority, connection=redis_client)
    queue.enqueue(...)
```

**Settings Presets**:
```python
# Add preset buttons in UI
PRESETS = {
    "conservative": {"mode": "top_n", "top_n_count": 5},
    "balanced": {"mode": "top_n", "top_n_count": 15},
    "aggressive": {"mode": "all"},
}
```

**Estimated Job Count Preview**:
```python
# Before training, show user: "This will queue ~47 suggestion jobs"
GET /api/v1/faces/sessions/{session_id}/estimate-suggestions
```

---

## 📞 Support and Troubleshooting

### Common Issues

**Issue**: Settings not loading in Admin Panel
**Solution**:
- Check backend running: `curl http://localhost:8000/health`
- Check API returns fields: `curl http://localhost:8000/api/v1/config/face-matching | jq`
- Check browser console for fetch errors

**Issue**: Training job not queuing suggestions
**Solution**:
- Check logs: `docker logs image-search-service | grep post-training`
- Verify settings: `redis-cli GET config:post_training_suggestions_mode`
- Check persons exist: `psql -c "SELECT COUNT(*) FROM persons;"`

**Issue**: Validation error "top_n_count must be between 1 and 100"
**Solution**: Update value in Admin Panel or via API:
```bash
curl -X PUT http://localhost:8000/api/v1/config/face-matching \
  -H "Content-Type: application/json" \
  -d '{"post_training_suggestions_top_n_count": 50}'
```

---

## ✅ Implementation Summary

This plan provides a **complete end-to-end implementation** for:

1. ✅ Database-backed settings (two keys: mode + top_n_count)
2. ✅ Backend ConfigService integration (defaults, validation)
3. ✅ API endpoints (GET/PUT with Pydantic validation)
4. ✅ Training job logic (reads settings, queues multi-proto jobs)
5. ✅ Frontend Admin Panel UI (radio buttons + conditional input)
6. ✅ Comprehensive testing (backend unit + integration, frontend component)
7. ✅ Documentation (API contract, README, admin guide)

**Estimated Implementation Time**: 6-8 hours
- Phase 1-2 (Backend DB + Config): 1 hour
- Phase 3 (Backend API): 1 hour
- Phase 4 (Training Job): 2 hours
- Phase 5-6 (Frontend): 2 hours
- Phase 7 (Testing): 2 hours

**Estimated Testing Time**: 2 hours
- Unit tests: 1 hour
- Integration testing: 1 hour

**Total**: ~10 hours from start to production-ready

---

**Last Updated**: 2026-01-12
**Status**: Ready for Implementation
**Approved By**: [Pending User Review]
