# Pytest Acceleration Techniques Research

**Date**: 2026-02-15
**Researcher**: techniques-researcher (test-acceleration-research team)
**Scope**: Modern pytest acceleration techniques for the image-search-service backend
**Classification**: Actionable

---

## Executive Summary

The image-search-service backend has **1,096 active test cases** (15 deselected postgres-marked tests) running in **~136 seconds** on the current machine. Tests are 100% async (pytest-asyncio auto mode) with SQLite in-memory databases. The dominant bottleneck is **fixture setup overhead** (0.6-0.8s per API test) rather than test execution itself. Only 2 tests exceed 1s of call time.

This research identifies **12 acceleration techniques** with estimated combined speedup potential of **4-8x** (target: 17-34 seconds), prioritized by effort-to-impact ratio.

---

## Table of Contents

1. [Current Baseline Analysis](#1-current-baseline-analysis)
2. [Technique 1: pytest-xdist Parallel Execution](#2-technique-1-pytest-xdist-parallel-execution)
3. [Technique 2: pytest-split CI Test Sharding](#3-technique-2-pytest-split-ci-test-sharding)
4. [Technique 3: Test Ordering Optimizations](#4-technique-3-test-ordering-optimizations)
5. [Technique 4: Fixture Scope Optimization](#5-technique-4-fixture-scope-optimization)
6. [Technique 5: Database Fixture Strategies](#6-technique-5-database-fixture-strategies)
7. [Technique 6: Mock Optimization](#7-technique-6-mock-optimization)
8. [Technique 7: pytest Cache and Incremental Testing](#8-technique-7-pytest-cache-and-incremental-testing)
9. [Technique 8: Test Profiling Tools](#9-technique-8-test-profiling-tools)
10. [Technique 9: CI-Specific Strategies](#10-technique-9-ci-specific-strategies)
11. [Technique 10: Import Optimization](#11-technique-10-import-optimization)
12. [Technique 11: pytest-xdist vs Alternatives](#12-technique-11-pytest-xdist-vs-alternatives)
13. [Technique 12: SQLite In-Memory Optimization](#13-technique-12-sqlite-in-memory-optimization)
14. [Prioritized Implementation Roadmap](#14-prioritized-implementation-roadmap)
15. [Sources](#15-sources)

---

## 1. Current Baseline Analysis

### Test Suite Profile

| Metric | Value |
|--------|-------|
| Total test cases | 1,111 (1,096 selected, 15 postgres deselected) |
| Total runtime | ~136 seconds |
| Average time per test | ~124ms |
| Async mode | `asyncio_mode = "auto"` |
| Database backend | SQLite in-memory (`sqlite+aiosqlite:///:memory:`) |
| Python version | 3.12.3 |
| pytest version | 9.0.2 |
| pytest-asyncio version | 1.3.0 |
| Coverage plugin | pytest-cov 7.0.0 |
| No parallel execution plugins installed | confirmed |

### Test Distribution by Directory

| Directory | Test Count | % of Total |
|-----------|-----------|------------|
| `tests/api/` | 384 | 35% |
| `tests/unit/` (all subdirs) | 342 + 73 + 66 + 64 = 545 | 50% |
| `tests/faces/` | 83 | 8% |
| `tests/integration/` | 37 | 3% |
| `tests/core/` | 31 | 3% |
| `tests/scripts/` | 12 | 1% |
| `tests/` (root) | 4 | <1% |

### Identified Bottlenecks

1. **Fixture setup dominates**: The `--durations` output shows most time is in `setup` phase (0.6-0.8s per API test), not `call` phase
2. **Only 2 tests > 1s call time**: `test_model_preload` (3.78s) and `test_sequential_job_processing` (1.49s)
3. **4 autouse fixtures per test**: `clear_settings_cache`, `use_test_settings`, `clear_embedding_cache`, `validate_embedding_dimensions` - all run for every single test
4. **Per-test DB engine creation**: Each test using `db_engine` creates a fresh async engine, runs `Base.metadata.create_all`, then drops all tables after
5. **Per-test app creation**: The `test_client` fixture calls `create_app()` for every test, which includes middleware setup, router registration, etc.
6. **No pytest-xdist installed**: Tests run purely sequentially on a single core

---

## 2. Technique 1: pytest-xdist Parallel Execution

### Overview

pytest-xdist distributes tests across multiple worker processes. Each worker is a completely independent pytest session with its own event loop, database, and fixtures.

### Configuration

```toml
# pyproject.toml
[dependency-groups]
dev = [
    # ... existing deps ...
    "pytest-xdist>=3.8.0",
]

[tool.pytest.ini_options]
# Add to existing config
addopts = "-m 'not postgres' -n auto --dist worksteal"
```

```bash
# Manual invocation with tuning
uv run pytest -n auto                    # auto-detect CPU cores
uv run pytest -n 4                       # fixed 4 workers
uv run pytest -n auto --dist worksteal   # best for varying test durations
uv run pytest -n auto --dist loadscope   # group by module (fixture reuse)
uv run pytest -n auto --dist loadfile    # group by file
```

### Distribution Mode Selection for This Project

| Mode | Best For | Recommendation |
|------|----------|----------------|
| `load` (default) | Equal-duration tests | Not ideal (our tests vary significantly) |
| **`worksteal`** | **Varying test durations** | **Best fit** - handles 3.78s outlier well |
| `loadscope` | Expensive shared fixtures | Good if we add module-scoped DB fixtures |
| `loadfile` | File-level isolation | Good for API tests that share setup |
| `loadgroup` | Custom grouping | Useful for integration tests |

### Compatibility with Current Setup

**Compatible**:
- SQLite `:memory:` databases are per-process (automatic isolation)
- In-memory Qdrant clients are per-process (automatic isolation)
- Mock embedding service is per-process (no shared state)
- `pytest-asyncio` auto mode works with xdist (each worker has its own event loop)

**Requires attention**:
- The `monkeypatch` fixture used in autouse fixtures is function-scoped (works fine per-worker)
- `tmp_path` is automatically unique per worker (pytest-xdist handles this)
- `conftest_postgres.py` imports (only for postgres-marked tests, already deselected)

### Expected Speedup

On a machine with N CPU cores:
- **Theoretical**: N x speedup
- **Practical (4 cores)**: 2.5-3.5x (overhead from worker startup, IPC, load imbalance)
- **Practical (8 cores)**: 4-6x
- **This project estimate**: **~3x on 4 cores** (136s → ~45s)

The overhead is relatively low because:
- Tests have no shared external services (all in-memory/mocked)
- Each test is independent (no inter-test dependencies)
- The worksteal scheduler handles the 3.78s outlier well

### Makefile Update

```makefile
test: ## Run pytest tests (fast SQLite tier only)
	uv run pytest

test-fast: ## Run pytest with parallel execution
	uv run pytest -n auto --dist worksteal

test-serial: ## Run pytest in serial mode (debugging)
	uv run pytest -n0
```

---

## 3. Technique 2: pytest-split CI Test Sharding

### Overview

pytest-split divides the test suite into N roughly-equal-duration groups based on historical timing data, allowing CI to run each group on a separate machine/container.

### Configuration

```toml
# pyproject.toml
[dependency-groups]
dev = [
    # ... existing deps ...
    "pytest-split>=0.9.0",
]
```

```bash
# Step 1: Generate timing data (run once, commit the file)
uv run pytest --store-durations

# Step 2: Run a specific split (in CI)
uv run pytest --splits 3 --group 1  # Worker 1 of 3
uv run pytest --splits 3 --group 2  # Worker 2 of 3
uv run pytest --splits 3 --group 3  # Worker 3 of 3
```

### GitHub Actions Matrix Example

```yaml
jobs:
  test:
    strategy:
      matrix:
        group: [1, 2, 3]
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: uv sync --dev
      - name: Run tests (shard ${{ matrix.group }}/3)
        run: uv run pytest --splits 3 --group ${{ matrix.group }}
```

### Combining with pytest-xdist

pytest-split and pytest-xdist can be combined: each CI shard runs a subset of tests in parallel:

```bash
# Each CI shard runs ~365 tests across 4 cores
uv run pytest --splits 3 --group 1 -n auto --dist worksteal
```

### Expected Speedup

- **3 CI shards**: ~3x reduction in wall-clock CI time
- **Combined with xdist (4 cores per shard)**: ~9-12x total speedup
- **This project estimate**: 136s → ~15s per shard (with xdist)

---

## 4. Technique 3: Test Ordering Optimizations

### Fail-Fast Mode (`-x` / `--maxfail`)

```bash
# Stop after first failure (fastest feedback during development)
uv run pytest -x

# Stop after 3 failures
uv run pytest --maxfail=3
```

### Run Last Failed First (`--lf`, `--ff`)

```bash
# Run ONLY previously failed tests
uv run pytest --lf

# Run failed tests FIRST, then the rest
uv run pytest --ff

# Combine: failed first + parallel
uv run pytest --ff -n auto --dist worksteal
```

### pytest-ordering / pytest-order

```toml
[dependency-groups]
dev = [
    "pytest-order>=1.3.0",
]
```

```python
# Mark slow tests to run last
@pytest.mark.order("last")
def test_model_preload():
    ...

# Mark critical tests to run first
@pytest.mark.order("first")
def test_health_endpoint():
    ...
```

### Custom pytest Hook (Fastest-First)

```python
# conftest.py
def pytest_collection_modifyitems(config, items):
    """Reorder tests: unit tests first, integration last."""
    unit_tests = []
    api_tests = []
    integration_tests = []
    other_tests = []

    for item in items:
        path = str(item.fspath)
        if "/unit/" in path:
            unit_tests.append(item)
        elif "/api/" in path:
            api_tests.append(item)
        elif "/integration/" in path:
            integration_tests.append(item)
        else:
            other_tests.append(item)

    items[:] = unit_tests + other_tests + api_tests + integration_tests
```

### Expected Speedup

- **`-x` (fail-fast)**: Near-instant feedback on first failure (development workflow)
- **`--ff` (failed-first)**: Seconds to re-verify fixes vs full 136s
- **Custom ordering**: No direct speedup, but faster feedback on failures
- **Recommended for**: Development workflow, not CI (CI should run all tests)

---

## 5. Technique 4: Fixture Scope Optimization

### Current Problem

The project has 4 `autouse=True` fixtures that run for **every single test** (1,096 times):

```python
# 1. clear_settings_cache - clears lru_cache before/after each test
# 2. use_test_settings - monkeypatches env vars + clears cache
# 3. clear_embedding_cache - clears lru_cache + monkeypatches methods
# 4. validate_embedding_dimensions - asserts dimensions match (every test!)
```

### Optimization Opportunities

#### 4a. Move `validate_embedding_dimensions` to Session Scope

This fixture only validates constants - it doesn't need to run per test:

```python
@pytest.fixture(autouse=True, scope="session")
def validate_embedding_dimensions():
    """Validate once at session start, not per test."""
    assert MockEmbeddingService.EMBEDDING_DIM == 768
    test_instance = MockEmbeddingService()
    assert test_instance.embedding_dim == 768
    yield
```

**Savings**: Eliminates ~1,096 redundant assertion+instantiation cycles.

#### 4b. Consolidate Settings Fixtures

Merge `clear_settings_cache` and `use_test_settings` into one fixture:

```python
@pytest.fixture(autouse=True)
def test_settings(monkeypatch):
    """Single fixture for all test settings."""
    from image_search_service.core.config import get_settings
    get_settings.cache_clear()
    monkeypatch.setenv("QDRANT_COLLECTION", "test_image_assets")
    monkeypatch.setenv("QDRANT_FACE_COLLECTION", "test_faces")
    get_settings.cache_clear()
    yield
    get_settings.cache_clear()
```

**Savings**: Eliminates 1 fixture call + 2 extra `cache_clear()` calls per test.

#### 4c. Session-Scoped Embedding Mock

If embedding service is stateless (it is - uses cached SemanticMockEmbeddingService):

```python
@pytest.fixture(autouse=True, scope="session")
def mock_embeddings():
    """Patch embedding service once for entire session."""
    semantic_mock = SemanticMockEmbeddingService()
    with unittest.mock.patch.multiple(
        "image_search_service.services.embedding.EmbeddingService",
        embed_text=lambda self, text: semantic_mock.embed_text(text),
        embed_image=lambda self, path: semantic_mock.embed_image(path),
        embed_images_batch=lambda self, imgs: semantic_mock.embed_images_batch(imgs),
    ):
        yield
```

**Note**: This requires switching from `monkeypatch` (function-scoped) to `unittest.mock.patch` (supports session scope). Must verify no test modifies the embedding mock behavior.

### Fixture Scoping Decision Matrix

| Fixture | Current Scope | Recommended | Reason |
|---------|--------------|-------------|--------|
| `clear_settings_cache` | function | **merge** | Consolidate with use_test_settings |
| `use_test_settings` | function | **function** | Must stay function - monkeypatch is function-scoped |
| `clear_embedding_cache` | function | **session** (with mock.patch) | Stateless, same every test |
| `validate_embedding_dimensions` | function | **session** | Only validates constants |
| `db_engine` | function | **module** (see DB section) | Expensive create_all |
| `test_client` | function | **module** (see DB section) | Expensive app creation |

### Expected Speedup

- **Session-scoped validation**: Negligible (saves microseconds per test)
- **Consolidated settings**: ~5% (less fixture overhead)
- **Session-scoped embedding mock**: ~5-10% (eliminates per-test patching)
- **Combined**: **~10-15% reduction** (136s → ~118s)

---

## 6. Technique 5: Database Fixture Strategies

### Current Architecture

```
Per test:
  1. create_async_engine(sqlite:///:memory:)     # New engine
  2. Base.metadata.create_all()                    # Create all tables
  3. async_sessionmaker() → session                # New session
  4. ... test runs ...
  5. session.rollback()                            # Rollback
  6. Base.metadata.drop_all()                      # Drop all tables
  7. engine.dispose()                              # Dispose engine
```

This means **every API test** creates and destroys the entire schema. With ~25+ SQLAlchemy models, this is the #1 bottleneck (0.6-0.8s setup per API test).

### Strategy A: Module-Scoped Engine + Savepoint Rollback

```python
@pytest.fixture(scope="module")
async def db_engine():
    """Create engine once per module, share across tests."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest.fixture
async def db_session(db_engine):
    """Per-test session with savepoint rollback for isolation."""
    async_session = async_sessionmaker(db_engine, expire_on_commit=False)
    async with async_session() as session:
        # Note: SQLite doesn't support nested transactions/savepoints natively
        # with aiosqlite, so we rely on full rollback
        yield session
        await session.rollback()
```

**Caveat**: SQLite has limited savepoint support with async drivers. For SQLite, the simpler approach is to use a module-scoped engine with function-scoped sessions that rollback.

### Strategy B: Named In-Memory SQLite with Shared Cache

```python
# Use a named in-memory database shared across connections
TEST_DATABASE_URL = "sqlite+aiosqlite:///file:testdb?mode=memory&cache=shared&uri=true"
```

This allows multiple connections to share the same in-memory database, which is essential for module-scoped engines with function-scoped sessions.

### Strategy C: Session-Scoped Engine with Table Truncation

```python
@pytest.fixture(scope="session")
async def db_engine():
    """Single engine for entire test session."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest.fixture(autouse=True)
async def clean_tables(db_engine):
    """Truncate all tables between tests (faster than recreating)."""
    yield
    async with db_engine.begin() as conn:
        for table in reversed(Base.metadata.sorted_tables):
            await conn.execute(table.delete())
```

### Expected Speedup

- **Strategy A (module-scoped)**: ~30-40% reduction in fixture setup time
  - 80 test files → 80 `create_all` instead of 1,096
  - **Estimate**: 136s → ~85-95s
- **Strategy C (session-scoped + truncate)**: ~40-50% reduction
  - 1 `create_all` for entire session
  - TRUNCATE is ~10-50x faster than DROP+CREATE
  - **Estimate**: 136s → ~70-80s

**Risk**: Test isolation. If any test mutates schema (unlikely in this project), it affects all subsequent tests. Mitigation: run `make test` in serial mode occasionally to verify isolation.

---

## 7. Technique 6: Mock Optimization

### Current Mock Architecture

The project uses a well-designed mock system:
1. **SemanticMockEmbeddingService**: Deterministic 768-dim vectors with semantic clustering
2. **In-memory Qdrant client**: Uses `:memory:` mode (no external server)
3. **monkeypatch-based patching**: Function-scoped, clean teardown

### Optimization Opportunities

#### 6a. Module-Level Patching with `unittest.mock.patch`

Replace per-test `monkeypatch.setattr` with module-level `mock.patch`:

```python
# Instead of monkeypatch (function-scoped only):
@pytest.fixture(autouse=True, scope="module")
def mock_embedding_module():
    """Patch embedding at module level."""
    mock = SemanticMockEmbeddingService()
    with (
        patch.object(EmbeddingService, "embed_text", side_effect=mock.embed_text),
        patch.object(EmbeddingService, "embed_image", side_effect=mock.embed_image),
        patch.object(EmbeddingService, "embed_images_batch", side_effect=mock.embed_images_batch),
    ):
        yield
```

#### 6b. Reduce Qdrant Fixture Setup

The `qdrant_client` fixture creates 2 collections + 5 payload indexes every test. Move to module scope:

```python
@pytest.fixture(scope="module")
def qdrant_client():
    """Create in-memory Qdrant client once per module."""
    client = QdrantClient(":memory:")
    settings = get_settings()
    client.create_collection(
        collection_name=settings.qdrant_collection,
        vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    )
    # ... create face collection + indexes ...
    return client
```

**Caveat**: Tests that insert data into Qdrant need cleanup between tests, or each test module gets a fresh client.

#### 6c. Pre-computed Mock Embeddings

The `SemanticMockEmbeddingService` uses `np.random.RandomState` and SHA-256 hashing for each call. Pre-computing common embeddings could save CPU:

```python
# Pre-compute embeddings for common test strings at module level
_PRECOMPUTED = {}
def _lazy_embed(text):
    if text not in _PRECOMPUTED:
        _PRECOMPUTED[text] = _original_embed(text)
    return _PRECOMPUTED[text]
```

Note: The existing `self._cache` dict already does this, so the improvement is minimal.

### Expected Speedup

- **Module-scoped mocks**: ~5-10% (less fixture overhead per test)
- **Module-scoped Qdrant**: ~10-15% (eliminates per-test collection creation)
- **Combined with DB optimization**: **~15-20% additional** on top of DB savings

---

## 8. Technique 7: pytest Cache and Incremental Testing

### Built-in pytest Cache (`--lf`, `--ff`)

```bash
# During development: run only last-failed tests
uv run pytest --lf                    # Only previously failed
uv run pytest --ff                    # Failed first, then rest
uv run pytest --cache-show            # Show cache contents
uv run pytest --cache-clear           # Clear cache
```

### pytest-testmon (Coverage-Based Selection)

```toml
[dependency-groups]
dev = [
    "pytest-testmon>=2.1.0",
]
```

```bash
# First run: collect coverage data (slower)
uv run pytest --testmon

# Subsequent runs: only run affected tests
uv run pytest --testmon
# Output: "2/1096 tests selected, 1094 deselected by testmon"
```

**How it works**: testmon traces which source files each test depends on (via coverage.py). When you change a file, only tests that imported/used that file are re-run.

**This project benefit**: When editing a single service file, typically only 10-50 tests need to re-run instead of 1,096.

### pytest-incremental

```toml
[dependency-groups]
dev = [
    "pytest-incremental>=0.6.0",
]
```

Lighter-weight than testmon: uses import graph analysis instead of line-level coverage. Faster initial setup, but less precise test selection.

### Expected Speedup (Development Workflow)

| Scenario | Tests Run | Time |
|----------|----------|------|
| Full suite | 1,096 | ~136s |
| `--lf` after 1 failure | 1 | <1s |
| `--ff` after 3 failures | 3 + 1,093 | ~136s but failures shown in <3s |
| `--testmon` (single file change) | ~20-50 | ~5-10s |
| `--testmon` (cross-cutting change) | ~200-500 | ~30-70s |

**Recommendation**: `--testmon` for development, full suite for CI.

---

## 9. Technique 8: Test Profiling Tools

### Built-in `--durations`

```bash
# Show 20 slowest tests (setup + call + teardown)
uv run pytest --durations=20

# Show all tests with duration > 0.5s
uv run pytest --durations=0 --durations-min=0.5

# Combine with verbose for detailed breakdown
uv run pytest --durations=20 -v
```

### pytest-profiling

```toml
[dependency-groups]
dev = [
    "pytest-profiling>=1.7.0",
]
```

```bash
# Generate cProfile data for each test
uv run pytest --profile

# Generate SVG flame graph
uv run pytest --profile-svg

# Profile specific test
uv run pytest tests/api/test_faces_routes.py --profile-svg
```

### pytest-benchmark

```toml
[dependency-groups]
dev = [
    "pytest-benchmark>=5.0.0",
]
```

```python
# Benchmark a specific operation
def test_embedding_performance(benchmark):
    mock = SemanticMockEmbeddingService()
    result = benchmark(mock.embed_text, "sunset on the beach")
    assert len(result) == 768
```

### pytest-monitor (CPU + Memory)

```toml
[dependency-groups]
dev = [
    "pytest-monitor>=1.6.0",
]
```

```bash
# Collect CPU and memory metrics
uv run pytest --monitor
# Outputs: pytest_monitor.db (SQLite database with metrics)
```

### Python Import Profiling

```bash
# Profile import time (find slow imports at collection time)
python -X importtime -c "import tests.conftest" 2>&1 | head -30

# Full import timing during test collection
uv run python -X importtime -m pytest --collect-only 2>&1 | sort -t: -k2 -rn | head -20
```

### Recommended Profiling Workflow

1. **Start**: `pytest --durations=20` to identify slowest tests
2. **Drill down**: `pytest --profile-svg` on slow test files to see where time is spent
3. **Import check**: `python -X importtime` to find slow imports at collection time
4. **Ongoing**: Add `--durations=10` to CI to catch regression

---

## 10. Technique 9: CI-Specific Strategies

### Strategy A: Test Sharding with GitHub Actions Matrix

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        group: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --dev
      - name: Run tests (shard ${{ matrix.group }}/4)
        run: uv run pytest --splits 4 --group ${{ matrix.group }} -n auto --dist worksteal
      - name: Upload timing data
        if: matrix.group == 1
        run: |
          uv run pytest --store-durations
          # Commit .test_durations back to repo or upload as artifact
```

### Strategy B: Tiered Test Execution

```yaml
jobs:
  fast-tests:
    runs-on: ubuntu-latest
    steps:
      - run: uv run pytest tests/unit/ tests/core/ -n auto --dist worksteal -x
    # ~40% of tests, runs in ~15s with xdist

  api-tests:
    runs-on: ubuntu-latest
    needs: fast-tests  # Only run if unit tests pass
    steps:
      - run: uv run pytest tests/api/ -n auto --dist worksteal

  integration-tests:
    runs-on: ubuntu-latest
    needs: fast-tests
    steps:
      - run: uv run pytest tests/integration/ tests/faces/ -v
```

### Strategy C: Coverage with `COVERAGE_CORE=sysmon`

```yaml
- name: Run tests with fast coverage
  env:
    COVERAGE_CORE: sysmon  # Python 3.12+ sys.monitoring API
  run: uv run pytest --cov=image_search_service --cov-report=xml -n auto
```

**Impact**: The Trail of Bits / PyPI case study showed **53% reduction** in coverage overhead by switching from traditional tracing to `sys.monitoring`.

### Expected CI Speedup

| Strategy | Wall-Clock Time | Parallelism |
|----------|----------------|-------------|
| Current (serial) | ~136s | 1 |
| + xdist (4 cores) | ~45s | 4 cores |
| + 4 shards | ~34s | 4 machines |
| + xdist per shard | ~12s | 4 machines x 4 cores |
| + sysmon coverage | ~8-10s | 4 machines x 4 cores |

---

## 11. Technique 10: Import Optimization

### Profiling Imports

```bash
# Profile pytest collection imports
python -X importtime -c "import image_search_service" 2>&1 | sort -t: -k2 -rn | head -20

# Check for heavy imports at test collection time
python -X importtime -m pytest --collect-only -q 2>&1 | head -40
```

### Current Import Concerns

The `conftest.py` imports at module level:
```python
import numpy as np                    # Heavy (~200ms)
from PIL import Image                 # Moderate (~50ms)
from qdrant_client import QdrantClient  # Moderate (~100ms)
from httpx import ASGITransport, AsyncClient  # Light
```

Additionally, it imports from `conftest_postgres.py` which imports `testcontainers`:
```python
from tests.conftest_postgres import (  # noqa: F401
    fresh_pg_database, pg_connection_url, ...
)
```

### Optimization: Lazy Fixture Imports

```python
# Move heavy imports inside fixtures instead of module level
@pytest.fixture
def qdrant_client():
    from qdrant_client import QdrantClient
    from qdrant_client.models import Distance, PayloadSchemaType, VectorParams
    # ...
```

### Optimization: Disable Unused Plugins

```toml
[tool.pytest.ini_options]
# Disable built-in plugins not needed
addopts = "-p no:pastebin -p no:doctest -p no:nose"
```

### PEP 810 (Python 3.14): Explicit Lazy Imports

Coming in Python 3.14, `lazy` keyword enables deferred imports:
```python
lazy import numpy as np  # Only loaded when first accessed
```

Not available yet in Python 3.12, but good to know for future.

### Expected Speedup

- **Collection time savings**: ~1-2s (from current 1.44s collection)
- **Lazy fixture imports**: ~0.5-1s per-session savings
- **Plugin reduction**: ~0.1-0.3s
- **Total**: ~5-10% of overall test time (**small but easy win**)

---

## 12. Technique 11: pytest-xdist vs Alternatives

### Comparison Matrix

| Feature | pytest-xdist | pytest-parallel | Custom (asyncio.gather) |
|---------|-------------|-----------------|------------------------|
| **Mechanism** | Process-based (fork/spawn) | Thread + process | Single-process async |
| **Isolation** | Full process isolation | Thread-shared state | Single event loop |
| **async test support** | Excellent (via pytest-asyncio) | Limited | Native |
| **Load balancing** | worksteal, loadscope, etc. | Basic | Manual |
| **Maintenance** | Active (3.8.0, 2024) | Inactive (0.0.10, 2020) | N/A |
| **PyPI downloads/month** | ~8M | ~200K | N/A |
| **Coverage support** | Yes (with config) | Limited | Yes |
| **CI integration** | Excellent | Basic | Manual |

### Recommendation: pytest-xdist

**pytest-xdist is the clear winner** for this project:

1. **Active maintenance**: Latest release 3.8.0, regular updates
2. **Process isolation**: Perfect for SQLite `:memory:` (each worker gets its own DB)
3. **pytest-asyncio compatibility**: Each worker runs its own event loop
4. **worksteal scheduler**: Handles the 3.78s outlier test well
5. **Ecosystem**: Works with pytest-split, pytest-cov, pytest-timeout
6. **Community standard**: 8M+ monthly downloads

**pytest-parallel is NOT recommended**:
- Last release was 2020 (effectively abandoned)
- Thread-based mode doesn't work with SQLite (thread safety issues)
- No worksteal or advanced scheduling
- Limited async support

**Custom asyncio.gather is NOT recommended**:
- Requires rewriting test infrastructure
- Single-process = limited to I/O-bound parallelism
- No ecosystem integration (coverage, CI, reporting)

---

## 13. Technique 12: SQLite In-Memory Optimization

### Current Configuration

```python
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
```

Each test creates a new in-memory database. This is the simplest but most expensive approach.

### Optimization A: Named In-Memory with Shared Cache

```python
# Shared across connections within the same process
TEST_DATABASE_URL = "sqlite+aiosqlite:///file:testdb?mode=memory&cache=shared&uri=true"
```

**Benefits**: Allows module/session-scoped engines to share the database across fixtures.

### Optimization B: WAL Mode for Concurrent Access

```python
# Enable WAL mode for better concurrent read performance
@pytest.fixture(scope="session")
async def db_engine():
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        echo=False,
        pool_size=5,
        max_overflow=10,
    )
    async with engine.begin() as conn:
        await conn.execute(text("PRAGMA journal_mode=WAL"))
        await conn.execute(text("PRAGMA synchronous=OFF"))
        await conn.execute(text("PRAGMA cache_size=-64000"))  # 64MB cache
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()
```

### Optimization C: Disable Unnecessary SQLite Features

```python
# Pragmas for maximum test performance
PRAGMAS = [
    "PRAGMA journal_mode=OFF",       # No journaling (we don't need crash recovery)
    "PRAGMA synchronous=OFF",        # No fsync (it's in-memory anyway)
    "PRAGMA cache_size=-64000",      # 64MB page cache
    "PRAGMA temp_store=MEMORY",      # Temp tables in memory
    "PRAGMA mmap_size=268435456",    # Memory-mapped I/O (256MB)
    "PRAGMA locking_mode=EXCLUSIVE", # Single-writer optimization
]
```

**Note**: Most of these pragmas have negligible effect on `:memory:` databases since there's no actual I/O. The main benefit is from shared cache mode enabling module-scoped engines.

### Expected Speedup

- **Shared cache + module-scoped engine**: **~30-40%** (the biggest win from eliminating per-test `create_all`)
- **Pragmas on in-memory DB**: ~1-2% (minimal since no real I/O)
- **Combined with fixture optimization**: **~40-50% total**

---

## 14. Prioritized Implementation Roadmap

### Phase 1: Quick Wins (1-2 hours, ~3x speedup)

| Priority | Technique | Effort | Expected Impact | Risk |
|----------|-----------|--------|-----------------|------|
| **P0** | Install pytest-xdist + `--dist worksteal` | 15 min | **3x** (136s → 45s) | Low |
| **P0** | Add `--durations=10` to CI | 5 min | Visibility | None |
| **P1** | Disable unused plugins (`-p no:pastebin -p no:doctest -p no:nose`) | 5 min | ~0.3s | None |
| **P1** | Session-scope `validate_embedding_dimensions` | 10 min | ~1% | None |
| **P1** | Consolidate settings fixtures | 15 min | ~3% | Low |

**Commands to implement Phase 1**:
```bash
cd image-search-service
uv add --group dev pytest-xdist
# Add to pyproject.toml:
# addopts = "-m 'not postgres' -n auto --dist worksteal -p no:pastebin -p no:doctest -p no:nose"
```

### Phase 2: Fixture Architecture (2-4 hours, additional ~1.5x)

| Priority | Technique | Effort | Expected Impact | Risk |
|----------|-----------|--------|-----------------|------|
| **P1** | Module-scoped DB engine | 2 hrs | **30-40%** | Medium |
| **P1** | Module-scoped Qdrant client | 1 hr | **10-15%** | Medium |
| **P2** | Session-scoped embedding mocks | 1 hr | **5-10%** | Low |
| **P2** | SQLite shared cache mode | 30 min | **5%** | Low |

### Phase 3: Development Workflow (1-2 hours)

| Priority | Technique | Effort | Expected Impact | Risk |
|----------|-----------|--------|-----------------|------|
| **P2** | Install pytest-testmon | 15 min | **10-50x for dev** | Low |
| **P2** | Add `--lf` and `--ff` Makefile targets | 10 min | Dev workflow | None |
| **P2** | pytest-profiling for ongoing monitoring | 15 min | Visibility | None |
| **P3** | Custom test ordering hook | 30 min | Faster failure feedback | Low |

### Phase 4: CI Optimization (2-4 hours)

| Priority | Technique | Effort | Expected Impact | Risk |
|----------|-----------|--------|-----------------|------|
| **P2** | pytest-split with GitHub Actions matrix | 2 hrs | **3-4x CI speedup** | Low |
| **P2** | `COVERAGE_CORE=sysmon` for coverage | 5 min | **30-50% coverage overhead reduction** | Low |
| **P3** | Tiered test execution in CI | 2 hrs | Faster feedback | Medium |

### Combined Speedup Projection

```
Baseline:                            136s
+ Phase 1 (xdist 4 cores):           ~45s  (3.0x)
+ Phase 2 (fixture optimization):    ~25s  (5.4x)
+ Phase 3 (dev: testmon):            ~5-10s per change (dev only)
+ Phase 4 (CI: 4 shards + sysmon):   ~8-10s wall-clock (CI)
```

### Makefile Additions

```makefile
test: ## Run pytest tests (fast, parallel)
	uv run pytest

test-serial: ## Run pytest in serial mode (debugging)
	uv run pytest -p no:xdist

test-fast: ## Run only previously failed tests
	uv run pytest --lf

test-failed-first: ## Run failed tests first, then rest
	uv run pytest --ff

test-affected: ## Run only tests affected by recent changes (requires testmon)
	uv run pytest --testmon

test-profile: ## Profile test durations
	uv run pytest --durations=20 --durations-min=0.5 -p no:xdist

test-cov: ## Run tests with fast coverage
	COVERAGE_CORE=sysmon uv run pytest --cov=image_search_service --cov-report=html -p no:xdist
```

---

## 15. Sources

### Parallel Execution
- [pytest-xdist Documentation](https://pytest-xdist.readthedocs.io/en/stable/distribution.html)
- [Parallel Testing Made Easy With pytest-xdist](https://pytest-with-eric.com/plugins/pytest-xdist/)
- [Pytest Parallel Execution for Large Test Suites in Python 2025](https://johal.in/pytest-parallel-execution-for-large-test-suites-in-python-2025/)
- [pytest-xdist on PyPI](https://pypi.org/project/pytest-xdist/)
- [How Parallel Testing with xdist Made Python Tests 5x Faster](https://techbytesdispatch.medium.com/unleashing-pytest-power-how-parallel-testing-with-xdist-made-python-tests-5x-faster-035c04e77187)

### CI Test Sharding
- [pytest-split on GitHub](https://github.com/jerry-git/pytest-split)
- [Blazing fast CI with pytest-split and GitHub Actions](https://blog.jerrycodes.com/pytest-split-and-github-actions/)
- [How to Distribute Tests with Pytest-Split for Faster CI/CD](https://medium.com/@krijnvanderburg/how-to-distribute-tests-in-ci-cd-for-faster-execution-zero-bs-1-b86d4d69b19d)

### Case Studies
- [Making PyPI's Test Suite 81% Faster (Trail of Bits)](https://blog.trailofbits.com/2025/05/01/making-pypis-test-suite-81-faster/)
- [How we made Python pytest suites 8.5x faster](https://habr.com/en/articles/972444/)
- [Faster Python Tests (Ramp Engineering)](https://engineering.ramp.com/post/faster-python-tests)

### General Optimization
- [13 Proven Ways To Improve Test Runtime With Pytest](https://pytest-with-eric.com/pytest-advanced/pytest-improve-runtime/)
- [awesome-pytest-speedup](https://github.com/zupo/awesome-pytest-speedup)
- [How to speed up pytest (BuildPulse)](https://buildpulse.io/blog/how-to-speed-up-pytest)

### Incremental Testing
- [pytest-testmon](https://www.testmon.org/)
- [Pytest and Testmon Magic: 2x Faster CI (Instawork)](https://engineering.instawork.com/test-impact-analysis-the-secret-to-faster-pytest-runs-e44021306603)

### Coverage Optimization
- [Coverage.py sys.monitoring support](https://coverage.readthedocs.io/)
- [SlipCover: Near Zero-Overhead Python Code Coverage](https://github.com/plasma-umass/slipcover)
- [PEP 810: Explicit Lazy Imports](https://peps.python.org/pep-0810/)

### Profiling
- [pytest-profiling on PyPI](https://pypi.org/project/pytest-profiling/)
- [pytest-benchmark Documentation](https://pytest-benchmark.readthedocs.io/en/latest/)

### Fixture Patterns
- [SQLAlchemy Nested Transaction Pattern for Tests](https://gist.github.com/kissgyorgy/e2365f25a213de44b9a2)
- [SQLite Shared-Cache Mode](https://sqlite.org/sharedcache.html)
- [SQLite In-Memory Databases](https://www.sqlite.org/inmemorydb.html)

### Speedup via Startup
- [Speedup pytest startup (gmpy)](https://gmpy.dev/blog/2025/speedup-pytest-startup)
- [Pytest Daemon (Discord's approach)](https://pytest-with-eric.com/pytest-advanced/pytest-improve-runtime/)
