# Phase 4: Fixture Architecture — PROTOTYPE ONLY

**Status**: DRAFT — Requires Go/No-Go data before implementation
**Risk Level**: HIGH
**Effort**: 8+ hours (including validation)
**Prerequisites**: Phases 1-3 complete + 2 weeks of `--durations` data from CI
**Scope**: MC3, MC4, MC5 from the synthesis plan

---

## 1. Risk Warning

**THIS PHASE IS THE HIGHEST-RISK CHANGE IN THE ENTIRE TEST ACCELERATION INITIATIVE.**

Read this section completely before proceeding. If you are uncomfortable with any of these risks, **skip this phase entirely** — Phases 1-3 already deliver 7-9x CI wall-clock improvement.

### Why this phase is dangerous

1. **You are trading test isolation for speed.** The current suite gives every test a pristine database, a fresh Qdrant client, and freshly patched embedding mocks. Module/session-scoping removes that guarantee.

2. **The hardest bugs in testing are state leakage bugs.** They don't reproduce when you run the test alone. They only fail in specific orderings. They create false positives (tests that pass but shouldn't). With function-scoped fixtures, these bugs are impossible.

3. **SQLite `:memory:` + async + connection pools = each connection gets its own empty database.** This is a fundamental constraint of SQLite's architecture. Working around it requires `StaticPool` (serializes all async) or `cache=shared` mode (has known issues with aiosqlite).

4. **160 `commit()` calls across API tests break rollback-based isolation.** After `commit()`, `rollback()` is a no-op — the data is already persisted. Table truncation is the only alternative, adding its own overhead and complexity.

5. **The projected savings are modest after Phases 1-3.** After xdist (2-2.5x), 55-65% of remaining time is irreducible test execution. Fixture optimization targets the remaining 35-45%, yielding ~15-20% of fixture time — roughly **8-15 seconds off a 55-68s parallel run**.

### Abandon criteria (stop immediately if any occur)

- Any flakiness in 50-run validation (even 1 failure in 50 = abandon)
- "Event loop is closed" errors from pytest-asyncio
- Tests that pass alone but fail in module context (state leakage)
- Tests that fail alone but pass in module context (false positive — most dangerous)
- Timing savings < 3 seconds for the prototype module
- Any interaction with `pytest-randomly` random ordering that causes failures

### What happens if you abandon

Nothing bad. Phases 1-3 already provide:
- **Local full suite**: ~55-60s (from ~136s)
- **CI wall-clock**: ~15-18s (with 4-shard split)
- **Iterative dev**: ~5-15s (with testmon)

Phase 4 targets saving an additional 8-15 seconds locally. The risk/reward ratio is poor unless your CI budget demands it.

---

## 2. Overview

### Goal

Prototype module-scoped fixture sharing for DB engine, Qdrant client, and embedding mock on a **single test module** (`tests/api/test_categories.py`). Validate stability, measure savings, and decide whether to expand.

### Expected impact

| Change | Projected Savings | Confidence |
|--------|-------------------|------------|
| MC3: Module-scoped DB engine | ~15-20% of fixture time (~8-12s across full suite) | Low |
| MC4: Module-scoped Qdrant client | ~5-10% of fixture time (~2-3s) | Low |
| MC5: Session-scoped embedding mock | ~5% (~3-4s) | Medium |
| **Combined (optimistic)** | **~13-19s off parallel run** | **Low** |

### Effort estimate

| Item | Time |
|------|------|
| MC3 prototype + 50-run validation | 4+ hours |
| MC4 prototype + validation | 2 hours |
| MC5 investigation + validation | 1 hour |
| **Total** | **7-8+ hours** |

### Prerequisites (all must be true)

- [ ] Phase 1 complete and merged (xdist, timeout, randomly)
- [ ] Phase 2 complete and merged (lazy imports, parametrization, testmon)
- [ ] Phase 3 complete and merged (CI sharding, sysmon coverage)
- [ ] 2+ weeks of `--durations=20` data from CI runs
- [ ] Go/No-Go criteria pass (see Section 3)
- [ ] `pytest-repeat` installed for 50-run validation

---

## 3. Go/No-Go Criteria

**Do NOT start this phase until you have data proving it's worth the complexity.**

### Required data

Run this command and collect output from at least 10 CI runs over 2 weeks:

```bash
uv run pytest --durations=20 --durations-min=0.1 -p no:xdist 2>&1 | grep "setup\|teardown"
```

### Go criteria (ALL must be true)

1. **Fixture setup/teardown is >25% of total test time** after Phases 1-3 applied.
   - If fixture overhead is <25%, the savings from module-scoping are <6 seconds. Not worth the complexity.

2. **`db_engine` setup appears in the top 10 slowest setup operations.**
   - If it's not in the top 10, it's not the bottleneck.

3. **No flaky tests detected by `pytest-randomly`** after 2 weeks of randomized CI runs.
   - If there are existing flaky tests, do NOT add fixture scoping complexity on top.

4. **Team agrees to accept the maintenance cost.**
   - Every new test author must understand module-scoped DB state.
   - Debugging test failures becomes harder.

### No-Go criteria (ANY one = skip Phase 4)

- Fixture overhead is <25% of total time
- Any flaky tests detected in randomized CI runs
- Phase 1-3 already achieved acceptable suite runtime (<45s local)
- Team does not want the maintenance burden

---

## 4. MC3: Module-Scoped DB Engine Prototype

### 4.1 Prototype target: `tests/api/test_categories.py`

**Why this module:**
- 21 tests — small enough to validate quickly, large enough to be meaningful
- Uses only 2 DB models: `Category` and `TrainingSession` — minimal table footprint
- 5 `commit()` calls — representative of the commit problem but manageable
- No Qdrant vector operations — isolates DB changes from vector state issues
- Self-contained fixtures (`default_category`, `test_category`, `category_with_session`) — no cross-module dependencies
- Tests are stateless CRUD operations — well-suited for truncation-based cleanup

### 4.2 SQLite shared-cache mode configuration

The fundamental problem: SQLite `:memory:` creates a new database per connection. An async engine with a connection pool gives each connection its own empty DB.

**Solution**: Use `StaticPool` to force a single connection reused across all async operations within the module.

```python
# In tests/api/test_categories.py — MODULE-SCOPED PROTOTYPE
# DO NOT copy to other files without completing the expansion criteria in Section 9

import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.pool import StaticPool

from image_search_service.db.models import Base


@pytest_asyncio.fixture(scope="module")
async def module_db_engine() -> AsyncGenerator[AsyncEngine, None]:
    """Module-scoped DB engine using StaticPool for connection reuse.

    WARNING: This is a PROTOTYPE fixture. StaticPool serializes all async
    operations through a single connection. This defeats true async concurrency
    but is the only reliable way to share SQLite :memory: across an async
    engine within a module.

    IMPORTANT: Uses check_same_thread=False because SQLAlchemy's async engine
    may access the connection from different threads via the asyncio executor.
    """
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        echo=False,
        poolclass=StaticPool,
        connect_args={"check_same_thread": False},
    )

    # Create all tables once for the module
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    # Teardown: drop all tables and dispose
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()
```

### 4.3 Table truncation cleanup fixture

Since `commit()` calls make rollback-based isolation impossible, we must truncate all tables between tests.

```python
@pytest_asyncio.fixture(autouse=True)
async def _truncate_tables(module_db_engine: AsyncEngine) -> AsyncGenerator[None, None]:
    """Truncate all tables between tests for isolation.

    WARNING: This runs DELETE FROM on every table in reverse dependency order
    after each test. This is necessary because 5 commit() calls in this module's
    fixtures write data that rollback() cannot undo.

    Costs ~1-2ms per table. With 18 tables, that's ~18-36ms per test.
    """
    yield

    # After each test: delete all data from all tables in reverse FK order
    async with module_db_engine.begin() as conn:
        # Disable FK checks for SQLite during cleanup
        await conn.execute(text("PRAGMA foreign_keys = OFF"))
        for table in reversed(Base.metadata.sorted_tables):
            await conn.execute(table.delete())
        await conn.execute(text("PRAGMA foreign_keys = ON"))
```

**Important**: Import `text` from SQLAlchemy:
```python
from sqlalchemy import text
```

### 4.4 Module-scoped session fixture

```python
@pytest_asyncio.fixture
async def db_session(module_db_engine: AsyncEngine) -> AsyncGenerator[AsyncSession, None]:
    """Per-test session from the module-scoped engine.

    Each test gets its own session, but they all share the same
    underlying database (via StaticPool). The _truncate_tables autouse
    fixture handles cleanup between tests.
    """
    session_factory = async_sessionmaker(module_db_engine, expire_on_commit=False)

    async with session_factory() as session:
        yield session
        # Rollback any uncommitted changes (committed data is handled by truncation)
        await session.rollback()
```

### 4.5 pytest-asyncio loop_scope configuration

Module-scoped async fixtures require the event loop to persist across the module. Add this to the **top of `test_categories.py`** (or to a module-level `conftest.py`):

```python
# Required for module-scoped async fixtures
# pytest-asyncio 0.23+ supports loop_scope
pytestmark = pytest.mark.asyncio(loop_scope="module")
```

**Version check**: The project requires `pytest-asyncio>=1.3.0` which supports `loop_scope`. Verify with:

```bash
uv run python -c "import pytest_asyncio; print(pytest_asyncio.__version__)"
```

If `loop_scope` is not supported, you must upgrade pytest-asyncio or **abandon MC3**.

### 4.6 ID-dependent assertion check

SQLite does not reset `ROWID` on `DELETE`. After truncation, auto-increment IDs continue from previous max+1. **Check if any test in `test_categories.py` asserts on specific ID values.**

Current scan: No test in `test_categories.py` asserts `data["id"] == 1` or any hardcoded ID. Tests use `test_category.id` (dynamic reference). **This is safe.**

If you expand to other modules, re-check this for every module. Tests like `assert data["id"] == 1` will break with module-scoped engines.

### 4.7 Putting it all together

The modified `test_categories.py` header should look like:

```python
"""Test category management endpoints — MODULE-SCOPED DB PROTOTYPE."""

from collections.abc import AsyncGenerator

import pytest
import pytest_asyncio
from httpx import AsyncClient
from sqlalchemy import text
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.pool import StaticPool

from image_search_service.db.models import Base, Category, TrainingSession

# Module-scoped event loop for async fixtures
pytestmark = pytest.mark.asyncio(loop_scope="module")


# --- Module-scoped engine (PROTOTYPE) ---

@pytest_asyncio.fixture(scope="module")
async def module_db_engine() -> AsyncGenerator[AsyncEngine, None]:
    # ... (code from 4.2 above)


@pytest_asyncio.fixture(autouse=True)
async def _truncate_tables(module_db_engine: AsyncEngine) -> AsyncGenerator[None, None]:
    # ... (code from 4.3 above)


@pytest_asyncio.fixture
async def db_session(module_db_engine: AsyncEngine) -> AsyncGenerator[AsyncSession, None]:
    # ... (code from 4.4 above)


# --- Existing fixtures (unchanged) ---

@pytest.fixture
async def default_category(db_session: AsyncSession) -> Category:
    # ... (existing code, unchanged)

# ... rest of existing test file
```

**NOTE**: The `test_client` fixture from `conftest.py` depends on `db_session`. By defining a local `db_session` fixture in this module, pytest's fixture resolution will prefer the local one. The module-scoped engine thus feeds into the existing `test_client` fixture **without modifying `conftest.py`**.

However — **critical caveat**: The `test_client` fixture also depends on `sync_db_session` and `qdrant_client`, which remain function-scoped from conftest. This means test_client will still create a fresh sync engine and Qdrant per test. For the MC3 prototype, this is acceptable — we're only testing the async DB engine scoping.

> **⚠️ REVIEW NOTE (plan-reviewer):** This caveat is critical for setting expectations. Since `test_client` (the most expensive fixture) also creates a **new FastAPI app**, a new sync DB engine, and a new Qdrant client **per test**, the actual savings from MC3 are limited to only the async `db_engine` creation (one `create_all` + `drop_all` per module instead of per test). For `test_categories.py` (21 tests), the savings may be as low as ~200-400ms total — not the ~15-20% projected for fixture time. **Measure carefully before expanding.**

### 4.8 Flakiness detection (50-run validation)

Install pytest-repeat if not already present:

```bash
uv add --group dev "pytest-repeat>=0.9.3"
```

Run the 50-iteration validation:

```bash
# Step 1: Run 50 times, fail on first error
uv run pytest tests/api/test_categories.py --count=50 -x -v 2>&1 | tee /tmp/mc3-validation.log

# Step 2: Check results
grep -c "PASSED\|FAILED\|ERROR" /tmp/mc3-validation.log
# Expected: 1050 PASSED, 0 FAILED, 0 ERROR (21 tests x 50 runs)
```

**Additionally**, run with random ordering to detect hidden order dependencies:

```bash
# Step 3: 50 runs with random test ordering within the module
uv run pytest tests/api/test_categories.py --count=50 -x -v -p randomly 2>&1 | tee /tmp/mc3-random-validation.log
```

### 4.9 Timing comparison

```bash
# Baseline (function-scoped, before changes):
uv run pytest tests/api/test_categories.py --durations=0 -v -p no:xdist 2>&1 | tee /tmp/mc3-baseline.log

# Module-scoped prototype:
# (after applying changes)
uv run pytest tests/api/test_categories.py --durations=0 -v -p no:xdist 2>&1 | tee /tmp/mc3-modulescoped.log

# Compare total time
tail -5 /tmp/mc3-baseline.log
tail -5 /tmp/mc3-modulescoped.log
```

**Expected**: Module-scoped should save ~15-20ms per test (avoiding `create_all`/`drop_all`), offset by ~18-36ms per test for truncation. Net savings for 21 tests: **~0-0.5s**. If savings are negative or negligible, **abandon MC3**.

### 4.10 Abandon criteria for MC3

**Stop and revert if ANY of these occur:**

| Condition | Action |
|-----------|--------|
| Any failure in 50-run validation | Revert immediately |
| Any failure in 50-run random-order validation | Revert immediately |
| Net timing savings < 100ms for the 21-test module | Abandon — not worth extrapolating |
| "Event loop is closed" errors | `loop_scope="module"` not working — abandon |
| `StaticPool` causes deadlocks or hangs | Abandon — SQLite serialization too aggressive |
| Test that passes alone but fails in module | State leakage — abandon |
| Test that fails alone but passes in module | False positive (MOST DANGEROUS) — abandon |

---

## 5. MC4: Module-Scoped Qdrant Client with Per-Test Cleanup

### 5.1 Overview

Share one in-memory Qdrant client across all tests in a module. Delete all points between tests instead of recreating collections.

### 5.2 Affected tests

Tests that insert vectors into Qdrant and assert on search results are the highest risk:

- `tests/api/test_face_suggestions.py` — asserts on specific similarity scores and result counts
- `tests/api/test_face_suggestion_cleanup.py` — tests cleanup of stale suggestions
- `tests/api/test_face_session_suggestions.py` — session-scoped suggestion tests
- `tests/api/test_vectors.py` — vector search result assertions
- `tests/api/test_search.py` — text search via vector similarity

**DO NOT prototype MC4 on these modules.** Start with `test_categories.py` which does NOT use Qdrant vectors directly (it only uses the Qdrant client via `test_client` dependency injection, and category CRUD tests don't insert vectors).

### 5.3 Module-scoped Qdrant fixture

```python
@pytest.fixture(scope="module")
def module_qdrant_client() -> QdrantClient:
    """Module-scoped Qdrant client. Collections created once, points deleted per test."""
    from image_search_service.core.config import get_settings

    client = QdrantClient(":memory:")
    settings = get_settings()

    # Create collections once for the module
    client.create_collection(
        collection_name=settings.qdrant_collection,
        vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    )
    client.create_collection(
        collection_name=settings.qdrant_face_collection,
        vectors_config=VectorParams(size=512, distance=Distance.COSINE),
    )

    # Create payload indexes (same as production bootstrap)
    for field, schema in [
        ("person_id", PayloadSchemaType.KEYWORD),
        ("cluster_id", PayloadSchemaType.KEYWORD),
        ("is_prototype", PayloadSchemaType.BOOL),
        ("asset_id", PayloadSchemaType.KEYWORD),
        ("face_instance_id", PayloadSchemaType.KEYWORD),
    ]:
        client.create_payload_index(
            collection_name=settings.qdrant_face_collection,
            field_name=field,
            field_schema=schema,
        )

    return client


@pytest.fixture(autouse=True)
def _clear_qdrant_points(module_qdrant_client: QdrantClient) -> Generator[None, None, None]:
    """Delete all points from both collections between tests.

    WARNING: This deletes by scroll+delete, which may be slow with many points.
    For modules with tests that insert thousands of vectors, this fixture could
    negate the savings from avoiding collection recreation.
    """
    yield

    from image_search_service.core.config import get_settings
    settings = get_settings()

    from qdrant_client.models import PointIdsList

    for collection_name in [settings.qdrant_collection, settings.qdrant_face_collection]:
        # Get all point IDs
        points, _ = module_qdrant_client.scroll(
            collection_name=collection_name,
            limit=10000,
            with_payload=False,
            with_vectors=False,
        )
        if points:
            # NOTE: Must use PointIdsList wrapper — raw list won't work
            module_qdrant_client.delete(
                collection_name=collection_name,
                points_selector=PointIdsList(points=[p.id for p in points]),
            )
```

### 5.4 Risk: Mock state leakage

Several face suggestion tests insert vectors into Qdrant and assert on:
- **Result count**: "search returns exactly 3 similar faces"
- **Similarity scores**: "nearest neighbor has score > 0.85"
- **Empty results**: "no faces found for new person"

If Test A inserts face vectors and the cleanup fixture fails to delete them, Test B sees stale vectors. This can cause:
- Tests that pass alone but fail in module (extra results from leaked state)
- Tests that fail alone but pass in module (false positive — relies on leaked data)

**Mitigation**: The `_clear_qdrant_points` fixture scrolls and deletes. Verify cleanup is complete:

```python
# Add an assertion in _clear_qdrant_points (debug mode):
for collection_name in [...]:
    info = module_qdrant_client.get_collection(collection_name)
    assert info.points_count == 0, f"Cleanup failed: {collection_name} has {info.points_count} points"
```

### 5.5 Validation

Same 50-run protocol as MC3:

```bash
uv run pytest tests/api/test_categories.py --count=50 -x -v -p randomly
```

Then if stable, try a module that actually uses Qdrant vectors (e.g., `test_face_suggestions.py` with 14 tests) as the second prototype target.

---

## 6. MC5: Session-Scoped Embedding Mock

### 6.1 Pre-check: Does any test modify EmbeddingService?

Before making the embedding mock session-scoped, you MUST verify no test patches `EmbeddingService` methods at the class level:

```bash
cd image-search-service

# Check for tests that monkeypatch/patch EmbeddingService methods
grep -rn "monkeypatch.*EmbeddingService\|patch.*EmbeddingService\|setattr.*EmbeddingService" tests/ --include="*.py" | grep -v conftest.py
```

**If any results appear**: Do NOT proceed with MC5. The function-scoped conftest fixture reverts monkeypatch after each test. If a test patches `EmbeddingService.embed_text` with a custom function while the session-scoped mock is active, you get a 3-layer stack:

```
Real method → session-scoped mock → per-test monkeypatch override
```

When monkeypatch reverts, it restores the session-scoped mock (not the real method). This probably works but is fragile, hard to reason about, and a debugging nightmare if it breaks.

**If no results**: You can proceed with MC5.

### 6.2 Session-scoped embedding mock implementation

Replace the current `clear_embedding_cache` autouse fixture with:

```python
@pytest.fixture(autouse=True, scope="session")
def _session_embedding_mock():
    """Session-scoped embedding mock to avoid re-patching per test.

    Uses unittest.mock.patch (not monkeypatch) because monkeypatch
    only supports function scope.

    WARNING: If any test modifies EmbeddingService at the class level,
    the modification will persist for ALL subsequent tests in the session.
    Run the pre-check grep command before enabling this fixture.
    """
    from unittest.mock import patch

    from image_search_service.services.embedding import EmbeddingService, get_embedding_service

    get_embedding_service.cache_clear()
    semantic_mock = SemanticMockEmbeddingService()

    with (
        patch.object(EmbeddingService, "embed_text", lambda self, text: semantic_mock.embed_text(text)),
        patch.object(EmbeddingService, "embed_image", lambda self, path: semantic_mock.embed_image(path)),
        patch.object(
            EmbeddingService,
            "embed_images_batch",
            lambda self, images: semantic_mock.embed_images_batch(images),
        ),
        patch.object(EmbeddingService, "embedding_dim", property(lambda self: semantic_mock.embedding_dim)),
    ):
        yield

    get_embedding_service.cache_clear()
```

### 6.3 Cache dict cleanup between tests

`SemanticMockEmbeddingService` has a `_cache` dict that grows across calls. Add a function-scoped autouse fixture to clear it:

```python
@pytest.fixture(autouse=True)
def _clear_embedding_mock_cache(_session_embedding_mock):
    """Clear the embedding mock's internal cache between tests.

    Prevents unbounded memory growth across 1,096 tests and ensures
    deterministic behavior (no test depends on another test's cached embeddings).
    """
    yield
    # The session mock's _cache is not directly accessible here.
    # Alternative: clear get_embedding_service cache to force re-creation.
    from image_search_service.services.embedding import get_embedding_service
    get_embedding_service.cache_clear()
```

**Note**: This approach clears the `lru_cache` but NOT the `_cache` dict inside the `SemanticMockEmbeddingService` instance. To clear the instance cache, you'd need to either:
1. Store the `semantic_mock` instance in a module-level variable accessible to the cleanup fixture, OR
2. Accept that the cache grows (it's deterministic and bounded by unique input strings)

Option 2 is probably fine for this project. The cache stores `{text: embedding_vector}` pairs. With ~1,096 tests each embedding at most a few strings, the cache will hold a few thousand 768-dim lists. Memory impact: ~20-50MB max. Acceptable.

### 6.4 Why the 3-layer patch stack is fragile

If any test does this:

```python
def test_custom_embedding(monkeypatch):
    monkeypatch.setattr(EmbeddingService, "embed_text", lambda self, t: [0.0] * 768)
    # ...test code...
```

The patch stack becomes:

```
Layer 1: Real EmbeddingService.embed_text (original)
Layer 2: session-scoped unittest.mock.patch (our mock)
Layer 3: function-scoped monkeypatch (test's override)
```

When the test ends:
- monkeypatch reverts Layer 3 → restoring Layer 2 (the session mock) — correct
- The session mock continues for subsequent tests — correct

BUT: if `unittest.mock.patch` internally stores the "original" as a reference and monkeypatch's `setattr` bypasses mock's internal tracking, the restoration chain can break.

**Bottom line**: It probably works with current pytest/mock versions. But it's a maintenance hazard. Document it clearly and add a test:

```python
def test_embedding_mock_integrity():
    """Verify the session-scoped embedding mock is still active."""
    from image_search_service.services.embedding import EmbeddingService

    svc = EmbeddingService.__new__(EmbeddingService)
    result = svc.embed_text("test")
    assert len(result) == 768
    assert isinstance(result, list)
```

### 6.5 Validation

```bash
# Run full suite to verify no test breaks
uv run pytest -p no:xdist -x -v 2>&1 | tee /tmp/mc5-validation.log

# Run 50 times on a representative module
uv run pytest tests/api/test_search.py --count=50 -x -v -p randomly
```

---

## 7. Validation Steps

### 7.1 Per-change validation protocol

For EACH change (MC3, MC4, MC5), run this validation sequence **before proceeding to the next change**:

```bash
# Step 1: 50-run stability check (single module)
uv run pytest <target_module> --count=50 -x -v
# MUST: 0 failures in 50 runs

# Step 2: 50-run random-order check
uv run pytest <target_module> --count=50 -x -v -p randomly
# MUST: 0 failures in 50 runs

# Step 3: Full suite check
uv run pytest -x -v
# MUST: Same pass count as before the change

# Step 4: Full suite with random ordering
uv run pytest -x -v -p randomly
# MUST: Same pass count as before

# Step 5: Timing comparison
uv run pytest <target_module> --durations=0 -p no:xdist  # Measure prototype
uv run pytest --durations=20 -p no:xdist                 # Measure full suite
# RECORD: timing data for comparison
```

### 7.2 Aggregate validation (after all MC3-MC5)

```bash
# Full suite, 5 runs, to detect intermittent failures
for i in $(seq 1 5); do
    echo "=== Run $i ==="
    uv run pytest -x -p randomly 2>&1 | tail -1
done

# Serial mode (no xdist) to verify in single-process
uv run pytest -p no:xdist -p randomly -x
```

---

## 8. Rollback Plan

### Rollback MC3 (Module-scoped DB engine)

1. Remove the `module_db_engine`, `_truncate_tables`, and local `db_session` fixtures from `test_categories.py`
2. Remove `pytestmark = pytest.mark.asyncio(loop_scope="module")` line
3. Remove the `from sqlalchemy import text` and `StaticPool` imports
4. The file reverts to using the function-scoped fixtures from `conftest.py`
5. Run `uv run pytest tests/api/test_categories.py -x -v` to verify

**Rollback time**: 5 minutes.

### Rollback MC4 (Module-scoped Qdrant)

1. Remove `module_qdrant_client` and `_clear_qdrant_points` fixtures from the prototype module
2. Tests revert to using the function-scoped `qdrant_client` from `conftest.py`
3. Run `uv run pytest <module> -x -v` to verify

**Rollback time**: 5 minutes.

### Rollback MC5 (Session-scoped embedding mock)

1. Restore the original `clear_embedding_cache` autouse fixture in `conftest.py` (revert to function-scoped monkeypatch approach)
2. Remove `_session_embedding_mock` and `_clear_embedding_mock_cache` fixtures
3. Run `uv run pytest -x -v` to verify full suite passes

**Rollback time**: 10 minutes.

### Emergency rollback (all Phase 4 changes)

```bash
# If you committed Phase 4 changes in a single commit:
git revert <commit-hash>

# If spread across multiple commits:
git revert <mc3-hash> <mc4-hash> <mc5-hash>

# Verify:
uv run pytest -x -v
```

---

## 9. Expansion Criteria

**Only consider expanding module-scoped fixtures beyond the prototype if ALL of these are true:**

### Hard requirements

1. **50-run validation passes with 0 failures** on the prototype module (MC3/MC4)
2. **50-run random-order validation passes with 0 failures** on the prototype module
3. **Full suite passes with random ordering** after prototype changes
4. **Net timing savings >300ms per module** for the prototype module (measured, not estimated)
5. **No false positives detected** — every test that passes in module context also passes in isolation:
   ```bash
   # For each test in the prototype module:
   uv run pytest tests/api/test_categories.py::test_<name> -x -v  # Must pass alone
   ```

### Soft requirements

6. **Team consensus** that the maintenance cost is acceptable
7. **No increase in CI flakiness** over 2 weeks of running with the prototype
8. **Documentation is complete** — every expanded module has a comment explaining the fixture scoping

### Expansion order (if approved)

Start with the simplest modules and work outward:

| Priority | Module | Tests | Reason |
|----------|--------|-------|--------|
| 1 | `test_categories.py` | 21 | Already prototyped |
| 2 | `test_health.py` | 2 | Trivial, no DB writes |
| 3 | `test_images.py` | 7 | Simple CRUD, few commits |
| 4 | `test_ingest.py` | 5 | Simple ingestion tests |
| 5+ | (evaluate based on data) | — | Only if 1-4 are stable for 2 weeks |

**Never expand to:**
- `test_face_suggestions.py` — heavy Qdrant vector assertions
- `test_search.py` — 27 tests with complex vector search assertions
- `test_faces_routes.py` — 49 tests, heaviest API test file

---

## 10. What NOT to Do

These are explicitly forbidden based on the synthesis and devil's advocate analyses:

| Action | Why It's Forbidden |
|--------|-------------------|
| **Apply module-scoped DB to all modules at once** | 160 `commit()` calls across 26 API test files break rollback isolation. Untested territory. Expand one module at a time after 50-run validation. |
| **Module-scope `test_client`** | `create_app()` + `dependency_overrides` assumes per-test isolation. Sharing an app means sharing override state. Used 815 times across 34 files — the blast radius is catastrophic. |
| **Session-scope `use_test_settings` (fixture #2)** | Prevents production Qdrant collection names. Cost: ~0.1ms/test. Blast radius of a bug: **total production data loss**. Keep function-scoped forever. |
| **Remove autouse fixtures #1 and #2** | They are safety guards, not just overhead. `clear_settings_cache` prevents cached production collection names from leaking. `use_test_settings` sets test-safe collection names. Both marked CRITICAL. |
| **Use `sqlite:///file:testdb?mode=memory&cache=shared&uri=true`** | The `cache=shared` mode has known issues with aiosqlite concurrent access. Use `StaticPool` instead — it's more reliable even though it serializes. |
| **Use rollback-based isolation with module-scoped DB** | 160 `commit()` calls make rollback a no-op. You MUST use table truncation. |
| **Use `TRUNCATE TABLE`** | SQLite does not support `TRUNCATE`. Use `DELETE FROM` instead. |
| **Skip the 50-run validation** | "It works on my machine" is not sufficient. State leakage bugs are probabilistic and may only appear in specific orderings. 50 runs with `pytest-randomly` is the minimum. |
| **Use `pytest-parallel`** | Abandoned project (last release 2020). Thread-based mode doesn't work with SQLite. Use `pytest-xdist` from Phase 1. |
| **Session-scope `clear_embedding_cache` broadly without pre-check** | The `_cache` dict grows unboundedly. A test that patches `embed_text` would affect all subsequent tests. The ~5% savings is not worth the risk without the grep pre-check passing. |
| **Custom test ordering hooks** | More complexity for marginal benefit. `pytest --ff` and `pytest-randomly` from Phase 1 are sufficient. |
| **Assert on auto-generated IDs (after truncation)** | SQLite `ROWID` does not reset on `DELETE`. Tests asserting `id == 1` will break after the first test in a module creates and deletes a row. |

---

## Appendix A: Quick Reference Commands

```bash
# Install pytest-repeat for 50-run validation
uv add --group dev "pytest-repeat>=0.9.3"

# MC3 50-run validation
uv run pytest tests/api/test_categories.py --count=50 -x -v
uv run pytest tests/api/test_categories.py --count=50 -x -v -p randomly

# MC5 pre-check: find tests that modify EmbeddingService
grep -rn "monkeypatch.*EmbeddingService\|patch.*EmbeddingService\|setattr.*EmbeddingService" \
  tests/ --include="*.py" | grep -v conftest.py

# Timing comparison (serial mode)
uv run pytest tests/api/test_categories.py --durations=0 -p no:xdist

# Full suite validation with random ordering
uv run pytest -x -v -p randomly

# Emergency rollback
git revert <phase4-commit-hash>
```

## Appendix B: Decision Log

| Decision | Chosen | Alternatives | Rationale |
|----------|--------|-------------|-----------|
| SQLite async strategy | `StaticPool` | `cache=shared` URI | StaticPool is more reliable; cache=shared has known aiosqlite issues |
| Isolation strategy | Table truncation (DELETE FROM) | Rollback, SAVEPOINT | 160 commit() calls make rollback a no-op; SQLite has no TRUNCATE |
| Prototype target | `test_categories.py` (21 tests) | `test_health.py` (2 tests) | 21 tests is meaningful sample; health tests too small to measure |
| Embedding mock scope | Session (with pre-check) | Module | Session saves more if pre-check passes; module is safer but saves less |
| Qdrant cleanup method | Scroll + delete all points | Recreate collections | Avoids collection setup; scroll+delete is fast for small point counts |
| Expansion approach | One module at a time, 2-week soak | Batch rollout | Safety first — state leakage bugs are probabilistic |
