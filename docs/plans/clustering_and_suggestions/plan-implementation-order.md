Suggested Implementation Order

Quick wins first (under 2 hours each):
1. C1 — 3 one-line fixes, 15 minutes, unblocks Find More features
2. C2 — Add Qdrant sync calls + repair script, 1-2 hours
3. C3 — Reverse deprecation order + use BUILDING status, 1-2 hours

Then medium effort:
4. H6 — Fix 9 hardcoded collection names, ~1 day
5. H2 — Terminology unification, ~3 days
6. H5 — Optimistic locking, ~3 days

Larger refactors:
7. H1 — Batch Qdrant retrieval, ~3.75 days (can be split into 4 independent PRs)
8. C4 — Centroid rejection persistence, ~4 days
9. C5 — Unify centroid computation, ~4 days
