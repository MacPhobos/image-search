# Phase 4: Fixture Architecture — STATUS: DEFERRED

**Date**: 2026-02-15
**Status**: DEFERRED pending Go/No-Go data collection
**Decision**: Requires 2 weeks of CI `--durations` data from Phases 1-3 before proceeding

---

## Why Deferred

Phase 4 (module-scoped DB engine, Qdrant client, session-scoped embedding mock) is the highest-risk change in the test acceleration initiative. Phases 1-3 already delivered substantial improvements, and Phase 4 targets only modest additional savings with significant complexity and maintenance cost.

### What Phases 1-3 Already Delivered

| Metric | Before | After Phases 1-3 | Improvement |
|--------|--------|-------------------|-------------|
| Local full suite | ~140s | ~39s | 3.6x faster |
| CI wall-clock (target) | ~140s | ~15-20s | 7-9x faster (4-shard split) |
| Iterative dev (testmon) | ~140s | ~5-15s | Affected tests only |

### What Phase 4 Would Add

| Change | Projected Savings | Risk |
|--------|-------------------|------|
| MC3: Module-scoped DB engine | ~8-12s | HIGH |
| MC4: Module-scoped Qdrant client | ~2-3s | HIGH |
| MC5: Session-scoped embedding mock | ~3-4s | MEDIUM |
| **Combined (optimistic)** | **~13-19s** | **HIGH** |

Phase 4 trades test isolation for speed. The risk/reward ratio is poor given Phases 1-3 results.

---

## Go/No-Go Criteria

**ALL of the following must be true before starting Phase 4:**

1. **Fixture overhead >25% of total test time** after Phases 1-3
   - If fixture overhead is <25%, savings from module-scoping are <6 seconds. Not worth the complexity.
   - Measure with: `uv run pytest --durations=20 --durations-min=0.1 -p no:xdist 2>&1 | grep "setup\|teardown"`

2. **`db_engine` setup appears in the top 10 slowest setup operations**
   - If it's not in the top 10, it's not the bottleneck.

3. **No flaky tests detected by `pytest-randomly`** after 2 weeks of randomized CI runs
   - If there are existing flaky tests, do NOT add fixture scoping complexity on top.

4. **Team consensus to accept the maintenance cost**
   - Every new test author must understand module-scoped DB state.
   - Debugging test failures becomes harder with shared fixtures.

### No-Go Criteria (ANY one = skip Phase 4)

- Fixture overhead is <25% of total time
- Any flaky tests detected in randomized CI runs
- Phases 1-3 already achieved acceptable suite runtime (<45s local)
- Team does not want the maintenance burden

---

## Data Collection Plan

Collect `--durations=20` output from at least 10 CI runs over 2 weeks:

```bash
# Run locally to preview:
uv run pytest --durations=20 --durations-min=0.1 -n0 -p no:randomly 2>&1 | grep "setup\|teardown"
```

Review CI shard logs for the `--durations=10` output already included in Phase 3's CI workflow.

---

## Next Steps

1. Let Phases 1-3 run in CI for 2+ weeks
2. Collect `--durations` data from CI shard logs
3. Evaluate Go/No-Go criteria with collected data
4. If Go: follow `phase-4-fixture-architecture.md` implementation plan
5. If No-Go: close Phase 4 as unnecessary — Phases 1-3 are sufficient

---

## Reference

- Full implementation plan: `phase-4-fixture-architecture.md`
- Phase 1 (Quick Wins): `phase-1-quick-wins.md`
- Phase 2 (Developer Experience): `phase-2-developer-experience.md`
- Phase 3 (CI Optimization): `phase-3-ci-optimization.md`
