# Devil's Advocate: Test Acceleration Risks, Tradeoffs, and Anti-Patterns

**Date**: 2026-02-15
**Role**: Devil's Advocate Analyst
**Team**: test-acceleration-research
**Inputs**: Architecture analysis, techniques research, dependency analysis
**Classification**: Actionable (risk mitigations should be implemented alongside optimizations)

---

## Executive Summary

The three research reports propose a compelling optimization path: from ~136s to ~8-10s via parallelization, fixture scoping, CI sharding, and development workflow tools. The combined projections promise a **4-8x speedup** at seemingly low cost.

I believe the proposals are generally sound but the reports consistently **understate risk and overstate speedup projections**. Specifically:

1. **Speedup projections are multiplicatively compounded** (Phase 1 "3x" × Phase 2 "1.5x" × Phase 4 "3x" = 13.5x) but real-world gains don't multiply cleanly due to fixed overhead and Amdahl's Law
2. **The 4 autouse fixtures marked CRITICAL are load-bearing safety guards**, not just overhead — removing them requires proving no test ever relies on the protection they provide
3. **Module-scoped DB engines with SQLite `:memory:` are the highest-risk proposal** due to SQLite's threading model and async connection pool constraints
4. **pytest-xdist is the only low-risk, high-impact change** — everything else requires careful validation

**Bottom line**: Implement xdist first. Measure. Then decide if the complexity of fixture restructuring is worth the marginal gain.

---

## 1. Parallelization Risks (pytest-xdist)

### What the reports say
- "Safe for parallelization" for all API and unit tests
- "Automatic isolation" via per-process in-memory DBs
- Expected 3x speedup on 4 cores

### What could go wrong

**Risk 1.1: Event loop isolation with pytest-asyncio**

Each xdist worker spawns its own process, which means its own event loop. But `asyncio_mode = "auto"` with pytest-asyncio 1.3.0 creates **one event loop per session** by default. If xdist's worker isolation doesn't cleanly separate event loops, you get cryptic "attached to a different event loop" errors.

- **Severity**: Medium — likely works, but failure mode is painful to debug
- **Mitigation**: Pin `asyncio_mode = "auto"` and test with `pytest -n 2` before going to `auto`. If issues arise, add `loop_scope = "function"` (pytest-asyncio 0.23+) to ensure per-test loops.

**Risk 1.2: `test_device.py` patches `os.environ` globally**

The dependency analysis notes that `test_device.py` uses `@patch.dict(os.environ, ...)` for 56 patches. In a single process this is safe (patches are thread-local). But `os.environ` mutations via `@patch.dict` in forked workers inherit the parent's env — if the fork happens mid-test, a half-applied patch can leak.

- **Severity**: Low — xdist workers are clean forks from the master process, not from each other
- **Mitigation**: Verify `test_device.py` passes with `-n 2`. If flaky, add `@pytest.mark.forked` to force process isolation per test.

**Risk 1.3: Import-time side effects in worker startup**

Each xdist worker re-imports conftest.py and all test modules. The architecture report identifies `core/device.py → import torch` as a 1.8s import. With 4 workers, that's 4 × 1.8s = 7.2s of combined startup overhead — **eating into the parallelism gains**.

- **Severity**: Medium — doesn't break anything, but erodes the "3x" speedup claim
- **Mitigation**: Fix the lazy import first (`torch` behind `if TYPE_CHECKING` or lazy import in function). This is an independent improvement that pays off regardless of parallelization.

**Risk 1.4: Worker startup overhead is underestimated**

The techniques report estimates "2.5-3.5x on 4 cores." But each worker must:
1. Fork the process (~50ms)
2. Re-import conftest.py and all test modules (~1.4s collection time × 4 workers)
3. Initialize numpy, PIL, qdrant_client in each worker
4. xdist IPC overhead for work stealing (~5-10ms per test)

Realistic estimate: **2-2.5x on 4 cores**, not 3x. Still worthwhile, but set expectations correctly.

- **Mitigation**: Measure with `pytest -n 4 --dist worksteal -v` and compare wall-clock time. Don't assume the theoretical speedup.

### Verdict: Low risk, do it. But expect 2-2.5x, not 3x.

---

## 2. The "Fast but Wrong" Trap

### What the reports say
- Fixture optimizations (module/session scoping) save 30-50%
- Mock optimizations save 5-15%
- Combined: significant speedup

### The fundamental problem

Every optimization that reduces per-test setup is trading **isolation for speed**. The current test suite runs every test with a pristine database, a fresh Qdrant client, and freshly patched embedding mocks. This is expensive but provides **an iron guarantee**: no test can affect another test.

The moment you share state across tests, you introduce the possibility that:
1. Test A leaves data in the DB that makes Test B pass (or fail) spuriously
2. Test B relies on the absence of data that Test A happened to create
3. A test failure in one test causes a cascade of failures in subsequent tests that share the fixture

These are **the hardest bugs in testing to debug** because:
- They don't reproduce when you run the test alone (`pytest tests/api/test_foo.py::test_bar` passes)
- They only fail in specific orderings (which xdist randomizes)
- They create false positives (tests that pass only because of leaked state)

### The insidious false positive

The reports focus on "tests that break" under new configurations. But the more dangerous outcome is **tests that pass but shouldn't**. If Test A inserts a person record and Test B queries for persons, Test B might pass because it finds Test A's data — even though the feature under test is broken and would return nothing in production.

With function-scoped fixtures, this bug is impossible. With module-scoped fixtures, it depends on test ordering.

- **Mitigation**: If adopting module-scoped DB engines, add `pytest-randomly` to randomize test order AND run the full suite periodically in serial mode to verify isolation. Consider adding a `--check-isolation` flag that runs each test twice: once normally, once in a fresh process.

---

## 3. Module-Scoped DB Engine: The Riskiest Proposal

### What the reports propose
- Create one SQLite engine per module, share it across all tests in that module
- Use rollback/truncation between tests for isolation
- Estimated 30-40% speedup

### Risk 3.1: SQLite + aiosqlite + shared engine = connection pool problems

SQLite in `:memory:` mode creates a database that exists only for the lifetime of a single connection. When you create an async engine with `create_async_engine("sqlite+aiosqlite:///:memory:")`, SQLAlchemy's connection pool can open multiple connections — **but each connection gets its own empty database**.

For module-scoped engines, you'd need `StaticPool` (single connection reused) or the `file:testdb?mode=memory&cache=shared&uri=true` URI. But:

- `StaticPool` means all async operations serialize through one connection — defeating the purpose of async
- `cache=shared` has known issues with aiosqlite and concurrent access
- **There are zero SAVEPOINT usages in this codebase** — the rollback strategy is untested territory

- **Severity**: High — this is the kind of change that seems to work in basic testing but fails under load (when multiple tests in a module do concurrent DB operations)
- **Mitigation**: If attempting this, prototype with **one module** (e.g., `test_categories.py` with 21 tests) and run it 50 times to check for flakiness. Use `StaticPool` and accept the serialization cost. Do NOT do this for the whole suite at once.

### Risk 3.2: `create_all` / `drop_all` actually validates the model schema

The current per-test `create_all` runs the full SQLAlchemy DDL. This is expensive (~5-10ms) but it also **validates that your models can actually create a working database**. If a migration breaks the schema, every test catches it immediately.

With module-scoped engines, a schema bug might only manifest in whichever module happens to run first.

- **Severity**: Low (schema bugs are rare and caught by migration tests)
- **Mitigation**: Keep the postgres integration tests running `create_all` per-test as the schema validation gate.

### Risk 3.3: Table truncation vs rollback

The techniques report proposes "truncate all tables between tests" as an alternative to per-test `create_all`:

```python
for table in reversed(Base.metadata.sorted_tables):
    await conn.execute(table.delete())
```

This is faster than `create_all`/`drop_all`, but:
- **Foreign key constraints**: `DELETE FROM` in dependency order can fail if the ordering is wrong. SQLAlchemy's `sorted_tables` usually handles this, but `ON DELETE CASCADE` behavior differs between SQLite and PostgreSQL.
- **Auto-increment counters**: SQLite doesn't reset `ROWID` on `DELETE`. If any test asserts on auto-generated IDs (e.g., "first person gets id=1"), it will fail in the second test when the counter starts at the previous max+1.
- **160 `await db_session.commit()` calls across API tests**: Each commit writes to the shared engine. If truncation misses a table or has ordering issues, stale data persists.

- **Mitigation**: Use `DELETE` (not `TRUNCATE` — SQLite doesn't support `TRUNCATE`). Add assertions in the truncation fixture to verify all tables are empty after cleanup. Check for ID-dependent assertions in tests.

---

## 4. Session-Scoped Fixture Dangers

### Risk 4.1: The embedding mock assumes statelessness — but does it?

The `clear_embedding_cache` autouse fixture creates a new `SemanticMockEmbeddingService` per test and monkeypatches `EmbeddingService` methods. The reports propose making this session-scoped.

But `SemanticMockEmbeddingService` has a `self._cache` dict that **accumulates entries across calls**:

```python
# From the class: "Uses cached embeddings for deterministic results"
self._cache: dict[str, list[float]] = {}
```

If session-scoped, this cache grows throughout the entire test session. With 1,096 tests, potentially thousands of cached embeddings accumulate in memory. This probably doesn't break anything (the cache is deterministic), but:

- If any test modifies the mock instance's behavior (e.g., patches `embed_text` to return a specific value), all subsequent tests in the session see the modified behavior
- Memory usage grows unboundedly

- **Severity**: Low (cache is deterministic, memory growth is small)
- **Mitigation**: If making session-scoped, add `_cache.clear()` to a function-scoped autouse fixture, or set a max cache size.

### Risk 4.2: Qdrant client state accumulation

The reports propose module-scoped Qdrant clients. But Qdrant's in-memory client accumulates all inserted vectors. After 10 API tests that each insert face vectors, the 11th test will see vectors from all 10 previous tests.

This fundamentally breaks test isolation for any test that:
- Searches Qdrant and asserts on result count
- Tests "no results" scenarios
- Validates distance/similarity scores (more vectors = different nearest neighbors)

- **Severity**: High — many face suggestion tests assert on specific search results
- **Mitigation**: Add per-test Qdrant cleanup (delete all points from collections between tests). But this partially negates the speedup from avoiding per-test collection creation.

### Risk 4.3: Hidden coupling through module execution order

With module-scoped fixtures, tests within a module run in declaration order. If Test A creates data and Test B (later in the file) queries it, the test suite appears to work. But if someone reorders the tests, adds a new test between them, or the randomization changes, Test B breaks.

- **Severity**: Medium — easy to introduce, hard to diagnose
- **Mitigation**: Use `pytest-randomly` to shake out ordering dependencies early. Run in random order as part of CI.

---

## 5. The 4 Autouse Fixtures: Safety Guards, Not Just Overhead

### The reports frame them as "overhead"

The architecture report says: "4 autouse fixtures run for EVERY test regardless of whether they're needed" and estimates "~17.5 seconds of pure autouse fixture overhead."

### What the reports understate

Read the docstrings carefully. Every single one is marked **CRITICAL**:

1. **`clear_settings_cache`**: "CRITICAL: This prevents tests from using cached production collection names, which could cause **deletion of live Qdrant data**."

2. **`use_test_settings`**: "CRITICAL: This ensures tests never use production collection names like `image_assets` or `faces`, preventing **accidental data deletion**."

3. **`clear_embedding_cache`**: "CRITICAL: This prevents tests from **loading the real OpenCLIP/SigLIP models**, which are expensive and not needed for unit tests."

4. **`validate_embedding_dimensions`**: Validates that mock dimensions match production (768-dim). A mismatch would mean tests pass with wrong dimensions and fail silently in production.

These aren't just performance fixtures. **Fixtures #1 and #2 are data safety guards.** If any test somehow loads real settings (perhaps through a code path that doesn't use the cached version), it could connect to production Qdrant and delete real data.

### The cost of "optimizing" them

The reports suggest:
- Merging #1 and #2 → Reasonable, same behavior in fewer calls
- Making #3 session-scoped → Removes per-test re-patching, meaning a test that modifies `EmbeddingService` at the class level would affect all subsequent tests
- Making #4 session-scoped → Reasonable, it only validates constants

The merge of #1 and #2 is safe. Making #4 session-scoped is safe. But making #3 session-scoped introduces a subtle risk: if any test does `monkeypatch.setattr(EmbeddingService, "embed_text", custom_fn)`, the function-scoped monkeypatch reverts after that test — but the session-scoped mock stays modified.

Wait — monkeypatch reverts the **original** method, not the session-scoped mock's method. So if the session-scoped `unittest.mock.patch` is active, monkeypatch would revert to the session-scoped mock, not the real method. This creates a three-layer stack: real method → session mock → per-test override. It probably works but it's fragile and hard to reason about.

- **Mitigation**: Merge #1 and #2 (no risk). Make #4 session-scoped (no risk). Leave #3 function-scoped unless you can prove no test modifies embedding behavior at the class level. The ~8ms-per-test cost of #3 is **not** the bottleneck (it's ~8.8s total across 1,096 tests — significant but not the main target). Focus on the test_client and db_engine fixture cost instead.

### Revised autouse savings estimate

| Change | Reports' Estimate | My Estimate | Risk |
|--------|------------------|-------------|------|
| Merge #1 + #2 | ~3% (~4s) | ~2% (~2.7s) | None |
| Session-scope #4 | ~1% | ~0.1% (negligible) | None |
| Session-scope #3 | ~5-10% | ~5% (~6.8s) | Medium |
| **Combined safe changes** | **~10-15%** | **~2-3%** | **None** |

The safe changes save ~2-3%, not 10-15%. The risky change (#3) saves ~5% but requires validation.

---

## 6. Maintenance Burden of Each Strategy

### Strategy: pytest-xdist

**Ongoing cost**: Low
- Configuration is one line in `pyproject.toml`
- Tests either work in parallel or they don't
- Debugging parallel failures: `pytest -n0 tests/failing_test.py` (just disable xdist)
- **Risk of flaky tests**: Low for this codebase (all in-memory, no shared resources)

### Strategy: Module-scoped DB fixtures

**Ongoing cost**: High
- Every new test file author must understand that DB state persists within a module
- Every test that inserts data needs a reviewer to check: "does this break other tests in this module?"
- When tests fail, developers must determine: is this a real failure or a fixture isolation issue?
- **Debugging is fundamentally harder**: `pytest tests/api/test_foo.py::test_bar` may pass alone but fail when run with the full module
- Migration changes now require verifying that module-scoped engines still create valid schemas

### Strategy: pytest-split CI sharding

**Ongoing cost**: Medium
- `.test_durations` file must be regenerated periodically (stale timing data → unbalanced shards)
- CI configuration is more complex (matrix strategy, artifact handling)
- When a test is added or removed, shards may become unbalanced until durations are updated
- Investigating CI failures requires knowing which shard ran which test

### Strategy: pytest-testmon

**Ongoing cost**: Medium
- `.testmondata` file must be maintained (can grow large)
- False negatives: testmon may skip tests that should run if the dependency graph is incomplete
- Import-level changes (adding an import) can trigger excessive test reruns
- **Not suitable for CI** — only for local development workflow

### Strategy: Tiered CI execution (unit → API → integration)

**Ongoing cost**: Low-Medium
- Need to maintain test markers or directory-based selection
- Adding new test categories requires updating CI config
- Risk of "tier 1 passes, tier 2 fails" being ignored by developers who only check fast tier

---

## 7. Diminishing Returns Analysis

### Where the real time goes

Let's decompose the 136s baseline:

| Component | Estimated Time | % of Total |
|-----------|---------------|------------|
| Import/collection overhead | ~3-4s | ~3% |
| Autouse fixture overhead (1,096 tests) | ~8-18s | ~6-13% |
| DB engine create_all/drop_all (per-test) | ~15-25s | ~11-18% |
| Qdrant client setup (per-test, API tests only) | ~3-5s | ~2-4% |
| test_client creation (356 API tests) | ~8-12s | ~6-9% |
| **Actual test execution** | **~75-95s** | **~55-70%** |

The reports focus heavily on fixture overhead (~30-45% of time). But **55-70% of time is actual test execution** — the `call` phase where test code runs, makes HTTP requests through the ASGI stack, queries the DB, and asserts results.

No fixture optimization can touch that 55-70%. Only parallelization can.

### The diminishing returns curve

```
Optimization                    Cumulative Speedup   Marginal Effort
─────────────────────────────────────────────────────────────────────
pytest-xdist (P0)              2-2.5x               Low (30 min)
Lazy torch import (P1)         2-2.5x (negligible)  Low (15 min)
Merge autouse fixtures (P1)    2.1-2.6x             Low (30 min)
Module-scoped DB engine (P2)   2.5-3x               HIGH (4+ hrs)
Module-scoped Qdrant (P2)      2.6-3.1x             Medium (2 hrs)
Session-scoped embedding (P2)  2.7-3.2x             Medium (1 hr)
CI sharding 4-way (P3)         8-10x wall-clock      Medium (2 hrs)
testmon (dev only) (P3)        N/A (dev workflow)    Low (15 min)
```

**The inflection point is after xdist.** Everything beyond that is incremental gain for significantly more complexity and risk. The CI sharding multiplier is real but applies to CI wall-clock time, not developer experience.

### Is module-scoped DB worth it?

The reports estimate module-scoped DB saves 30-40%. Applied after xdist (which already divides by 2-2.5x), the module-scoped optimization saves **12-20 seconds** off a **55-68s** parallel run (xdist).

Question: Is saving 12-20 seconds worth:
- 4+ hours of refactoring
- Permanent increase in test architecture complexity
- Risk of test isolation bugs
- Harder debugging for all future developers?

If this is a CI-bound project where every second counts, maybe. If developers run `pytest --lf` or `--testmon` for iterative work, probably not.

---

## 8. Infrastructure Costs

### CI sharding costs

Each CI shard is a separate GitHub Actions runner. The free tier gives 2,000 minutes/month for private repos. Going from 1 runner to 4 runners per PR means:
- 4x more CI minutes consumed
- 4x more Docker layer pulls, dependency installs
- Need caching (uv cache, Docker layer cache) to amortize

**Estimated CI cost increase**: 3-4x per run, partially offset by 3-4x faster completion. Net effect: roughly neutral in minutes, but more parallel resource usage.

### pytest-split timing database

The `.test_durations` file needs periodic regeneration. If stale:
- Shards become unbalanced (one shard gets all slow tests)
- CI times regress to near-serial
- Requires a dedicated CI step to update durations

**Estimated maintenance**: 30 min/month to verify shard balance, plus a CI job that regenerates durations on the default branch.

### testmon data management

`.testmondata` can grow to 10-50MB for large test suites. It must be:
- Excluded from git (`.gitignore`)
- Regenerated when switching branches
- Invalidated when dependencies change

**Estimated maintenance**: Low but occasional "testmon selected 0 tests" confusion when data is stale.

---

## 9. Alternative Perspective: Fewer, Better Tests

### The question nobody asked

The reports assume all 1,096 tests are necessary and valuable. But the dependency analysis reveals:
- **50 duplicate fixture definitions** (copy-pasted across files)
- **Zero parametrized tests** despite obvious parametrization candidates
- **476 redundant `@pytest.mark.asyncio` markers**
- **56 `@patch` decorators in a single file** (`test_device.py`)
- **10+ near-identical copies of `mock_image_asset`**

This suggests the test suite has **grown organically without systematic review**. Before optimizing execution speed, consider:

1. **Are there redundant tests?** Multiple test files test overlapping functionality (e.g., face suggestion acceptance appears in at least 4 files: `test_face_suggestions.py`, `test_face_suggestion_cleanup.py`, `test_face_session_suggestions.py`, `test_unknown_persons_accept.py`). Are these truly different scenarios or copy-paste variants?

2. **Could parametrization reduce fixture overhead more than scoping?** If 10 boundary tests in `test_temporal_service.py` become 2 parametrized tests (with 5 cases each), that's 8 fewer fixture setups per run. Across the whole suite, parametrization could eliminate 100-200 fixture setups — equivalent to the gains from module-scoped fixtures but with **zero isolation risk**.

3. **Should `test_device.py` (31 tests, 56 patches) be this complex?** This file tests PyTorch device detection with elaborate mocking. If device detection is a stable, well-understood function, maybe 5-10 parametrized tests suffice. That's 20 fewer fixture setups and a more readable test file.

### The test diet approach

| Action | Tests Eliminated | Fixture Saves | Risk |
|--------|-----------------|---------------|------|
| Parametrize `test_device.py` | ~20 | ~20 × 16ms = 320ms | Low |
| Parametrize `test_temporal_service.py` | ~30 | ~30 × 16ms = 480ms | Low |
| Parametrize `test_config_unknown_person.py` | ~10 | ~10 × 16ms = 160ms | Low |
| Parametrize `test_face_schemas.py` | ~10 | ~10 × 16ms = 160ms | Low |
| Deduplicate face fixture tests | ~15-20 | ~15 × 50ms = 750ms | Low |
| **Total** | **~85-90** | **~1.9s** | **Low** |

1.9 seconds isn't dramatic, but the **maintenance benefit is enormous**: fewer tests to understand, update, and debug. This is the kind of improvement that makes every future optimization cheaper.

---

## 10. Specific Risks for THIS Project

### Risk 10.1: FastAPI async tests with module-scoped engines

FastAPI's `TestClient` (or `AsyncClient` via httpx) runs the ASGI app in the same event loop as the test. When `db_engine` is module-scoped but `db_session` is function-scoped, the session must be created from the module-scoped engine **within the test's event loop**.

With `asyncio_mode = "auto"` and pytest-asyncio, each test function potentially gets its own event loop (depending on `loop_scope`). A module-scoped async engine created in one loop can't be used in a different loop.

- **This is the #1 risk for this specific project.** SQLAlchemy's async engine is tied to an event loop. Module-scoped async fixtures require `loop_scope = "module"` in pytest-asyncio 0.23+.
- **Current version: pytest-asyncio 1.3.0** — need to verify `loop_scope` support

- **Mitigation**: If module-scoping async engines, you MUST also set the event loop scope to `module`. Test thoroughly for "event loop is closed" errors.

### Risk 10.2: SQLAlchemy session cleanup between tests

The current fixture does:
```python
async with async_session() as session:
    yield session
    await session.rollback()
```

If switching to module-scoped engines with per-test sessions, the rollback ensures test isolation. But if any test calls `db_session.commit()` (and **160 of them do across 26 API test files**), the commit writes to the shared engine's database. `rollback()` after `commit()` is a no-op — the data is already persisted.

This means **every test that commits data will leave state in the shared module-scoped database**. The truncation approach (delete all rows between tests) is the only viable alternative, but it adds its own overhead and complexity.

- **Mitigation**: For module-scoped DB, you MUST use the truncation approach, not rollback. Budget ~1-2ms per table × 25+ tables × 1,096 tests = ~27-55ms per test. This partially negates the savings from avoiding `create_all`.

### Risk 10.3: Qdrant mock state leakage

Several face suggestion tests insert vectors into Qdrant and assert on similarity search results. With a module-scoped Qdrant client:

- Test A inserts face vectors for person_1
- Test B queries for similar faces and finds person_1's vectors (from Test A)
- Test B passes but would fail with a fresh Qdrant client

This is especially dangerous for the face suggestion tests (`test_face_suggestions.py`, `test_face_suggestion_cleanup.py`, etc.) which test specific similarity thresholds and result counts.

- **Mitigation**: Per-test point deletion from Qdrant collections. But this requires knowing which points were inserted (or deleting all points), which adds complexity.

### Risk 10.4: The `clear_embedding_cache` autouse fixture catches a real class of bug

The fixture clears `get_embedding_service.cache_clear()` before AND after each test. This prevents:
1. A cached **real** `EmbeddingService` instance from leaking into tests (which would try to load OpenCLIP)
2. A cached **mock** instance from one test leaking into another test that expects a different mock

If made session-scoped, scenario #2 becomes possible. Is it likely? Probably not with the current test suite. But any future test that customizes the embedding mock would silently affect all subsequent tests.

- **Mitigation**: If session-scoping, add a comment warning future developers. Or add a function-scoped autouse fixture that verifies the mock is in the expected state (fast assertion, no re-initialization).

### Risk 10.5: The `use_test_settings` fixture protects against production data deletion

This fixture sets `QDRANT_COLLECTION=test_image_assets` and `QDRANT_FACE_COLLECTION=test_faces`. Without it, `get_settings()` would return production collection names (`image_assets`, `faces`).

If this fixture were accidentally broken or scoped incorrectly:
- Tests could connect to production Qdrant
- `qdrant_client.create_collection()` in the fixture would **DELETE and recreate** the production collection
- All production vectors would be lost

The current function-scoped approach is bulletproof: every test starts with test collection names. A session-scoped approach should be equally safe (env vars are set once and never cleared during the session), but the **blast radius of a bug is total data loss**.

- **Mitigation**: This fixture should remain autouse and should clear + reset env vars per test. The cost is negligible (~0.1ms) and the protection is priceless. Do NOT session-scope this one.

---

## 11. Recommended Implementation Order (Risk-Adjusted)

| Priority | Action | Speedup | Risk | Effort |
|----------|--------|---------|------|--------|
| **P0** | Add pytest-xdist (`-n auto --dist worksteal`) | ~2-2.5x | Low | 30 min |
| **P0** | Add `pytest-timeout --timeout=30` | Safety net | None | 5 min |
| **P1** | Lazy-import torch in `core/device.py` | ~1.5s once | Low | 15 min |
| **P1** | Merge `clear_settings_cache` + `use_test_settings` | Negligible | None | 15 min |
| **P1** | Session-scope `validate_embedding_dimensions` | Negligible | None | 5 min |
| **P1** | Add `pytest-randomly` to detect hidden ordering deps | Safety net | Low | 10 min |
| **P2** | Parametrize `test_device.py` | ~20 fewer tests | Low | 1 hr |
| **P2** | Parametrize `test_temporal_service.py` | ~30 fewer tests | Low | 1 hr |
| **P2** | Consolidate duplicate fixtures | Maintenance | None | 2 hrs |
| **P2** | pytest-testmon for dev workflow | Dev experience | Low | 15 min |
| **P3** | CI sharding with pytest-split | CI wall-clock | Low | 2 hrs |
| **P3** | `COVERAGE_CORE=sysmon` in CI | Coverage speed | None | 5 min |
| **P4** | Module-scoped DB engine (prototype 1 module) | ~15-20% | **High** | 4+ hrs |
| **P4** | Module-scoped Qdrant client | ~5-10% | **High** | 2 hrs |
| **P4** | Session-scoped embedding mock | ~5% | **Medium** | 1 hr |

### What I would NOT do (or defer significantly)

1. **Module-scoped DB engine as a broad change**: The risk-to-reward ratio is poor. Save 15-20 seconds but introduce permanent isolation complexity. Prototype with one module first, measure, then decide.

2. **Session-scoped `clear_embedding_cache`**: The fixture is marked CRITICAL for a reason. The 5% savings (~6.8s) isn't worth the risk of subtle mock leakage.

3. **Custom test ordering hooks**: More complexity for marginal benefit. `pytest --ff` is built-in and sufficient.

4. **Module-scoped `test_client`**: The `create_app()` + dependency override pattern fundamentally assumes per-test isolation. Sharing an app instance means sharing `dependency_overrides` state. This is fragile.

---

## 12. Summary: What the Reports Get Right and Wrong

### Right
- pytest-xdist is the single highest-impact, lowest-risk change
- The autouse fixtures are over-aggressive (running for pure unit tests)
- Fixture duplication is a real maintenance problem
- Parametrization is underutilized
- CI sharding provides real wall-clock speedup
- testmon is excellent for dev workflow

### Overstated
- Speedup projections are too optimistic (multiplicative compounding doesn't work in practice)
- Module-scoped fixture savings (30-40%) don't account for cleanup overhead
- Autouse fixture overhead (~17.5s) uses high-end estimates; likely closer to 8-10s
- "Low risk" assessment of module-scoped DB changes understates async event loop complications

### Understated
- The CRITICAL safety role of autouse fixtures #1 and #2
- SQLite + async + module-scoped engine = event loop hell
- The 160 `db_session.commit()` calls that break rollback-based isolation
- Qdrant state leakage risk for face similarity tests
- The maintenance burden of module-scoped fixtures on future developers

### Missing
- `pytest-timeout` (the suite has no timeouts — a hung test blocks forever)
- `pytest-randomly` (no test ordering randomization — hidden dependencies may exist)
- A **measurement-first approach**: add `--durations=20` to CI NOW, measure for 2 weeks, THEN optimize based on data instead of estimates
- The option of **fewer, better tests** through parametrization and deduplication

---

## Final Word

The test suite runs in 136 seconds. With just pytest-xdist, it'll run in ~55-68 seconds. With CI sharding, wall-clock time drops to ~15-20 seconds.

That's already a **7-9x improvement in CI wall-clock time** with minimal risk. Everything beyond that is chasing the last 10-20 seconds at the cost of permanent architectural complexity.

**The best test optimization is the one you don't have to maintain.** pytest-xdist + CI sharding gives 90% of the benefit at 10% of the complexity. Ship that, measure, and revisit fixture scoping only if the numbers demand it.
