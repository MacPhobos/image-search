# Executive Synthesis: Clustering & Face Suggestions System Analysis

## Research Team
- **UI/UX Researcher**: Frontend usability analysis of 18 Svelte components, 7 route pages, and the 1,652-line API client
- **Backend Researcher**: Service architecture analysis of FastAPI routes, data models, queue jobs, vector clients, and configuration
- **Clustering Researcher**: Algorithm & ML deep-dive into HDBSCAN, centroid computation, training pipelines, and suggestion generation
- **Devil's Advocate**: Cross-cutting concerns, failure mode mapping, and adversarial scalability analysis

## Date: 2026-02-06

---

## Critical Findings (Cross-Referenced)

### CRITICAL -- Must Fix (Data Corruption / Feature Broken)

#### C1. Qdrant person_id Not Updated on Suggestion Accept -- Silent Data Desynchronization

**Description**: When a user accepts a face suggestion via the API, the `face.person_id` is updated in PostgreSQL but the corresponding `person_id` payload in Qdrant is never updated. Over time, Qdrant's person_id data diverges completely from the database, causing duplicate suggestions, incorrect face counts, and broken exclude-self filters in centroid searches.

**Researchers**: Devil's Advocate (Bomb #1, primary discovery), Backend (identified missing locking but not sync gap)

**Location**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`, lines 526-533

**Evidence**: Grep for `update_person_ids` in `face_suggestions.py` returns zero matches. The `assigner.py:237` correctly calls `qdrant.update_person_ids()`, proving the pattern is known but was not applied to the suggestion acceptance path.

**Impact**: Every accepted suggestion leaves Qdrant in an inconsistent state. Cumulative, irreversible without full Qdrant re-sync. Affects every user who accepts suggestions.

**Fix**: Add a single `qdrant.update_person_ids([face.qdrant_point_id], suggestion.suggested_person_id)` call after the DB commit in the accept_suggestion endpoint.

---

#### C2. Qdrant Payload Key Mismatch -- find-more and Centroid Suggestions Silently Return Zero Results

**Description**: The Qdrant `faces` collection stores face IDs under the key `face_instance_id` (set in `face_qdrant.py:154`). Three job functions look for the key `face_id` instead. Since the key does not exist, `payload.get("face_id")` returns `None`, the result is skipped, and zero suggestions are created.

**Researchers**: Devil's Advocate (Bomb #2, primary discovery), Backend (identified as "Payload Field Missing" at face_jobs.py:1438)

**Locations**:
- `queue/face_jobs.py:1438` -- `find_more_suggestions_job`
- `queue/face_jobs.py:1665` -- `propagate_person_label_multiproto_job`
- `queue/face_jobs.py:2062` -- `find_more_centroid_suggestions_job`

**Impact**: The multi-prototype suggestion, centroid suggestion, and find-more features are silently broken. They execute without errors but produce no suggestions. Post-training suggestion queuing (which calls these jobs) produces nothing.

**Fix**: Replace `result.payload.get("face_id")` with `result.payload.get("face_instance_id")` in all three locations.

---

#### C3. Duplicated Centroid Computation Logic -- Divergent Algorithm Behavior

**Description**: Centroid computation exists in two separate locations with different algorithms. The formal `centroid_service.py` applies robust outlier trimming and versioning. The inline implementation in `find_more_centroid_suggestions_job` (face_jobs.py:1944-2003) uses simple mean without trimming (`trim_outliers=False`). Bug fixes or algorithm improvements in one location do not propagate to the other.

**Researchers**: Backend (Critical finding), Clustering (Priority 1 recommendation), Devil's Advocate (identified as part of Bomb #7 -- two centroid systems)

**Locations**:
- `services/centroid_service.py:258` -- `compute_centroids_for_person()` (full-featured)
- `queue/face_jobs.py:1944-2003` -- inline centroid computation (simplified)
- `faces/dual_clusterer.py:150-170` -- simple mean centroids in clustering (no trimming, no versioning)

**Impact**: The same person can have different centroid vectors depending on which code path computed them, leading to inconsistent matching behavior. The inline job version computes lower-quality centroids.

**Fix**: Extract centroid computation from `find_more_centroid_suggestions_job` and call `centroid_service.compute_centroids_for_person()` instead. Unify the dual_clusterer to use the same robust centroid computation.

---

#### C4. Missing Optimistic Locking on Face Assignment -- Concurrent Modification Risk

**Description**: The face assignment endpoints read a `FaceInstance`, set `person_id`, then commit without `SELECT FOR UPDATE` or version checking. Two concurrent suggestion acceptances for the same face could cause inconsistent state.

**Researchers**: Backend (Critical finding), Devil's Advocate (confirmed via TOCTOU analysis)

**Locations**:
- `api/routes/faces.py:1616` -- `assign_face`
- `api/routes/face_suggestions.py:527` -- `accept_suggestion`

**Impact**: Under concurrent use (multiple users reviewing suggestions simultaneously), the same face could be assigned to different persons, with the last write winning and no conflict detection.

**Fix**: Add `SELECT FOR UPDATE` or an optimistic locking version column to `FaceInstance`.

---

#### C5. Hardcoded Collection Name `"faces"` Bypasses Test Safety Guards

**Description**: The `face_qdrant.py` module provides configurable `_get_face_collection_name()` with a test safety guard, but at least 12 call sites bypass this by hardcoding the string `"faces"`.

**Researchers**: Backend (Critical: hardcoded in `clusterer.py:88`), Devil's Advocate (Bomb #4: expanded to 12 locations across 6 files)

**Locations**:
- `assigner.py:274, 344`
- `clusterer.py:88`
- `dual_clusterer.py:352`
- `trainer.py:416`
- `face_jobs.py:1027, 1048`

**Impact**: Integration tests running against a real Qdrant instance bypass the safety guard and could mutate or delete production face embeddings.

**Fix**: Replace all hardcoded `"faces"` with calls to `_get_face_collection_name()`.

---

### HIGH -- Should Fix (Significant UX/Performance Impact)

#### H1. N+1 Qdrant Embedding Retrieval -- 7+ Call Sites

**Description**: Embedding retrieval from Qdrant uses individual HTTP requests per face in loops across 7+ locations. The Qdrant `retrieve()` API supports batch IDs but is never used with batches.

**Researchers**: Backend (Critical finding: 7 call sites), Clustering (Priority 2 recommendation), Devil's Advocate (Bomb #5: performance analysis)

**Locations**:
- `centroid_service.py:206`
- `face_jobs.py:1423, 1629, 1929`
- `faces.py:1902`
- `face_clustering_service.py:90`
- `dual_clusterer.py` (via `_get_face_embedding()`)

**Impact**: For a person with 500 faces, centroid computation makes 500 sequential HTTP requests. At 20ms latency per request, this is 10 seconds of pure network overhead. This is the dominant performance bottleneck for all face operations.

**Fix**: Add `get_embeddings_batch(point_ids: list[UUID])` to `FaceQdrantClient` using Qdrant's batch `retrieve()` API.

---

#### H2. Two Parallel Centroid Systems Operating Simultaneously

**Description**: The codebase contains two separate centroid storage and computation systems that can produce conflicting results for the same person.

**Researchers**: Devil's Advocate (Bomb #7, primary discovery), Clustering (Section 6.5, item 2), Backend (indirectly via configuration inconsistency)

**Systems**:
- **System A (Legacy)**: `assigner.py:286-390` stores centroids in the `faces` Qdrant collection using `PersonPrototype` with `role=CENTROID` and `is_prototype=True` payload. Simple mean, no versioning.
- **System B (New)**: `centroid_service.py` + `centroid_qdrant.py` stores centroids in dedicated `person_centroids` Qdrant collection using `PersonCentroid` table. Robust computation with outlier trimming, versioning, staleness detection.

**Impact**: System A centroids (in faces collection) are searched by the assigner's prototype matching. System B centroids (in person_centroids collection) are used by centroid-based suggestions. Different computation methods produce different vectors, causing inconsistent matching.

**Fix**: Deprecate System A (`assigner.compute_person_centroids()`) in favor of the formal `CentroidService` pipeline.

---

#### H3. O(n^2) Memory for HDBSCAN Distance Matrix

**Description**: The dual-mode clusterer builds a full N*N cosine distance matrix for HDBSCAN clustering. With the default `max_faces=50000`, this requires approximately 20GB of contiguous memory.

**Researchers**: Backend (Moderate finding), Clustering (Section 1.2), Devil's Advocate (C1: detailed memory analysis)

**Location**: `dual_clusterer.py:379` -- `cosine_distances(embeddings)`

**Memory estimates**:
| Faces | Memory |
|-------|--------|
| 10,000 | 800 MB |
| 50,000 | 20 GB |
| 100,000 | 80 GB |

**Impact**: Clustering operations with large face counts will OOM-kill the worker process on typical deployments.

**Fix**: Add memory-bounded clustering (incremental HDBSCAN or switch to approximate methods for large N). Alternatively, reduce `max_faces` default or add a memory check before computing the matrix.

---

#### H4. Centroid Deprecation Gap -- "Deprecate Before Create" Can Leave Zero Active Centroids

**Description**: `compute_centroids_for_person` deprecates ALL existing active centroids BEFORE creating the new one. If any subsequent step fails (numpy computation, DB flush, Qdrant upsert), the person is left with zero active centroids.

**Researchers**: Devil's Advocate (Bomb #3, primary discovery), Backend (Moderate: Qdrant/DB consistency gap)

**Location**: `centroid_service.py:325-394`

**Impact**: A Qdrant outage during centroid computation leaves persons permanently without centroids until manually triggered again.

**Fix**: Create the new centroid first (with BUILDING status), then deprecate old centroids, then activate the new one. Or wrap in a transaction that rolls back deprecation on failure.

---

#### H5. Terminology Inconsistency -- Largest Usability Barrier

**Description**: The same concepts have 3-5 different labels depending on the page, creating significant user confusion.

**Researchers**: UI/UX (Critical finding #1), Devil's Advocate (Section B: mental model mismatches)

**Key inconsistencies**:
| Concept | Terms Used |
|---------|-----------|
| Unidentified face groups | "Unknown Faces", "Clusters", "Needs Names", "Unidentified" |
| Low-quality ungrouped faces | "Noise", "Review", "Unknown Faces" |
| Matching confidence | "X% match", "X% confidence", "Score", "Similarity" |
| Face suggestion | "Suggestion", "Centroid Suggestion", "Match" |

**Impact**: Users must learn multiple terms for the same concept across different pages. This is the single largest barrier to usability for face management workflows.

**Fix**: Establish a glossary and apply consistently across all components: `UnifiedPersonCard.svelte`, `ClusterCard.svelte`, `CentroidResultsDialog.svelte`, `ComputeCentroidsDialog.svelte`, clusters `+page.svelte`, people `+page.svelte`.

---

#### H6. Dual-Store Architecture Without Reconciliation

**Description**: Face data is maintained across PostgreSQL and two Qdrant collections with no transactional guarantees, no CDC, no periodic reconciliation, and no checksums to detect divergence.

**Researchers**: Devil's Advocate (Section A1, primary analysis), Backend (dual-storage consistency concerns), Clustering (Section 6.5)

**Impact**: Every write path touching both systems is a potential consistency divergence point. The only repair mechanism is a full reset-and-rebuild requiring downtime.

**Fix**: Implement a periodic reconciliation job that compares person_id/cluster_id between PostgreSQL and Qdrant, flagging and optionally repairing divergences.

---

### MODERATE -- Improvement Opportunities

#### M1. Modal Stacking Reaches 4+ Levels Deep

**Researchers**: UI/UX (Finding #3)

Users can navigate through ComputeCentroidsDialog > CentroidResultsDialog > SuggestionDetailModal > PersonAssignmentModal, creating disorientation and fragile z-index management (`z-[60]`).

**Fix**: Convert CentroidResults and SuggestionDetail into full pages or slide-over panels.

---

#### M2. Native `confirm()` Dialogs in 7 Locations

**Researchers**: UI/UX (Finding #4)

All destructive operations use native browser `confirm()` instead of shadcn/ui Dialog, breaking visual consistency and preventing contextual information display.

**Files**: 7 files listed in UI/UX report Section 3.4.

---

#### M3. "Find More" Feature Hidden Behind Unlabeled Icon

**Researchers**: UI/UX (Finding #5)

The centroid-based "Find More" feature is triggered by an unlabeled icon button with tooltip-only description. Hidden entirely when prerequisite (2+ labeled faces) is not met.

---

#### M4. Centroid Reject is Cosmetic Only -- Not Persisted

**Researchers**: UI/UX (Finding in Section 2.4, item 2)

`CentroidResultsDialog.svelte:221-226`: The reject handler shows a toast "Suggestion removed from list" but does not persist to backend. Re-running centroids surfaces the same faces.

---

#### M5. Noise Faces Are Display-Only Dead Ends

**Researchers**: UI/UX (Finding #2)

Noise faces appear on the People page with a misleading "Review" badge but are not clickable and have no actionable path forward.

---

#### M6. Sequential API Calls for Centroid Batch Accepts

**Researchers**: UI/UX (Finding #6), Devil's Advocate (G3: unbounded blast radius)

`CentroidResultsDialog.svelte:121-139` uses sequential `assignFaceToPerson()` calls in a for-loop. No batch endpoint exists.

---

#### M7. Double-Load Risk on Suggestions Page

**Researchers**: UI/UX (Finding #7)

`$effect` fires on mount AND on filter change, potentially causing duplicate API calls and flickering.

---

#### M8. Full Collection Scan for Unlabeled Faces in Qdrant

**Researchers**: Backend (Moderate), Clustering (Section 6.2), Devil's Advocate (C2)

`face_qdrant.py:573-646`: Scrolls ALL faces and filters in Python due to Qdrant lacking native "is null" filter. For 100k faces, transfers ~200MB of vector data.

**Fix**: Add a boolean `has_person_id` payload field and index it for Qdrant-side filtering.

---

#### M9. Silent Exception Swallowing in Embedding Retrieval

**Researchers**: Devil's Advocate (Bomb #8)

All embedding retrieval methods catch all exceptions and return None, transforming Qdrant outages into "face not found" results. Centroids computed during partial outages are silently degraded.

---

#### M10. Post-Training Job Queue Flood

**Researchers**: Devil's Advocate (C3), Clustering (Priority 3, item 6)

After training, one suggestion job is queued per person with no rate limiting. 500 persons creates 500 concurrent DB sessions + thousands of Qdrant queries.

---

#### M11. Multiple Configuration Sources Create Hidden Coupling

**Researchers**: Backend (Section 4.4), Clustering (Section 7.2)

Thresholds come from env vars (pydantic-settings), SystemConfig DB table (via ConfigService), and hardcoded defaults in job function signatures. Same parameter may have different values depending on which source is read.

---

#### M12. Prototype Count Always Shows 0 on Person Detail

**Researchers**: UI/UX (Section 4.4, item 1)

`PersonDetailResponse` lacks `prototypeCount`, so the header always displays "0 prototypes" even when prototypes exist.

---

#### M13. EventSource No-Reconnect for SSE Progress

**Researchers**: Devil's Advocate (Bomb #9)

`faces.ts:808-810`: EventSource error handler immediately closes the connection instead of auto-reconnecting. Users lose progress updates on any transient error during long-running sessions.

---

#### M14. fetchAllPersons Loads All Persons Into Memory

**Researchers**: Devil's Advocate (Bomb #10), UI/UX (Section 4.4, item 3)

`faces.ts:384-403` loops with `pageSize=1000` and spread operator, creating quadratic GC pressure. For 10k+ persons, causes browser memory spikes.

---

#### M15. source_face_id Semantic Overload in FaceSuggestion

**Researchers**: Devil's Advocate (Section A3)

The `source_face_id` column stores face IDs, prototype IDs, or centroid UUIDs depending on context. The FK constraint to `face_instances` will fail for centroid-origin suggestions.

---

## Cross-Cutting Themes

### Theme 1: Qdrant Synchronization (4/4 reports)

All four researchers identified Qdrant consistency as a systemic concern. The dual-store architecture (PostgreSQL + Qdrant) has no transactional guarantees, no reconciliation, and multiple code paths that update one store but not the other. The suggestion acceptance bug (C1) is the most acute manifestation, but the pattern is endemic.

### Theme 2: Duplicated/Divergent Code Paths (3/4 reports)

Backend, Clustering, and Devil's Advocate all identified the centroid computation duplication (C3) and the dual centroid systems (H2). The sync/async split between API routes and worker jobs forces business logic duplication. The two clustering modes (FaceClusterer vs DualModeClusterer) use different metrics and centroid approaches.

### Theme 3: Terminology and Mental Model Confusion (2/4 reports)

UI/UX and Devil's Advocate both identified that users face a confusing array of terms for the same concepts. The "confidence" metric means different things in different contexts (detection confidence, prototype similarity, centroid similarity, aggregate confidence), and users have no way to know what they are comparing.

### Theme 4: Silent Failures (3/4 reports)

Backend, UI/UX, and Devil's Advocate identified multiple locations where errors are silently swallowed: embedding retrieval returns None on any exception, suggestion generation jobs produce zero results without errors (C2), post-training suggestion queuing failures are logged but not surfaced to users, and frontend error handlers log to console without user notification.

### Theme 5: Performance Bottlenecks at Scale (3/4 reports)

Backend, Clustering, and Devil's Advocate identified the N+1 Qdrant pattern (H1), O(n^2) distance matrix memory (H3), full collection scans (M8), and post-training job queue floods (M10) as scalability ceilings.

### Theme 6: God Objects (2/4 reports)

Backend and UI/UX both identified oversized files: `faces.py` (2,450 lines, 28+ endpoints), `face_jobs.py` (2,126 lines, 15 jobs), `SuggestionDetailModal.svelte` (1,008 lines, 8+ responsibilities), `person detail page` (1,911 lines).

---

## System Architecture Summary

The image-search system implements a face recognition pipeline with the following architecture:

**Detection Layer**: InsightFace `buffalo_l` model produces 512-dimensional ArcFace embeddings. Face detection runs in batched RQ background jobs with resume/pause support, using a producer-consumer pattern (ThreadPoolExecutor for I/O, main thread for GPU inference). Detected faces are stored as `FaceInstance` records in PostgreSQL with embeddings upserted to the `faces` Qdrant collection.

**Clustering Layer**: Two clustering modes exist -- a basic `FaceClusterer` (HDBSCAN with euclidean metric on normalized vectors) and a `DualModeClusterer` (Phase 1: supervised matching against known person centroids, Phase 2: HDBSCAN/DBSCAN/Agglomerative on remaining unknowns). Cluster IDs are written to both PostgreSQL and Qdrant.

**Suggestion Layer**: Four distinct strategies generate `FaceSuggestion` records: (1) single-prototype propagation when a face is labeled, (2) multi-prototype aggregation using all person prototypes, (3) centroid-based search using pre-computed person centroids, and (4) quality-weighted random sampling ("Find More"). A two-tier threshold system separates auto-assignment (high confidence) from suggestion creation (medium confidence). Suggestions are reviewed by users through a grouped UI with bulk accept/reject.

**Centroid Layer**: Person centroids are computed with outlier trimming (5-10% depending on face count) and stored in a dedicated `person_centroids` Qdrant collection. Staleness detection uses model version, algorithm version, and a SHA-256 hash of source face IDs. A legacy centroid system also exists in the assigner, storing centroids in the faces collection as prototype records.

**Frontend**: SvelteKit 2 + Svelte 5 application with approximately 25 face-related components, a 1,652-line API client (`faces.ts`), and pages for People, Clusters, Suggestions, and Person Detail views.

**Background Processing**: Redis/RQ with synchronous workers (due to macOS fork-safety). 15 job functions handle detection, clustering, assignment, training, and suggestion generation.

---

## UX Improvement Roadmap

### User Mental Model Issues

The system's internal pipeline (Images > Faces > Clusters > Persons) is not communicated to users. There is no onboarding, help text, or in-app documentation explaining:
- What "Identified", "Needs Name", and "Unknown Faces" categories mean
- Why a cluster exists and what actions to take
- How confidence scores are computed and what values to trust
- The relationship between the People page, Clusters page, and Suggestions page

**Recommendation**: Add contextual help tooltips to section headers explaining each category. Consider a first-run guided tour for the face management workflow.

### Terminology Inconsistencies

Standardize on a consistent glossary:
- **Person**: A named identity (not "Individual" or "Identity")
- **Face Group**: An unlabeled cluster of similar faces (pick one term, not "Cluster" in some places and "Unknown Faces" in others)
- **Ungrouped Face**: A face that could not be confidently grouped (not "Noise", "Review", or "Unknown")
- **Match Confidence**: The similarity score between a face and a person reference (not "score", "similarity", or "match %")
- **Suggestion**: A proposed match for user review (not "Centroid Suggestion" or "Match")

Files requiring updates: `UnifiedPersonCard.svelte`, `ClusterCard.svelte`, `CentroidResultsDialog.svelte`, `ComputeCentroidsDialog.svelte`, clusters `+page.svelte`, people `+page.svelte`, `SuggestionGroupCard.svelte`.

### Navigation & Flow Improvements

1. **Eliminate the People/Clusters overlap**: The People page (`/people`) is a superset of the Clusters page (`/faces/clusters`). Either remove the Clusters nav item and make it a filtered view within People, or clearly differentiate their purposes (e.g., People = named identities, Clusters = unlabeled groups for review).

2. **Fix cross-page navigation**: Clicking an unidentified person on the People page navigates to `/faces/clusters/{id}` without breadcrumb context. Add "Back to People" breadcrumb on cluster detail when navigated from People.

3. **Add a Suggestions tab to Person Detail**: Users currently must navigate to the global Suggestions page to see suggestions for a specific person. A "Suggestions" tab on the Person detail page would consolidate all person-related information.

4. **Make noise faces actionable**: Add a review flow for noise faces (either a dedicated page or expandable in-place cards) and remove the misleading "Review" badge if no action is available.

5. **Default to showing unidentified clusters on People page**: The `showUnidentified` toggle defaults to false, hiding the primary action items from new users.

---

## Technical Debt Inventory

| ID | Category | Description | Location | Severity |
|----|----------|-------------|----------|----------|
| TD1 | God Object | `faces.py` route file: 2,450 lines, 28+ endpoints | `api/routes/faces.py` | Structural |
| TD2 | God Object | `face_jobs.py`: 2,126 lines, 15 jobs | `queue/face_jobs.py` | Structural |
| TD3 | God Object | `SuggestionDetailModal.svelte`: 1,008 lines, 8+ responsibilities | UI component | Structural |
| TD4 | Duplication | Sync/async session split forces business logic duplication | API routes vs worker jobs | Structural |
| TD5 | Duplication | Two centroid systems (legacy prototype + formal PersonCentroid) | `assigner.py` vs `centroid_service.py` | High |
| TD6 | Duplication | Inline centroid computation in job vs service | `face_jobs.py:1944-2003` vs `centroid_service.py` | High |
| TD7 | Inconsistency | Centroid collection name not configurable (hardcoded constant) | `centroid_qdrant.py:34` | Moderate |
| TD8 | Inconsistency | Missing test safety guard on centroid collection reset | `centroid_qdrant.py` | Moderate |
| TD9 | Inconsistency | Cluster ID format varies across clustering paths | `clusterer.py` vs `dual_clusterer.py` | Low |
| TD10 | Inconsistency | Raw `fetch()` for centroid API functions, bypassing `apiRequest()` | `faces.ts:1563-1651` | Moderate |
| TD11 | Inconsistency | Mixed styling (custom CSS vs Tailwind) on person detail page | `people/[personId]/+page.svelte` | Low |
| TD12 | Inconsistency | Query parameter aliasing (camelCase vs snake_case) | Various API endpoints | Low |
| TD13 | Dead Code | Deprecated `faces_assigned` field still in model | `db/models.py` | Low |
| TD14 | Dead Code | `_resolve_asset_path()` checks nonexistent attributes | `faces/service.py` | Low |
| TD15 | Dead Code | `recluster_after_training_job` is a trivial pass-through | `face_jobs.py:395` | Low |
| TD16 | Incomplete | Time-bucket clustering parameter accepted but not implemented | `clusterer.py:80` | Low |
| TD17 | Incomplete | `exclude_prototypes` filter commented out (is_prototype not in DB model) | `face_centroids.py:381` | Low |
| TD18 | Inconsistency | Redundant suggestion response building (helper exists but not used) | `face_suggestions.py:441-499` | Low |
| TD19 | Inconsistency | Tab URL redirect `faces -> photos` suggests incomplete rename | `people/[personId]/+page.svelte:42` | Low |
| TD20 | Missing | No observability metrics (Qdrant latency, consistency counters, job queue depth) | System-wide | Moderate |

---

## Prioritized Action Plan

### Phase 1: Critical Bug Fixes (Week 1)

1. **Fix Qdrant sync on suggestion accept** (C1): Add `qdrant.update_person_ids([face.qdrant_point_id], suggestion.suggested_person_id)` after the DB commit in `face_suggestions.py:533`. Also add the same call to `bulk_suggestion_action` endpoint. **Effort**: 1-2 hours. **Files**: `face_suggestions.py`.

2. **Fix payload key mismatch** (C2): Replace `result.payload.get("face_id")` with `result.payload.get("face_instance_id")` in `face_jobs.py` at lines 1438, 1665, and 2062. **Effort**: 30 minutes. **Files**: `face_jobs.py`.

3. **Add optimistic locking to face assignment** (C4): Add a `version` column to `FaceInstance` or use `SELECT FOR UPDATE` in `faces.py:assign_face` and `face_suggestions.py:accept_suggestion`. **Effort**: 2-4 hours. **Files**: `faces.py`, `face_suggestions.py`, `models.py` (if adding version column), new Alembic migration.

4. **Fix hardcoded "faces" collection names** (C5): Replace all 12 occurrences of hardcoded `"faces"` with `_get_face_collection_name()`. **Effort**: 1 hour. **Files**: `assigner.py`, `clusterer.py`, `dual_clusterer.py`, `trainer.py`, `face_jobs.py`.

5. **Run a one-time Qdrant reconciliation** to repair existing data corruption from C1: Script to sync all `person_id` values from PostgreSQL `face_instances` to the Qdrant `faces` collection. **Effort**: 2-4 hours (write + test + execute).

### Phase 2: UX Improvements (Weeks 2-3)

1. **Unify terminology** (H5): Establish glossary, update all components and page titles to use consistent terms. **Effort**: 4-6 hours. **Files**: 6-8 Svelte components + route pages.

2. **Make "Find More" discoverable**: Add text label "Find More Matches" to the button in `SuggestionGroupCard.svelte`. Show disabled state with tooltip when < 2 labeled faces instead of hiding. **Effort**: 1-2 hours.

3. **Replace all 7 native `confirm()` dialogs** (M2): Create a reusable `ConfirmDialog.svelte` using shadcn/ui AlertDialog and replace all native confirm calls with contextual information. **Effort**: 3-4 hours. **Files**: 7 files listed in UI/UX Section 3.4.

4. **Add actionable path for noise faces** (M5): Make noise face cards expandable to show individual faces with assignment buttons, or link to a dedicated review page. Remove misleading "Review" badge if no action is available. **Effort**: 4-8 hours.

5. **Default `showUnidentified` to true** on People page: Show unidentified clusters by default so new users see primary action items. **Effort**: 30 minutes. **File**: `routes/people/+page.svelte`.

6. **Fix prototype count display** (M12): Derive count from loaded prototypes list (`prototypes.length`) instead of missing `prototypeCount` field. **Effort**: 30 minutes. **File**: `routes/people/[personId]/+page.svelte:125`.

7. **Add empty state guidance**: When cluster/suggestion lists are empty, explain why and link to next action (e.g., "Run face detection on the Sessions page to generate clusters"). **Effort**: 2-3 hours. **Files**: Clusters, Suggestions, People `+page.svelte`.

8. **Persist centroid rejections to backend** (M4): Store rejection so re-running centroids does not resurface the same faces. **Effort**: 2-4 hours. **Files**: `CentroidResultsDialog.svelte`, new or updated backend endpoint.

### Phase 3: Architecture Improvements (Weeks 3-5)

1. **Eliminate legacy centroid system** (H2): Deprecate `assigner.compute_person_centroids()` and migrate all centroid operations to the formal `CentroidService` + `PersonCentroid` pipeline. Remove centroid-as-prototype storage in faces collection. **Effort**: 8-12 hours. **Files**: `assigner.py`, `face_jobs.py:199`.

2. **Unify centroid computation** (C3): Remove inline centroid code from `find_more_centroid_suggestions_job`. Create a sync wrapper for `centroid_service.compute_centroids_for_person()` for use in worker jobs. **Effort**: 4-6 hours. **Files**: `face_jobs.py:1944-2003`, potentially `centroid_service.py` (add sync interface).

3. **Fix centroid deprecation gap** (H4): Implement "create-then-deprecate" pattern: create new centroid with BUILDING status, verify success, then deprecate old centroids, then activate new one. **Effort**: 3-4 hours. **File**: `centroid_service.py`.

4. **Split `faces.py` route file** (TD1): Decompose into `face_clusters.py`, `face_persons.py`, `face_detection.py`, `face_assignment.py`, `face_prototypes.py`, `face_training.py`. **Effort**: 4-6 hours.

5. **Split `face_jobs.py`** (TD2): Decompose into `detection_jobs.py`, `clustering_jobs.py`, `suggestion_jobs.py`, `training_jobs.py`. **Effort**: 3-4 hours.

6. **Standardize centroid API error handling** (TD10): Refactor centroid API functions in `faces.ts` to use `apiRequest()` instead of raw `fetch()`. **Effort**: 2 hours. **File**: `faces.ts:1563-1651`.

7. **Add DB/Qdrant reconciliation job**: Periodic job that compares `person_id` and `cluster_id` between PostgreSQL and Qdrant, logging divergences and optionally auto-repairing. **Effort**: 6-8 hours.

8. **Fix SSE EventSource reconnection** (M13): Implement standard reconnection with exponential backoff in `subscribeFaceDetectionProgress`. **Effort**: 1-2 hours. **File**: `faces.ts:808-810`.

### Phase 4: Performance & Scalability (Weeks 5-8)

1. **Add batch embedding retrieval** (H1): Implement `get_embeddings_batch(point_ids)` in `FaceQdrantClient` using Qdrant's batch `retrieve()` API. Update all 7+ call sites. **Effort**: 4-6 hours. **Files**: `face_qdrant.py`, `centroid_service.py`, `face_jobs.py`, `face_clustering_service.py`, `dual_clusterer.py`, `faces.py`.

2. **Add `has_person_id` boolean index to Qdrant** (M8): Replace full-collection scan for unlabeled faces with Qdrant-side boolean filter. Update the field whenever `person_id` changes. **Effort**: 4-6 hours. **Files**: `face_qdrant.py`.

3. **Add memory-bounded clustering** (H3): For face counts above a threshold (e.g., 20,000), switch to approximate methods, incremental HDBSCAN, or reduce max_faces with clear documentation. **Effort**: 6-8 hours. **Files**: `dual_clusterer.py`, `clusterer.py`, `face_jobs.py`.

4. **Add rate limiting to post-training suggestion queuing** (M10): Batch multiple persons per job (10-20 per job) instead of one job per person. Add configurable concurrency limits. **Effort**: 3-4 hours. **File**: `face_jobs.py:849-892`.

5. **Reuse Redis connections in job progress updates**: Pass Redis connection as parameter or use module-level factory instead of creating new connection per update. **Effort**: 1-2 hours. **File**: `face_jobs.py`.

6. **Replace `fetchAllPersons` with paginated search** (M14): Use server-side search/filter for person selection dropdowns instead of loading all persons into browser memory. **Effort**: 4-6 hours. **Files**: `faces.ts`, components that use person dropdowns.

7. **Reduce modal stacking depth** (M1): Convert CentroidResultsDialog and SuggestionDetailModal into full pages or slide-over panels with URL-based deep linking. **Effort**: 8-12 hours. **Files**: `CentroidResultsDialog.svelte`, `SuggestionDetailModal.svelte`, route definitions.

8. **Add keyboard shortcuts for suggestion review**: Implement A/Enter (accept), R/Backspace (reject), arrow keys (navigate) in `SuggestionDetailModal.svelte`. **Effort**: 3-4 hours.

---

## Individual Report Summaries

### 01 -- UI/UX Analysis

The UI/UX researcher conducted a systematic review of 18 Svelte components and 7 route pages, tracing user journeys through the suggestion review pipeline from clusters page through detail modals to person assignment. The analysis identified terminology inconsistency as the single largest usability barrier, with the same concepts labeled 3-5 different ways across pages. The researcher discovered 7 locations using native `confirm()` dialogs, noise faces as dead-end UI elements, modal stacking reaching 4+ levels, and the hidden "Find More" feature behind an unlabeled icon. The report also identified a double-load risk from `$effect` and `onMount` both calling the same load function, and the `SuggestionDetailModal.svelte` exceeding the project's 300-line component guideline at 1,008 lines. The centroid API functions using raw `fetch()` instead of the centralized `apiRequest()` helper was flagged as an error handling gap.

### 02 -- Backend Architecture Analysis

The Backend researcher analyzed 12 core backend files totaling over 10,000 lines of Python code. The most impactful finding was the N+1 Qdrant embedding retrieval pattern across 7 call sites, where individual HTTP requests are made per face instead of using batch retrieval. The researcher identified a likely functional bug in the find-more jobs (payload key `face_id` vs `face_instance_id` mismatch), missing optimistic locking on face assignment, and a hardcoded collection name in the clusterer bypassing test safety. The analysis documented the complete API surface (40+ endpoints across 4 route files), the 15 background job functions, and the dual-storage architecture's consistency gaps. The report highlighted three structural issues: the 2,450-line faces.py god object, the 2,126-line face_jobs.py, and the sync/async session duality forcing business logic duplication.

### 03 -- Clustering System Analysis

The Clustering researcher provided the deepest technical analysis of the ML pipeline, documenting the complete clustering algorithm parameters, centroid computation with outlier trimming thresholds, four distinct suggestion generation strategies, and the two-tier threshold system. The analysis traced the full training pipeline through its three phases (detection, auto-assignment, clustering) and post-training suggestion generation. Key contributions include the detailed centroid staleness detection mechanism documentation, the O(n^2) pairwise confidence calculation concern, the comprehensive configuration map of 20+ thresholds from both env vars and DB-backed SystemConfig, and the threshold decision tree showing how faces flow from detection through assignment to clustering to suggestions. The researcher recommended unifying centroid computation paths, eliminating dual centroid storage, and adding centroid consistency validation.

### 04 -- Devil's Advocate Analysis

The Devil's Advocate researcher deliberately sought failure modes and hidden assumptions through adversarial analysis. The most critical finding was the Qdrant sync bug on suggestion acceptance (confirmed by grep showing zero matches for `update_person_ids` in `face_suggestions.py`). The researcher independently confirmed the payload key mismatch bug and expanded the hardcoded collection name issue from 1 location (Backend report) to 12 locations across 6 files. Unique contributions include the centroid deprecation race condition, the `source_face_id` semantic overload (FK to face_instances stores centroid UUIDs), the dual centroid systems analysis, the EventSource no-reconnect pattern, and the `fetchAllPersons` pagination bomb. The report also provided the most thorough scalability analysis, including memory estimates for the distance matrix at various face counts and the post-training job queue flood scenario. The observability and testing gaps sections highlighted systemic issues: no metrics exist to detect the identified bugs, and the test framework's constraint against external dependencies prevents integration tests for cross-system consistency paths.

---

## Appendix: Issue Cross-Reference Matrix

| Issue | UI/UX | Backend | Clustering | Devil's Advocate |
|-------|:-----:|:-------:|:----------:|:---------------:|
| Qdrant sync on suggestion accept (C1) | | | | X (primary) |
| Payload key mismatch face_id vs face_instance_id (C2) | | X | | X (confirmed) |
| Duplicated centroid computation (C3) | | X | X | X |
| Missing optimistic locking on face assignment (C4) | | X (primary) | | X (confirmed) |
| Hardcoded "faces" collection name (C5) | | X (1 site) | | X (12 sites) |
| N+1 Qdrant embedding retrieval (H1) | | X (primary) | X | X |
| Two parallel centroid systems (H2) | | | X | X (primary) |
| O(n^2) distance matrix memory (H3) | | X | X | X |
| Centroid deprecation race (H4) | | X (partial) | | X (primary) |
| Terminology inconsistency (H5) | X (primary) | | | X |
| Dual-store no reconciliation (H6) | | X | X | X (primary) |
| Modal stacking 4+ levels (M1) | X (primary) | | | |
| Native confirm() dialogs x7 (M2) | X (primary) | | | |
| Find More not discoverable (M3) | X (primary) | | | |
| Centroid reject cosmetic only (M4) | X (primary) | | | |
| Noise face dead ends (M5) | X (primary) | | | |
| Sequential centroid accepts (M6) | X | | | X |
| Double-load risk on suggestions (M7) | X (primary) | | | |
| Full collection scan unlabeled (M8) | | X | X | X |
| Silent exception swallowing (M9) | | | | X (primary) |
| Post-training job queue flood (M10) | | | X | X (primary) |
| Multiple config sources (M11) | | X (primary) | X | |
| Prototype count shows 0 (M12) | X (primary) | | | |
| EventSource no reconnect (M13) | | | | X (primary) |
| fetchAllPersons memory (M14) | X | | | X (primary) |
| source_face_id semantic overload (M15) | | | | X (primary) |
| God object: faces.py 2450 lines | | X (primary) | | |
| God object: face_jobs.py 2126 lines | | X (primary) | | |
| God object: SuggestionDetailModal 1008 lines | X (primary) | | | |
| Sync/async session duality | | X (primary) | | X |
| Centroid raw fetch in frontend | X (primary) | | | X |
| Confidence means different things | X | | | X (primary) |
| Unidentified hidden by default | X (primary) | | | |
| Invisible state transitions post-training | | | | X (primary) |
| Non-deterministic suggestion ordering | | | | X (primary) |
| Missing empty state guidance | X (primary) | | | |
| No keyboard shortcuts for review | X (primary) | | | |
