# C5: Unify Centroid Computation Paths

**Priority**: Critical
**Severity**: Critical (Silent Data Divergence)
**Estimated Effort**: 3-4 days
**Date**: 2026-02-06

---

## References

| Artifact | Path |
|----------|------|
| Research: Executive Synthesis | `docs/research/clustering_and_suggestions/00-executive-synthesis.md` |
| Research: Clustering System Analysis | `docs/research/clustering_and_suggestions/03-clustering-system-analysis.md` |
| Research: Devil's Advocate | `docs/research/clustering_and_suggestions/04-devils-advocate-analysis.md` |
| Formal Centroid Service | `image-search-service/src/image_search_service/services/centroid_service.py` |
| Inline Centroid (Job) | `image-search-service/src/image_search_service/queue/face_jobs.py` (line 1810) |
| Assigner Centroid | `image-search-service/src/image_search_service/faces/assigner.py` (line 286) |
| Dual Clusterer Centroid | `image-search-service/src/image_search_service/faces/dual_clusterer.py` (line 158) |
| Centroid Qdrant Client | `image-search-service/src/image_search_service/vector/centroid_qdrant.py` |
| Face Qdrant Client | `image-search-service/src/image_search_service/vector/face_qdrant.py` |

---

## 1. Problem Statement

The codebase contains **four separate centroid computation implementations** that produce different results for the same input data. When the formal centroid service (`centroid_service.py`) is improved -- for example, by adding outlier trimming, staleness detection, or versioning -- the other implementations do not benefit from those changes. This creates silent data divergence where:

1. Two users looking at the same person's centroid may see different embeddings depending on which code path computed it.
2. Bug fixes to centroid math in one path do not propagate to others.
3. Testing and validation efforts are multiplied across all four implementations.

---

## 2. Root Cause Analysis

### The Four Centroid Computation Paths

**Path A: Formal CentroidService (Full-featured)**
- **File**: `image-search-service/src/image_search_service/services/centroid_service.py`
- **Function**: `compute_global_centroid()` (lines 36-109)
- **Algorithm**: Robust mean with outlier trimming, cosine distance-based filtering, L2 normalization
- **Features**:
  - Outlier trimming via configurable thresholds (`trim_threshold_small=0.05`, `trim_threshold_large=0.10`)
  - Staleness detection (`is_centroid_stale()`, lines 127-174)
  - Versioning and deprecation (`deprecate_centroids()`, lines 220-255)
  - Source hash tracking for change detection
  - Qdrant storage in dedicated `person_centroids` collection
- **Called by**: `compute_centroids_for_person()` (lines 258-396) via API routes
- **Session type**: Async (AsyncSession)

**Path B: Inline Computation in find_more_centroid_suggestions_job (Simplified)**
- **File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
- **Function**: `find_more_centroid_suggestions_job()` (lines 1951-1956)
- **Algorithm**: Simple mean, NO outlier trimming, L2 normalization
- **Code** (lines 1951-1956):
  ```python
  embeddings_array = np.array(embeddings, dtype=np.float32)

  # Compute mean and normalize
  centroid_mean = np.mean(embeddings_array, axis=0)
  norm = np.linalg.norm(centroid_mean)
  centroid_vector = (centroid_mean / norm).astype(np.float32)
  ```
- **Missing features**:
  - No outlier trimming (`build_params` explicitly records `"trim_outliers": False` at line 1979)
  - No staleness detection
  - No versioning (creates version but does not check against existing)
  - No deprecation of old centroids
- **Called by**: RQ job when no active centroid exists
- **Session type**: Sync (SyncSession via `get_sync_session()`)

**Path C: FaceAssigner.compute_person_centroids (Legacy)**
- **File**: `image-search-service/src/image_search_service/faces/assigner.py`
- **Function**: `compute_person_centroids()` (lines 286-390)
- **Algorithm**: Simple mean of all face embeddings, L2 normalization
- **Code** (lines 330-331):
  ```python
  centroid = np.mean(embeddings, axis=0)
  centroid = centroid / np.linalg.norm(centroid)  # Re-normalize
  ```
- **Missing features**:
  - No outlier trimming
  - No staleness detection
  - No versioning
  - Stores in `faces` collection (not `person_centroids`)
  - Creates `PersonPrototype` with `role=CENTROID` (different schema from `PersonCentroid`)
- **Session type**: Sync (SyncSession)

**Path D: DualModeClusterer.assign_to_known_people (Inline)**
- **File**: `image-search-service/src/image_search_service/faces/dual_clusterer.py`
- **Function**: `assign_to_known_people()` (lines 158-164)
- **Algorithm**: Simple mean per person's labeled faces, L2 normalization
- **Code** (lines 161-163):
  ```python
  centroid = np.mean(embeddings, axis=0)
  # Re-normalize
  centroid = centroid / np.linalg.norm(centroid)
  ```
- **Missing features**:
  - No outlier trimming
  - No persistence (ephemeral, used only during clustering run)
  - No staleness detection, versioning, or Qdrant storage
- **Session type**: Sync (SyncSession)

### Comparison Matrix

| Feature | Path A (CentroidService) | Path B (face_jobs.py) | Path C (assigner.py) | Path D (dual_clusterer.py) |
|---------|-------------------------|----------------------|---------------------|--------------------------|
| Outlier trimming | Yes (configurable) | No (`trim_outliers: False`) | No | No |
| L2 normalization | Yes | Yes | Yes | Yes |
| Staleness detection | Yes | No | No | No |
| Versioning | Yes (centroid_version) | Partial (writes version, no check) | No | No |
| Deprecation | Yes | No | No | No |
| Source hash | Yes | Yes | No | No |
| Storage collection | `person_centroids` | `person_centroids` | `faces` (wrong!) | None (ephemeral) |
| Session type | Async | Sync | Sync | Sync |
| Min faces check | Configurable | Hardcoded 5 | Hardcoded 2 | No minimum |

### Why the Divergence Exists

The devil's advocate analysis (Section A2) explains the root cause: the formal `CentroidService` uses **async** database operations (`AsyncSession`), but RQ background jobs require **sync** operations. Rather than creating a sync wrapper for the async service, developers duplicated the centroid logic inline using sync sessions. This happened independently in three different locations, each with different subsets of the formal service's features.

---

## 3. Current Behavior vs Expected Behavior

### Current Behavior

1. `compute_centroids_for_person()` (API route) uses full outlier trimming, producing a robust centroid.
2. `find_more_centroid_suggestions_job()` (background job) uses simple mean, producing a potentially skewed centroid if outlier faces exist.
3. `FaceAssigner.compute_person_centroids()` uses simple mean and stores in the wrong Qdrant collection.
4. `DualModeClusterer` uses simple mean with no persistence.
5. For a person with 1 outlier face out of 20, Paths A and B produce **measurably different centroid vectors**, leading to different suggestion results depending on which path was used.

### Expected Behavior

1. ALL centroid computations use the SAME algorithm (outlier trimming, normalization).
2. ALL persistent centroids are stored in the `person_centroids` collection with consistent metadata.
3. Algorithm improvements to `compute_global_centroid()` automatically benefit all call sites.
4. A single sync-compatible entry point exists for background jobs to compute centroids.
5. Staleness detection and versioning are applied consistently.

---

## 4. Implementation Plan

### Approach

Create a **sync wrapper** for the core centroid computation that can be called from both async API routes and sync RQ jobs. The formal `compute_global_centroid()` function is already a pure function (takes numpy arrays, returns numpy array) -- it does not use async. The async dependency is only in the surrounding `compute_centroids_for_person()` function that fetches data from the database.

The plan:
1. Extract the pure computation into a standalone sync function (already done -- `compute_global_centroid()` is sync).
2. Create a sync entry point for the full pipeline (fetch faces, compute centroid, store in Qdrant).
3. Replace all inline computations with calls to this sync entry point or the pure function.

### Step 1: Create Sync Centroid Pipeline

**File**: `image-search-service/src/image_search_service/services/centroid_service.py`

Add a new sync function that mirrors `compute_centroids_for_person()` but uses sync database sessions:

```python
def compute_centroids_for_person_sync(
    db: SyncSession,
    qdrant: FaceQdrantClient,
    centroid_qdrant: CentroidQdrantClient,
    person_id: uuid.UUID,
    force_rebuild: bool = False,
    min_faces: int = 5,
    trim_outliers: bool = True,
) -> PersonCentroid | None:
    """Sync version of compute_centroids_for_person for use in RQ jobs.

    This function provides the same centroid computation as the async version,
    ensuring consistent algorithm application across all code paths.

    Args:
        db: Synchronous database session
        qdrant: Face Qdrant client for retrieving embeddings
        centroid_qdrant: Centroid Qdrant client for storing centroids
        person_id: Person UUID
        force_rebuild: Force rebuild even if centroid is not stale
        min_faces: Minimum number of faces required
        trim_outliers: Whether to apply outlier trimming (default: True)

    Returns:
        PersonCentroid record if created/updated, None if insufficient faces
    """
    from image_search_service.db.models import FaceInstance, PersonCentroid, CentroidType

    # 1. Get face instances for this person
    query = select(FaceInstance).where(FaceInstance.person_id == person_id)
    faces = list(db.execute(query).scalars().all())

    if len(faces) < min_faces:
        logger.warning(
            f"Only {len(faces)} faces for person {person_id} "
            f"(minimum {min_faces} required)"
        )
        return None

    # 2. Retrieve embeddings from Qdrant (N+1 pattern -- to be fixed in H1)
    face_ids = []
    embeddings_list = []

    for face in faces:
        if face.qdrant_point_id:
            embedding = qdrant.get_embedding_by_point_id(face.qdrant_point_id)
            if embedding is not None:
                face_ids.append(face.id)
                embeddings_list.append(embedding)

    if len(embeddings_list) < min_faces:
        logger.warning(
            f"Only {len(embeddings_list)} embeddings found for person {person_id}"
        )
        return None

    # 3. Compute centroid using the CANONICAL algorithm
    embeddings_array = np.array(embeddings_list, dtype=np.float32)
    centroid_vector = compute_global_centroid(
        embeddings_array,
        trim_outliers=trim_outliers,
    )

    # 4. Compute source hash for staleness detection
    import hashlib
    sorted_ids = sorted(str(fid) for fid in face_ids)
    hash_input = ":".join(sorted_ids)
    source_hash = hashlib.sha256(hash_input.encode()).hexdigest()[:16]

    # 5. Check if existing centroid is still valid
    if not force_rebuild:
        existing_query = (
            select(PersonCentroid)
            .where(
                PersonCentroid.person_id == person_id,
                PersonCentroid.status == "active",
                PersonCentroid.centroid_type == CentroidType.GLOBAL,
            )
            .order_by(PersonCentroid.created_at.desc())
            .limit(1)
        )
        existing = db.execute(existing_query).scalar_one_or_none()

        if existing and existing.source_face_ids_hash == source_hash:
            logger.info(f"Centroid for person {person_id} is still valid (hash match)")
            return existing

    # 6. Deprecate old centroids
    deprecate_query = (
        select(PersonCentroid)
        .where(
            PersonCentroid.person_id == person_id,
            PersonCentroid.status == "active",
        )
    )
    old_centroids = list(db.execute(deprecate_query).scalars().all())
    for old in old_centroids:
        old.status = "deprecated"

    # 7. Create new centroid record
    settings = get_settings()
    centroid_id = uuid.uuid4()

    centroid = PersonCentroid(
        centroid_id=centroid_id,
        person_id=person_id,
        qdrant_point_id=centroid_id,
        model_version=settings.centroid_model_version,
        centroid_version=settings.centroid_algorithm_version,
        centroid_type=CentroidType.GLOBAL,
        cluster_label="global",
        n_faces=len(face_ids),
        status="active",
        source_face_ids_hash=source_hash,
        build_params={
            "trim_outliers": trim_outliers,
            "n_faces_used": len(face_ids),
            "trim_threshold_small": 0.05,
            "trim_threshold_large": 0.10,
        },
    )
    db.add(centroid)
    db.flush()

    # 8. Store in Qdrant
    centroid_qdrant.upsert_centroid(
        centroid_id=centroid_id,
        vector=centroid_vector.tolist(),
        payload={
            "person_id": person_id,
            "centroid_id": centroid_id,
            "model_version": settings.centroid_model_version,
            "centroid_version": settings.centroid_algorithm_version,
            "centroid_type": "global",
            "cluster_label": "global",
            "n_faces": len(face_ids),
            "source_hash": source_hash,
            "build_params": centroid.build_params,
        },
    )

    db.commit()
    logger.info(
        f"Created centroid {centroid_id} for person {person_id} "
        f"from {len(face_ids)} faces (trim_outliers={trim_outliers})"
    )

    return centroid
```

### Step 2: Replace Inline Centroid in find_more_centroid_suggestions_job

**File**: `image-search-service/src/image_search_service/queue/face_jobs.py`
**Lines to replace**: 1901-2008

**Before** (lines 1901-2008, simplified):
```python
if not centroid:
    # Compute centroid on-demand (sync operation)
    # ... 107 lines of inline centroid computation ...
    embeddings_array = np.array(embeddings, dtype=np.float32)
    centroid_mean = np.mean(embeddings_array, axis=0)
    norm = np.linalg.norm(centroid_mean)
    centroid_vector = (centroid_mean / norm).astype(np.float32)
    # ... create PersonCentroid record with trim_outliers=False ...
```

**After**:
```python
if not centroid:
    # Compute centroid on-demand using canonical algorithm
    logger.info(f"[{job_id}] No active centroid found, computing via CentroidService...")

    from image_search_service.services.centroid_service import (
        compute_centroids_for_person_sync,
    )
    from image_search_service.vector.centroid_qdrant import get_centroid_qdrant_client

    face_qdrant = get_face_qdrant_client()
    centroid_qdrant_client = get_centroid_qdrant_client()

    centroid = compute_centroids_for_person_sync(
        db=db_session,
        qdrant=face_qdrant,
        centroid_qdrant=centroid_qdrant_client,
        person_id=person_uuid,
        force_rebuild=False,
        min_faces=5,
        trim_outliers=True,  # Use outlier trimming (was False)
    )

    if not centroid:
        logger.warning(f"[{job_id}] Could not compute centroid for person {person_id}")
        return {"status": "error", "message": "Could not compute centroid"}
```

This replaces ~107 lines of inline code with a ~15-line call to the canonical service.

### Step 3: Replace Centroid Computation in FaceAssigner

**File**: `image-search-service/src/image_search_service/faces/assigner.py`
**Function**: `compute_person_centroids()` (lines 286-390)

**Before** (lines 329-331):
```python
# Compute centroid (mean of normalized vectors, then re-normalize)
centroid = np.mean(embeddings, axis=0)
centroid = centroid / np.linalg.norm(centroid)  # Re-normalize
```

**After**:
```python
# Compute centroid using canonical algorithm with outlier trimming
from image_search_service.services.centroid_service import compute_global_centroid

embeddings_array = np.array(embeddings, dtype=np.float32)
centroid = compute_global_centroid(embeddings_array, trim_outliers=True)
```

Additionally, this function stores centroids in the `faces` collection as `PersonPrototype` records. This should be updated to store in `person_centroids` collection using `CentroidQdrantClient`, or this function should be deprecated in favor of calling `compute_centroids_for_person_sync()` directly.

**Recommended approach**: Deprecate `FaceAssigner.compute_person_centroids()` entirely and replace its callers with `compute_centroids_for_person_sync()`:

```python
def compute_person_centroids(self) -> dict:
    """Compute centroid embeddings for all persons.

    DEPRECATED: Use centroid_service.compute_centroids_for_person_sync() instead.
    This method is maintained for backward compatibility.
    """
    import warnings
    warnings.warn(
        "compute_person_centroids() is deprecated. "
        "Use centroid_service.compute_centroids_for_person_sync() instead.",
        DeprecationWarning,
        stacklevel=2,
    )

    from image_search_service.services.centroid_service import (
        compute_centroids_for_person_sync,
    )
    from image_search_service.vector.centroid_qdrant import get_centroid_qdrant_client
    from image_search_service.vector.face_qdrant import get_face_qdrant_client

    qdrant = get_face_qdrant_client()
    centroid_qdrant = get_centroid_qdrant_client()

    # Get all active persons
    from image_search_service.db.models import Person
    persons_query = select(Person).where(Person.status == "active")
    persons = self.db.execute(persons_query).scalars().all()

    centroids_computed = 0
    for person in persons:
        result = compute_centroids_for_person_sync(
            db=self.db,
            qdrant=qdrant,
            centroid_qdrant=centroid_qdrant,
            person_id=person.id,
            min_faces=2,
            trim_outliers=True,
        )
        if result:
            centroids_computed += 1

    return {
        "persons_processed": len(persons),
        "centroids_computed": centroids_computed,
    }
```

### Step 4: Replace Centroid Computation in DualModeClusterer

**File**: `image-search-service/src/image_search_service/faces/dual_clusterer.py`
**Function**: `assign_to_known_people()` (lines 158-164)

**Before** (lines 160-164):
```python
# Calculate centroid for each person
person_centroids = {}
for person_id, embeddings in person_embeddings.items():
    centroid = np.mean(embeddings, axis=0)
    # Re-normalize
    centroid = centroid / np.linalg.norm(centroid)
    person_centroids[person_id] = centroid
```

**After**:
```python
# Calculate centroid for each person using canonical algorithm
from image_search_service.services.centroid_service import compute_global_centroid

person_centroids = {}
for person_id, embeddings in person_embeddings.items():
    embeddings_array = np.array(embeddings, dtype=np.float32)
    if len(embeddings_array) >= 3:
        # Use outlier trimming for sufficient sample sizes
        centroid = compute_global_centroid(embeddings_array, trim_outliers=True)
    else:
        # Too few for trimming; use simple mean
        centroid = compute_global_centroid(embeddings_array, trim_outliers=False)
    person_centroids[person_id] = centroid
```

Note: The DualModeClusterer uses ephemeral centroids (not persisted). This is acceptable because these centroids are only used within a single clustering run. The key improvement is that the computation algorithm is now consistent with the formal service.

---

## 5. Data Repair

### Existing Centroids May Have Different Vectors

After unifying the computation, existing centroids in the `person_centroids` collection may have been computed without outlier trimming. These should be rebuilt:

```python
# One-time migration script or management command
from image_search_service.services.centroid_service import compute_centroids_for_person_sync

def rebuild_all_centroids(db_session, qdrant, centroid_qdrant):
    """Rebuild all centroids using the unified algorithm."""
    from image_search_service.db.models import Person

    persons = db_session.execute(
        select(Person).where(Person.status == "active")
    ).scalars().all()

    rebuilt = 0
    for person in persons:
        result = compute_centroids_for_person_sync(
            db=db_session,
            qdrant=qdrant,
            centroid_qdrant=centroid_qdrant,
            person_id=person.id,
            force_rebuild=True,  # Force rebuild all
            trim_outliers=True,
        )
        if result:
            rebuilt += 1

    return {"total_persons": len(persons), "centroids_rebuilt": rebuilt}
```

This can be run as a one-time job after deployment. It is non-destructive (old centroids are marked as `deprecated`, not deleted).

### PersonPrototype CENTROID Records

Centroids created by `FaceAssigner.compute_person_centroids()` (Path C) were stored as `PersonPrototype` records in the `faces` collection. After this fix:
- These records should be identified and either migrated to `PersonCentroid` records or deleted.
- Query: `SELECT * FROM person_prototypes WHERE role = 'CENTROID'`
- Safe to delete since `compute_centroids_for_person_sync()` will recreate proper centroids.

---

## 6. Verification Checklist

### Unit Tests

- [ ] Test `compute_global_centroid()` with outlier trimming produces different result than simple mean for datasets with outliers.
- [ ] Test `compute_centroids_for_person_sync()` creates a `PersonCentroid` record with correct metadata.
- [ ] Test `compute_centroids_for_person_sync()` skips computation when existing centroid has matching source hash.
- [ ] Test `compute_centroids_for_person_sync()` deprecates old centroids when creating new ones.
- [ ] Test `compute_centroids_for_person_sync()` returns None when insufficient faces.

### Integration Tests

- [ ] Run `find_more_centroid_suggestions_job` and verify the resulting centroid has `build_params.trim_outliers = True`.
- [ ] Run centroid computation via API route and via RQ job for the same person. Verify both produce identical centroid vectors.
- [ ] Verify `DualModeClusterer` assigns faces using trimmed centroids.

### Algorithm Consistency Tests

- [ ] Create a test dataset with known outliers (e.g., 19 similar faces + 1 very different face).
- [ ] Compute centroid via all four paths. Verify all produce the SAME vector.
- [ ] Verify the outlier-trimmed centroid is closer to the 19 similar faces than the simple mean.

### Manual Testing

- [ ] Trigger centroid computation via the API.
- [ ] Trigger centroid computation via Find More (centroid mode).
- [ ] Compare results: both should produce the same centroid vector for the same person.
- [ ] Verify no regression in suggestion quality.

---

## 7. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Outlier trimming changes suggestion quality | Medium | Medium | Compare suggestion results before/after for 10 persons; run A/B comparison |
| Sync wrapper introduces session management bugs | Low | High | Extensive unit tests; follow existing sync session patterns from face_jobs.py |
| Import circular dependency between services | Low | Medium | Use lazy imports inside functions (already standard pattern in codebase) |
| Rebuild job takes too long for large person collections | Low | Low | Process persons in batches; add progress tracking |
| Breaking change to FaceAssigner callers | Low | Low | Deprecation wrapper maintains backward compatibility |

### Rollback Plan

1. The sync wrapper is additive -- it does not modify the async version.
2. If issues are found, revert the call sites in `face_jobs.py`, `assigner.py`, and `dual_clusterer.py` to their inline implementations.
3. The data repair (centroid rebuild) creates new centroid versions without deleting old ones, so rollback does not lose data.

---

## 8. Estimated Effort

| Task | Effort |
|------|--------|
| Step 1: Create `compute_centroids_for_person_sync()` | 1 day |
| Step 2: Replace inline in `face_jobs.py` | 0.5 day |
| Step 3: Deprecate `FaceAssigner.compute_person_centroids()` | 0.5 day |
| Step 4: Update `DualModeClusterer` | 0.25 day |
| Data repair script | 0.25 day |
| Testing (unit + integration + algorithm consistency) | 1 day |
| Code review + QA | 0.5 day |
| **Total** | **~4 days** |

---

## 9. File Change Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `image-search-service/.../services/centroid_service.py` | Modify | Add `compute_centroids_for_person_sync()` function |
| `image-search-service/.../queue/face_jobs.py` | Modify | Replace inline centroid computation (lines 1901-2008) with call to sync service |
| `image-search-service/.../faces/assigner.py` | Modify | Replace `compute_person_centroids()` with delegation to sync service; add deprecation |
| `image-search-service/.../faces/dual_clusterer.py` | Modify | Replace inline centroid mean with call to `compute_global_centroid()` |
| `tests/unit/services/test_centroid_service.py` | Add/Modify | Tests for sync centroid pipeline and algorithm consistency |
| `scripts/rebuild_centroids.py` (optional) | Add | One-time script to rebuild all centroids with unified algorithm |
