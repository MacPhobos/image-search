# Amendments: Production Isolation Safety Fixes

**Date**: 2026-02-06
**Source**: `docs/research/service-testing/production-isolation-audit.md`
**Purpose**: Specific text changes needed for existing plans to address safety gaps

---

## Amendment 1: P0-01 - Add Qdrant Collection Name Guards to CI Workflow

**File**: `docs/plans/service-testing/p0-01-enable-ci-gating.md`
**Section**: CI workflow YAML (env vars section)
**Reason**: The CI workflow sets `QDRANT_URL` but does not explicitly set collection names. While the autouse fixture handles this at runtime, belt-and-suspenders protection requires CI-level env var overrides to match.

### Current Text (locate the `env:` block in the GitHub Actions workflow YAML)

```yaml
env:
  DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test_image_search
  REDIS_URL: redis://localhost:6379
  QDRANT_URL: http://localhost:6333
```

### Amended Text

```yaml
env:
  DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test_image_search
  REDIS_URL: redis://localhost:6379
  QDRANT_URL: http://localhost:6333
  # SAFETY: Force test collection names even if autouse fixtures fail
  QDRANT_COLLECTION: test_image_assets
  QDRANT_FACE_COLLECTION: test_faces
```

### Rationale

The autouse `use_test_settings` fixture in `tests/conftest.py` sets these env vars at runtime. However, if a test file bypasses conftest (e.g., using `TestClient(app)` directly) or if the autouse fixture fails to apply for any reason, the default collection names from `config.py` would be `image_assets` and `faces` (production names). Setting them at the CI level provides an additional safety layer.

---

## Amendment 2: P0-02 - Add Safety Note to Category D Fix

**File**: `docs/plans/service-testing/p0-02-fix-35-failing-tests.md`
**Section**: Section 6 (Category D: Config Post-Training Suggestions)
**Reason**: The fix description should explicitly call out the production safety implication of the `TestClient(app)` pattern, not just the test correctness issue.

### Insert After Line 377 (after "### Root Cause")

```markdown
**SAFETY NOTE**: This `TestClient(app)` pattern is also a production isolation risk.
When `TestClient(app)` is used without dependency overrides, the `create_app()` call
loads settings from `.env` which contains real service URLs:
- `DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:15432/image_search`
- `QDRANT_URL=http://localhost:6333`

If the tested endpoints accessed the database or Qdrant (these config endpoints do not),
the tests would connect to real services. The `test_config_unknown_clustering.py` file
has the same issue (see Amendment 3 below).

This fix should be prioritized as a SAFETY fix, not just a test correctness fix.
```

---

## Amendment 3: P0-02 - Add Category for test_config_unknown_clustering.py

**File**: `docs/plans/service-testing/p0-02-fix-35-failing-tests.md`
**Section**: After Section 6 (Category D), before Section 7 (Category H)
**Reason**: The audit identified `test_config_unknown_clustering.py` as having the same `TestClient(app)` bypass pattern as Category D. While these tests may currently pass, they have the same architectural flaw and should be fixed alongside Category D.

### New Section to Insert

```markdown
## 6b. Category D-2: Config Unknown Clustering Tests (Safety Refactor)

### Affected Tests

**File**: `tests/unit/api/test_config_unknown_clustering.py`

| Test Method |
|-------------|
| `TestUnknownClusteringConfig::test_get_unknown_clustering_config_returns_defaults` |
| `TestUnknownClusteringConfig::test_get_unknown_clustering_config_uses_camel_case` |
| `TestUnknownClusteringConfig::test_put_unknown_clustering_config_accepts_valid_values` |
| `TestUnknownClusteringConfig::test_put_unknown_clustering_config_validates_confidence_range` |
| `TestUnknownClusteringConfig::test_put_unknown_clustering_config_validates_cluster_size` |
| `TestUnknownClusteringConfig::test_put_unknown_clustering_config_rejects_missing_fields` |
| `TestUnknownClusteringConfig::test_put_unknown_clustering_config_accepts_snake_case_input` |
| `TestUnknownClusteringConfig::test_put_unknown_clustering_config_accepts_boundary_values` |

### Root Cause

Same pattern as Category D: uses `TestClient(app)` at module level (line 7)
bypassing dependency overrides from conftest. These tests currently PASS because
the config endpoints operate on in-memory settings, not database records. However,
the pattern is architecturally unsafe and should be fixed for consistency.

### Fix

Same approach as Category D: refactor to use the `test_client` fixture.

**Before (unsafe pattern):**
```python
from fastapi.testclient import TestClient
from image_search_service.main import app
client = TestClient(app)
```

**After (safe pattern):**
```python
import pytest

class TestUnknownClusteringConfig:
    @pytest.mark.asyncio
    async def test_get_unknown_clustering_config_returns_defaults(self, test_client):
        response = await test_client.get("/api/v1/config/face-clustering-unknown")
        assert response.status_code == 200
        # ...
```

### Verification

```bash
uv run pytest tests/unit/api/test_config_unknown_clustering.py -v --tb=short
```
```

---

## Amendment 4: P1-03 - Note on Qdrant Mock Safety in Race Condition Tests

**File**: `docs/plans/service-testing/p1-03-race-condition-tests.md`
**Section**: Section 5.1 (Concurrency Test Helpers)
**Reason**: The race condition tests mock Qdrant at the route level using `patch()`. If the patch path is wrong, the real Qdrant client could be invoked. The plan should note the safety implication.

### Insert After Section 5.1, Before Section 5.2

```markdown
### 5.1.1 Safety Note on Qdrant Mocking

All race condition tests MUST patch `get_face_qdrant_client` at the correct import
path. If the patch fails to intercept:

1. The `autouse` fixture `use_test_settings` still protects collection names
   (they will be `test_faces`, not `faces`)
2. The Qdrant connection will fail if no Qdrant server is running (safe failure)
3. If Qdrant IS running, operations target `test_` collections (safe)

However, verify each test's patch is effective by checking that mock assertions
(like `mock_qdrant.update_person_ids.assert_called_once()`) actually fire. If
a mock assertion never fires, the real code path may be executing.
```

---

## Amendment 5: P1-04 - CORS Test Uses TestClient Without Overrides

**File**: `docs/plans/service-testing/p1-04-security-testing.md`
**Section**: Section 3.2 (CORS Configuration Tests), specifically `test_cors_preflight_request_behavior`
**Reason**: The proposed CORS preflight test at line 679 creates `TestClient(app)` without dependency overrides. While the test only calls OPTIONS on `/api/v1/health` (which does not require DB), it should note the pattern and explain why it is acceptable.

### Insert After Line 679 (`client = TestClient(app, raise_server_exceptions=False)`)

```markdown
        # NOTE: TestClient(app) without dependency overrides is intentional here.
        # CORS preflight only calls OPTIONS /api/v1/health which does NOT access
        # the database, Qdrant, or Redis. The test specifically needs the raw app
        # to verify middleware configuration.
        # This pattern should NOT be copied for tests that exercise DB/Qdrant endpoints.
```

---

## Amendment 6: P2-06 - Flag Hardcoded Collection Name as Safety Issue

**File**: `docs/plans/service-testing/p2-06-critical-path-coverage.md`
**Section**: Section 2, Module A (dual_clusterer.py), subsection A.2 Testing Strategy
**Reason**: The plan notes the hardcoded `collection_name="faces"` as a "bug that would be caught by the test" but does not flag it as a production safety issue.

### Current Text (locate the note about hardcoded collection name)

```markdown
Note: The `_get_face_embedding` method (line 344-366) hardcodes `collection_name="faces"` instead of using `_get_face_collection_name()`. This is a bug that would be caught by the test (in-memory Qdrant uses `test_faces` per the test settings fixture). The test should either patch to use `"faces"` as collection name or identify this as a bug to fix.
```

### Amended Text

```markdown
**SAFETY NOTE**: The `_get_face_embedding` method (line 344-366) hardcodes
`collection_name="faces"` instead of using `_get_face_collection_name()`. This is
both a **bug** and a **production safety risk**:

1. **Bug**: The in-memory Qdrant test collection is named `test_faces` (per the
   autouse `use_test_settings` fixture), so this code would fail to find the
   collection during tests.

2. **Safety risk**: If the in-memory Qdrant override fails to inject (e.g., test
   not using the `test_client` fixture) AND a real Qdrant server is running, this
   code would read from the production `faces` collection. The `use_test_settings`
   fixture does NOT protect against hardcoded collection names.

**Recommendation**: Fix the hardcoded collection name in production code BEFORE
writing tests for `dual_clusterer.py`:

```python
# Before (unsafe):
collection_name = "faces"

# After (safe):
collection_name = _get_face_collection_name()
```

This ensures tests validate the correct behavior AND the safety guard works.
```

---

## Summary of Amendments

| # | Plan | Change Type | Safety Impact |
|---|------|-------------|---------------|
| 1 | P0-01 | Add env vars | Prevents collection name defaults in CI |
| 2 | P0-02 | Add safety note | Elevates Category D fix priority |
| 3 | P0-02 | Add new category | Fixes second TestClient bypass file |
| 4 | P1-03 | Add safety note | Documents mock verification practice |
| 5 | P1-04 | Add code comment | Documents intentional TestClient use |
| 6 | P2-06 | Amend text | Flags hardcoded collection as safety issue |

---

## How to Apply These Amendments

Each amendment is designed to be applied as a text edit to the referenced plan file.
They do not change the plans' scope or effort estimates. They add safety context
and (in Amendment 3) one additional category of work to P0-02.

**Priority order for implementation**:
1. Amendment 1 (CI env vars) -- Immediate, prevents CI-level safety gap
2. Amendment 3 (new P0-02 category) -- Should be done alongside Category D fix
3. Amendment 6 (hardcoded collection) -- Should be done before P2-06 implementation
4. Amendments 2, 4, 5 -- Documentation-only, apply when editing plans
