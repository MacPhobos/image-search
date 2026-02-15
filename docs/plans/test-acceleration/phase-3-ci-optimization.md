# Phase 3: CI Optimization — pytest-split Sharding + Fast Coverage

**Date**: 2026-02-15
**Status**: Ready for implementation
**Prerequisites**: Phases 1 and 2 must be completed first
**Effort Estimate**: 2-3 hours
**Risk Level**: Low-Medium

---

## 1. Overview

### Goal

Reduce CI wall-clock time from ~55-68s (post-Phase 1 xdist) to ~15-20s by splitting the test suite across 4 parallel GitHub Actions runners (shards), each running pytest-xdist internally.

### Expected Impact

| Metric | Before Phase 3 | After Phase 3 | Improvement |
|--------|----------------|---------------|-------------|
| CI wall-clock (test step) | ~55-68s | ~15-20s | 3-4x |
| Coverage collection overhead | Baseline | 30-50% less | via sysmon |
| CI machine-minutes consumed | ~1 min | ~4 x 0.3 min = ~1.2 min | Slight increase |

### Why This Is Safe

- **pytest-split** only changes _which tests run on which runner_. It does not modify test code, fixtures, or configuration.
- Each shard runs a standard `uv run pytest` with the same isolation guarantees.
- `COVERAGE_CORE=sysmon` is a standard Python 3.12+ feature that uses `sys.monitoring` instead of `sys.settrace` — same coverage data, lower overhead.
- `fail-fast: false` ensures all shards run to completion, so you always see the full failure picture.

### Architecture

```
GitHub Actions "test" job (matrix: 4 shards)
    |
    +-- Shard 1: ~274 tests (balanced by duration)
    |     \-- pytest-xdist: -n auto --dist worksteal
    |
    +-- Shard 2: ~274 tests
    |     \-- pytest-xdist: -n auto --dist worksteal
    |
    +-- Shard 3: ~274 tests
    |     \-- pytest-xdist: -n auto --dist worksteal
    |
    +-- Shard 4: ~274 tests
          \-- pytest-xdist: -n auto --dist worksteal

Each shard: ~274 tests / 2 cores (GH runner) ≈ 15-20s
```

---

## 2. Changes

### MC1: CI Sharding with pytest-split

#### 2.1 Install pytest-split

**File**: `image-search-service/pyproject.toml`

Add `pytest-split` to the dev dependency group:

```toml
[dependency-groups]
dev = [
    "aiosqlite>=0.22.0",
    "mypy>=1.19.1",
    "pillow>=12.0.0",
    "pytest>=9.0.2",
    "pytest-asyncio>=1.3.0",
    "pytest-cov>=7.0.0",
    "pytest-split>=0.9.0",       # <-- ADD THIS
    "ruff>=0.14.10",
    "testcontainers[postgres]>=4.0.0",
]
```

Note: After Phase 1 is applied, this section will also include `pytest-xdist`, `pytest-timeout`, and `pytest-randomly`. Add `pytest-split` alongside those.

Then run:

```bash
cd image-search-service
uv sync --dev
```

#### 2.2 Generate Initial Test Timing Data

**File to create**: `image-search-service/.test_durations` (auto-generated, then committed)

pytest-split uses a JSON file mapping test node IDs to their durations. This file must be generated locally and committed to the repository so CI can use it for balanced shard assignment.

```bash
cd image-search-service

# Run tests serially (no xdist) to get accurate per-test timings
uv run pytest --store-durations -p no:xdist -p no:randomly -q

# This creates .test_durations in the current directory
# Verify it was created and looks reasonable:
wc -l .test_durations           # Should show ~1096 entries
head -5 .test_durations         # JSON format: {"test_id": duration, ...}
python3 -c "
import json
with open('.test_durations') as f:
    data = json.load(f)
print(f'Total tests: {len(data)}')
print(f'Total duration: {sum(data.values()):.1f}s')
print(f'Per-shard target: {sum(data.values())/4:.1f}s')
"
```

**Important**: Run `--store-durations` WITHOUT `-n auto` and WITHOUT `--randomly` to get clean, deterministic timing data. xdist and random ordering can distort individual test timings.

#### 2.3 Commit the Timing Data

```bash
cd image-search-service
git add .test_durations
git commit -m "chore(tests): add pytest-split timing data for CI sharding"
```

The `.test_durations` file should be committed to the repo. It's a JSON file (~50-100KB) that enables balanced test splitting. It does NOT need to be regenerated on every run — see Section 8 for the maintenance cadence.

#### 2.4 Update GitHub Actions CI Workflow

**File**: `image-search-service/.github/workflows/ci.yml`

> **⚠️ REVIEW NOTE (plan-reviewer):** The existing CI runs commands from the repo root without `cd image-search-service`, relying on uv's auto-discovery. This proposed workflow explicitly adds `cd image-search-service` to install steps and `working-directory` to the test step, which is more correct for the monorepo structure. Verify existing CI behavior before applying this change.

Replace the entire file with this updated workflow:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python 3.12
        run: uv python install 3.12

      - name: Install dependencies
        run: |
          cd image-search-service
          uv sync --dev

      - name: Run linter
        run: |
          cd image-search-service
          uv run ruff check .

      - name: Run type checker
        run: |
          cd image-search-service
          uv run mypy src/

  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      fail-fast: false
      matrix:
        group: [1, 2, 3, 4]

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
        run: |
          cd image-search-service
          uv sync --dev

      - name: Run tests (shard ${{ matrix.group }}/4)
        working-directory: image-search-service
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/image_search_test
          REDIS_URL: redis://localhost:6379/0
          QDRANT_URL: http://localhost:6333
          COVERAGE_CORE: sysmon
        run: >-
          uv run pytest
          --splits 4
          --group ${{ matrix.group }}
          --splitting-algorithm least_duration
          --durations=10
```

#### Key Design Decisions in the Workflow

**1. Separate `lint` and `test` jobs**:
- Linting + type checking runs once (not 4x), saving ~2 minutes of CI time.
- Tests only run if lint passes (`needs: lint`).

**2. `fail-fast: false`**:
- All 4 shards always run to completion.
- If shard 1 fails, you still see results from shards 2-4.
- This is critical for debugging — a test failure in one shard shouldn't hide failures in others.

**3. `--splitting-algorithm least_duration`**:
- This is pytest-split's default and most balanced algorithm.
- It assigns tests to shards greedily, filling the shard with the least total duration.
- Results in near-equal shard durations even with varying test times.

**4. xdist flags are NOT repeated**:
- Phase 1 adds `-n auto --dist worksteal` to the `addopts` in `pyproject.toml`.
- These apply automatically. No need to pass them again on the command line.
- pytest-split selects which tests each shard runs; xdist parallelizes within the shard.

**5. `--durations=10` on each shard**:
- Provides visibility into the 10 slowest tests per shard in CI logs.
- Helps identify shard imbalance or individual slow tests.

**6. Services (postgres, redis) remain**:
- Even though the default test run deselects postgres-marked tests, the services are kept for forward compatibility.
- If you want to save CI time, you can remove these services since the sharded tests don't use them. See the "Optional: Remove Unused Services" section below.

#### Optional: Remove Unused CI Services

If you're confident the sharded test run will never include postgres-marked tests (Phase 1 configures `addopts = "-m 'not postgres'"`), you can remove the `services` block from the `test` job entirely:

```yaml
  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      fail-fast: false
      matrix:
        group: [1, 2, 3, 4]

    # No services block — SQLite in-memory only
    steps:
      # ... same steps ...
```

This saves ~15-30s of service startup per shard. However, keeping the services is harmless and makes it easier to add postgres tests later.

#### Optional: Add a Separate Postgres Test Job

If you want to run the 15 postgres-marked tests as well, add a separate job:

```yaml
  test-postgres:
    runs-on: ubuntu-latest
    needs: lint
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
    steps:
      - uses: actions/checkout@v4
      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"
      - name: Set up Python 3.12
        run: uv python install 3.12
      - name: Install dependencies
        run: |
          cd image-search-service
          uv sync --dev
      - name: Run PostgreSQL tests
        working-directory: image-search-service
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/image_search_test
        run: uv run pytest -m postgres -v --tb=short -p no:xdist
```

---

### MC2: COVERAGE_CORE=sysmon for Faster Coverage

#### What It Does

Python 3.12 introduced `sys.monitoring` (PEP 669) — a low-overhead mechanism for code instrumentation. Setting `COVERAGE_CORE=sysmon` tells `coverage.py` (used by `pytest-cov`) to use this instead of the traditional `sys.settrace` approach.

Real-world benchmarks show **30-53% reduction** in coverage collection overhead:
- Trail of Bits / PyPI case study: 53% reduction
- Typical Python projects: 30-50% depending on test profile

#### Implementation

This is already included in the CI workflow above via the environment variable:

```yaml
env:
  COVERAGE_CORE: sysmon
```

No code changes required. No new dependencies. Works with the existing `pytest-cov>=7.0.0` already in the dev dependencies.

#### Requirements

- Python >= 3.12 (already satisfied: `requires-python = ">=3.12,<3.13"`)
- coverage.py >= 7.4 (already satisfied via `pytest-cov>=7.0.0` which pulls `coverage>=7.0`)

#### Local Usage

To use fast coverage locally:

```bash
cd image-search-service
COVERAGE_CORE=sysmon uv run pytest --cov=image_search_service --cov-report=html -p no:xdist
```

#### Add Makefile Target

**File**: `image-search-service/Makefile`

Add this target (after Phase 1's Makefile changes):

```makefile
test-cov: ## Run tests with fast coverage collection
	COVERAGE_CORE=sysmon uv run pytest --cov=image_search_service --cov-report=html -p no:xdist
```

---

## 3. Generating Test Timing Data

The `.test_durations` file is the critical input for balanced sharding. Here's the complete procedure.

### Initial Generation

```bash
cd image-search-service

# Step 1: Ensure all dependencies are installed (Phase 1 + Phase 3)
uv sync --dev

# Step 2: Run full suite serially with timing capture
# IMPORTANT: Disable xdist and randomly for accurate timings
uv run pytest --store-durations -p no:xdist -p no:randomly -q

# Step 3: Verify the file
ls -la .test_durations
python3 -c "
import json, sys
with open('.test_durations') as f:
    data = json.load(f)
total = sum(data.values())
tests = len(data)
print(f'Tests:          {tests}')
print(f'Total duration: {total:.1f}s')
print(f'Avg per test:   {total/tests*1000:.0f}ms')
print(f'Target per shard (4 shards): {total/4:.1f}s')
# Show projected shard balance
sorted_tests = sorted(data.items(), key=lambda x: x[1], reverse=True)
print(f'Longest test:   {sorted_tests[0][0]} ({sorted_tests[0][1]:.2f}s)')
print(f'Shortest test:  {sorted_tests[-1][0]} ({sorted_tests[-1][1]:.4f}s)')
"

# Step 4: Commit
git add .test_durations
git commit -m "chore(tests): add pytest-split timing data for CI sharding"
```

### Verifying Shard Balance Locally

Before pushing to CI, verify the splits are balanced:

```bash
cd image-search-service

# Dry-run each shard to see which tests it would run
for i in 1 2 3 4; do
  echo "=== Shard $i ==="
  uv run pytest --splits 4 --group $i --collect-only -q -p no:xdist 2>/dev/null | tail -1
done
```

Expected output: each shard should have roughly equal test counts (±10%). Duration balance matters more than count balance — pytest-split optimizes for duration.

### What Happens When Tests Are Added/Removed

- **New tests not in `.test_durations`**: pytest-split assigns them a default duration (the median of existing tests) and distributes them to the least-loaded shard. This works well enough until timings are refreshed.
- **Removed tests in `.test_durations`**: Ignored silently. The stale entries don't cause errors.
- **Significantly changed test durations**: Shards may become imbalanced until `.test_durations` is refreshed.

This means sharding works even with a stale `.test_durations` file — it just becomes less balanced over time.

---

## 4. Validation Steps

### Step 1: Verify pytest-split Works Locally

```bash
cd image-search-service

# Generate timing data if not already done
uv run pytest --store-durations -p no:xdist -p no:randomly -q

# Run shard 1 of 4 (should run ~25% of tests)
uv run pytest --splits 4 --group 1 -p no:xdist -v 2>&1 | tail -5

# Run shard 2 of 4
uv run pytest --splits 4 --group 2 -p no:xdist -v 2>&1 | tail -5

# Verify: total tests across all shards == total tests without splitting
TOTAL_SPLIT=0
for i in 1 2 3 4; do
  COUNT=$(uv run pytest --splits 4 --group $i --collect-only -q -p no:xdist 2>/dev/null | tail -1 | grep -o '[0-9]*')
  echo "Shard $i: $COUNT tests"
  TOTAL_SPLIT=$((TOTAL_SPLIT + COUNT))
done
echo "Total (split): $TOTAL_SPLIT"

TOTAL=$(uv run pytest --collect-only -q -p no:xdist 2>/dev/null | tail -1 | grep -o '[0-9]*')
echo "Total (unsplit): $TOTAL"

# These numbers MUST match
```

### Step 2: Verify All Shards Pass

```bash
cd image-search-service

# Run each shard individually and confirm all pass
for i in 1 2 3 4; do
  echo "=== Running shard $i/4 ==="
  uv run pytest --splits 4 --group $i -p no:xdist -q
  echo "=== Shard $i exit code: $? ==="
done
```

### Step 3: Verify xdist + pytest-split Combination

```bash
cd image-search-service

# Run shard 1 with xdist enabled (how CI will run it)
uv run pytest --splits 4 --group 1 -v 2>&1 | tail -10

# Confirm pass count and timing
time uv run pytest --splits 4 --group 1
```

### Step 4: Verify COVERAGE_CORE=sysmon

```bash
cd image-search-service

# Run with sysmon coverage and verify report generates
COVERAGE_CORE=sysmon uv run pytest --cov=image_search_service --cov-report=term-missing -p no:xdist -q 2>&1 | tail -20

# Compare coverage output (should be identical results, just faster)
# Normal:
time uv run pytest --cov=image_search_service -p no:xdist -q 2>/dev/null
# With sysmon:
time COVERAGE_CORE=sysmon uv run pytest --cov=image_search_service -p no:xdist -q 2>/dev/null
```

### Step 5: Push and Verify CI

After local validation:

```bash
git push origin <branch>
# Open PR and verify:
# 1. Lint job runs once and passes
# 2. Four test shards start in parallel
# 3. All four shards pass
# 4. Each shard completes in ~15-20s (check the "Run tests" step timing)
# 5. No test appears in multiple shards (check logs for unique test names)
```

---

## 5. Rollback Plan

### Full Rollback (Revert to Pre-Phase 3)

If Phase 3 causes issues in CI:

**Step 1**: Revert the CI workflow to the single-job version:

```yaml
# .github/workflows/ci.yml — reverted to pre-Phase 3
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
        run: |
          cd image-search-service
          uv sync --dev

      - name: Run linter
        run: |
          cd image-search-service
          uv run ruff check .

      - name: Run type checker
        run: |
          cd image-search-service
          uv run mypy src/

      - name: Run tests
        working-directory: image-search-service
        run: uv run pytest
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/image_search_test
          REDIS_URL: redis://localhost:6379/0
          QDRANT_URL: http://localhost:6333
```

**Step 2**: Remove pytest-split from dependencies (optional — it's harmless to keep):

```bash
cd image-search-service
# Edit pyproject.toml: remove "pytest-split>=0.9.0" from [dependency-groups] dev
uv sync --dev
```

**Step 3**: The `.test_durations` file is harmless and can be left in the repo.

### Partial Rollback (Disable Sharding, Keep sysmon)

If sharding causes issues but sysmon is fine:

```yaml
# In ci.yml test job — remove the matrix and split flags:
jobs:
  test:
    runs-on: ubuntu-latest
    # Remove: strategy/matrix block
    steps:
      # ... same setup ...
      - name: Run tests
        working-directory: image-search-service
        env:
          COVERAGE_CORE: sysmon  # Keep this — it's safe
        run: uv run pytest --durations=10
```

### Partial Rollback (Reduce Shard Count)

If 4 shards cause imbalance, reduce to 2 or 3:

```yaml
strategy:
  matrix:
    group: [1, 2]  # or [1, 2, 3]
# ...
run: uv run pytest --splits 2 --group ${{ matrix.group }}  # match count
```

---

## 6. Success Criteria

### Primary Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| CI wall-clock (test step, slowest shard) | **< 20 seconds** | GitHub Actions job timing |
| All 4 shards pass | **100%** | All matrix jobs green |
| Total test count across shards | **= total test count** | Sum of `--collect-only` across shards |
| No test runs in multiple shards | **0 duplicates** | Grep CI logs for duplicate test IDs |
| Shard duration balance | **< 30% variance** | Compare slowest vs fastest shard |

### Secondary Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Coverage with sysmon matches baseline | **Same lines covered** | Compare coverage reports |
| CI total machine-minutes | **< 5 min** (4 shards x ~1.2 min) | GitHub Actions billing |
| Lint job passes independently | **Yes** | Lint job green before test starts |

### Validation Commands (Post-Deploy)

```bash
# After first CI run with sharding:

# 1. Check shard balance in CI logs
# Look for "X passed" in each shard's output
# Ideal: each shard runs ~274 tests (1096/4)

# 2. Check wall-clock per shard
# In GitHub Actions UI: click each matrix job, check "Run tests" step duration
# Target: all 4 shards complete in 15-20s

# 3. Verify no test was skipped
# Sum of "X passed" across all 4 shards should equal total test count
# Expected: 274 + 274 + 274 + 274 ≈ 1096

# 4. Locally verify shard completeness
cd image-search-service
TOTAL_SPLIT=0
for i in 1 2 3 4; do
  COUNT=$(uv run pytest --splits 4 --group $i --collect-only -q -p no:xdist 2>/dev/null | tail -1 | grep -o '[0-9]*')
  TOTAL_SPLIT=$((TOTAL_SPLIT + COUNT))
done
TOTAL=$(uv run pytest --collect-only -q -p no:xdist 2>/dev/null | tail -1 | grep -o '[0-9]*')
echo "Split total: $TOTAL_SPLIT, Actual total: $TOTAL"
# These MUST be equal
```

---

## 7. Known Risks

### Risk 1: Shard Imbalance

**Probability**: Medium
**Impact**: Low (worst case: one shard takes 2x longer than others)
**Cause**: `.test_durations` data becomes stale as tests are added/modified.
**Mitigation**: Regenerate `.test_durations` monthly (see Section 8).
**Detection**: In CI logs, compare "Run tests" step duration across all 4 matrix jobs. If the slowest shard takes >2x the fastest, regenerate timing data.

### Risk 2: Stale Timing Data

**Probability**: High (will happen naturally over time)
**Impact**: Low (sharding still works, just less balanced)
**Cause**: New tests don't appear in `.test_durations`; removed tests are ignored.
**Mitigation**: pytest-split assigns unknown tests the median duration. This is good enough for weeks. Regenerate timing data when you notice imbalance.
**Detection**: `python3 -c "import json; d=json.load(open('.test_durations')); print(len(d))"` — compare to `uv run pytest --collect-only -q | tail -1`.

### Risk 3: pytest-split + xdist Interaction

**Probability**: Low
**Impact**: Medium (tests could be skipped or duplicated)
**Cause**: pytest-split filters tests at collection time; xdist distributes at execution time. These are independent mechanisms that compose well.
**Mitigation**: The validation step (Section 4, Step 1) explicitly checks that total tests across shards equals total tests without splitting.
**Detection**: Sum of pass counts across shards != expected total.

### Risk 4: CI Cost Increase

**Probability**: Certain (by design)
**Impact**: Low (GitHub Actions free tier: 2,000 minutes/month)
**Cause**: 4 shards x ~1.2 min each = ~4.8 min of machine-time vs ~1.1 min for a single job.
**Mitigation**: The wall-clock savings (68s -> 20s) is worth the machine-time trade-off. For private repos, monitor GitHub Actions usage.
**Detection**: GitHub Settings > Billing > Actions usage.

### Risk 5: Coverage Report Merging Not Addressed

**Probability**: Certain (if `--cov` is later added to sharded CI)
**Impact**: Medium (4 partial coverage reports, no combined view)
**Cause**: Each shard runs a subset of tests and generates coverage for only those tests.
**Mitigation**: If coverage is needed in sharded CI, add a post-job that:
1. Uploads each shard's `.coverage` file as a GitHub Actions artifact
2. Runs `coverage combine` and `coverage report` in a final job
**Detection**: Coverage report showing <100% of expected coverage.

> **⚠️ REVIEW NOTE (plan-reviewer):** The current CI workflow does NOT include `--cov`, so this is not an immediate issue. But when coverage is added to sharded CI, a coverage combination step will be required. Plan this now to avoid surprises.

### Risk 6: Empty Shard Not Detected

**Probability**: Low
**Impact**: Medium (CI shows "green" but a shard ran 0 tests, meaning some tests were skipped)
**Cause**: Stale `.test_durations` after major test file renames/deletions.
**Mitigation**: Add a CI step that fails if any shard runs fewer than 50 tests:
```yaml
- name: Verify shard completeness
  run: |
    COUNT=$(grep -c "passed\|PASSED" test-output.txt || echo "0")
    if [ "$COUNT" -lt 50 ]; then
      echo "ERROR: Shard ${{ matrix.group }} ran only $COUNT tests (expected ~274)"
      exit 1
    fi
```

### Risk 7: Missing `.test_durations` in CI

**Probability**: Low (one-time, if someone forgets to commit it)
**Impact**: Medium (shards will be unbalanced — tests distributed by filename instead)
**Cause**: `.test_durations` not committed or accidentally gitignored.
**Mitigation**: Verify `.test_durations` is NOT in `.gitignore`. The CI workflow does not generate timing data — it reads the committed file.
**Detection**: pytest-split prints a warning if `.test_durations` is missing: `"No test durations found, using fallback splitting"`. Check CI logs for this warning.

---

## 8. Maintenance

### Refreshing Timing Data

**Frequency**: Monthly, or when test count changes by >10%.

```bash
cd image-search-service

# Regenerate timing data
uv run pytest --store-durations -p no:xdist -p no:randomly -q

# Verify the update
python3 -c "
import json
with open('.test_durations') as f:
    data = json.load(f)
print(f'Tests: {len(data)}, Total: {sum(data.values()):.1f}s, Per-shard: {sum(data.values())/4:.1f}s')
"

# Commit
git add .test_durations
git commit -m "chore(tests): refresh pytest-split timing data"
```

### When to Regenerate

| Trigger | Action |
|---------|--------|
| Monthly cadence | Regenerate `.test_durations` |
| >50 new tests added since last refresh | Regenerate |
| Shard imbalance >30% (slowest vs fastest) | Regenerate |
| Major test refactoring (Phase 2 parametrization) | Regenerate immediately |
| After Phase 1 is applied | Generate for the first time |

### Automated Timing Data Refresh (Optional)

If you want to automate timing data refresh, add a scheduled workflow:

```yaml
# .github/workflows/refresh-test-timings.yml
name: Refresh Test Timings

on:
  schedule:
    - cron: '0 6 1 * *'  # First of every month at 6 AM UTC
  workflow_dispatch:       # Manual trigger

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python 3.12
        run: uv python install 3.12

      - name: Install dependencies
        run: |
          cd image-search-service
          uv sync --dev

      - name: Regenerate timing data
        working-directory: image-search-service
        run: uv run pytest --store-durations -p no:xdist -p no:randomly -q

      - name: Commit updated timings
        working-directory: image-search-service
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .test_durations
          git diff --cached --quiet || git commit -m "chore(tests): auto-refresh pytest-split timing data"
          git push
```

This workflow runs monthly and commits updated timing data. The `git diff --cached --quiet ||` guard prevents empty commits when nothing changes.

### Monitoring Shard Health

Add this check to your PR review process:

```bash
# Quick check: are shards balanced?
cd image-search-service
for i in 1 2 3 4; do
  COUNT=$(uv run pytest --splits 4 --group $i --collect-only -q -p no:xdist 2>/dev/null | tail -1 | grep -o '[0-9]*')
  echo "Shard $i: $COUNT tests"
done
```

If any shard has >40% more tests than the smallest shard, regenerate timing data.

---

## Appendix A: Complete File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `image-search-service/pyproject.toml` | MODIFY | Add `pytest-split>=0.9.0` to dev dependencies |
| `image-search-service/.test_durations` | CREATE | Auto-generated timing data (commit to repo) |
| `image-search-service/.github/workflows/ci.yml` | MODIFY | Split into lint + sharded test jobs |
| `image-search-service/Makefile` | MODIFY | Add `test-cov` target |

## Appendix B: pytest-split CLI Reference

```bash
# Generate/update timing data
uv run pytest --store-durations

# Run specific shard
uv run pytest --splits N --group G              # G in [1..N]

# Use specific splitting algorithm
uv run pytest --splits 4 --group 1 --splitting-algorithm least_duration    # default, best balance
uv run pytest --splits 4 --group 1 --splitting-algorithm round_robin       # simple, less balanced
uv run pytest --splits 4 --group 1 --splitting-algorithm duration_based_chunks  # contiguous blocks

# Combine with xdist (CI pattern)
uv run pytest --splits 4 --group 1 -n auto --dist worksteal

# Dry-run: see which tests are in each shard
uv run pytest --splits 4 --group 1 --collect-only -q
```

## Appendix C: Troubleshooting

### "No test durations found" Warning

**Cause**: `.test_durations` file missing or in wrong directory.
**Fix**:
```bash
cd image-search-service
ls -la .test_durations  # Should exist in the service root
# If missing:
uv run pytest --store-durations -p no:xdist -p no:randomly -q
git add .test_durations && git commit -m "chore: add test timing data"
```

### Shard Shows 0 Tests

**Cause**: `.test_durations` references test IDs that no longer exist (renamed/deleted).
**Fix**: Regenerate timing data:
```bash
uv run pytest --store-durations -p no:xdist -p no:randomly -q
```

### One Shard Takes Much Longer Than Others

**Cause**: Stale timing data or a single very slow test.
**Fix**:
1. Check `--durations=10` output in the slow shard's CI log
2. If a single test dominates: investigate that test
3. If generally imbalanced: regenerate timing data

### Coverage Report Differs with sysmon

**Cause**: `COVERAGE_CORE=sysmon` uses `sys.monitoring` which has slightly different line-level precision in rare edge cases (e.g., multi-line expressions).
**Fix**: This is expected and negligible. If exact coverage parity is required, remove the `COVERAGE_CORE: sysmon` env var from CI.

### pytest-split Conflicts with pytest-randomly

**Cause**: `pytest-randomly` reorders tests after `pytest-split` selects them. This is fine — it only changes order within the shard, not which tests are in each shard.
**Fix**: No fix needed. They compose correctly. However, when generating `.test_durations`, always use `-p no:randomly` to get deterministic timing.
