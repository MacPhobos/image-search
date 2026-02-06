# Devil's Advocate Analysis: Face Clustering & Suggestions System

**Date**: 2026-02-06
**Researcher**: Devil's Advocate (Agent 4 of 4)
**Scope**: Adversarial analysis of face clustering, suggestion, and assignment systems
**Method**: Cross-system consistency analysis, failure mode mapping, scalability stress testing

---

## Executive Summary

This analysis examines the face recognition pipeline -- from detection through clustering, suggestion, and assignment -- through an adversarial lens, deliberately seeking failure modes, hidden assumptions, and architectural weaknesses that optimistic design reviews typically miss.

The system's most dangerous characteristic is its **dual-store architecture** (PostgreSQL + Qdrant) with **no transactional guarantees across stores**. Every write path that touches both systems is a potential consistency bomb. Combined with a sync/async split between API routes (async) and worker jobs (sync), three distinct suggestion generation strategies (single-prototype, multi-prototype, centroid-based), and a frontend that makes optimistic assumptions about backend state, the system has numerous latent failure modes that will manifest under load or during partial outages.

**Severity Assessment**: 3 critical issues that can cause silent data corruption, 4 high-severity issues that degrade system reliability, and multiple moderate issues affecting maintainability and user experience.

---

## Top 10 Ticking Time Bombs

### 1. CRITICAL: Suggestion Acceptance Silently Desynchronizes Qdrant

**Location**: `image-search-service/src/image_search_service/api/routes/face_suggestions.py`, lines 526-533

**The Bug**: When a user accepts a face suggestion, the route updates `face.person_id` in PostgreSQL and commits. It **never** updates the `person_id` payload in Qdrant. Every subsequent Qdrant similarity search that filters by `person_id` will return stale results.

```python
# Line 527: DB updated
face.person_id = suggestion.suggested_person_id

# Line 530-531: Suggestion marked accepted
suggestion.status = FaceSuggestionStatus.ACCEPTED.value
suggestion.reviewed_at = datetime.now(UTC)

# Line 533: Committed to DB
await db.commit()

# MISSING: qdrant.update_person_ids([face.qdrant_point_id], suggestion.suggested_person_id)
```

**Impact**: Every accepted suggestion leaves Qdrant in an inconsistent state. Over time, Qdrant's `person_id` data diverges completely from PostgreSQL. This means:
- Future centroid-based searches (`search_faces_with_centroid` at `centroid_qdrant.py:275`) with `exclude_person_id` will fail to exclude faces that were already assigned.
- The same face will appear as a suggestion repeatedly for the same person.
- Face counts per person in Qdrant-based queries will be wrong.

**Confirmation**: I grepped for `update_person_ids` and `update_payload.*person` in `face_suggestions.py` and found **zero matches**. The assigner at `assigner.py:237` correctly calls `qdrant.update_person_ids()`, proving the pattern is known -- it was simply not applied in the suggestion acceptance path.

**Blast Radius**: Every user who accepts a suggestion. Cumulative, irreversible without a full Qdrant re-sync operation.

---

### 2. CRITICAL: Phantom Face IDs -- Qdrant Payload Key Mismatch

**Location**: Multiple files in `queue/face_jobs.py` vs `vector/face_qdrant.py`

**The Bug**: The Qdrant face payload stores faces with the key `face_instance_id` (see `face_qdrant.py:154`, `upsert_face` method). However, three separate job functions look for the key `face_id` instead:

- `find_more_suggestions_job` at `face_jobs.py:1438`: `result.payload.get("face_id")`
- `propagate_person_label_multiproto_job` at `face_jobs.py:1665`: `result.payload.get("face_id")`
- `find_more_centroid_suggestions_job` at `face_jobs.py:2062`: `result.payload.get("face_id")`

While the single-prototype `propagate_person_label_job` at `face_jobs.py:1060` correctly uses `qdrant_point_id` lookup, the multi-prototype and centroid paths look for a key that does not exist.

**Impact**: Every call to these three job functions will:
1. Get search results from Qdrant (working correctly)
2. Try to extract `face_id` from each result payload
3. Get `None` because the key is `face_instance_id`, not `face_id`
4. Skip the result with `if not face_id_str: continue`
5. Return 0 suggestions created

**This means the find-more and centroid-based suggestion features are silently broken.** They appear to work (no errors thrown) but produce no results. The only code paths that correctly look up `face_instance_id` are: `face_qdrant.py:621`, `clusterer.py:101`, `clusterer.py:255`, and `face_centroids.py:371`.

**Blast Radius**: All centroid-based and multi-prototype suggestion generation. Post-training suggestion queuing (which calls these jobs) silently produces nothing.

---

### 3. CRITICAL: Centroid Deprecation Gap -- "Deprecate Before Create" Race

**Location**: `image-search-service/src/image_search_service/services/centroid_service.py`, line 325

**The Bug**: The `compute_centroids_for_person` function deprecates ALL existing active centroids for a person **before** creating the new one:

```python
# Line 325: Deprecate first
await deprecate_centroids(db, centroid_qdrant, person_id)

# Lines 328-361: Compute new centroid (can fail)
embeddings_array = np.array(embeddings, dtype=np.float32)
centroid_vector = compute_global_centroid(...)

# Lines 349-364: Create DB record (can fail)
centroid = PersonCentroid(...)
db.add(centroid)
await db.flush()

# Lines 380-385: Store in Qdrant (can fail)
centroid_qdrant.upsert_centroid(...)
```

If **any step after deprecation fails** (numpy computation, DB flush, Qdrant upsert), the person is left with **zero active centroids**. The deprecation is already flushed to the DB.

Furthermore, at `centroid_service.py:389-394`, if the Qdrant upsert specifically fails, the code marks the centroid as FAILED and raises. But the `await db.flush()` that changes the status to FAILED may not be committed by the caller, depending on error handling in the call chain.

**Impact**: A Qdrant outage during centroid computation leaves persons without any centroid. The on-demand centroid creation in `find_more_centroid_suggestions_job` at `face_jobs.py:1901-2008` duplicates centroid logic without using the service, further increasing the risk of inconsistent centroid records.

---

### 4. HIGH: Hardcoded "faces" Collection Name Bypasses Test Safety

**Location**: 12 locations across 6 files (assigner.py, clusterer.py, dual_clusterer.py, trainer.py, face_jobs.py, bootstrap_qdrant.py)

**The Finding**: The `face_qdrant.py` module provides a configurable `_get_face_collection_name()` that reads from settings, specifically to allow tests to use a different collection name and prevent accidental deletion of production data. It even has a safety guard at `face_qdrant.py:814-819` that checks for `PYTEST_CURRENT_TEST`.

However, **at least 12 call sites bypass this** by directly using the string `"faces"`:

```
assigner.py:274       collection_name="faces"
assigner.py:344       collection_name="faces"
clusterer.py:88       collection_name="faces"
dual_clusterer.py:352 collection_name="faces"
trainer.py:416        collection_name="faces"
face_jobs.py:1027     collection_name="faces"
face_jobs.py:1048     collection_name="faces"
```

**Impact**: If tests using these code paths run against the real Qdrant instance (e.g., integration tests), they bypass the safety guard and could mutate or delete production face embeddings. The safety guard in `reset_collection()` protects only the reset path; individual reads/writes go directly to production.

---

### 5. HIGH: N+1 Embedding Retrieval -- O(n) Qdrant Round-Trips

**Location**: `assigner.py:128`, `dual_clusterer.py:150+173+229`, `centroid_service.py:205`

**The Pattern**: Every face-processing pipeline retrieves embeddings one at a time from Qdrant:

```python
# assigner.py:128
for face in faces:
    embedding = self._get_face_embedding(face.qdrant_point_id)

# centroid_service.py:205
for face in faces:
    embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
```

Each call triggers an HTTP round-trip to Qdrant. For a person with 500 faces, centroid computation makes 500 sequential HTTP requests. For an assignment batch of 1000 faces, the assigner makes 1000 sequential requests.

Qdrant's `retrieve` API accepts a list of IDs. A batch call could reduce 500 round-trips to 1.

**Impact**: At 5ms per Qdrant round-trip on localhost, 1000 faces takes 5 seconds. On a remote Qdrant with 20ms latency, it takes 20 seconds. This becomes the dominant bottleneck for all face operations. For the `get_unlabeled_faces_with_embeddings` method at `face_qdrant.py:573`, Qdrant's lack of native "is null" filtering forces full collection scrolling with Python-side filtering, compounding the issue.

---

### 6. HIGH: Dual Commit Pattern -- DB+Qdrant Without Rollback

**Location**: `assigner.py:234-239`, `dual_clusterer.py:335-341`, `face_jobs.py:646`

**The Pattern**: Throughout the codebase, a common pattern appears:

```python
# Step 1: Update database records
self.db.execute(stmt)

# Step 2: Update Qdrant
qdrant.update_person_ids(qdrant_point_ids, person_id)

# Step 3: Commit database
self.db.commit()
```

In `assigner.py:224-239`, if the Qdrant update at step 2 fails mid-batch (e.g., after updating person A's faces but before person B's), the subsequent `self.db.commit()` at step 3 is never reached. But the DB session has uncommitted changes from step 1 that will be discarded. However, the Qdrant changes from person A are already persisted.

Result: Qdrant has person A's faces updated, PostgreSQL does not. The reverse scenario is also possible: if step 2 succeeds but step 3 fails (rare but possible), PostgreSQL lacks the update while Qdrant has it.

**Impact**: Every assignment/clustering batch operation is a potential consistency divergence point. There is no reconciliation mechanism to detect or repair these divergences.

---

### 7. HIGH: Two Centroid Systems Operating in Parallel

**Location**: `assigner.py:286-390` (PersonPrototype CENTROID role) vs `services/centroid_service.py` (PersonCentroid table + dedicated Qdrant collection)

**The Finding**: The codebase contains TWO separate centroid systems:

**System A -- "Legacy" Prototype Centroids** (`assigner.py:286-390`):
- Stores centroids in the `faces` Qdrant collection alongside real face embeddings
- Uses `PersonPrototype` with `role=CENTROID` in PostgreSQL
- Uses `is_prototype=True` payload flag in Qdrant
- Simple mean computation, no outlier trimming
- No versioning or staleness detection

**System B -- "New" PersonCentroid Service** (`services/centroid_service.py` + `vector/centroid_qdrant.py`):
- Stores centroids in a dedicated `person_centroids` Qdrant collection
- Uses `PersonCentroid` table in PostgreSQL
- Robust computation with outlier trimming
- Full versioning, staleness detection, deprecation lifecycle

Both systems are actively called:
- `compute_centroids_job` at `face_jobs.py:199` calls System A via `assigner.compute_person_centroids()`
- `find_more_centroid_suggestions_job` at `face_jobs.py:1810` uses System B's PersonCentroid table
- Post-training flow at `face_jobs.py:849-860` queues System B centroid suggestions

**Impact**: The two systems can produce conflicting centroids for the same person. System A's centroid (in the faces collection with `is_prototype=True`) is searched by the assigner's prototype matching. System B's centroid (in person_centroids collection) is used by centroid-based suggestions. They may produce different matching results because they use different computation methods (simple mean vs. trimmed mean).

---

### 8. MODERATE: Silent Exception Swallowing in Embedding Retrieval

**Location**: `assigner.py:282-284`, `face_qdrant.py:517-519`, `centroid_qdrant.py:271-273`

**The Pattern**: All embedding retrieval methods catch ALL exceptions and return None:

```python
# assigner.py:282
except Exception as e:
    logger.error(f"Error retrieving embedding for {qdrant_point_id}: {e}")
    return None

# face_qdrant.py:517
except Exception as e:
    logger.error(f"Failed to retrieve embedding for point {point_id}: {e}")
    return None
```

This transforms **every Qdrant failure** (network timeout, collection deleted, authentication error, disk full) into a "face not found" result. The caller silently skips the face.

**Impact**: During a Qdrant partial outage, the system silently produces degraded results rather than failing fast. A centroid computed from 100/500 faces (because 400 returned None due to network errors) will be mathematically wrong. No alerting or circuit breaker exists to detect this degradation.

---

### 9. MODERATE: EventSource No-Reconnect Pattern

**Location**: `image-search-ui/src/lib/api/faces.ts`, lines 808-810

**The Pattern**: The `subscribeFaceDetectionProgress` function creates an EventSource for SSE streaming, but its error handler immediately closes the connection:

```typescript
eventSource.onerror = () => {
    eventSource.close();
    // No reconnection attempt
};
```

Standard SSE behavior is to auto-reconnect on transient errors. By closing on any error, the frontend loses progress updates after a single blip (e.g., a brief network hiccup, a proxy timeout, or a backend restart).

**Impact**: Users monitoring long-running face detection sessions (which can run for hours) will lose real-time progress on any transient error. The UI will appear frozen, prompting users to restart sessions unnecessarily.

---

### 10. MODERATE: fetchAllPersons Pagination Bomb

**Location**: `image-search-ui/src/lib/api/faces.ts`, lines 384-403

**The Pattern**: The `fetchAllPersons()` function loops with `pageSize=1000`, fetching ALL persons into memory:

```typescript
export async function fetchAllPersons(): Promise<PersonSummary[]> {
    let allPersons: PersonSummary[] = [];
    let offset = 0;
    const pageSize = 1000;

    while (true) {
        const response = await getPersons({ limit: pageSize, offset });
        allPersons = [...allPersons, ...response.persons];
        // ...
    }
    return allPersons;
}
```

Each loop iteration creates a new array (spread operator), meaning garbage collection pressure grows quadratically. With 10,000 persons, this creates ~10 intermediate arrays of increasing size.

**Impact**: For installations with thousands of persons, this function can cause browser memory spikes and UI freezes. It is called from components that need person lists for dropdowns/filters, making it a user-facing performance issue.

---

## Detailed Analysis by Category

### A. Data Flow Consistency Risks

#### A1. Three-Way Data Split Without Reconciliation

The system maintains face data across three systems with no consistency mechanism:

| Data Store | What It Holds | Update Mechanism |
|---|---|---|
| PostgreSQL `face_instances` | person_id, cluster_id, quality_score | SQLAlchemy ORM (async for API, sync for workers) |
| Qdrant `faces` collection | person_id, cluster_id, is_prototype, embedding | HTTP API via qdrant-client |
| Qdrant `person_centroids` | centroid embedding, person_id, metadata | HTTP API via CentroidQdrantClient |

There is no:
- Change Data Capture (CDC) to detect divergences
- Periodic reconciliation job
- Transaction spanning both systems
- Checksums or version numbers to compare states

The only repair mechanism is a full reset-and-rebuild, which requires downtime and reprocessing all face data.

#### A2. Sync/Async Session Split Creates Subtle Bugs

**API Routes** use `AsyncSession` (via FastAPI `Depends`):
- `face_suggestions.py`, `face_centroids.py`, etc.

**Worker Jobs** use `SyncSession` (via `get_sync_session()`):
- `face_jobs.py`, `assigner.py`, `dual_clusterer.py`

The `centroid_service.py` uses `AsyncSession` but is called from the API layer. However, `find_more_centroid_suggestions_job` at `face_jobs.py:1901-2008` **reimplements centroid computation** using synchronous operations rather than calling the service, because the service is async.

This means any bug fix to `centroid_service.py` must be **manually mirrored** in the inline sync implementation in `face_jobs.py`. Currently, they already diverge:
- `centroid_service.py` uses outlier trimming (line 329-334)
- `face_jobs.py:1951-1956` uses simple mean without trimming

#### A3. FaceSuggestion.source_face_id Semantic Overload

The `source_face_id` column in `FaceSuggestion` (DB model at `models.py:751-754`) has multiple semantic meanings:

1. In `propagate_person_label_job` (face_jobs.py:1098): It's the face that was just labeled by the user
2. In `propagate_person_label_multiproto_job` (face_jobs.py:1771): It's the highest-quality prototype
3. In `find_more_centroid_suggestions_job` (face_jobs.py:2090): It's the **centroid UUID** (not a face at all!)
4. In `assigner.py:194-204`: It defaults to `face.id` (the face itself!) if no prototype is found

Foreign key constraint: `ForeignKey("face_instances.id", ondelete="CASCADE")` at `models.py:753`. The centroid-based suggestion stores a centroid UUID in a column that has a FK constraint to `face_instances`. This will either:
- Cause an FK violation error (if the centroid UUID doesn't match any face instance)
- Silently work if UUIDs happen to collide (astronomically unlikely but semantically wrong)

---

### B. User Mental Model Mismatches

#### B1. "Confidence" Means Different Things in Different Contexts

The system presents "confidence" scores to users, but the number means different things:

| Context | What "Confidence" Actually Is | Range | Source |
|---|---|---|---|
| Face detection | RetinaFace detection probability | 0.0-1.0 | `detection_confidence` in FaceInstance |
| Prototype suggestion | Cosine similarity to single prototype | 0.0-1.0 | `FaceSuggestion.confidence` |
| Multi-prototype suggestion | Cosine similarity to best prototype | 0.0-1.0 | `FaceSuggestion.confidence` |
| Aggregate confidence | MAX cosine similarity across prototypes | 0.0-1.0 | `FaceSuggestion.aggregate_confidence` |
| Centroid suggestion | Cosine similarity to centroid | 0.0-1.0 | `FaceSuggestion.confidence` |

A "0.85 confidence" from a centroid search is **not comparable** to "0.85 confidence" from a single-prototype search. Centroids smooth out individual variation, so centroid similarities are generally **higher** than prototype similarities for the same face pair. Users sorting by confidence will see centroid suggestions ranked higher even when prototype suggestions might be more reliable.

#### B2. "Find More" vs "Suggestions" Conceptual Confusion

The system has two separate mechanisms that appear identical to users:

1. **Auto-generated suggestions** (created during face detection sessions via assigner)
2. **Find-more suggestions** (user-triggered via "Find More" button)

Both create `FaceSuggestion` records with `status=pending`. Both appear in the same suggestion review UI. But:
- Auto-suggestions use the assigner's two-tier threshold (different thresholds)
- Find-more suggestions use config-based threshold (potentially different)
- Post-training suggestions use hardcoded `min_similarity=0.70` (face_jobs.py:857)

A user who adjusts the suggestion threshold in settings affects auto-suggestions but NOT the already-queued find-more jobs that used the old threshold value.

#### B3. Invisible State Transitions

When a face detection session completes, it triggers a cascade of background operations (face_jobs.py:716-912):
1. Auto-assignment of faces (assigner)
2. Clustering of unlabeled faces (clusterer)
3. Queuing of per-person suggestion jobs (centroid or prototype)

Each step can independently fail without the user being informed. The session status shows "completed" even if clustering failed, assignment failed, or suggestion queuing failed. The user sees "Session Complete: 1000 faces detected" but has no visibility into:
- How many faces were auto-assigned vs suggested vs uncategorized
- Whether clustering ran or failed
- Whether find-more suggestions were queued or errored

---

### C. Scalability Concerns

#### C1. O(n^2) Distance Matrix in Clustering

**Location**: `dual_clusterer.py:379`

```python
from sklearn.metrics.pairwise import cosine_distances
distance_matrix = cosine_distances(embeddings)
```

For `n` faces, this creates an `n x n` float64 matrix. Memory usage: `n^2 * 8 bytes`.

| Faces | Matrix Size | Memory |
|---|---|---|
| 1,000 | 1M entries | 8 MB |
| 10,000 | 100M entries | 800 MB |
| 50,000 | 2.5B entries | 20 GB |
| 100,000 | 10B entries | 80 GB |

The `max_faces` parameter in the clustering job defaults to 50,000 (`face_jobs.py:98`), which would require 20GB of contiguous memory for the distance matrix alone. This is likely to cause OOM kills on typical deployments.

#### C2. Full Collection Scan for "Unlabeled" Faces

**Location**: `face_qdrant.py:573-646`

The comment at `face_qdrant.py:588-590` reveals the issue:

```python
# Note: Qdrant doesn't have a native "is null" filter, so we scroll
# all records and filter in Python
```

To find unlabeled faces, the system scrolls through **every face in the collection** and checks if `person_id` is missing in the payload. For 100,000 faces with 90% labeled, it downloads 100,000 records (with embeddings) to find 10,000 unlabeled ones. This is approximately 100,000 * 512 * 4 bytes = 200MB of vector data transferred over the network.

#### C3. Post-Training Job Queue Flood

**Location**: `face_jobs.py:849-892`

After a face detection session completes, the system queues one suggestion job per person:

```python
for person in persons:
    if use_centroids and person.face_count >= min_faces_for_centroid:
        job = queue.enqueue("find_more_centroid_suggestions_job", ...)
    else:
        job = queue.enqueue("propagate_person_label_multiproto_job", ...)
```

With 500 persons, this enqueues 500 background jobs. Each job:
- Opens a new DB session
- Makes multiple Qdrant queries (N+1 per face for multi-proto)
- Creates suggestion records

This creates a burst of 500 concurrent sessions + thousands of Qdrant queries that can overwhelm both PostgreSQL connection pool and Qdrant. There is no rate limiting, batching, or backpressure mechanism.

---

### D. Hidden State Machine Issues

#### D1. FaceDetectionSession Status Machine Has Invisible Edges

The `FaceDetectionSessionStatus` enum defines: PENDING, PROCESSING, COMPLETED, FAILED, PAUSED, CANCELLED.

The state machine has these transitions in code:
```
PENDING -> PROCESSING (face_jobs.py:504)
PROCESSING -> PAUSED (checked at face_jobs.py:578)
PROCESSING -> CANCELLED (checked at face_jobs.py:594)
PROCESSING -> COMPLETED (face_jobs.py:916)
PROCESSING -> FAILED (face_jobs.py:947)
```

But the **resume path** (face_jobs.py:510-536) allows: PAUSED -> PROCESSING. This creates a problem: when a session is paused and then resumed, a NEW RQ job is created (the old one returned at line 583). If Redis doesn't properly clear the old job, two workers could simultaneously process the same session.

The code at face_jobs.py:576 does `db_session.refresh(session)` to check current status, but there's a TOCTOU race: between the refresh and the next batch start, another process could change the status.

#### D2. Centroid Status Machine Lacks "Building" Enforcement

The `CentroidStatus` enum includes BUILDING, but `compute_centroids_for_person` never sets status to BUILDING:

```python
# centroid_service.py:349-361
centroid = PersonCentroid(
    ...
    status=CentroidStatus.ACTIVE,  # Goes straight to ACTIVE
    ...
)
```

This means if two concurrent calls compute centroids for the same person, both will:
1. Deprecate existing centroids (line 325)
2. Create new ACTIVE centroids
3. The unique constraint `ix_person_centroid_unique_active` will cause one to fail

The BUILDING status was designed to prevent this, but it's never used.

---

### E. Error Recovery Failures

#### E1. Redis Failure During Post-Processing Is Fire-and-Forget

**Location**: `face_jobs.py:846-912`

After face detection completes, suggestion jobs are queued via Redis/RQ. If Redis is unavailable:

```python
try:
    redis_client = get_redis()
    queue = Queue("default", connection=redis_client)
    # ... queue jobs ...
except Exception as e:
    logger.error(f"[{job_id}] Post-training suggestion queuing error (non-fatal): {e}")
    # Don't fail the whole job if suggestion queuing fails
```

The session is marked COMPLETED (line 916-917) regardless. There is no mechanism to:
- Retry the suggestion queuing later
- Mark that suggestions are pending generation
- Notify the user that suggestions won't appear

The user sees "Session Complete" and expects suggestions. They never come.

#### E2. No Idempotency Keys for Background Jobs

None of the face jobs implement idempotency. If `find_more_suggestions_job` is accidentally enqueued twice for the same person (e.g., due to Redis retry), it will:
1. Create duplicate suggestions (checked individually at line 1503, but between two concurrent jobs, race condition)
2. Double the Qdrant query load
3. Potentially exceed the `max_suggestions` limit in aggregate

The `propagate_person_label_job` does check for existing pending suggestions (face_jobs.py:1082-1090), but the check-then-insert is not atomic -- two concurrent jobs can both pass the check and create duplicates.

#### E3. Cleanup Jobs Don't Handle Large Volumes

`expire_old_suggestions_job` at `face_jobs.py:1162-1173` loads ALL expired suggestions into memory:

```python
old_suggestions = result.scalars().all()  # Loads entire result set

expired_count = 0
for suggestion in old_suggestions:
    suggestion.status = FaceSuggestionStatus.EXPIRED.value
```

With 100,000 stale suggestions, this loads all of them into the SQLAlchemy identity map, consuming significant memory. A batch-update SQL statement would be orders of magnitude more efficient.

---

### F. Frontend-Backend Inconsistencies

#### F1. Inconsistent Error Handling: apiRequest vs Raw Fetch

**Location**: `image-search-ui/src/lib/api/faces.ts`

The vast majority of API functions use the `apiRequest()` helper (imported from `client.ts`), which provides centralized error handling, response parsing, and timeout management.

However, the centroid API functions (lines 1563-1651) use raw `fetch()`:

```typescript
// Lines 1568-1572 (centroid functions)
const response = await fetch(`${getApiBase()}/api/v1/faces/centroids/...`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ... }),
});
```

This means centroid operations:
- Don't get automatic timeout handling
- Don't get centralized error parsing
- Don't trigger global error notifications
- Silently fail if the response isn't manually checked

#### F2. Mixed Type Sources

The `faces.ts` file defines types both inline (lines 60-255, ~200 lines of local type definitions) and from the generated OpenAPI types (`components['schemas']` at line 411+). Some types shadow generated types with slightly different shapes.

If the backend API changes, the generated types update via `make gen-api`, but the inline types in `faces.ts` remain stale. There is no compile-time check that these stay in sync.

#### F3. Snake_case vs CamelCase in Request Bodies

Some API call sites send snake_case keys (matching the Python backend), while others send camelCase (matching TypeScript conventions). For example:

- `face_jobs.py:857`: `min_similarity=0.70` (Python parameter, snake_case)
- `faces.ts:1568`: `{ min_similarity: 0.70 }` (TypeScript body, snake_case for this one)
- Other functions in `faces.ts` use camelCase

FastAPI uses Pydantic which handles alias mapping, but only if configured. Any mismatch silently drops the parameter, causing the backend to use defaults.

---

### G. UX Anti-Patterns

#### G1. Non-Deterministic Suggestion Ordering

The `find_more_suggestions_job` uses `random.choices` with weighted selection (face_jobs.py:1401-1405). This means:
- Running "Find More" twice for the same person produces different suggestions
- Users cannot reproduce or compare results
- The ordering of suggestions in the UI varies between runs

While randomness helps explore the face space, it violates the principle of least surprise. A user who sees a suggestion, navigates away, and returns may find different suggestions.

#### G2. No Progress Granularity for Multi-Step Operations

The post-training flow (detection -> assignment -> clustering -> suggestion queuing) shows a single "Session Complete" at the end. During operation, the progress bar shows detection progress but not:
- Which phase is running (detection vs assignment vs clustering)
- How many persons had suggestions queued
- Whether any phase was skipped due to errors

#### G3. Bulk Accept All Has Unbounded Blast Radius

The suggestion review UI allows "Accept All" for a person's suggestions. The bulk action at `face_suggestions.py:662-694` processes each suggestion individually with N+1 database queries. For a person with 200 suggestions, this means 200+ DB round-trips in a single HTTP request.

Furthermore, the accepted suggestions trigger find-more job queuing (line 695-706), which can cascade into hundreds more background jobs. There is no confirmation dialog warning about the downstream effects.

---

## Cross-Cutting Concerns

### Observability Gap

The system has extensive logging but no metrics. There are no:
- Qdrant query latency histograms
- Cross-system consistency counters (DB vs Qdrant face counts)
- Suggestion acceptance/rejection rate tracking
- Centroid freshness monitoring
- Job queue depth/latency metrics

Without metrics, the phantom face ID bug (Bomb #2) and the Qdrant sync bug (Bomb #1) would go undetected indefinitely.

### Testing Gap

The `CLAUDE.md` for the backend states: "No external deps - tests must pass without Postgres/Redis/Qdrant running" (line 114). This means integration tests for the most critical code paths (cross-system consistency, Qdrant sync) cannot exist in the current test framework. The bugs identified in this analysis are all in cross-system interaction paths that unit tests with mocked dependencies would not catch.

---

## Recommendations Priority Matrix

| Priority | Issue | Effort | Impact |
|---|---|---|---|
| P0 | Fix Qdrant sync on suggestion accept (Bomb #1) | Low (add 1 line) | Prevents cumulative data corruption |
| P0 | Fix payload key mismatch `face_id` vs `face_instance_id` (Bomb #2) | Low (rename 3 keys) | Unblocks centroid/multiproto suggestions |
| P1 | Add reconciliation job for DB<->Qdrant consistency | Medium | Detects and repairs all sync issues |
| P1 | Remove legacy centroid system (Bomb #7) | Medium | Eliminates parallel centroid confusion |
| P1 | Batch embedding retrieval (Bomb #5) | Medium | 10-100x performance improvement |
| P2 | Fix centroid deprecate-before-create (Bomb #3) | Low (reorder operations) | Prevents centroid gaps |
| P2 | Replace hardcoded "faces" collection names (Bomb #4) | Low (use function) | Improves test safety |
| P2 | Add circuit breaker for Qdrant calls (Bomb #8) | Medium | Prevents silent degradation |
| P3 | Add idempotency to background jobs | Medium | Prevents duplicate suggestions |
| P3 | Implement SSE reconnection (Bomb #9) | Low | Improves monitoring UX |
| P3 | Replace fetchAllPersons with paginated component (Bomb #10) | Medium | Prevents browser memory issues |

---

## Methodology Notes

This analysis was conducted by:

1. **Reading core source files** (7 backend modules, 2 frontend modules, 2 Qdrant clients)
2. **Cross-referencing data flow paths** (tracing face_instance_id/person_id across DB, Qdrant, frontend)
3. **Grepping for pattern violations** (hardcoded collection names, payload key mismatches)
4. **Mapping state machines** against enum definitions
5. **Analyzing error handling chains** (what happens when each step fails)

Files analyzed:
- `image-search-service/src/image_search_service/db/models.py` (971 lines)
- `image-search-service/src/image_search_service/faces/assigner.py` (410 lines)
- `image-search-service/src/image_search_service/faces/dual_clusterer.py` (427 lines)
- `image-search-service/src/image_search_service/services/centroid_service.py` (426 lines)
- `image-search-service/src/image_search_service/vector/face_qdrant.py` (868 lines)
- `image-search-service/src/image_search_service/vector/centroid_qdrant.py` (531 lines)
- `image-search-service/src/image_search_service/queue/face_jobs.py` (2127 lines)
- `image-search-service/src/image_search_service/api/routes/face_suggestions.py` (928 lines)
- `image-search-ui/src/lib/api/faces.ts` (1652 lines)
