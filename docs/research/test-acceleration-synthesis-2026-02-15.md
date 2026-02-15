# Test Acceleration Synthesis: image-search-service

**Date**: 2026-02-15
**Type**: Synthesis of 4 research tracks
**Classification**: Actionable
**Inputs**:
- Architecture analysis (test structure, config, bottleneck ranking)
- Techniques research (12 acceleration techniques with estimated impact)
- Dependency analysis (fixture weight, mocking patterns, duplication)
- Devil's advocate (risk analysis, tradeoffs, counterarguments)

---

## 1. Executive Summary

### Current State

The `image-search-service` backend has **1,096 active pytest tests** (15 postgres-marked tests deselected) across **87 test files**, running in **~136 seconds** on the current machine. Tests run entirely in-memory (SQLite + in-memory Qdrant), with no external service dependencies. The suite has zero parallelization, zero parametrized tests, and function-scoped fixtures throughout.

### Target State

| Scenario | Target Time | Speedup | Confidence |
|----------|-------------|---------|------------|
| Local development (full suite) | ~55-68s | 2-2.5x | **High** |
| Local development (iterative) | ~5-15s | 10-25x | **High** |
| CI wall-clock (with sharding) | ~15-20s | 7-9x | **Medium-High** |

### Confidence Assessment

**High confidence**: pytest-xdist parallelization and developer workflow tools (testmon, --lf/--ff) deliver substantial improvements with minimal risk.

**Medium confidence**: CI sharding delivers wall-clock gains but adds configuration complexity.

**Low confidence**: Fixture scope restructuring (module-scoped DB, Qdrant) delivers projected gains. The devil's advocate analysis identifies serious risks with SQLite async engines and test isolation that make projected 30-40% savings unlikely to materialize cleanly.

**Bottom line**: Ship xdist + workflow tools first. Measure for 2 weeks. Then decide if fixture restructuring is worth the permanent architectural complexity.

---

## 2. Findings Summary

### Track 1: Architecture Analysis

- **1,096 tests**: 48.5% async, 51.5% sync. 356 API tests use the heavyweight `test_client` fixture; 426 are pure unit tests needing no DB/Qdrant.
- **4 autouse fixtures** run for every test (~8-18s total overhead depending on estimate). Two are data-safety guards marked CRITICAL.
- **Function-scoped `test_client`** creates a complete FastAPI app + SQLite DB + Qdrant collections per test (~30-50ms each, ~10-18s total for API tests).
- **Torch import** at module level in `core/device.py` costs ~1.8s at session startup.
- **Zero parametrization** and zero parallel execution configured.

### Track 2: Acceleration Techniques

- **pytest-xdist** is the highest-impact single change (estimated 2-2.5x on 4 cores after adjusting for worker startup overhead).
- **CI sharding** (pytest-split + GitHub Actions matrix) provides additional wall-clock reduction for CI.
- **Module-scoped DB engine** projected 30-40% fixture savings, but requires SQLite shared-cache mode and careful session management.
- **pytest-testmon** enables 10-50x speedup for iterative development by running only affected tests.
- **Import optimization** (lazy torch) saves ~1.8s one-time per session.

### Track 3: Dependency & Fixture Analysis

- **184 total fixture definitions** (14 in conftest, 134 inline in test files, 36 in helpers). Zero parametrized tests.
- **50+ duplicate fixture definitions** (`mock_image_asset` in 8 locations, `mock_person` in 8, `mock_face_instance` in 6, `mock_qdrant_client` in 10).
- **777 `db_session.add/commit/flush` calls** across 47 test files. 160 `commit()` calls in API tests — these break rollback-based isolation strategies.
- **`test_client`** is the single most expensive fixture: creates app + 2 DB engines + Qdrant + 5 dependency overrides per invocation.
- **SemanticMockEmbeddingService** has a `_cache` dict that grows across calls (relevant if session-scoped).

### Track 4: Devil's Advocate

- **xdist speedup is overstated**: Expect 2-2.5x on 4 cores, not 3x, due to worker startup overhead (each worker re-imports conftest + torch).
- **Module-scoped DB engine is the highest-risk proposal**: SQLite `:memory:` with async + connection pools = each connection gets its own empty database. Requires `StaticPool` or `cache=shared` URI, both with significant caveats.
- **Autouse fixtures #1 and #2 are safety-critical**: They prevent tests from using production Qdrant collection names. Removing per-test reset risks production data deletion.
- **55-70% of test time is actual test execution**, not fixture overhead. No fixture optimization can touch that portion — only parallelization can.
- **Speedup projections don't compound**: Phase 1 "2.5x" + Phase 2 "30-40%" does not equal 3.5x. After xdist, fixture savings apply only to the fixture portion (~30-45% of remaining time).

---

## 3. Risk-Adjusted Recommendations

### Quick Wins (< 1 hour total, low risk)

These changes are safe, independent, and can be shipped immediately.

| # | Change | Effort | Expected Impact | Risk |
|---|--------|--------|-----------------|------|
| QW1 | Install pytest-xdist, configure `-n auto --dist worksteal` | 15 min | **2-2.5x** (136s -> ~55-68s) | Low |
| QW2 | Add `pytest-timeout --timeout=30` as safety net | 5 min | Prevents hung tests blocking CI forever | None |
| QW3 | Add `--durations=10` to CI runs | 5 min | Visibility into slowest tests | None |
| QW4 | Disable unused pytest plugins (`-p no:pastebin -p no:doctest -p no:nose`) | 5 min | ~0.1-0.3s collection time | None |
| QW5 | Session-scope `validate_embedding_dimensions` (fixture #4) | 10 min | Negligible time savings; removes 1,095 redundant assertion cycles | None |
| QW6 | Merge `clear_settings_cache` + `use_test_settings` into one autouse fixture | 15 min | ~2% (~2.7s) | None |
| QW7 | Add `pytest-randomly` to detect hidden ordering dependencies | 10 min | Safety net for future changes | Low |

**Combined Quick Wins**: ~55-68s (2-2.5x from xdist) with safety nets in place.

### Medium Effort (1-4 hours, moderate risk)

These provide meaningful improvements but require more careful implementation.

| # | Change | Effort | Expected Impact | Risk |
|---|--------|--------|-----------------|------|
| ME1 | Lazy-import `torch` in `core/device.py` | 15 min | ~1.8s one-time session savings; also reduces xdist worker startup by ~1.8s each | Low |
| ME2 | Parametrize `test_device.py` (31 tests -> ~10) | 1 hr | ~20 fewer test functions, fewer fixture setups, cleaner code | Low |
| ME3 | Parametrize `test_temporal_service.py` (57 tests -> ~20) | 1 hr | ~30 fewer test functions | Low |
| ME4 | Consolidate duplicate fixtures into shared conftest | 2 hrs | Maintenance improvement; eliminates ~50 redundant definitions | None |
| ME5 | Install pytest-testmon for dev workflow | 15 min | 10-50x for iterative development (only re-runs affected tests) | Low |
| ME6 | Add `--lf` / `--ff` Makefile targets | 10 min | Developer experience improvement | None |

**Cumulative after Medium Effort**: ~50-62s locally (marginal time improvement from fewer tests + faster worker startup), but significant maintenance and developer experience gains.

### Major Changes (1+ days, higher risk) — Prototype First

These are the largest potential gains but carry the most risk. **Do not apply broadly.** Prototype with a single test module, run 50 times, then evaluate.

| # | Change | Effort | Expected Impact | Risk | Prerequisite |
|---|--------|--------|-----------------|------|-------------|
| MC1 | CI sharding with pytest-split (4 shards) | 2 hrs | CI wall-clock: ~15-20s | Low-Med | QW1 (xdist), CI config |
| MC2 | `COVERAGE_CORE=sysmon` in CI | 5 min | 30-50% coverage overhead reduction | Low | Python 3.12+ |
| MC3 | Module-scoped DB engine (prototype 1 module) | 4+ hrs | ~15-20% of remaining fixture time | **High** | See prototyping guidance below |
| MC4 | Module-scoped Qdrant client with per-test cleanup | 2 hrs | ~5-10% | **High** | Requires per-test point deletion |
| MC5 | Session-scoped embedding mock (fixture #3) | 1 hr | ~5% (~3-4s) | **Medium** | Must prove no test modifies mock |

#### Prototyping Guidance for MC3 (Module-Scoped DB Engine)

This is the riskiest proposal. The devil's advocate analysis identified these blockers:

1. **SQLite `:memory:` creates a new database per connection**. A module-scoped engine with a connection pool will give each connection its own empty DB. You must use either:
   - `StaticPool` (serializes all async through one connection — defeats async), or
   - `sqlite+aiosqlite:///file:testdb?mode=memory&cache=shared&uri=true` (shared-cache mode — has known issues with aiosqlite)

2. **160 `commit()` calls across API tests** mean rollback-based isolation won't work. You must use table truncation between tests, which adds ~1-2ms per table x 25+ tables per test.

3. **Event loop scope**: Module-scoped async engines require `loop_scope = "module"` in pytest-asyncio. Verify this works with pytest-asyncio 1.3.0.

**Prototype steps**:
```bash
# 1. Pick a small, self-contained API test module
#    Candidate: tests/api/test_categories.py (21 tests)

# 2. Create a module-scoped engine fixture in that file
# 3. Add table truncation cleanup
# 4. Run the module 50 times to check for flakiness:
uv run pytest tests/api/test_categories.py --count=50 -x -v

# 5. Compare timing:
uv run pytest tests/api/test_categories.py --durations=0 -v  # module-scoped
uv run pytest tests/api/test_categories.py --durations=0 -v -p no:xdist  # baseline

# 6. If stable: expand to 2-3 more modules. If flaky: abandon.
```

---

## 4. Effort vs Impact Matrix

| Technique | Effort | Expected Speedup | Risk | Confidence |
|-----------|--------|------------------|------|------------|
| **QW1**: pytest-xdist (`-n auto --dist worksteal`) | 15 min | **2-2.5x** | Low | High |
| **QW2**: pytest-timeout (safety net) | 5 min | N/A (prevents hangs) | None | High |
| **QW3**: `--durations=10` in CI | 5 min | N/A (visibility) | None | High |
| **QW6**: Merge autouse settings fixtures | 15 min | ~2% | None | High |
| **QW7**: pytest-randomly | 10 min | N/A (safety net) | Low | High |
| **ME1**: Lazy torch import | 15 min | ~1.8s session + worker startup | Low | High |
| **ME2-3**: Parametrize test_device + test_temporal | 2 hrs | ~50 fewer tests, cleaner code | Low | High |
| **ME4**: Consolidate duplicate fixtures | 2 hrs | Maintenance improvement | None | High |
| **ME5**: pytest-testmon (dev workflow) | 15 min | **10-50x for iterative dev** | Low | Medium |
| **MC1**: CI sharding (4 shards) | 2 hrs | CI wall-clock **3-4x** | Low-Med | Medium |
| **MC2**: `COVERAGE_CORE=sysmon` | 5 min | 30-50% coverage overhead reduction | Low | Medium |
| **MC3**: Module-scoped DB engine | 4+ hrs | ~15-20% of fixture time | **High** | Low |
| **MC4**: Module-scoped Qdrant client | 2 hrs | ~5-10% | **High** | Low |
| **MC5**: Session-scoped embedding mock | 1 hr | ~5% (~3-4s) | **Medium** | Medium |

---

## 5. Phased Implementation Plan

### Phase 1: Quick Wins (Day 1, < 1 hour)

**Goal**: 2-2.5x speedup with safety nets. Zero risk to test isolation.

```bash
cd image-search-service

# Step 1: Install xdist and timeout
uv add --group dev "pytest-xdist>=3.8.0" "pytest-timeout>=2.3.0" "pytest-randomly>=3.16.0"
```

Update `pyproject.toml`:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "postgres: marks tests as requiring PostgreSQL (deselect with '-m \"not postgres\"')",
]
addopts = "-m 'not postgres' -n auto --dist worksteal --timeout=30 -p randomly -p no:pastebin -p no:doctest -p no:nose"
```

Merge autouse settings fixtures in `tests/conftest.py` — combine `clear_settings_cache` and `use_test_settings` into a single `test_settings` fixture. Keep it function-scoped and autouse.

Session-scope `validate_embedding_dimensions`:
```python
@pytest.fixture(autouse=True, scope="session")
def validate_embedding_dimensions():
    """Validate embedding dimensions once at session start."""
    from tests.conftest import MockEmbeddingService
    assert MockEmbeddingService.EMBEDDING_DIM == 768
    test_instance = MockEmbeddingService()
    assert test_instance.embedding_dim == 768
```

Update Makefile:
```makefile
test: ## Run pytest (parallel by default)
	uv run pytest

test-serial: ## Run pytest serially (for debugging)
	uv run pytest -p no:xdist -p no:randomly

test-fast: ## Run only previously failed tests
	uv run pytest --lf -p no:xdist

test-failed-first: ## Run failed tests first, then rest
	uv run pytest --ff

test-profile: ## Show slowest tests
	uv run pytest --durations=20 --durations-min=0.5 -p no:xdist
```

**Validate**:
```bash
uv run pytest  # Should complete in ~55-68s with xdist
uv run pytest -p no:xdist  # Verify same pass count as before (~136s)
```

### Phase 2: Developer Experience + Code Quality (Week 1, 2-4 hours)

**Goal**: Faster iterative development, cleaner test code.

**Step 2a**: Lazy-import torch in `core/device.py`
```python
# Before:
import torch  # 1.8s import at module level

# After:
def get_device() -> str:
    import torch  # Deferred to first call
    ...
```

**Step 2b**: Install testmon for dev workflow
```bash
uv add --group dev "pytest-testmon>=2.1.0"
```
Add Makefile target:
```makefile
test-affected: ## Run only tests affected by recent changes
	uv run pytest --testmon -p no:xdist
```

**Step 2c**: Parametrize `test_device.py`
- Current: 31 tests, 56 `@patch` decorators
- Target: ~10 parametrized tests covering the same matrix of device/env combinations
- Template:
```python
@pytest.mark.parametrize("cuda_avail,mps_avail,env_override,expected", [
    (True, False, None, "cuda"),
    (False, True, None, "mps"),
    (False, False, None, "cpu"),
    (False, False, "cuda", "cuda"),
    # ... etc
])
def test_get_device(cuda_avail, mps_avail, env_override, expected):
    ...
```

**Step 2d**: Parametrize `test_temporal_service.py`
- Current: 57 tests across 8 classes
- Target: ~20 parametrized tests
- Each class with repeated input/output patterns becomes a single parametrized test

**Step 2e**: Consolidate duplicate fixtures
- Move `mock_image_asset`, `mock_person`, `mock_face_instance` into `tests/conftest.py` (or a new `tests/fixtures.py` imported by conftest)
- Remove the 50+ duplicate definitions from individual test files
- Verify no test breaks: `uv run pytest -p no:xdist -x`

### Phase 3: CI Optimization (Week 2, 2-4 hours)

**Goal**: Minimize CI wall-clock time.

**Step 3a**: Generate test timing data
```bash
uv run pytest --store-durations -p no:xdist  # Creates .test_durations
```

**Step 3b**: Add pytest-split
```bash
uv add --group dev "pytest-split>=0.9.0"
```

**Step 3c**: Update GitHub Actions (if applicable)
```yaml
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
```

**Step 3d**: Enable fast coverage
```yaml
- name: Run tests with coverage
  env:
    COVERAGE_CORE: sysmon
  run: uv run pytest --cov=image_search_service --cov-report=xml -n auto
```

### Phase 4: Fixture Architecture (Only if Phases 1-3 are insufficient)

**Prerequisite**: 2 weeks of `--durations=20` data from CI. Only proceed if the data shows fixture setup is still a meaningful bottleneck after xdist.

**Step 4a**: Prototype module-scoped DB engine with `test_categories.py` (21 tests)
- Use `sqlite+aiosqlite:///file:testdb?mode=memory&cache=shared&uri=true`
- Use `StaticPool` for the async engine
- Add table truncation between tests (DELETE FROM all tables)
- Set `loop_scope = "module"` for pytest-asyncio
- Run 50 iterations: `uv run pytest tests/api/test_categories.py --count=50 -x`
- If stable for 50 runs: expand to 2-3 more modules
- If any flakiness: **abandon this approach**

**Step 4b**: Prototype module-scoped Qdrant client
- Add per-test cleanup fixture that deletes all points from both collections
- Verify face suggestion tests still pass (they assert on specific result counts and similarity scores)

**Step 4c**: Evaluate session-scoped embedding mock
- Grep for any test that modifies `EmbeddingService` at the class level
- If none found: switch `clear_embedding_cache` from monkeypatch to `unittest.mock.patch` at session scope
- Add a function-scoped autouse fixture that clears the `_cache` dict between tests

---

## 6. What NOT to Do

| Recommendation from Research | Why to Defer/Avoid |
|-------------------------------|-------------------|
| **Session-scope `use_test_settings` (fixture #2)** | This fixture prevents tests from using production Qdrant collection names. The cost is ~0.1ms/test. The blast radius of a bug is **total production data loss**. Keep function-scoped forever. |
| **Session-scope `clear_embedding_cache` (fixture #3) broadly** | Marked CRITICAL — prevents loading real OpenCLIP/SigLIP models. The `_cache` dict grows unboundedly if session-scoped. A test that patches `embed_text` would affect all subsequent tests. The ~5% savings (~3-4s) is not worth the risk without thorough validation. |
| **Module-scoped `test_client`** | The `create_app()` + `dependency_overrides` pattern assumes per-test isolation. Sharing an app instance means sharing override state. The dependency analysis found `test_client` referenced 815 times across 34 files — the blast radius is enormous. |
| **Module-scoped DB engine as a broad rollout** | 160 `commit()` calls break rollback-based isolation. SQLite `:memory:` + async + connection pools = each connection gets its own empty database. Prototype with ONE module first; do not apply globally without proven stability. |
| **Custom test ordering hooks** | More complexity for marginal benefit. `pytest --ff` is built-in and sufficient for development workflow. |
| **Removing the 4 autouse fixtures** | Fixtures #1 and #2 are safety-critical (prevent production data deletion). Fixture #3 prevents loading expensive ML models. Fixture #4 can be session-scoped (safe), but the others must remain per-test or merged carefully. |
| **pytest-parallel** | Abandoned project (last release 2020). Thread-based mode doesn't work with SQLite. Use pytest-xdist instead. |

---

## 7. Success Metrics

### Primary Metrics

| Metric | Baseline | Phase 1 Target | Phase 2 Target | How to Measure |
|--------|----------|----------------|----------------|----------------|
| Full suite wall-clock (local) | ~136s | ~55-68s | ~50-62s | `time uv run pytest` |
| Full suite wall-clock (CI) | ~136s | ~55-68s | ~15-20s (sharded) | CI job duration |
| Iterative dev feedback time | ~136s | ~136s (no change) | ~5-15s (testmon) | `time uv run pytest --testmon` |
| Test count | 1,096 | 1,096 | ~1,000 (parametrized) | `pytest --co -q \| tail -1` |
| Slowest test duration | 3.78s | 3.78s | 3.78s | `pytest --durations=5` |

### Monitoring Commands

```bash
# After Phase 1: Compare parallel vs serial
time uv run pytest                    # Parallel (should be ~55-68s)
time uv run pytest -p no:xdist       # Serial baseline (~136s)

# Ongoing: Track slowest tests in CI
uv run pytest --durations=20 --durations-min=0.5

# Weekly: Verify no test ordering dependencies
uv run pytest -p randomly --randomly-seed=last -x  # Reproduce last random order
uv run pytest -p randomly -x                        # New random order

# Monthly: Full isolation check
uv run pytest -p no:xdist -p randomly -x  # Serial + random order

# Check xdist worker balance
uv run pytest -n 4 -v 2>&1 | grep "worker" | sort | uniq -c
```

### Regression Detection

Add to CI (if not already):
```toml
# pyproject.toml
[tool.pytest.ini_options]
# Fail if any test exceeds 10s (catches accidental real service connections)
timeout = 30
```

---

## 8. Realistic Projections

### Scenario Analysis

Projections are based on the conservative estimates from the devil's advocate analysis, not the optimistic compounding from the techniques research.

#### Conservative Scenario (Minimum Expected)

Only Quick Wins (Phase 1) implemented.

| Component | Time |
|-----------|------|
| xdist on 4 cores (2x speedup) | ~68s |
| Autouse fixture merge | saves ~2.7s -> ~65s |
| Lazy torch import (helps worker startup) | saves ~1s -> ~64s |
| **Local full suite** | **~64-68s** |
| **CI with 4 shards** | **~17-20s** |

#### Likely Scenario (Phases 1 + 2)

Quick Wins + Developer Experience + CI Sharding.

| Component | Time |
|-----------|------|
| xdist on 4 cores (2.2x speedup) | ~62s |
| Autouse + lazy import savings | ~58s |
| Parametrization (~50 fewer tests) | ~55s |
| **Local full suite** | **~55-60s** |
| **CI with 4 shards** | **~15-18s** |
| **Iterative dev (testmon)** | **~5-15s** |

#### Optimistic Scenario (All Phases, fixture restructuring succeeds)

Phases 1 + 2 + 3 + successful Phase 4 prototyping.

| Component | Time |
|-----------|------|
| xdist on 4 cores (2.2x speedup) | ~62s |
| Autouse + lazy import savings | ~58s |
| Parametrization (~50 fewer tests) | ~55s |
| Module-scoped DB (saves ~15% of fixture time) | ~48-50s |
| Module-scoped Qdrant (saves ~5%) | ~46-48s |
| **Local full suite** | **~46-50s** |
| **CI with 4 shards + sysmon coverage** | **~12-15s** |
| **Iterative dev (testmon)** | **~3-10s** |

### Key Insight: Where the Time Goes After xdist

After applying xdist (2-2.5x), the remaining ~55-68s breaks down as:

| Component | Est. Time | % | Can Optimize? |
|-----------|-----------|---|--------------|
| Actual test execution (call phase) | ~35-45s | ~55-65% | **No** (only more cores help) |
| DB fixture setup/teardown | ~8-12s | ~12-18% | Module-scoping (risky) |
| Qdrant fixture setup | ~2-3s | ~3-5% | Module-scoping (risky) |
| Autouse fixtures | ~4-5s | ~6-8% | Already optimized in Phase 1 |
| test_client creation | ~4-6s | ~6-9% | Tied to DB/Qdrant scoping |
| Import/collection overhead | ~2-3s | ~3-5% | Lazy imports |
| Other | ~2-4s | ~3-6% | Diminishing returns |

**55-65% of the remaining time is irreducible test execution.** This is why fixture optimization after xdist yields diminishing returns — you're optimizing the 35-45% that's fixture overhead, not the 55-65% that's actual test code.

---

## Appendix A: Decision Log

| Decision | Chosen Option | Alternatives Considered | Rationale |
|----------|--------------|------------------------|-----------|
| Parallelization tool | pytest-xdist | pytest-parallel, custom asyncio.gather | xdist: active maintenance (8M+ downloads/mo), process isolation, worksteal scheduler |
| xdist distribution mode | `worksteal` | `load`, `loadscope`, `loadfile` | Best for varying test durations (3.78s outlier) |
| Autouse fixture #1+#2 | Merge, keep function-scoped | Session-scope, remove | Safety-critical: prevent production data deletion |
| Autouse fixture #3 | Keep function-scoped (for now) | Session-scope with mock.patch | Risk of mock state leakage; 3-layer patch stack is fragile |
| Autouse fixture #4 | Session-scope | Keep function-scoped | Only validates constants; no risk from running once |
| Module-scoped DB | Defer to Phase 4 (prototype only) | Implement broadly | SQLite async + `:memory:` + connection pools = each connection gets own DB; 160 commits break rollback isolation |
| CI sharding | pytest-split (4 shards) | Manual directory splitting, Tiered execution | Data-driven shard balancing; combine with xdist for max throughput |
| Dev workflow | pytest-testmon + --lf/--ff | pytest-incremental | testmon is more precise (line-level coverage vs import-graph) |

## Appendix B: Research Document Index

| Document | Focus | Key Finding |
|----------|-------|-------------|
| `test-acceleration-architecture-analysis-2026-02-15.md` | Test structure, config, bottlenecks | 4 autouse fixtures cost ~8-18s; function-scoped test_client costs ~10-18s; torch import costs 1.8s |
| `test-acceleration-techniques-research-2026-02-15.md` | 12 acceleration techniques | xdist 3x (revised to 2-2.5x), module-scoped DB 30-40%, CI sharding 3-4x, testmon 10-50x for dev |
| `test-acceleration-dependency-analysis-2026-02-15.md` | Fixture weight, duplication, mocking | 50+ duplicate fixtures, 777 DB write ops, 160 commits break rollback, zero parametrized tests |
| `test-acceleration-devils-advocate-2026-02-15.md` | Risks and counterarguments | Module-scoped DB is highest risk; autouse fixtures #1/#2 are safety guards; 55-70% of time is actual execution |
