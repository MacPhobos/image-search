# Phase 1: Test Acceleration Quick Wins

**Date**: 2026-02-15
**Effort**: < 1 hour
**Risk Level**: Low (zero changes to test logic or isolation)
**Expected Impact**: 2-2.5x full-suite speedup (136s -> ~55-68s)

---

## 1. Overview

### Goal

Achieve a 2-2.5x speedup on the `image-search-service` test suite through zero-risk, additive-only changes: parallel execution, safety nets, observability, and minor fixture optimizations.

### Current State

| Metric | Value |
|--------|-------|
| Total tests | 1,096 (15 postgres-marked deselected) |
| Full suite time (serial) | ~136s |
| Parallelization | None |
| Test timeout | None (hung tests block forever) |
| Slowest-test visibility | None |
| Autouse fixtures | 4 (function-scoped, run for every test) |
| Dev dependency group | pytest, pytest-asyncio, pytest-cov, ruff, mypy, aiosqlite |

### Expected Outcome After Phase 1

| Metric | Target |
|--------|--------|
| Full suite time (parallel) | ~55-68s |
| Test count | 1,096 (unchanged) |
| Test pass rate | 100% (unchanged) |
| Hung-test protection | 30s per-test timeout |
| Ordering-dependency detection | Random ordering via pytest-randomly |
| Slowest-test visibility | Top 10 durations shown by default |

### Prerequisites

- Python 3.12+ with `uv` package manager
- Working test suite: `cd image-search-service && uv run pytest` passes all 1,096 tests
- No external services required (tests use SQLite + in-memory Qdrant)

---

## 2. Changes

### QW1: Install pytest-xdist and Configure Parallel Execution

**What**: Add `pytest-xdist` to enable multi-process parallel test execution using the `worksteal` scheduler.

**Why**: This is the single highest-impact change. On a 4-core machine, `worksteal` distributes tests across worker processes, targeting a 2-2.5x speedup. The `worksteal` scheduler is preferred over `load` or `loadscope` because the suite has a wide range of test durations (0.01s to 3.78s) and worksteal dynamically rebalances.

**Why this is safe**: Each xdist worker runs in a separate process with its own memory space. Tests already use in-memory SQLite and in-memory Qdrant per-fixture invocation, so there is zero shared state between workers.

**File**: `image-search-service/pyproject.toml`

**Before** (lines 80-90):
```toml
[dependency-groups]
dev = [
    "aiosqlite>=0.22.0",
    "mypy>=1.19.1",
    "pillow>=12.0.0",
    "pytest>=9.0.2",
    "pytest-asyncio>=1.3.0",
    "pytest-cov>=7.0.0",
    "ruff>=0.14.10",
    "testcontainers[postgres]>=4.0.0",
]
```

**After**:
```toml
[dependency-groups]
dev = [
    "aiosqlite>=0.22.0",
    "mypy>=1.19.1",
    "pillow>=12.0.0",
    "pytest>=9.0.2",
    "pytest-asyncio>=1.3.0",
    "pytest-cov>=7.0.0",
    "pytest-randomly>=3.16.0",
    "pytest-timeout>=2.3.0",
    "pytest-xdist>=3.8.0",
    "ruff>=0.14.10",
    "testcontainers[postgres]>=4.0.0",
]
```

**File**: `image-search-service/pyproject.toml`

**Before** (lines 71-78):
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "postgres: marks tests as requiring PostgreSQL (deselect with '-m \"not postgres\"')",
]
# Default: exclude postgres tests from fast runs
addopts = "-m 'not postgres'"
```

**After**:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "postgres: marks tests as requiring PostgreSQL (deselect with '-m \"not postgres\"')",
]
# Parallel execution with safety nets
# -n auto: use all available CPU cores for parallel test execution
# --dist worksteal: dynamic work-stealing scheduler (best for varied test durations)
# --timeout=30: kill any test that runs longer than 30s (safety net)
# --durations=10: show 10 slowest tests after run (observability)
# -p randomly: randomize test order to detect ordering dependencies
# -p no:pastebin -p no:doctest -p no:nose: disable unused plugins
addopts = "-m 'not postgres' -n auto --dist worksteal --timeout=30 --durations=10 -p randomly -p no:pastebin -p no:doctest -p no:nose"
```

**Install command**:
```bash
cd image-search-service
uv add --group dev "pytest-xdist>=3.8.0" "pytest-timeout>=2.3.0" "pytest-randomly>=3.16.0"
uv sync --dev
```

> Note: The `uv add` command updates `pyproject.toml` automatically. Verify the final `[dependency-groups] dev` section matches the "After" block above. If `uv add` places entries differently, manually adjust to keep entries sorted alphabetically.

---

### QW2: pytest-timeout (Safety Net)

**What**: The `--timeout=30` flag in `addopts` (added in QW1 above) kills any individual test that runs longer than 30 seconds.

**Why**: Without a timeout, a hung test (e.g., accidental real network call, deadlock) blocks CI indefinitely. The current slowest test is 3.78s, so 30s provides a generous 8x margin.

**Why this is safe**: The timeout only kills tests that exceed 30s. No passing test comes close to this threshold. The timeout uses `signal.SIGALRM` on Linux (process-level, not thread-level), which is reliable.

**File change**: Already included in QW1's `addopts` change above.

**No additional file changes required.**

---

### QW3: --durations=10 (Observability)

**What**: The `--durations=10` flag in `addopts` (added in QW1 above) prints the 10 slowest tests after every test run.

**Why**: Provides ongoing visibility into which tests are slowest, enabling data-driven optimization decisions in future phases. Without this, slow-test regressions go unnoticed.

**Why this is safe**: Pure reporting — no effect on test execution or results.

> **⚠️ REVIEW NOTE (plan-reviewer):** With xdist enabled, `--durations` reports controller-measured wall-clock time (includes scheduling overhead), not pure per-test execution time. For accurate profiling, use `make test-profile` which runs serially with `-p no:xdist`.

**File change**: Already included in QW1's `addopts` change above.

**No additional file changes required.**

---

### QW4: Disable Unused Pytest Plugins

**What**: The `-p no:pastebin -p no:doctest -p no:nose` flags in `addopts` (added in QW1 above) skip loading three built-in pytest plugins that this project does not use.

**Why**: Saves ~0.1-0.3s in collection time. More importantly, with xdist each worker re-initializes plugins, so the savings multiply by worker count.

**Why this is safe**: The project has zero doctests, zero nose-style tests, and never uses pastebin. Disabling unused plugins cannot affect test results.

**File change**: Already included in QW1's `addopts` change above.

**No additional file changes required.**

---

### QW5: Session-Scope `validate_embedding_dimensions`

**What**: Change the `validate_embedding_dimensions` fixture from `autouse=True, scope="function"` (default) to `autouse=True, scope="session"`.

**Why**: This fixture only validates that `MockEmbeddingService.EMBEDDING_DIM == 768` — a class-level constant that cannot change between tests. Running it 1,096 times is pure waste. Running it once per session (or once per xdist worker) is sufficient.

**Why this is safe**: The fixture checks compile-time constants (`EMBEDDING_DIM = 768` and the `embedding_dim` property). These values are deterministic and immutable. No test modifies `MockEmbeddingService.EMBEDDING_DIM`. The fixture has no teardown logic (the `yield` is followed by a comment saying "no additional validation needed").

**File**: `image-search-service/tests/conftest.py`

**Before** (lines 134-163):
```python
@pytest.fixture(autouse=True)
def validate_embedding_dimensions():
    """Validate that test embedding dimensions match expected values.

    This guard catches the most common mock accuracy issue: dimension
    mismatch between test and production embedding services.

    Expected dimensions:
    - Image search (CLIP/SigLIP): 768
    - Face recognition (InsightFace/ArcFace): 512

    This runs at import time before any tests execute to fail fast.
    """
    # Pre-test validation - check class constant
    assert MockEmbeddingService.EMBEDDING_DIM == 768, (
        f"Mock embedding service dimension mismatch: "
        f"got {MockEmbeddingService.EMBEDDING_DIM}, expected 768 (CLIP/SigLIP). "
        f"Make sure MockEmbeddingService = SemanticMockEmbeddingService in conftest.py"
    )

    # Also verify instance dimension property
    test_instance = MockEmbeddingService()
    assert test_instance.embedding_dim == 768, (
        f"Mock embedding instance dimension mismatch: "
        f"got {test_instance.embedding_dim}, expected 768 (CLIP/SigLIP)"
    )

    yield

    # Post-test: no additional validation needed since pre-test catches issues
```

**After**:
```python
@pytest.fixture(autouse=True, scope="session")
def validate_embedding_dimensions():
    """Validate that test embedding dimensions match expected values.

    This guard catches the most common mock accuracy issue: dimension
    mismatch between test and production embedding services.

    Expected dimensions:
    - Image search (CLIP/SigLIP): 768
    - Face recognition (InsightFace/ArcFace): 512

    Session-scoped: only runs once per session (or once per xdist worker).
    These are compile-time constants that cannot change between tests.
    """
    # Pre-test validation - check class constant
    assert MockEmbeddingService.EMBEDDING_DIM == 768, (
        f"Mock embedding service dimension mismatch: "
        f"got {MockEmbeddingService.EMBEDDING_DIM}, expected 768 (CLIP/SigLIP). "
        f"Make sure MockEmbeddingService = SemanticMockEmbeddingService in conftest.py"
    )

    # Also verify instance dimension property
    test_instance = MockEmbeddingService()
    assert test_instance.embedding_dim == 768, (
        f"Mock embedding instance dimension mismatch: "
        f"got {test_instance.embedding_dim}, expected 768 (CLIP/SigLIP)"
    )

    yield

    # Post-test: no additional validation needed since pre-test catches issues
```

**Only change**: Line 134 — `@pytest.fixture(autouse=True)` becomes `@pytest.fixture(autouse=True, scope="session")`.

---

### QW6: Merge `clear_settings_cache` + `use_test_settings` Into One Fixture

**What**: Combine the two separate autouse fixtures `clear_settings_cache` and `use_test_settings` into a single `test_settings` fixture. Both manipulate `get_settings` cache and must run for every test (function-scoped, autouse).

**Why**: Currently, every test invokes both fixtures sequentially. `clear_settings_cache` calls `get_settings.cache_clear()` twice (before + after). Then `use_test_settings` calls `get_settings.cache_clear()` THREE more times (before, after env set, after yield). That's 5 `cache_clear()` calls per test x 1,096 tests = 5,480 redundant cache clears. Merging eliminates ~2,740 redundant calls and saves ~2% (~2.7s).

**Why this is safe**: The merged fixture does exactly the same work in the same order: clear cache, set env vars, clear cache, yield, clear cache. No behavior change — just fewer fixture invocations per test.

> **⚠️ REVIEW NOTE (plan-reviewer):** Two test fixtures reference `use_test_settings` **by name** as a dependency:
> - `tests/faces/test_dual_clusterer.py:49` — `mock_qdrant_for_clusterer(monkeypatch, use_test_settings, qdrant_client)`
> - `tests/unit/test_dual_clusterer_batch.py:14` — `mock_qdrant_for_batch(monkeypatch, use_test_settings, qdrant_client)`
>
> **You MUST also update these fixtures** to reference the new name `test_settings`, or keep a backward-compatible alias:
> ```python
> @pytest.fixture(autouse=True)
> def use_test_settings(test_settings):
>     """Backward-compatible alias. Depends on test_settings (does nothing extra)."""
>     pass
> ```
> Without this fix, renaming breaks those 2 test files immediately.

**File**: `image-search-service/tests/conftest.py`

**Before** (lines 47-79 — two separate fixtures):
```python
@pytest.fixture(autouse=True)
def clear_settings_cache():
    """Clear settings cache before and after each test to prevent production settings leaking.

    CRITICAL: This prevents tests from using cached production collection names,
    which could cause deletion of live Qdrant data.
    """
    from image_search_service.core.config import get_settings

    get_settings.cache_clear()
    yield
    get_settings.cache_clear()


@pytest.fixture(autouse=True)
def use_test_settings(monkeypatch):
    """Override settings with test-safe collection names.

    CRITICAL: This ensures tests never use production collection names
    like "image_assets" or "faces", preventing accidental data deletion.
    """
    from image_search_service.core.config import get_settings

    get_settings.cache_clear()

    # Set environment variables for test collection names
    monkeypatch.setenv("QDRANT_COLLECTION", "test_image_assets")
    monkeypatch.setenv("QDRANT_FACE_COLLECTION", "test_faces")

    # Clear cache again so new env vars are picked up
    get_settings.cache_clear()
    yield
    get_settings.cache_clear()
```

**After** (replace both fixtures with one):
```python
@pytest.fixture(autouse=True)
def test_settings(monkeypatch):
    """Clear settings cache and override with test-safe collection names.

    CRITICAL: This fixture serves two safety purposes:
    1. Clears the settings LRU cache to prevent production settings from leaking between tests
    2. Sets test-safe Qdrant collection names to prevent accidental deletion of live data

    This MUST remain function-scoped and autouse. Every test must get fresh settings
    with test-safe collection names. Removing or session-scoping this fixture risks
    production data deletion.

    Replaces the former clear_settings_cache + use_test_settings fixtures.
    """
    from image_search_service.core.config import get_settings

    # Clear any cached settings from previous test
    get_settings.cache_clear()

    # Set environment variables for test collection names
    monkeypatch.setenv("QDRANT_COLLECTION", "test_image_assets")
    monkeypatch.setenv("QDRANT_FACE_COLLECTION", "test_faces")

    # Clear cache again so new env vars are picked up
    get_settings.cache_clear()
    yield
    # Clear cache after test so no test-specific settings leak
    get_settings.cache_clear()
```

**What changed**:
- Deleted `clear_settings_cache` fixture entirely
- Deleted `use_test_settings` fixture entirely
- Added `test_settings` fixture that does the combined work
- Reduced from 5 `cache_clear()` calls per test to 3

---

### QW7: Add pytest-randomly (Ordering Dependency Detection)

**What**: The `-p randomly` flag in `addopts` (added in QW1 above) randomizes test execution order on every run. The `pytest-randomly` package is installed via `uv add` in QW1.

**Why**: Hidden test-ordering dependencies are the #1 risk when enabling parallel execution. If test B only passes because test A ran first and set up some shared state, xdist will catch this intermittently (hard to debug). pytest-randomly catches it deterministically by printing the random seed, allowing reproduction with `--randomly-seed=LAST`.

**Why this is safe**: If any test fails under random ordering, it reveals a pre-existing isolation bug — not a bug introduced by this change. The seed is printed in the test output header for reproducibility.

**File change**: Already included in QW1's dependency and `addopts` changes above.

**No additional file changes required.**

---

## 3. New Makefile Targets

**File**: `image-search-service/Makefile`

**Before** (lines 22-31):
```makefile
test: ## Run pytest tests (fast SQLite tier only)
	uv run pytest

test-postgres: ## Run PostgreSQL integration tests (requires Docker)
	@echo "Starting PostgreSQL integration tests (requires Docker)..."
	uv run pytest tests/ -m "postgres" -v --tb=short

test-all: ## Run all tests (SQLite + PostgreSQL)
	@echo "Running all tests..."
	uv run pytest tests/ -m "" -v --tb=short
```

**After** (replace the `test:` target and add new targets between `test:` and `test-postgres:`):
```makefile
test: ## Run pytest tests (parallel, randomized, with timeout)
	uv run pytest

test-serial: ## Run pytest serially (for debugging test isolation issues)
	uv run pytest -p no:xdist -p no:randomly --timeout=60

test-fast: ## Re-run only previously failed tests (serial, for quick iteration)
	uv run pytest --lf -x -p no:xdist --timeout=60

test-failed-first: ## Run failed tests first, then all remaining tests
	uv run pytest --ff

test-profile: ## Show slowest 20 tests (serial, for profiling)
	uv run pytest --durations=20 --durations-min=0.5 -p no:xdist -p no:randomly --timeout=60

test-postgres: ## Run PostgreSQL integration tests (requires Docker)
	@echo "Starting PostgreSQL integration tests (requires Docker)..."
	uv run pytest tests/ -m "postgres" -v --tb=short

test-all: ## Run all tests (SQLite + PostgreSQL)
	@echo "Running all tests..."
	uv run pytest tests/ -m "" -v --tb=short
```

**New targets explained**:

| Target | Purpose | When to Use |
|--------|---------|-------------|
| `test` | Full suite, parallel, randomized, 30s timeout | Default: CI and local pre-commit |
| `test-serial` | Serial execution, deterministic order, 60s timeout | Debugging a test failure that only appears in parallel |
| `test-fast` | Re-run only the tests that failed last time | Quick iteration after a failure |
| `test-failed-first` | Run failures first, then everything else | Verify a fix then confirm nothing else broke |
| `test-profile` | Show top 20 slowest tests, serial for accurate timing | Identifying optimization targets |

**Also update the `.PHONY` line at the top of the Makefile**:

**Before** (line 1):
```makefile
.PHONY: help dev api lint format typecheck test db-up db-down migrate makemigrations worker ingest \
```

**After**:
```makefile
.PHONY: help dev api lint format typecheck test test-serial test-fast test-failed-first test-profile db-up db-down migrate makemigrations worker ingest \
```

---

## 4. Validation Steps

Run these commands **in order** after making all changes:

### Step 1: Install dependencies

```bash
cd image-search-service
uv add --group dev "pytest-xdist>=3.8.0" "pytest-timeout>=2.3.0" "pytest-randomly>=3.16.0"
uv sync --dev
```

### Step 2: Verify packages installed

```bash
uv run pip list | grep -iE "xdist|timeout|randomly"
```

Expected output (version numbers may vary):
```
pytest-randomly   3.16.0
pytest-timeout    2.3.1
pytest-xdist      3.8.0
```

### Step 3: Run serial baseline (verify no regressions from fixture changes)

```bash
uv run pytest -p no:xdist -p no:randomly --timeout=60 -v 2>&1 | tail -5
```

Expected: All 1,096 tests pass (15 postgres deselected). This confirms QW5 (session-scoped validate_embedding_dimensions) and QW6 (merged settings fixture) did not break anything.

### Step 4: Run parallel suite (verify xdist works)

```bash
time uv run pytest
```

Expected:
- All 1,096 tests pass
- Wall-clock time: ~55-68s (down from ~136s)
- Output header shows: `[gw0] ... [gw3]` or similar (worker processes)
- Output header shows: `Using --randomly-seed=NNNNNNN` (random seed)
- Output footer shows: `slowest 10 durations` (durations report)

### Step 5: Verify Makefile targets work

```bash
make test              # Parallel (should be ~55-68s)
make test-serial       # Serial (should be ~136s, all pass)
make test-profile      # Shows slowest 20 tests
```

### Step 6: Verify random ordering reproducibility

```bash
# Run once, note the seed from output header: "Using --randomly-seed=NNNNNNN"
uv run pytest --co -q 2>&1 | head -3

# Re-run with same seed to verify deterministic ordering
uv run pytest --randomly-seed=LAST --co -q 2>&1 | head -3
```

### Step 7: Verify timeout works

```bash
# Create a temporary test that sleeps for 35s (should be killed at 30s)
echo 'import time
def test_timeout_works():
    time.sleep(35)' > /tmp/test_timeout_check.py

uv run pytest /tmp/test_timeout_check.py --timeout=30 -p no:xdist 2>&1 | tail -5
# Expected: FAILED with "Timeout" message

rm /tmp/test_timeout_check.py
```

---

## 5. Rollback Plan

Every change in Phase 1 is independently reversible.

### Rollback QW1-QW4 (xdist, timeout, durations, plugin disabling)

Revert `addopts` in `pyproject.toml` to the original:

```toml
addopts = "-m 'not postgres'"
```

Optionally uninstall packages:
```bash
uv remove --group dev pytest-xdist pytest-timeout pytest-randomly
```

### Rollback QW5 (session-scoped validate_embedding_dimensions)

Change `scope="session"` back to function scope:

```python
# In tests/conftest.py
@pytest.fixture(autouse=True)  # Remove scope="session"
def validate_embedding_dimensions():
```

### Rollback QW6 (merged settings fixtures)

Replace the merged `test_settings` fixture with the original two fixtures (copy the "Before" code from QW6 above back into `tests/conftest.py`).

### Rollback QW7 (pytest-randomly)

Remove `-p randomly` from `addopts` and optionally uninstall:
```bash
uv remove --group dev pytest-randomly
```

### Rollback Makefile changes

Revert the Makefile `test:` target and remove `test-serial`, `test-fast`, `test-failed-first`, `test-profile` targets. Restore the original:

```makefile
test: ## Run pytest tests (fast SQLite tier only)
	uv run pytest
```

### Full rollback (all changes at once)

```bash
cd image-search-service
git checkout -- pyproject.toml tests/conftest.py Makefile
uv sync --dev
uv run pytest  # Verify original behavior restored
```

---

## 6. Success Criteria

### Must-Have (all required for Phase 1 to be considered complete)

| Criteria | How to Verify |
|----------|---------------|
| All 1,096 tests pass in parallel mode | `uv run pytest` exits 0, shows 1,096 passed |
| All 1,096 tests pass in serial mode | `uv run pytest -p no:xdist -p no:randomly` exits 0 |
| Wall-clock time < 70s in parallel | `time uv run pytest` shows real < 70s |
| No test count change | `uv run pytest --co -q \| tail -1` shows "1096 tests" |
| Timeout protection active | `--timeout=30` in addopts, verified via Step 7 above |
| Random ordering active | `Using --randomly-seed=` appears in test output header |
| Slowest tests visible | `slowest 10 durations` appears in test output footer |
| All new Makefile targets work | `make test-serial`, `make test-fast`, `make test-profile` run without error |

### Nice-to-Have

| Criteria | How to Verify |
|----------|---------------|
| Wall-clock time < 60s in parallel | `time uv run pytest` shows real < 60s |
| Zero test failures under 5 consecutive random-order runs | `for i in $(seq 5); do uv run pytest -x; done` |
| Worker load balanced (no worker doing 2x others) | `uv run pytest -n 4 -v 2>&1 \| grep "PASSED\|FAILED" \| sed 's/.*\[gw\([0-9]\).*/gw\1/' \| sort \| uniq -c` |

---

## 7. Known Risks

### Risk 1: Hidden Test-Ordering Dependencies (Low probability, Medium impact)

**What**: Some tests may fail when run in a different order than the default alphabetical collection.

**Detection**: `pytest-randomly` will surface these on the first run. The seed is printed for reproduction.

**Mitigation**: If a test fails:
1. Reproduce: `uv run pytest --randomly-seed=LAST -x`
2. Isolate: Run the failing test alone: `uv run pytest tests/path/test_file.py::test_name -v`
3. If it passes alone but fails in random order: there's a pre-existing isolation bug. Fix the test, not the tooling.
4. Temporary workaround: `uv run pytest -p no:randomly` to run in default order while fixing the test.

### Risk 2: xdist Worker Startup Overhead (Low probability, Low impact)

**What**: Each xdist worker process re-imports all modules including conftest.py and torch. On machines with < 4 cores or slow disk, the overhead may reduce the speedup below 2x.

**Detection**: Compare `time uv run pytest` (parallel) vs `time uv run pytest -p no:xdist` (serial). If parallel is not at least 1.5x faster, xdist overhead is too high.

**Mitigation**: Reduce worker count: change `-n auto` to `-n 2` in `addopts`. Or defer torch optimization to Phase 2 (ME1: lazy torch import).

### Risk 3: pytest-asyncio Compatibility with xdist (Low probability, High impact)

**What**: The project uses `pytest-asyncio>=1.3.0` with `asyncio_mode = "auto"`. Older versions of pytest-asyncio had issues with xdist worker isolation.

**Detection**: If async tests fail with `RuntimeError: Event loop is closed` or similar, this is the cause.

**Mitigation**:
1. Verify `pytest-asyncio` version: `uv run pip show pytest-asyncio` (should be >= 0.24.0 as specified in pyproject.toml optional deps; actual installed version from dependency-groups is >= 1.3.0)
2. If issues persist, pin: `uv add --group dev "pytest-asyncio>=1.3.0,<2.0.0"`
3. Last resort: disable xdist for async-heavy test files using `pytest.ini` markers

### Risk 4: Timeout Kills Legitimately Slow Tests (Very low probability)

**What**: A test that normally takes 4s might occasionally take >30s under xdist load, causing a false timeout failure.

**Detection**: Timeout failures show `Timeout >30.0s` in the error message.

**Mitigation**: The current slowest test is 3.78s, giving an 8x safety margin. If a test legitimately needs more time, mark it:
```python
@pytest.mark.timeout(120)  # Override default 30s timeout for this test
def test_legitimately_slow():
    ...
```

### Risk 5: CI Environment Differences (Low probability, Low impact)

**What**: CI machines may have different core counts. `-n auto` detects cores automatically, but a single-core CI runner would get `-n 1` (no speedup) plus xdist overhead (net slowdown).

**Detection**: CI logs will show `[gw0]` only (single worker).

**Mitigation**: In CI config, explicitly set worker count: `uv run pytest -n 4` regardless of available cores. Or use the existing `make test` which inherits `addopts` with `-n auto`.

---

## Appendix: Complete File Diffs

### Summary of files changed

| File | Type of Change |
|------|---------------|
| `image-search-service/pyproject.toml` | Add 3 dev deps, update `addopts` |
| `image-search-service/tests/conftest.py` | Merge 2 fixtures into 1, session-scope 1 fixture |
| `image-search-service/Makefile` | Update `test:` comment, add 4 new targets, update `.PHONY` |

### Total lines changed

- ~5 lines in `pyproject.toml` (3 new deps + 1 line `addopts` replacement)
- ~35 lines removed, ~20 lines added in `conftest.py` (net -15 lines)
- ~12 lines added in `Makefile` (4 new targets + `.PHONY` update)
