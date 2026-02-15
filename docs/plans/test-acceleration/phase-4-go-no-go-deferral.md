# Phase 4: Fixture Architecture — Deferred

**Date**: 2026-02-15
**Status**: DEFERRED pending data collection
**Decision**: Do NOT implement until Go/No-Go criteria are met

---

## Current State After Phases 1-3

| Metric | Before | After Phases 1-3 | Improvement |
|--------|--------|-------------------|-------------|
| Local full suite (parallel) | ~136s | ~39-42s | ~3.3x |
| CI wall-clock (slowest shard) | ~68s | ~15-20s (projected) | ~3.5-4.5x |
| Iterative dev (testmon) | ~136s | ~5-15s (affected only) | ~9-27x |
| Test count | ~1096 | 1105 (parametrized) | +9 boundary tests |

These improvements are already substantial. Phase 4 targets an additional ~8-15s locally with HIGH risk.

---

## Why Phase 4 Is Deferred

1. **Risk/reward is poor**: Phase 4 trades test isolation for speed. Module-scoped fixtures introduce state leakage risks that are the hardest class of test bugs to diagnose.

2. **Insufficient data**: The plan requires 2+ weeks of `--durations=20` data from CI to prove fixture overhead is >25% of total time. This data does not exist yet.

3. **Phases 1-3 may be sufficient**: If the team finds ~40s local / ~20s CI acceptable, Phase 4 is unnecessary.

---

## Go/No-Go Criteria Checklist

**ALL Go criteria must be true to proceed. ANY No-Go criterion = skip Phase 4.**

### Go Criteria

- [ ] **Phases 1-3 merged and stable for 2+ weeks**
- [ ] **Fixture setup/teardown is >25% of total test time** (measured from `--durations=20 --durations-min=0.1` over 10+ CI runs)
- [ ] **`db_engine` setup appears in top 10 slowest setup operations**
- [ ] **No flaky tests detected** by pytest-randomly after 2 weeks of randomized CI runs
- [ ] **Team accepts maintenance cost** of module-scoped DB state for all future test authors

### No-Go Criteria (any one = skip)

- [ ] Fixture overhead is <25% of total time
- [ ] Any flaky tests detected in randomized CI runs
- [ ] Phase 1-3 already achieved acceptable runtime (<45s local)
- [ ] Team does not want the maintenance burden

---

## Data Collection Plan

After Phases 1-3 are merged, collect this data over 2 weeks:

```bash
# Run weekly to track fixture overhead trend
uv run pytest --durations=20 --durations-min=0.1 -n0 -p no:randomly 2>&1 | grep "setup\|teardown"
```

Check CI `--durations=10` output in each shard's logs for the slowest operations.

---

## Decision Points

| Timeframe | Action |
|-----------|--------|
| Now | Merge Phases 1-3. Begin data collection from CI durations. |
| +2 weeks | Review Go/No-Go criteria with CI data. |
| +2 weeks (Go) | Implement MC3 prototype on `test_categories.py` only. Validate with 50 randomized runs. |
| +2 weeks (No-Go) | Close Phase 4. Phases 1-3 are the final state. |

---

## Abandon Criteria (if Phase 4 is started)

Stop immediately if any occur during prototyping:

- Any failure in 50-run validation (even 1/50 = abandon)
- "Event loop is closed" errors from pytest-asyncio
- Tests pass alone but fail in module context (state leakage)
- Tests fail alone but pass in module context (false positive)
- Timing savings < 3 seconds for the prototype module
- Any interaction with pytest-randomly that causes failures

---

## Reference

Full Phase 4 plan: `docs/plans/test-acceleration/phase-4-fixture-architecture.md`
Review notes: `docs/plans/test-acceleration/review-notes.md`
