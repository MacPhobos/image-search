# P0-01: Enable CI Gating for Test Suite

**Priority**: P0 (Critical)
**Effort**: 2-4 hours
**Risk**: Low (no behavioral changes to production code)
**Blocking**: All other test improvements depend on this

---

## 1. Problem Statement

The image-search-service has a CI workflow (`.github/workflows/ci.yml`) that runs
lint, typecheck, and tests on every push/PR to `main`. However, **35 tests are
currently failing** and **5 tests produce errors**, meaning CI is either:

1. Not configured as a required status check (PRs merge despite failures), or
2. Already failing on every run (broken window effect)

Either way, the test suite provides **zero regression protection**. New code can
break additional tests without anyone noticing. The devil's advocate review
(see References) correctly identifies this as the single highest-risk item in
the entire testing posture: a test suite that runs but whose results are ignored
is worse than no suite at all, because it creates an illusion of quality.

### Current State

| Metric | Value |
|--------|-------|
| Total tests | 795 |
| Passing | 744 (93.6%) |
| Failing | 35 (4.4%) |
| Errors | 5 (0.6%) |
| Skipped | 11 (1.4%) |
| Coverage | ~48% |
| CI workflow exists | Yes |
| CI is a merge gate | **No (or ineffective)** |

### Desired State

| Metric | Target |
|--------|--------|
| CI blocks merge on failure | **Yes** |
| Known failures marked `xfail` | 35 + 5 = 40 tests |
| New failures caught immediately | **Yes** |
| Coverage threshold enforced | 45% (floor, prevents regression) |

---

## 2. Rationale: Triage Before Fix

The natural instinct is to fix all 35 failing tests first, then enable gating.
This is the wrong order for three reasons:

1. **Time gap**: Fixing 35 tests takes 1-3 days. During that time, more tests
   can break undetected. The value of gating is realized *today*, not after fixes.

2. **pytest `xfail` exists for exactly this**: Mark known failures as
   `@pytest.mark.xfail(reason="...")` so they do not fail the suite, but
   automatically alert when they start passing (xpass detection).

3. **Incremental progress becomes visible**: Each fixed test removes an `xfail`
   marker. The team can track progress. Without gating, fixes have no impact
   on CI status.

The strategy is: **mark known failures as xfail -> enable strict CI gating ->
then fix tests one category at a time (P0-02)**.

---

## 3. Implementation Plan

### Step 1: Inventory All Failing Tests (15 min)

Run the full test suite and capture exact test IDs for every failure and error.

```bash
cd image-search-service
uv run pytest --tb=no -q 2>&1 | tail -50
```

Cross-reference with the test execution analysis. The expected 40 failures/errors
are distributed across these categories:

| Category | Count | Test File(s) |
|----------|-------|-------------|
| A: FaceClusterer constructor | 7 | `tests/faces/test_clusterer.py` |
| B: FaceProcessingService batch | 2 | `tests/faces/test_service.py` |
| C: Restart workflow integration | 8 | `tests/integration/test_restart_workflows.py` |
| D: Config post-training suggestions | 5 | `tests/unit/api/test_config_post_training_suggestions.py` |
| E: Multi-prototype propagation | 8 | `tests/unit/queue/test_multiproto_propagation.py` |
| F: Prototype route mismatch | 2 | `tests/api/test_faces_routes.py` |
| G: Pin prototype quota | 1 | `tests/api/test_prototype_endpoints.py` |
| H: Config keys sync | 2 | `tests/unit/test_config_keys_sync.py` |
| I: Unified progress stubs | 5 | `tests/api/test_training_unified_progress.py` |
| **Total** | **40** | |

### Step 2: Apply `xfail` Markers to All Known Failures (30 min)

For each failing test, add an `@pytest.mark.xfail` decorator with a descriptive
reason. Group by category for easy tracking.

**Pattern to apply:**

```python
@pytest.mark.xfail(
    reason="Category A: FaceClusterer constructor missing qdrant_client param (P0-02)",
    strict=True,
)
```

**Why `strict=True`**: With `strict=True`, if a test unexpectedly *passes*, it
will be reported as `XPASS` (unexpected pass), which *fails* the suite. This is
exactly what we want: when someone fixes the underlying bug, the `xfail` marker
must be removed so the test counts as a real pass. This prevents stale xfail
markers from hiding fixed tests.

**Files to modify and test IDs:**

**A. `tests/faces/test_clusterer.py` (7 tests)**

Apply to each test method in the `TestFaceClusterer` class that instantiates
`FaceClusterer(mock_session, ...)` without `qdrant_client`:

```python
# Lines 27, 49, 82, 130, 151, 187, 200
@pytest.mark.xfail(
    reason="FaceClusterer.__init__ requires qdrant_client as 2nd arg (P0-02-A)",
    strict=True,
)
```

Tests:
- `test_cluster_unlabeled_faces_empty`
- `test_cluster_unlabeled_faces_with_results`
- `test_cluster_unlabeled_faces_applies_hdbscan`
- `test_cluster_unlabeled_faces_updates_db`
- `test_cluster_unlabeled_faces_with_time_bucket`
- `test_cluster_unlabeled_faces_with_noise`
- `test_cluster_unknown_faces_returns_summary`

**B. `tests/faces/test_service.py` (2 tests)**

```python
@pytest.mark.xfail(
    reason="Mock path mismatch for FaceProcessingService batch (P0-02-B)",
    strict=True,
)
```

Tests:
- `test_process_batch_success`
- `test_process_batch_partial_failure`

**C. `tests/integration/test_restart_workflows.py` (8 tests)**

```python
@pytest.mark.xfail(
    reason="Restart services use AsyncSession but tests have session mismatch (P0-02-C)",
    strict=True,
)
```

Tests:
- `TestRestartStateValidation::test_cannot_restart_running_training_session`
- `TestRestartStateValidation::test_cannot_restart_running_face_detection`
- `TestTrainingRestart::test_training_restart_failed_only`
- `TestTrainingRestart::test_training_restart_full`
- `TestFaceDetectionRestart::test_face_detection_restart_basic`
- `TestFaceDetectionRestart::test_face_detection_restart_deletes_persons`
- `TestClusteringRestart::test_clustering_restart_preserves_manual_assignments`
- `TestRestartWorkflowIntegration::test_normal_workflow_unchanged`

**D. `tests/unit/api/test_config_post_training_suggestions.py` (5 tests)**

```python
@pytest.mark.xfail(
    reason="Uses TestClient(app) bypassing dependency overrides (P0-02-D)",
    strict=True,
)
```

Tests:
- `TestPostTrainingSuggestionsConfig::test_get_config_returns_mode_and_top_n`
- `TestPostTrainingSuggestionsConfig::test_update_mode_to_top_n`
- `TestPostTrainingSuggestionsConfig::test_update_mode_to_none`
- `TestPostTrainingSuggestionsConfig::test_update_top_n_count`
- `TestPostTrainingSuggestionsConfig::test_update_invalid_mode_returns_422`

**E. `tests/unit/queue/test_multiproto_propagation.py` (8 tests)**

```python
@pytest.mark.xfail(
    reason="Qdrant mock not intercepted: job acquires client internally (P0-02-E)",
    strict=True,
)
```

Tests:
- `test_creates_suggestions_with_single_prototype`
- `test_creates_suggestions_with_multiple_prototypes`
- `test_aggregates_scores_across_prototypes`
- `test_respects_max_suggestions_limit`
- `test_preserves_existing_suggestions`
- `test_replaces_suggestions_when_not_preserving`
- `test_handles_prototype_without_embedding`
- `test_expires_old_suggestions`

**F. `tests/api/test_faces_routes.py` (2 tests)**

```python
@pytest.mark.xfail(
    reason="Route removed or changed: prototype endpoint mismatch (P0-02-F)",
    strict=True,
)
```

Identify the 2 specific failing tests by running:
```bash
uv run pytest tests/api/test_faces_routes.py --tb=short -q 2>&1 | grep FAILED
```

**G. `tests/api/test_prototype_endpoints.py` (1 test)**

```python
@pytest.mark.xfail(
    reason="Assertion string mismatch for pin quota error (P0-02-G)",
    strict=True,
)
```

Test:
- `TestPinPrototype::test_pin_quota_exceeded`

The test asserts `"Maximum 3 PRIMARY prototypes"` but the actual error message
may have changed.

**H. `tests/unit/test_config_keys_sync.py` (2 tests)**

```python
@pytest.mark.xfail(
    reason="Missing migration INSERTs for 4 new config keys (P0-02-H)",
    strict=True,
)
```

Tests:
- `test_all_defaults_have_migration_inserts`
- `test_all_defaults_keys_are_documented_in_migration`

These fail because `ConfigService.DEFAULTS` contains 12 keys (including 4
post-training/centroid keys added recently) but migration INSERT statements
only cover the original 8 keys:
- `post_training_suggestions_mode`
- `post_training_suggestions_top_n_count`
- `post_training_use_centroids`
- `centroid_min_faces_for_suggestions`

**I. `tests/api/test_training_unified_progress.py` (5 errors)**

```python
@pytest.mark.xfail(
    reason="Wrong fixture name (async_client -> test_client) + empty bodies (P0-02-I)",
    strict=True,
)
```

Tests:
- `test_unified_progress_basic_response`
- `test_unified_progress_with_phases`
- `test_unified_progress_completed_session`
- `test_unified_progress_no_session`
- `test_unified_progress_phase_details`

All 5 tests reference an `async_client` fixture that does not exist (the correct
fixture is `test_client`), and all 5 have `pass` as their body.

### Step 3: Verify xfail Markers Work Correctly (10 min)

Run the full test suite and confirm:

```bash
cd image-search-service

# Full run -- should now pass (0 failures, 40 xfail)
uv run pytest --tb=short -q

# Expected output:
# 744 passed, 40 xfailed, 11 skipped
```

If any test still FAILS (not xfail), investigate and add the missing marker.
If any test shows XPASS (unexpected pass), remove the xfail marker -- the test
was already fixed.

### Step 4: Add Coverage Threshold to pytest Configuration (10 min)

Update `pyproject.toml` to add coverage enforcement:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
# Fail if coverage drops below current floor
# This prevents coverage regression without requiring improvement
addopts = "--strict-markers"
```

If pytest-cov is already a dependency, also add:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--strict-markers"

[tool.coverage.run]
source = ["src/image_search_service"]

[tool.coverage.report]
fail_under = 45
show_missing = true
```

Check if pytest-cov is available:

```bash
uv run pip list | grep pytest-cov
```

If not installed, add it:

```bash
uv add --dev pytest-cov
```

### Step 5: Update CI Workflow (20 min)

Modify `.github/workflows/ci.yml` to:
1. Run pytest with coverage reporting
2. Report xfail counts for visibility
3. Upload coverage report as artifact

**Updated `.github/workflows/ci.yml`:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: image_search_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python 3.12
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --dev

      - name: Run linter
        run: uv run ruff check .

      - name: Run type checker
        run: uv run mypy src/

      - name: Run tests with coverage
        run: |
          uv run pytest \
            --tb=short \
            --cov=src/image_search_service \
            --cov-report=term-missing \
            --cov-report=xml:coverage.xml \
            --cov-fail-under=45 \
            -q
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/image_search_test
          REDIS_URL: redis://localhost:6379/0
          QDRANT_URL: http://localhost:6333

      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.xml
          retention-days: 30
```

### Step 6: Enable Branch Protection (10 min)

Configure GitHub branch protection rules for `main`:

1. Navigate to **Settings > Branches > Branch protection rules**
2. Click **Add rule** for branch name pattern: `main`
3. Enable:
   - **Require status checks to pass before merging**: ON
   - **Require branches to be up to date before merging**: ON
   - Search for and add required status check: `test`
4. Save changes

**If using GitHub CLI:**

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews=null \
  --field restrictions=null
```

### Step 7: Update Makefile (5 min)

Add a `test-ci` target that matches CI behavior:

```makefile
# In Makefile:

test-ci:  ## Run tests exactly as CI does (with coverage + strict markers)
	uv run pytest \
		--tb=short \
		--cov=src/image_search_service \
		--cov-report=term-missing \
		--cov-fail-under=45 \
		--strict-markers \
		-q
```

Update the existing `test` target to also use strict markers:

```makefile
test:  ## Run tests
	uv run pytest --strict-markers
```

### Step 8: Validate End-to-End (15 min)

1. Run the full suite locally to confirm green:
   ```bash
   make test-ci
   ```

2. Create a test PR to verify CI gates correctly:
   ```bash
   git checkout -b ci/enable-test-gating
   git add -A
   git commit -m "ci: enable test gating with xfail markers for 40 known failures"
   git push -u origin ci/enable-test-gating
   gh pr create --title "ci: enable test gating" --body "..."
   ```

3. Verify the PR shows the `test` check as required
4. Verify the check passes (all 40 known failures are xfail)
5. Merge the PR

---

## 4. Rollout Strategy

### Phase 1: Local Validation (Day 1, Morning)

- Apply all xfail markers
- Run `make test` locally and confirm 0 failures
- Verify xfail count matches expectations (40)

### Phase 2: CI Validation (Day 1, Afternoon)

- Push branch with updated CI workflow
- Verify CI passes on the branch
- Add branch protection rule

### Phase 3: Merge Gate Active (Day 1, End of Day)

- Merge the gating PR
- Confirm branch protection is active
- Any subsequent PR that breaks a test will be blocked

### Rollback Plan

If CI gating causes unexpected disruption:

1. **Immediate**: Remove branch protection rule via GitHub Settings
2. **Investigate**: Check if additional tests need xfail markers
3. **Re-enable**: Add missing markers and re-enable protection

This is a low-risk rollback because xfail markers are purely additive and
do not change any production code behavior.

---

## 5. Success Criteria

| Criterion | How to Verify |
|-----------|--------------|
| CI runs lint + typecheck + test | Check workflow output |
| All 40 known failures are xfail | `pytest --tb=no -q` shows 0 FAILED |
| CI blocks PRs with new failures | Create PR with intentionally broken test |
| Coverage floor is enforced (45%) | Check `--cov-fail-under=45` in CI output |
| xfail markers have descriptive reasons | Grep for `reason=` in test files |
| Each xfail references P0-02 category | Grep for `P0-02-` in test files |
| `make test-ci` matches CI behavior | Compare local and CI output |

---

## 6. Effort Estimate

| Task | Time |
|------|------|
| Inventory failing tests | 15 min |
| Apply xfail markers (9 files) | 30 min |
| Verify suite passes with markers | 10 min |
| Add coverage configuration | 10 min |
| Update CI workflow | 20 min |
| Enable branch protection | 10 min |
| Update Makefile | 5 min |
| End-to-end validation | 15 min |
| **Total** | **~2 hours** |

Buffer for unexpected issues: +1-2 hours (e.g., additional failing tests not
captured in the analysis, CI environment differences, xfail edge cases).

---

## 7. Dependencies

- **Prerequisite**: None (this is the first task)
- **Blocks**: P0-02 (Fix 35 Failing Tests) -- xfail markers provide the
  scaffolding for incremental test fixes
- **Required access**: GitHub repository admin (for branch protection rules)
- **Required packages**: `pytest-cov` if not already installed

---

## 8. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Additional unknown failing tests | Low | Low | Run full suite before marking; add xfail for any new failures |
| CI environment differs from local | Medium | Low | CI already has postgres+redis services; tests use SQLite in-memory |
| xfail markers mask permanently broken tests | Low | Medium | `strict=True` ensures tests that start passing are noticed |
| Coverage floor too high/low | Low | Low | Start at 45% (below current ~48%); adjust after P0-02 |
| Branch protection blocks emergency fixes | Low | Medium | Admin override available; can temporarily disable |

---

## 9. References

- **Test Execution Analysis**: `docs/research/service-testing/test-execution-analysis.md`
  - Source of truth for the 35 failure / 5 error count and 9 categories
- **Devil's Advocate Review**: `docs/research/service-testing/devils-advocate-review.md`
  - Argues CI gating should be P0, not P3; identifies "illusion of quality" risk
- **Service Code Analysis**: `docs/research/service-testing/service-code-analysis.md`
  - Maps testable surfaces and coverage gaps
- **Test Case Analysis**: `docs/research/service-testing/test-case-analysis.md`
  - Documents test infrastructure, fixtures, and patterns
- **CI Workflow**: `image-search-service/.github/workflows/ci.yml`
  - Current workflow (exists but does not gate merges)
- **Pytest Configuration**: `image-search-service/pyproject.toml` lines 71-73
  - Current pytest settings (asyncio_mode = "auto", testpaths = ["tests"])
- **Fix Plan**: `docs/plans/service-testing/p0-02-fix-35-failing-tests.md`
  - Companion document: detailed fixes for each of the 40 xfail'd tests
