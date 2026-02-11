# Unknown Person Face Detection -- Research Synthesis

**Date**: 2026-02-07
**Status**: Complete
**Sources**: 5 parallel research streams (General, Backend, Database/Qdrant, UI/UX, Devil's Advocate)
**Purpose**: Decision-ready synthesis for the team to start implementation

---

## 1. Executive Summary

The image-search system has a mature face recognition pipeline (InsightFace detection, 512-dim ArcFace embeddings in Qdrant, HDBSCAN clustering, dual-mode supervised/unsupervised assignment, multi-role prototypes) but a critical workflow gap: **there is no path from "unknown face cluster" to "new Person entity"** without tedious manual navigation through the Clusters view. The Face Suggestions system only matches faces to *already-known* persons. Users who want to discover new people must browse anonymous clusters, manually create Person records, and assign faces one cluster at a time.

All five research streams agree on the core problem. Where they diverge is on the solution approach -- specifically whether to build a new view with dynamic threshold controls or to enhance the existing Clusters view with streamlined workflows. After weighing the evidence, this synthesis recommends a **phased hybrid approach**: start by enhancing the Clusters view with "Create Person" workflows and pre-computed confidence scoring (Phase 0-1), then add the tabbed "Unlabeled Groups" sub-view within Suggestions with pre-computed groupings (Phase 2), and defer dynamic threshold re-clustering to Phase 3 only if user demand warrants it.

**Key numbers**: The existing system provides ~80% of the backend building blocks. Phase 1 requires no new database tables. The primary technical risk is O(N^2) memory for real-time clustering, which the phased approach avoids by pre-computing. Estimated MVP delivery: 2-3 weeks (Phases 0-1).

---

## 2. Consensus Points

All five research streams agree on the following:

### 2.1 The Gap Is Real and Well-Defined

The system can match faces to *existing* persons but cannot propose *new* persons. `FaceSuggestion` requires a non-null `suggested_person_id` (Doc 01, Doc 02). After clustering, faces receive generic `cluster_id` labels like `unknown_cluster_3` but are never promoted to `Person` entities automatically (Doc 01). The Clusters UI shows grouped faces but lacks a streamlined "create person from this cluster" workflow (Doc 01, Doc 04, Doc 05).

### 2.2 The Backend Building Blocks Already Exist

All streams confirm that 80%+ of needed infrastructure is in place:

| Capability | Location | Status |
|---|---|---|
| 512-dim ArcFace embeddings | Qdrant `faces` collection | Production |
| HDBSCAN clustering of unlabeled faces | `FaceClusterer`, `DualModeClusterer` | Production |
| Cluster confidence calculation | `FaceClusteringService.calculate_cluster_confidence()` | Production |
| Cluster-to-person labeling | `POST /faces/clusters/{cluster_id}/label` | Production |
| Centroid computation + search | `CentroidService`, `CentroidQdrantClient` | Production |
| Background job infrastructure | Redis + RQ, 4-tier priority queues | Production |
| Grouped pagination pattern | `FaceSuggestionsGroupedResponse` | Production |
| Unified people view | `PersonService.get_all_people()` with PersonType enum | Production |

### 2.3 Pre-Computation Is Preferred Over Real-Time Clustering

All streams agree that HDBSCAN clustering should run as a background job, not on every API request. Real-time re-clustering is O(N^2) in memory and seconds-to-minutes in latency (Doc 03: 5,000 faces = ~35s, ~200MB distance matrix; 10,000 faces = ~100s, ~800MB). Pre-computed results enable sub-200ms API responses (Doc 02).

### 2.4 The Cluster Labeling Flow Should Be the Core Interaction

The existing `POST /faces/clusters/{cluster_id}/label` endpoint already creates a Person, assigns all cluster faces, creates prototypes, and syncs to Qdrant (Doc 02). This is the exact "accept unknown person" flow needed. All streams agree this should be reused, not reimplemented.

### 2.5 Singleton Faces Are a Known, Unsolved Problem

Faces appearing in only 1-2 photos never cluster with HDBSCAN's `min_cluster_size >= 3` (Doc 01, Doc 05). In typical collections, 40-60% of unique faces are one-timers (Doc 05). No proposed approach fully solves this; the consensus is to surface singletons separately (quality-gated, optional toggle) rather than lowering clustering thresholds and risking false merges.

### 2.6 Industry Pattern: Conservative First Pass, Then Refine

Google Photos, Apple Photos, and Immich all use multi-pass strategies: high-precision auto-grouping first, followed by user-driven merge/refine (Doc 01). The consensus is to follow this pattern rather than exposing raw threshold controls.

---

## 3. Key Tensions and Resolutions

### 3.1 New View vs. Enhance Existing Clusters View

**Tension**: Doc 04 proposes a new tabbed sub-view within the Suggestions page. Doc 05 argues the Clusters view already covers most of this use case and a parallel view creates UX confusion and duplicate code paths.

**Resolution**: **Both, phased.** Phase 0-1 enhances the Clusters view with "Create Person" workflows, confidence scoring, and sort/filter -- this is low-risk and delivers immediate value. Phase 2 adds the tabbed sub-view in Suggestions *only after* the backend API is proven and the team validates that users want a discovery-oriented view separate from the cluster management view. This avoids building a UI that might be unnecessary while still planning for it.

**Rationale**: Doc 05 correctly identifies that the Clusters view already shows the same data. However, Doc 04 correctly identifies that the Clusters view's UX is optimized for cluster *management* (splitting, inspecting), not person *discovery* (create person, dismiss, merge candidates). The two views serve different mental models: "manage my clusters" vs. "who are these people?" Both have value, but the cluster management enhancements should come first since they are lower-risk.

### 3.2 Dynamic Threshold Slider vs. Pre-Computed Groupings

**Tension**: Doc 04 proposes a similarity threshold slider with presets (0.70-0.95). Doc 05 argues the slider is a leaky abstraction that exposes implementation details and causes unpredictable "phase transitions" in groupings.

**Resolution**: **Semantic presets for Phase 1-2, defer numeric slider to Phase 3.** Replace the numeric threshold with semantic labels: "High confidence" / "Standard" / "Exploratory" (mapping to internal thresholds 0.90/0.85/0.75). Pre-compute clusters at these three levels during the background job. The Clusters view already has a confidence threshold filter (presets + custom); the unlabeled groups view should use the simpler semantic version.

**Rationale**: Doc 05 is right that cosine similarity in 512-dim space is unintuitive to users. Doc 04's preset buttons are a reasonable middle ground, but even numbered presets (85%, 80%, 75%) carry implementation semantics. Semantic labels abstract this away while still giving users control. Pre-computing at 3 levels (rather than arbitrary values) keeps the backend simple and avoids the O(N^2) real-time problem.

### 3.3 HDBSCAN vs. Agglomerative Clustering for Threshold Control

**Tension**: Doc 03 recommends Agglomerative clustering as the default because it provides exact distance threshold control (deterministic). The existing system uses HDBSCAN which does not directly map to a user-specified threshold.

**Resolution**: **Use HDBSCAN for background batch clustering (Phase 1), add Agglomerative as an option for on-demand refinement (Phase 3).** HDBSCAN is already battle-tested in production via `DualModeClusterer`. Agglomerative clustering adds value when exact threshold mapping is needed, but this is only relevant if the dynamic threshold feature is implemented.

**Rationale**: Switching the default clustering algorithm introduces risk. HDBSCAN's density-based approach handles variable-quality face collections well. Agglomerative's strength (exact threshold) only matters when users adjust thresholds, which is deferred to Phase 3.

### 3.4 `is_assigned` Sentinel Value vs. Current Absent-Key Pattern

**Tension**: Doc 03 strongly recommends adding an `is_assigned` boolean payload field to Qdrant to eliminate the O(N) full-scan in `get_unlabeled_faces_with_embeddings()`. Doc 05 notes that `IsEmptyCondition` already exists in `clusterer.py` (line 77-78) and is not used in the retrieval function.

**Resolution**: **Fix the immediate bug first (use `IsEmptyCondition` consistently), then add `is_assigned` sentinel as a Phase 1 optimization.** The `IsEmptyCondition` filter is a quick fix that eliminates the Python-side filtering problem. The `is_assigned` boolean is a stronger optimization (indexed boolean lookup vs. empty-key check) but requires a data migration. Both should happen, sequentially.

### 3.5 New Database Tables vs. Reuse Existing `cluster_id`

**Tension**: Doc 02 proposes optional `UnknownPersonDiscovery` and `ClusterMetadata` tables in Phase 2. Doc 05 argues against creating parallel data structures and recommends reusing `FaceInstance.cluster_id`.

**Resolution**: **Reuse `FaceInstance.cluster_id` for Phase 0-1. Add `ClusterMetadata` (not `UnknownPersonDiscovery`) in Phase 2 only if pre-computed confidence caching proves necessary for performance.** The discovery job metadata can be stored in Redis (ephemeral) rather than PostgreSQL.

### 3.6 Where to Place the New UI: Suggestions Page vs. Standalone

**Tension**: Doc 04 proposes tabs within the Suggestions page. The user's original prompt mentions a "component that allows to flip between the two views" on the Suggestions page.

**Resolution**: **Tabbed interface within Suggestions page, as originally specified.** The user's requirement is clear. The tabs approach (shadcn/ui `Tabs` component) is well-supported by the existing UI framework and keeps related face management functionality together. The existing Clusters view remains as a separate, more technical cluster management tool.

---

## 4. Critical Technical Decisions (Ranked by Impact)

### Decision 1: Clustering Algorithm and Execution Model

**Impact**: CRITICAL -- determines performance ceiling, memory requirements, and UX responsiveness.

| Option | Pros | Cons | Recommendation |
|---|---|---|---|
| HDBSCAN background batch (current) | Battle-tested, handles variable density | Not threshold-adjustable, O(N^2) memory | **Phase 0-2 default** |
| Agglomerative on-demand | Exact threshold mapping, deterministic | O(N^2) memory, seconds-to-minutes latency | Phase 3 option |
| Pre-compute at 3 semantic levels | Fast API, no real-time compute | Storage x3, staleness risk | **Phase 2 enhancement** |

**Decision**: Use HDBSCAN for batch clustering (existing infrastructure). Pre-compute at 3 confidence levels in Phase 2. Defer on-demand Agglomerative to Phase 3.

### Decision 2: Qdrant Unlabeled Face Retrieval Strategy

**Impact**: HIGH -- current O(N) full-scan is the primary performance bottleneck for any approach involving unassigned faces.

| Option | Effort | Performance Gain | Risk |
|---|---|---|---|
| Fix `get_unlabeled_faces_with_embeddings` to use `IsEmptyCondition` | Low (code exists) | Moderate (server-side filter) | Very low |
| Add `is_assigned` boolean payload + index | Medium (migration) | High (indexed boolean lookup) | Low |
| Separate `unassigned_faces` collection | High | Highest | High (sync complexity) |

**Decision**: Apply both fixes sequentially. Step 1 (Phase 0): use `IsEmptyCondition` consistently. Step 2 (Phase 1): add `is_assigned` boolean sentinel with payload index.

### Decision 3: New View Architecture

**Impact**: HIGH -- determines UX complexity, code duplication, and maintenance burden.

| Option | Risk | Effort | User Value |
|---|---|---|---|
| Enhance Clusters view only | Low | Medium | Medium |
| New tabbed sub-view in Suggestions | Medium | Medium-High | High |
| Both (phased) | Low-Medium | Medium (split across phases) | Highest |

**Decision**: Phase 0-1: enhance Clusters view. Phase 2: add tabbed sub-view in Suggestions. The Clusters view enhancements are prerequisites (backend API, confidence scoring) that the Suggestions sub-view will consume.

### Decision 4: Threshold Exposure to Users

**Impact**: MEDIUM -- determines whether the feature feels intuitive or confusing.

| Option | Complexity | User Understanding | Recommendation |
|---|---|---|---|
| Numeric slider (0.50-1.00) | High (non-linear behavior) | Low | Defer to Phase 3 |
| Numeric presets (85%, 80%, etc.) | Medium | Medium | Available in Clusters view |
| Semantic presets ("High/Standard/Exploratory") | Low | High | **Phase 2 default** |
| No user control (system decides) | Lowest | Depends on accuracy | Phase 0-1 approach |

**Decision**: Phase 0-1: no user threshold control (present system's best groupings). Phase 2: semantic presets. Phase 3: numeric slider for power users.

### Decision 5: Singleton Face Handling

**Impact**: MEDIUM -- affects 40-60% of unique faces in typical collections.

| Strategy | Coverage | Noise Risk | Effort |
|---|---|---|---|
| Ignore singletons (min_cluster_size >= 3) | ~50% of persons | None | Zero |
| Quality-gated singletons (quality > 0.8) | ~65% of persons | Low | Low |
| "Show singletons" toggle in UI | ~80% of persons | Medium | Low |
| NN-based singleton grouping (find similar) | ~90% of persons | Low | Medium |

**Decision**: Phase 1: ignore singletons (current behavior). Phase 2: add "Show singletons" toggle with quality gate. Phase 3: NN-based singleton grouping via existing "Find More" pattern.

### Decision 6: Cluster-to-Person Promotion Model

**Impact**: MEDIUM -- determines data model complexity and suggestion system integration.

| Option | Complexity | Integration | Recommendation |
|---|---|---|---|
| Extend `FaceSuggestion` with null `suggested_person_id` | Low | Tight (reuses suggestion UX) | Considered but rejected (muddies existing model) |
| New `ClusterPromotion` model | Medium | Loose (separate workflow) | Deferred to Phase 3 |
| Reuse existing `label_cluster` endpoint | Lowest | Direct | **Phase 0-1** |
| New `/unknown-persons/candidates/{id}/accept` endpoint | Low-Medium | Clean API surface | **Phase 2** |

**Decision**: Phase 0-1: reuse existing `POST /faces/clusters/{cluster_id}/label`. Phase 2: add dedicated `/unknown-persons/candidates/{id}/accept` that delegates to the same service layer but provides a cleaner API for the new UI.

---

## 5. Consolidated Architecture Recommendation

### System Architecture

```
                 EXISTING (unchanged)                     NEW (phased)
                 ====================                     ============

  Photo Upload --> Face Detection --> Qdrant faces
                                        |
                                        v
                 DualModeClusterer (background job)
                   |                    |
                   v                    v
         Phase 1: Supervised    Phase 2: Unsupervised
         (assign to known)      (HDBSCAN clustering)
                                        |
                                        v
                              FaceInstance.cluster_id
                                        |
                    +-------------------+-------------------+
                    |                                       |
                    v                                       v
           [PHASE 0-1: Enhanced]                  [PHASE 2: New View]
           Clusters View                          Unlabeled Groups Tab
           - Confidence scoring                   in Suggestions page
           - "Create Person" button               - Pre-computed groups
           - Sort by confidence/size              - Semantic threshold
           - Dismiss cluster                      - Create/Dismiss/Merge
                    |                                       |
                    v                                       v
           POST /clusters/{id}/label              POST /unknown-persons/
              (existing)                          candidates/{id}/accept
                    |                                       |
                    +-------------------+-------------------+
                                        |
                                        v
                              Shared Service Layer:
                              ClusterLabelingService
                              - Create Person
                              - Assign faces
                              - Create prototypes
                              - Sync Qdrant
                              - Trigger find-more
```

### Backend Components (New)

| Component | Phase | Description |
|---|---|---|
| `ClusterLabelingService` | 0 | Extracted from `faces.py` route handler. Shared by Clusters and Unknown Persons views. |
| `UnknownPersonsRouter` | 2 | New route file: `api/routes/unknown_persons.py` (~6 endpoints). |
| `unknown_person_schemas.py` | 2 | Pydantic schemas for candidate groups. |
| `discover_unknown_persons_job` | 2 | Background job: cluster + score + store metadata. |
| `ClusterMetadata` table | 2 | Pre-computed cluster confidence, representative face, dismissal state. |
| `is_assigned` Qdrant sentinel | 1 | Boolean payload field + index for fast unassigned-face filtering. |

### Frontend Components (New)

| Component | Phase | Estimated Size |
|---|---|---|
| `SimilarityThresholdControl.svelte` | 1 | ~80-100 lines |
| `UnlabeledGroupsView.svelte` | 2 | ~250-300 lines |
| `UnlabeledGroupCard.svelte` | 2 | ~150-180 lines |
| `CreatePersonFromGroupDialog.svelte` | 2 | ~150-200 lines |
| `MergeGroupsDialog.svelte` | 2 | ~100-120 lines |
| Modifications to `+page.svelte` (Suggestions) | 2 | ~30-40 additional lines |

---

## 6. Implementation Roadmap

### Phase 0: Foundation Fixes (1-2 days)

**Goal**: Fix known issues and extract shared services. No new features visible to users.

| Task | Type | Effort | Details |
|---|---|---|---|
| Fix `get_unlabeled_faces_with_embeddings()` | Bug fix | Small | Use `IsEmptyCondition` (already exists in `clusterer.py` line 77-78) instead of Python-side filtering |
| Extract `ClusterLabelingService` | Refactor | Small | Move cluster-to-person logic from `faces.py` route handler into a shared service class |
| Add batch embedding retrieval | Optimization | Small | Replace 1-at-a-time Qdrant calls in `dual_clusterer.py` with batch `retrieve()` calls (100 per batch) |

**Deliverables**: Faster unlabeled face retrieval, reusable cluster labeling service, improved clustering performance.

### Phase 1: Enhanced Clusters View (1-2 weeks)

**Goal**: Make the existing Clusters view effective for unknown person discovery. This phase delivers the MVP.

| Task | Type | Effort | Details |
|---|---|---|---|
| Add `is_assigned` sentinel to Qdrant faces | Backend | Medium | Add boolean payload field, create index, data migration for existing faces |
| Add cluster confidence to `list_clusters` response | Backend | Small | Pre-compute or sample-based confidence already exists via `calculate_cluster_confidence()` |
| Add "Create Person" inline action to ClusterCard | Frontend | Small | Button that opens `LabelClusterModal` or inline name input |
| Add "Dismiss Cluster" action | Backend + Frontend | Small | Set `cluster_id = '-1'` or new dismissed marker |
| Add sort by confidence, size, quality | Frontend | Small | Extend existing sort controls in Clusters view |
| Extract `SimilarityThresholdControl` component | Frontend | Small | Reusable preset+custom threshold control from Clusters view pattern |
| Add `GET /unknown-persons/stats` endpoint | Backend | Small | Count unassigned, clustered, noise, unclustered faces |
| Update Clusters view confidence threshold to use presets | Frontend | Small | Standardize threshold UI with the new shared component |

**Deliverables**: Users can browse clusters sorted by confidence, create persons with one click, dismiss noise clusters. Stats endpoint provides dashboard-ready data.

### Phase 2: Unlabeled Groups Tab in Suggestions (2-3 weeks)

**Goal**: Add the discovery-oriented "Unlabeled Groups" tab within the Suggestions page.

| Task | Type | Effort | Details |
|---|---|---|---|
| Create `UnlabeledGroupsView.svelte` | Frontend | Medium | Container component with state management, data fetching, group orchestration |
| Create `UnlabeledGroupCard.svelte` | Frontend | Small | Presentation card: dashed border, thumbnail grid, actions |
| Create `CreatePersonFromGroupDialog.svelte` | Frontend | Medium | Name input, face selection grid, create+assign flow |
| Create `MergeGroupsDialog.svelte` | Frontend | Small | Confirmation dialog for group merging |
| Add Tabs wrapper to Suggestions `+page.svelte` | Frontend | Small | shadcn/ui Tabs with "Known Persons" and "Unlabeled Groups" |
| Create `UnknownPersonsRouter` | Backend | Medium | 6 endpoints: discover, list candidates, detail, accept, dismiss, stats |
| Create `unknown_person_schemas.py` | Backend | Small | CamelCase Pydantic models for request/response |
| Create `discover_unknown_persons_job` | Backend | Medium | Background job: cluster at 3 semantic levels, compute confidence, store metadata |
| Add `ClusterMetadata` table | Backend | Small | Pre-computed cluster confidence, representative face, dismissal tracking |
| Add semantic threshold presets to API | Backend | Small | Map "high/standard/exploratory" to internal thresholds |
| Update API contract | Docs | Small | Add new endpoints to `docs/api-contract.md` |
| Regenerate frontend types | Frontend | Small | `npm run gen:api` after backend changes |

**Deliverables**: Full tabbed interface, pre-computed groupings at 3 confidence levels, create/dismiss/merge workflows, background discovery job with SSE progress.

### Phase 3: Advanced Features (Future, scope TBD)

**Goal**: Add dynamic threshold control, singleton handling, and ML enhancements. Only implement if Phase 2 user feedback warrants it.

| Task | Type | Effort | Details |
|---|---|---|---|
| On-demand Agglomerative re-clustering | Backend | Medium | Compute groupings at arbitrary user-specified threshold |
| Numeric threshold slider | Frontend | Small | Expose custom threshold input with debounced re-fetch |
| NN-based singleton grouping | Backend | Medium | Use Qdrant similarity search from each singleton to find potential matches |
| Cross-cluster merge suggestions | Backend | Medium | Compute pairwise cluster centroid similarities, suggest merges |
| Incremental clustering | Backend | Large | Add new faces to existing clusters without full re-clustering |
| Temporal-aware clustering | Backend | Medium | Use `taken_at` metadata as a soft signal for same-person detection |
| Active learning from user feedback | Backend | Large | Track accept/dismiss patterns to calibrate thresholds |
| Group stability indicators | Frontend | Small | Visual indicator for groups likely to change with threshold adjustment |

---

## 7. Risk Register

Top 10 risks from the Devil's Advocate analysis (Doc 05) cross-referenced with technical findings from all streams, ranked by severity and likelihood.

| # | Risk | Severity | Likelihood | Phase Affected | Mitigation |
|---|---|---|---|---|---|
| 1 | **O(N^2) memory for real-time clustering** -- 10K faces = ~800MB distance matrix, 50K = ~10GB | Critical | High (if Phase 3 attempted) | Phase 3 | Pre-compute in background job. Cap `max_faces` at 10K. Use sampling for larger sets. Defer on-demand clustering to Phase 3 only if needed. |
| 2 | **Qdrant full-scan in `get_unlabeled_faces_with_embeddings`** -- scrolls ALL faces, filters in Python | High | Certain (exists today) | Phase 0 | Fix in Phase 0: use `IsEmptyCondition` (already in codebase). Add `is_assigned` sentinel in Phase 1. |
| 3 | **Singleton faces invisible** -- 40-60% of unique faces never cluster (min_cluster_size >= 3) | High | Certain | Phase 1-2 | Quality-gated singleton section (Phase 2). NN-based singleton matching (Phase 3). Document limitation to users. |
| 4 | **DB-Qdrant consistency gaps** -- non-blocking Qdrant sync after DB commit means faces can reappear as "unlabeled" | High | High | Phase 1-2 | Add reconciliation check before displaying unlabeled groups. Consider making Qdrant sync blocking for assignment operations. Add periodic reconciliation job. |
| 5 | **Threshold slider causes unpredictable phase transitions** -- small changes cause catastrophic group reorganization | Medium | High (if slider built) | Phase 3 | Replace with semantic presets (Phase 2). Only expose numeric slider in Phase 3 with stability warnings. Pre-compute at fixed levels to ensure stability. |
| 6 | **Duplicate/conflicting views** -- Clusters view and Unlabeled Groups show overlapping data, confusing users | Medium | High | Phase 2 | Clear UX differentiation: Clusters = technical management tool, Unlabeled Groups = discovery workflow. Consider linking between views. Document purpose of each. |
| 7 | **Temporal face variation splits same person** -- same person photographed over 10 years appears as 2-3 groups | Medium | Medium | Phase 2-3 | Temporal-aware clustering (Phase 3). Cross-cluster merge suggestions. Display temporal spread metadata to help users identify merges. |
| 8 | **Privacy/GDPR compliance** -- automated face grouping constitutes biometric data processing | Medium | Medium (jurisdiction-dependent) | All phases | Self-hosted mitigates some risk. Add opt-in toggle for face grouping feature. Document biometric data handling. Consider "right to be forgotten" for dismissed faces. |
| 9 | **Background job race conditions** -- clustering job runs while user is actively reviewing groups | Medium | Medium | Phase 2 | Add cluster versioning. Optimistic locking on group operations. Display "results may have changed" warning if version mismatch. |
| 10 | **Decision fatigue** -- 500+ unknown groups overwhelms users | Low | High | Phase 2 | Default to high-confidence groups only. Sort by size and confidence. Pagination. Consider triage queue UX (Doc 05, Alternative 7.5) for future enhancement. |

---

## 8. Open Questions Requiring User Input

These decisions cannot be resolved by technical analysis alone and require product/user input.

### 8.1 Naming UX (Affects Phase 1-2)

**Question**: When creating a person from a cluster, should we require a name immediately or allow "Unknown Person #N" with deferred naming?

**Context**: Apple Photos uses unnamed groups; Google Photos prompts for names; Immich requires naming. The existing `POST /faces/clusters/{cluster_id}/label` endpoint requires a name (non-empty string).

**Options**:
- A: Require name immediately (current behavior, cleanest data model)
- B: Allow placeholder names like "Unknown Person 1" (faster workflow but creates naming debt)
- C: Allow unnamed persons initially, prompt for names later (most flexible but adds state management)

**Impact**: Determines the `AcceptUnknownPersonRequest` schema and the Create Person dialog design.

### 8.2 Minimum Cluster Size for Display (Affects Phase 1-2)

**Question**: What should the minimum face count be for surfacing a cluster as a "candidate person"?

**Context**: HDBSCAN's `min_cluster_size` is already 3-5. Setting the display minimum lower captures more persons but shows more noise. Setting it higher means cleaner groups but misses persons with few photos.

**Options**:
- A: 2 faces minimum (catches most real persons, more noise)
- B: 3 faces minimum (current HDBSCAN default, balanced)
- C: 5 faces minimum (very clean, misses people with few photos)
- D: User-configurable (adds UI complexity)

### 8.3 Confidence Visibility (Affects Phase 1-2)

**Question**: Should cluster cohesion scores be shown to users?

**Context**: Consumer apps (Google Photos, Apple Photos) hide confidence. Power users (the likely audience for this self-hosted tool) may appreciate it. The Clusters view already shows confidence values.

**Options**:
- A: Show confidence prominently (badge on each group)
- B: Show on hover/tooltip only (available but not prominent)
- C: Hide entirely (simplest UX, rely on sort order)

### 8.4 Cluster Persistence Across Runs (Affects Phase 1)

**Question**: Should clusters be stable identifiers (surviving re-clustering) or ephemeral (recalculated each run)?

**Context**: Current implementation assigns new `cluster_id` values each run, which means a user who dismissed "Group A" may see the same faces reappear as "Group B" after re-clustering. Stable IDs require tracking cluster membership hashes.

**Options**:
- A: Ephemeral (simplest, current behavior, but dismissed groups reappear)
- B: Stable via membership hash (faces in cluster determine ID, dismissed state persists)
- C: Versioned (cluster_version counter, UI shows "updated since last visit")

### 8.5 Post-Creation Behavior (Affects Phase 1-2)

**Question**: After creating a person from a cluster, should the system automatically trigger "Find More" to discover additional faces for the new person?

**Context**: The existing suggestion system has `auto_find_more` capability. Auto-triggering creates a virtuous cycle (more faces -> better prototypes -> more matches) but also generates background job load and may surface false positives immediately after creation.

**Options**:
- A: Always auto-trigger find-more (maximize recall)
- B: Offer as a toggle, default on (user control)
- C: Never auto-trigger (user manually triggers when ready)

### 8.6 Relationship Between Clusters View and Unlabeled Groups (Affects Phase 2)

**Question**: When the tabbed Unlabeled Groups view is built (Phase 2), should the existing Clusters view be deprecated, maintained as-is, or evolved into a "power user" tool?

**Context**: Doc 05 argues these are fundamentally the same data with different UX. Maintaining both creates documentation and maintenance burden. However, the Clusters view has capabilities (split cluster, recluster) that the Unlabeled Groups view does not need.

**Options**:
- A: Keep both views indefinitely (different audiences, different purposes)
- B: Deprecate Clusters view after Phase 2 launch (reduce surface area)
- C: Evolve Clusters into "Advanced" mode accessible from Unlabeled Groups (single entry point)

---

## Appendix A: Document Cross-Reference Matrix

| Topic | Doc 01 (General) | Doc 02 (Backend) | Doc 03 (DB/Qdrant) | Doc 04 (UI/UX) | Doc 05 (Devil's Advocate) |
|---|---|---|---|---|---|
| Gap analysis | Sections 2-3 | Section 1 | Section 1 | Section 1 | Section 1.1 |
| Clustering approach | Section 4, Decision 1 | Section 5.1-5.3 | Section 4 (Options A-D) | -- | Section 3.1 |
| Qdrant limitations | Section 2.4 | Section 2.3 | Section 3.2 | -- | Section 3.2 |
| API design | Section 3.4 | Section 4.2 (full spec) | -- | Section 7.4 | Section 5.4 |
| UI design | -- | -- | -- | Sections 4-7 (full spec) | Sections 4.1-4.4 |
| Singleton problem | Section 4, Decision 3 | -- | -- | Section 4.6 | Section 2.1 |
| Threshold strategy | Section 4, Decision 2 | Section 6.3 | Section 4 | Section 4.2 | Section 1.2 |
| Risk assessment | -- | Section 8.3 | Section 5 | -- | Sections 2-3 (comprehensive) |
| Alternative approaches | Section 5 | Section 5.1 | Section 6 | -- | Section 7 (5 alternatives) |
| Industry patterns | Section 3 | -- | -- | Section 8 | -- |
| Privacy/GDPR | Section 5, Q8 | -- | -- | -- | Section 6 |

## Appendix B: Threshold Reference (All Sources)

| Threshold | Current Value | Used By | Phase | Notes |
|---|---|---|---|---|
| `person_match_threshold` | 0.7 | DualModeClusterer Phase 1 | Existing | Supervised assignment to known persons |
| `face_auto_assign_threshold` | Configurable | FaceAssigner | Existing | Auto-assign without review |
| `face_suggestion_threshold` | Configurable | FaceAssigner | Existing | Create suggestion for review |
| `unknown_min_cluster_size` | 3 | DualModeClusterer Phase 2 | Existing | HDBSCAN min cluster size |
| `min_cluster_size` | 5 | FaceClusterer | Existing | Standalone HDBSCAN |
| `quality_threshold` | 0.5 | FaceClusterer | Existing | Min quality for inclusion |
| `unknown_eps` | 0.5 | DualModeClusterer | Existing | DBSCAN/Agglomerative distance |
| Semantic "High" | 0.90 (proposed) | Unknown persons discovery | Phase 2 | Highest confidence groups only |
| Semantic "Standard" | 0.85 (proposed) | Unknown persons discovery | Phase 2 | Default confidence level |
| Semantic "Exploratory" | 0.75 (proposed) | Unknown persons discovery | Phase 2 | Lower confidence, more groups |
| `cluster_merge_threshold` | 0.55-0.65 (proposed) | Cross-cluster merge detection | Phase 3 | Suggest merging similar clusters |

## Appendix C: Performance Budget

| Operation | Target | Strategy | Phase |
|---|---|---|---|
| `GET /unknown-persons/candidates` (pre-computed) | < 200ms | SQL query + cached confidence | Phase 2 |
| `GET /faces/clusters` (with confidence) | < 500ms | Sample-based confidence (max 20 faces) | Phase 1 |
| `GET /unknown-persons/candidates/{id}` | < 500ms | Direct cluster query | Phase 2 |
| `POST /unknown-persons/discover` | Immediate (async) | Returns job_id, work in background | Phase 2 |
| `POST /candidates/{id}/accept` | < 1s | Transaction: create person + assign faces | Phase 2 |
| Background discovery job (5K faces) | < 60s | HDBSCAN + confidence computation | Phase 2 |
| Background discovery job (10K faces) | < 120s | With sampling for confidence | Phase 2 |
| Qdrant unlabeled face retrieval (with `is_assigned` index) | < 2s for 10K faces | Indexed boolean filter | Phase 1 |

---

*This synthesis represents the consolidated recommendation from 5 parallel research streams. All technical decisions should be reviewed against this document before implementation begins. Open questions (Section 8) should be resolved with the product owner before Phase 1 work commences.*
