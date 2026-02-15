# Plan 6: Cover Critical Paths with Zero Test Coverage

**Priority**: P2 (Important)
**Effort**: Large (5-7 days)
**Risk Level**: HIGH -- production-critical code has 0-13% coverage
**Depends On**: None (can be implemented independently)
**Research References**:
- `docs/research/service-testing/test-execution-analysis.md` -- Coverage numbers per file
- `docs/research/service-testing/service-code-analysis.md` -- Architecture of untested modules
- `docs/research/service-testing/devils-advocate-review.md` Section 6.3 "testability problem"

---

## 1. Problem Statement

The following production-critical modules have zero or near-zero test coverage:

| Module | Coverage | Statements Missed | Severity |
|--------|----------|-------------------|----------|
| `faces/dual_clusterer.py` | **0%** | All (~410 lines) | CRITICAL -- production face clustering |
| `faces/trainer.py` | **0%** | All (~459 lines) | CRITICAL -- training pipeline |
| `queue/face_jobs.py` | **13%** | 656 statements | CRITICAL -- all face background jobs |
| `queue/training_jobs.py` | **10%** | 312 statements | HIGH -- training session orchestrator |
| `api/routes/faces.py` | **30%** | 590 statements | HIGH -- 30+ endpoints, many untested |

These modules handle the core face recognition pipeline: detection, clustering, training, suggestion generation, and all user-facing face management endpoints. A bug in any of these paths is invisible to the test suite.

### Key Challenge: Testability Architecture

As identified in the devil's advocate review (Section 6.3), these modules have a structural testability problem:
- `DualModeClusterer` calls `get_face_qdrant_client()` internally via `_get_face_embedding()` and `save_dual_mode_results()` -- module-level singleton access, not dependency injection.
- `FaceTrainer` does the same via `_get_face_embedding()` and `get_face_qdrant_client()`.
- `face_jobs.py` functions call `get_sync_session()`, `get_face_qdrant_client()`, `FaceDetector()`, and other singletons directly.
- `training_jobs.py` functions use `EmbeddingService()`, `PIL.Image`, `ThreadPoolExecutor`, and Redis directly.

The testing strategy must use `monkeypatch` or `unittest.mock.patch` to intercept these internal calls without requiring production dependencies.

---

## 2. Implementation Plan by Module

### Module A: `faces/dual_clusterer.py` (0% -> target 70%+)

**Estimated effort**: 1 day

#### A.1 Current Architecture

```
DualModeClusterer(db_session, person_match_threshold, unknown_min_cluster_size, unknown_method, unknown_eps)
    |
    +-- cluster_all_faces(max_faces)
    |       |-- query FaceInstance from DB
    |       |-- separate labeled vs unlabeled
    |       |-- assign_to_known_people(unlabeled, labeled)
    |       |-- cluster_unknown_faces(still_unknown)
    |       |-- save_dual_mode_results(assigned, clusters)
    |
    +-- assign_to_known_people(unlabeled, labeled)
    |       |-- _get_face_embedding() x N (Qdrant calls)
    |       |-- compute person centroids (np.mean + normalize)
    |       |-- cosine similarity matching
    |       |-- threshold-based assignment
    |
    +-- cluster_unknown_faces(unknown_faces)
    |       |-- _get_face_embedding() x N (Qdrant calls)
    |       |-- _cluster_hdbscan() | _cluster_dbscan() | _cluster_agglomerative()
    |
    +-- save_dual_mode_results(assigned, clusters)
    |       |-- DB update (person_id, cluster_id)
    |       |-- Qdrant update (person_ids, cluster_ids)
    |       |-- commit
    |
    +-- _get_face_embedding(qdrant_point_id)
            |-- get_face_qdrant_client().client.retrieve()
```

#### A.2 Testing Strategy

The key dependency to mock is `get_face_qdrant_client()`. Two approaches:

**Approach 1 (Preferred): Monkeypatch at module level**
```python
@pytest.fixture
def mock_qdrant_for_clusterer(monkeypatch):
    """Inject in-memory Qdrant into dual_clusterer module."""
    client = QdrantClient(":memory:")
    client.create_collection("faces", VectorParams(size=512, distance=Distance.COSINE))

    face_qdrant = FaceQdrantClient()
    face_qdrant._client = client

    monkeypatch.setattr(
        "image_search_service.faces.dual_clusterer.get_face_qdrant_client",
        lambda: face_qdrant,
    )
    return client, face_qdrant
```

**Approach 2 (Alternative): Patch at the vector module level**
```python
with patch("image_search_service.vector.face_qdrant.get_face_qdrant_client") as mock:
    mock.return_value = face_qdrant
```

Note: The `_get_face_embedding` method (line 344-366) hardcodes `collection_name="faces"` instead of using `_get_face_collection_name()`. This is a bug that would be caught by the test (in-memory Qdrant uses `test_faces` per the test settings fixture). The test should either patch to use `"faces"` as collection name or identify this as a bug to fix.

#### A.3 Test Cases

**File**: `image-search-service/tests/faces/test_dual_clusterer.py` (new file)

| Test Function | What It Tests | Key Assertions |
|---------------|---------------|----------------|
| `test_cluster_all_faces_empty_db` | No faces in DB | Returns `{total_processed: 0, ...}` |
| `test_cluster_all_faces_all_labeled` | All faces have person_id | No unknown clustering needed |
| `test_cluster_all_faces_all_unlabeled` | No labeled faces | Skip supervised, go to unsupervised |
| `test_assign_to_known_people_above_threshold` | Unlabeled face close to person centroid | Face assigned to correct person |
| `test_assign_to_known_people_below_threshold` | Unlabeled face far from all persons | Face stays in `still_unknown` |
| `test_assign_to_known_people_no_embedding` | Qdrant returns None for face | Face goes to `still_unknown` |
| `test_cluster_unknown_faces_hdbscan` | 10+ unlabeled faces, two natural clusters | Two clusters created, noise handled |
| `test_cluster_unknown_faces_dbscan` | Same as above with method="dbscan" | Clusters created |
| `test_cluster_unknown_faces_agglomerative` | Same as above with method="agglomerative" | Clusters created |
| `test_cluster_unknown_faces_too_few` | Fewer faces than min_cluster_size | All labeled as noise |
| `test_cluster_unknown_faces_invalid_method` | method="invalid" | Raises `ValueError` |
| `test_save_dual_mode_results_assigned` | 3 faces assigned to 2 people | DB `person_id` updated, Qdrant `person_id` updated |
| `test_save_dual_mode_results_clusters` | 5 faces in 2 clusters | DB `cluster_id` updated, Qdrant `cluster_id` updated |
| `test_save_dual_mode_results_commits` | Any results | `db.commit()` called |
| `test_full_dual_mode_pipeline` | Mix of labeled and unlabeled faces | End-to-end: assign + cluster + save |

**Embedding setup for tests**: Pre-populate the in-memory Qdrant with known embeddings for test faces. Use numpy to generate clusters:

```python
def make_clustered_embeddings(n_per_cluster=5, n_clusters=3, dim=512):
    """Generate embeddings with known cluster structure for testing."""
    embeddings = []
    labels = []
    for cluster_idx in range(n_clusters):
        center = np.random.randn(dim)
        center = center / np.linalg.norm(center)
        for _ in range(n_per_cluster):
            noise = np.random.randn(dim) * 0.05  # Small noise
            emb = center + noise
            emb = emb / np.linalg.norm(emb)  # Re-normalize
            embeddings.append(emb)
            labels.append(cluster_idx)
    return embeddings, labels
```

---

### Module B: `faces/trainer.py` (0% -> target 80%+)

**Estimated effort**: 1 day

#### B.1 Current Architecture

```
TripletFaceDataset(embeddings_by_person, triplets_per_person=100)
    |
    +-- generate_triplets() -> list[(anchor, positive, negative)]
    |       |-- for each person: sample anchor, sample positive (same person)
    |       |-- _select_hard_negative(anchor, person_id) -> most similar from other person
    |
    +-- _select_hard_negative(anchor, anchor_person_id)
            |-- iterate all other persons' embeddings
            |-- compute cosine similarity
            |-- return most similar (hardest negative)

FaceTrainer(db_session, margin=0.2, epochs=20, batch_size=32, learning_rate=0.0001)
    |
    +-- get_labeled_faces_by_person() -> {person_id: [{face_id, qdrant_point_id, embedding}]}
    |       |-- query FaceInstance WHERE person_id IS NOT NULL
    |       |-- _get_face_embedding() for each (Qdrant call)
    |
    +-- fine_tune_for_person_clustering(min_faces_per_person=5, checkpoint_path=None)
    |       |-- get_labeled_faces_by_person()
    |       |-- filter persons with >= min_faces
    |       |-- generate triplets via TripletFaceDataset
    |       |-- compute triplet loss per batch per epoch
    |       |-- save_checkpoint() if path provided
    |       |-- return {epochs, final_loss, persons_trained, triplets_used}
    |
    +-- compute_triplet_loss(anchor, positive, negative) -> float
    |       |-- cosine_distance(anchor, positive)
    |       |-- cosine_distance(anchor, negative)
    |       |-- max(0, d_ap - d_an + margin)
    |
    +-- save_checkpoint(path, epoch, loss)
    +-- load_checkpoint(path) -> dict
    |
    +-- _get_face_embedding(qdrant_point_id) -> np.array | None
            |-- get_face_qdrant_client().client.retrieve()
```

#### B.2 Test Cases

**File**: `image-search-service/tests/faces/test_trainer.py` (new file)

**TripletFaceDataset tests** (pure numpy, no mocking needed):

| Test Function | What It Tests | Key Assertions |
|---------------|---------------|----------------|
| `test_generate_triplets_basic` | 3 persons, 5 faces each | Returns list of (anchor, positive, negative) tuples |
| `test_generate_triplets_count` | triplets_per_person=10, 3 persons | ~30 triplets generated |
| `test_generate_triplets_skip_single_face_person` | Person with 1 face | That person skipped, warning logged |
| `test_generate_triplets_two_persons_min` | Only 2 persons | Triplets generated (need at least 2) |
| `test_generate_triplets_single_person` | Only 1 person | No triplets (no negatives available) |
| `test_select_hard_negative_returns_most_similar` | Known embeddings with calculable similarity | Returns the most similar face from different person |
| `test_select_hard_negative_no_other_persons` | Only 1 person | Returns None |
| `test_triplet_shapes` | Generated triplets | Each element is correct shape (dim,) |

**FaceTrainer tests** (require mocking Qdrant + DB):

| Test Function | What It Tests | Key Assertions |
|---------------|---------------|----------------|
| `test_compute_triplet_loss_satisfied` | d(a,p) + margin < d(a,n) | Loss is 0.0 |
| `test_compute_triplet_loss_violated` | d(a,p) + margin > d(a,n) | Loss > 0.0 |
| `test_compute_triplet_loss_identical_anchor_positive` | anchor == positive | d(a,p) = 0, loss = max(0, margin - d(a,n)) |
| `test_compute_triplet_loss_margin_effect` | Same embeddings, different margins | Higher margin -> higher loss |
| `test_get_labeled_faces_by_person_empty_db` | No labeled faces | Returns empty dict |
| `test_get_labeled_faces_by_person_groups_correctly` | 3 persons, multiple faces | Grouped by person_id |
| `test_fine_tune_no_labeled_faces` | Empty DB | Returns {epochs: 0, ...} |
| `test_fine_tune_insufficient_faces` | All persons have < min_faces | Returns {epochs: 0, persons_trained: 0} |
| `test_fine_tune_success` | 3 persons, 10 faces each | Returns {epochs: 20, final_loss: float, persons_trained: 3} |
| `test_fine_tune_saves_checkpoint` | checkpoint_path provided | JSON file created with correct content |
| `test_save_checkpoint_creates_directory` | Non-existent parent dir | Directory created, file written |
| `test_load_checkpoint_success` | Valid checkpoint file | Returns dict with epoch, loss, margin |
| `test_load_checkpoint_missing_file` | Non-existent path | Returns empty dict |

**Fixture for trainer tests**:

```python
@pytest.fixture
def trainer_with_mock_qdrant(sync_db_session, monkeypatch):
    """Create FaceTrainer with mocked Qdrant client."""
    client = QdrantClient(":memory:")
    client.create_collection("faces", VectorParams(size=512, distance=Distance.COSINE))

    face_qdrant = FaceQdrantClient()
    face_qdrant._client = client

    monkeypatch.setattr(
        "image_search_service.faces.trainer.get_face_qdrant_client",
        lambda: face_qdrant,
    )

    trainer = FaceTrainer(
        db_session=sync_db_session,
        margin=0.2,
        epochs=5,  # Fewer epochs for fast tests
        batch_size=8,
    )
    return trainer, client, face_qdrant
```

**Helper to populate test data**:

```python
def populate_labeled_faces(db_session, qdrant_client, persons_config):
    """Create labeled faces in DB + Qdrant for training tests.

    Args:
        persons_config: list of (person_name, n_faces, base_embedding)
    """
    for name, n_faces, base_embedding in persons_config:
        person = Person(name=name)
        db_session.add(person)
        db_session.flush()

        for i in range(n_faces):
            point_id = uuid.uuid4()
            # Add noise to base embedding
            emb = base_embedding + np.random.randn(512) * 0.05
            emb = emb / np.linalg.norm(emb)

            face = FaceInstance(
                asset_id=...,  # Need to create ImageAsset first
                bbox_x=i*10, bbox_y=0, bbox_w=50, bbox_h=50,
                detection_confidence=0.95,
                qdrant_point_id=point_id,
                person_id=person.id,
            )
            db_session.add(face)

            qdrant_client.upsert(
                collection_name="faces",
                points=[PointStruct(id=str(point_id), vector=emb.tolist(), payload={})],
            )

        db_session.flush()
    db_session.commit()
```

---

### Module C: `queue/face_jobs.py` (13% -> target 40%+)

**Estimated effort**: 2 days

This is the largest untested module (2127 lines, 656 missed statements, 15+ job functions). Prioritize by production usage and risk.

#### C.1 Priority Ranking of Job Functions

| Priority | Function | Lines | Risk | Reason |
|----------|----------|-------|------|--------|
| **P1** | `detect_faces_for_session_job` | ~700 | CRITICAL | Core face detection pipeline with pause/cancel/resume |
| **P1** | `cluster_dual_job` | ~50 | HIGH | Wrapper that creates DualModeClusterer |
| **P1** | `propagate_person_label_multiproto_job` | ~250 | HIGH | Suggestion generation, 8/12 tests broken |
| **P2** | `train_person_matching_job` | ~50 | MEDIUM | Training pipeline trigger |
| **P2** | `compute_centroids_job` | ~30 | MEDIUM | Centroid computation |
| **P2** | `find_more_centroid_suggestions_job` | ~100 | MEDIUM | Centroid-based suggestions |
| **P3** | `expire_old_suggestions_job` | ~60 | LOW | Cleanup job |
| **P3** | `cleanup_orphaned_suggestions_job` | ~60 | LOW | Cleanup job |
| **P3** | `backfill_faces_job` | ~50 | LOW | Batch backfill |

#### C.2 Testing Strategy: Monkepatching Internal Dependencies

All face_jobs functions call `get_sync_session()` for database access and various singleton functions internally. The testing approach:

```python
@pytest.fixture
def face_job_fixtures(sync_db_session, monkeypatch):
    """Set up all dependencies for face_jobs testing."""
    # Mock database session
    monkeypatch.setattr(
        "image_search_service.queue.face_jobs.get_sync_session",
        lambda: contextmanager(lambda: (yield sync_db_session))(),
    )

    # Mock Qdrant
    qdrant_client = QdrantClient(":memory:")
    qdrant_client.create_collection("faces", VectorParams(size=512, distance=Distance.COSINE))
    face_qdrant = FaceQdrantClient()
    face_qdrant._client = qdrant_client
    monkeypatch.setattr(
        "image_search_service.queue.face_jobs.get_face_qdrant_client",
        lambda: face_qdrant,
    )

    # Mock Redis (for progress tracking)
    mock_redis = MagicMock()
    monkeypatch.setattr(
        "image_search_service.queue.face_jobs.redis.Redis.from_url",
        lambda *a, **kw: mock_redis,
    )

    # Mock FaceDetector (InsightFace)
    mock_detector = MagicMock()
    mock_detector.detect_faces.return_value = [
        {"bbox": [10, 20, 60, 70], "det_score": 0.95, "embedding": np.random.randn(512).tolist()},
    ]
    monkeypatch.setattr(
        "image_search_service.queue.face_jobs.FaceDetector",
        lambda **kwargs: mock_detector,
    )

    return {
        "db_session": sync_db_session,
        "qdrant_client": qdrant_client,
        "face_qdrant": face_qdrant,
        "mock_redis": mock_redis,
        "mock_detector": mock_detector,
    }
```

#### C.3 Test Cases by Function

**File**: `image-search-service/tests/unit/queue/test_face_jobs_coverage.py` (new file)

**detect_faces_for_session_job** (P1, ~10 tests):

| Test Function | What It Tests | Key Assertions |
|---------------|---------------|----------------|
| `test_detect_faces_session_not_found` | Invalid session_id | Returns error result, no crash |
| `test_detect_faces_session_already_completed` | Session status=completed | Skips processing, returns early |
| `test_detect_faces_basic_flow` | 3 images, 1 face each | 3 faces detected, session completed |
| `test_detect_faces_no_faces_detected` | Image with no faces | processed_images incremented, faces_detected=0 |
| `test_detect_faces_cancellation` | Cancel during processing | Session status=cancelled, partial results saved |
| `test_detect_faces_pause_resume` | Pause then resume | Resumes from correct asset index |
| `test_detect_faces_auto_assignment` | Known person exists | Faces auto-assigned to matching person |
| `test_detect_faces_progress_tracking` | Multiple batches | Redis progress updated per batch |
| `test_detect_faces_error_handling` | Detector throws on one image | That image failed, others processed |
| `test_detect_faces_suggestion_generation` | Post-detection suggestions | Suggestions created for unassigned faces |

**cluster_dual_job** (P1, ~3 tests):

| Test Function | What It Tests |
|---------------|---------------|
| `test_cluster_dual_job_basic` | Calls DualModeClusterer with correct params |
| `test_cluster_dual_job_custom_params` | Custom thresholds passed through |
| `test_cluster_dual_job_empty_db` | No faces, returns zero counts |

**propagate_person_label_multiproto_job** (P1, ~5 tests):

| Test Function | What It Tests |
|---------------|---------------|
| `test_propagate_creates_suggestions` | Face near known person generates suggestion |
| `test_propagate_respects_threshold` | Below threshold, no suggestion |
| `test_propagate_multi_prototype_scoring` | Multiple prototypes, aggregate score computed |
| `test_propagate_skip_already_assigned` | Faces with person_id skipped |
| `test_propagate_handles_no_prototypes` | No prototypes for person, no crash |

**compute_centroids_job** (P2, ~3 tests):

| Test Function | What It Tests |
|---------------|---------------|
| `test_compute_centroids_basic` | Creates centroids for persons with faces |
| `test_compute_centroids_insufficient_faces` | Person with < min_faces, skipped |
| `test_compute_centroids_updates_existing` | Re-computation bumps version |

**expire_old_suggestions_job / cleanup_orphaned_suggestions_job** (P3, ~4 tests):

| Test Function | What It Tests |
|---------------|---------------|
| `test_expire_old_suggestions` | Suggestions older than threshold expired |
| `test_expire_no_old_suggestions` | Nothing to expire, no crash |
| `test_cleanup_orphaned_no_person` | Suggestion where person deleted |
| `test_cleanup_orphaned_no_face` | Suggestion where face_instance deleted |

---

### Module D: `queue/training_jobs.py` (10% -> target 40%+)

**Estimated effort**: 1 day

#### D.1 Current Architecture

```
train_session(session_id, batch_size=32)  [lines 129-319]
    |-- get session from DB
    |-- discover pending jobs (TrainingJob records)
    |-- process in batches via train_batch()
    |-- handle cancellation/pause
    |-- update session status
    |-- auto-trigger face detection on completion

train_batch(jobs, session_id, batch_size=32)  [lines 322-653]
    |-- Producer thread: load images via PIL (with _image_load_lock)
    |-- Consumer: batch GPU embedding via EmbeddingService.embed_images_batch()
    |-- Qdrant upsert for embeddings
    |-- TrainingEvidence creation
    |-- Periodic GC for MPS memory

train_single_asset(job_id, session_id)  [lines 655-805]
    |-- Load image via PIL
    |-- embed_image() via EmbeddingService
    |-- Qdrant upsert
    |-- TrainingEvidence creation
    |-- _build_evidence_metadata()

_build_evidence_metadata(image_path, processing_time_ms, ...)  [lines 808-886]
    |-- File stat (size, modified_at)
    |-- EXIF extraction
    |-- SHA256 hash
    |-- Return metadata dict
```

#### D.2 Testing Strategy

Training jobs depend on PIL, EmbeddingService, Qdrant, Redis, and the filesystem. Mock all external dependencies:

```python
@pytest.fixture
def training_job_fixtures(sync_db_session, tmp_path, monkeypatch):
    """Set up dependencies for training_jobs testing."""
    # Mock sync session
    monkeypatch.setattr(
        "image_search_service.queue.training_jobs.get_sync_session",
        lambda: contextmanager(lambda: (yield sync_db_session))(),
    )

    # Mock embedding service
    mock_embed = MagicMock()
    mock_embed.embed_image.return_value = [0.1] * 512
    mock_embed.embed_images_batch.return_value = [[0.1] * 512]
    mock_embed.embed_image_from_pil.return_value = [0.1] * 512
    mock_embed.embedding_dim = 512
    monkeypatch.setattr(
        "image_search_service.queue.training_jobs.get_embedding_service",
        lambda: mock_embed,
    )

    # Mock Qdrant
    mock_qdrant = MagicMock()
    monkeypatch.setattr(
        "image_search_service.queue.training_jobs.get_qdrant_client",
        lambda: mock_qdrant,
    )

    # Create test images on disk
    test_images = {}
    for name in ["photo1.jpg", "photo2.jpg", "photo3.jpg"]:
        img_path = tmp_path / name
        Image.new("RGB", (100, 100)).save(img_path)
        test_images[name] = img_path

    return {
        "db_session": sync_db_session,
        "mock_embed": mock_embed,
        "mock_qdrant": mock_qdrant,
        "test_images": test_images,
        "tmp_path": tmp_path,
    }
```

#### D.3 Test Cases

**File**: `image-search-service/tests/unit/queue/test_training_jobs_coverage.py` (new file)

**train_session** (~6 tests):

| Test Function | What It Tests |
|---------------|---------------|
| `test_train_session_not_found` | Invalid session_id, returns error |
| `test_train_session_no_pending_jobs` | Session exists but no pending TrainingJobs |
| `test_train_session_basic_flow` | 3 pending jobs, all complete successfully |
| `test_train_session_cancellation` | Session cancelled mid-batch |
| `test_train_session_pause` | Session paused, resumes from correct position |
| `test_train_session_partial_failure` | 1 of 3 jobs fails, others succeed |

**train_single_asset** (~5 tests):

| Test Function | What It Tests |
|---------------|---------------|
| `test_train_single_asset_success` | Image embedded, Qdrant upserted, evidence created |
| `test_train_single_asset_file_not_found` | Missing image file, job marked failed |
| `test_train_single_asset_corrupt_image` | PIL cannot open, job marked failed |
| `test_train_single_asset_embedding_error` | EmbeddingService throws, job marked failed |
| `test_train_single_asset_evidence_metadata` | Verify TrainingEvidence fields populated |

**_build_evidence_metadata** (~4 tests, pure function, no mocking needed):

| Test Function | What It Tests |
|---------------|---------------|
| `test_build_evidence_metadata_basic` | Returns dict with expected keys |
| `test_build_evidence_metadata_with_exif` | JPEG with EXIF data, metadata includes EXIF |
| `test_build_evidence_metadata_no_exif` | PNG without EXIF, metadata has null EXIF fields |
| `test_build_evidence_metadata_file_stats` | File size and modification time captured |

---

### Module E: `api/routes/faces.py` (30% -> target 50%+)

**Estimated effort**: 1-2 days

#### E.1 Untested Endpoints Audit

The faces route file has 30+ endpoints. From grep output and the existing `test_faces_routes.py`, the following endpoints lack test coverage:

| Endpoint | Method | Coverage Status |
|----------|--------|-----------------|
| `POST /faces/cluster/dual` | POST | No tests |
| `POST /faces/train` | POST | No tests |
| `POST /faces/persons/{id}/merge` | POST | Minimal tests |
| `POST /faces/persons/{id}/photos/bulk-remove` | POST | No tests |
| `POST /faces/persons/{id}/photos/bulk-move` | POST | No tests |
| `DELETE /faces/faces/{id}/person` | DELETE | No tests |
| `GET /faces/faces/{id}/suggestions` | GET | Tested separately |
| `POST /faces/detect/{asset_id}` | POST | No route tests |
| `POST /faces/cluster` | POST | No route tests |
| `GET /faces/persons/{id}/photos` | GET | Partial |
| Prototype management endpoints | Various | 2 broken tests |
| Suggestion regeneration endpoints | Various | Separate files |

#### E.2 Test Cases for Untested Endpoints

**File**: `image-search-service/tests/api/test_faces_routes_extended.py` (new file)

The existing `test_faces_routes.py` has ~35 tests. Add coverage for untested endpoints:

| Test Function | Endpoint | What It Tests |
|---------------|----------|---------------|
| `test_cluster_dual_enqueues_job` | `POST /cluster/dual` | Job enqueued with correct params |
| `test_cluster_dual_custom_params` | `POST /cluster/dual` | Custom thresholds passed |
| `test_train_enqueues_job` | `POST /train` | Training job enqueued |
| `test_train_custom_params` | `POST /train` | Custom epochs/margin/lr passed |
| `test_merge_persons` | `POST /persons/{id}/merge` | Two persons merged, faces reassigned |
| `test_merge_persons_not_found` | `POST /persons/{id}/merge` | 404 for missing person |
| `test_merge_persons_self_merge` | `POST /persons/{id}/merge` | 400 cannot merge with self |
| `test_bulk_remove_faces` | `POST /persons/{id}/photos/bulk-remove` | Faces unassigned from person |
| `test_bulk_remove_faces_empty_list` | `POST /persons/{id}/photos/bulk-remove` | 400 or no-op |
| `test_bulk_move_faces` | `POST /persons/{id}/photos/bulk-move` | Faces moved to target person |
| `test_bulk_move_faces_target_not_found` | `POST /persons/{id}/photos/bulk-move` | 404 for missing target |
| `test_unassign_face_from_person` | `DELETE /faces/{id}/person` | face.person_id set to NULL |
| `test_unassign_face_not_found` | `DELETE /faces/{id}/person` | 404 |
| `test_detect_single_asset` | `POST /detect/{asset_id}` | Detection result returned |
| `test_detect_single_asset_not_found` | `POST /detect/{asset_id}` | 404 |
| `test_cluster_basic` | `POST /cluster` | Clustering enqueued |
| `test_person_photos_pagination` | `GET /persons/{id}/photos` | Offset/limit work correctly |
| `test_person_photos_temporal_grouping` | `GET /persons/{id}/photos` | Photos grouped by date |

**Setup pattern** for these tests (use existing `test_client` fixture):

```python
@pytest.mark.asyncio
async def test_bulk_move_faces(test_client, db_session):
    """Test POST /api/v1/faces/persons/{id}/photos/bulk-move."""
    # Setup: create 2 persons, 3 faces assigned to person1
    person1 = Person(name="Source Person")
    person2 = Person(name="Target Person")
    db_session.add_all([person1, person2])
    await db_session.flush()

    asset = ImageAsset(path="/test/photo.jpg")
    db_session.add(asset)
    await db_session.flush()

    faces = []
    for i in range(3):
        face = FaceInstance(
            asset_id=asset.id,
            bbox_x=i*100, bbox_y=0, bbox_w=50, bbox_h=50,
            detection_confidence=0.9,
            qdrant_point_id=uuid.uuid4(),
            person_id=person1.id,
        )
        faces.append(face)
    db_session.add_all(faces)
    await db_session.commit()

    # Act
    response = await test_client.post(
        f"/api/v1/faces/persons/{person1.id}/photos/bulk-move",
        json={
            "face_instance_ids": [str(f.id) for f in faces[:2]],
            "target_person_id": str(person2.id),
        },
    )

    # Assert
    assert response.status_code == 200
    data = response.json()
    assert data["moved_count"] == 2
```

---

## 3. Test Count Summary

| Module | New Tests | Target Coverage |
|--------|-----------|----------------|
| `faces/dual_clusterer.py` | 15 | 70%+ |
| `faces/trainer.py` | 21 | 80%+ |
| `queue/face_jobs.py` | 25 | 40%+ |
| `queue/training_jobs.py` | 15 | 40%+ |
| `api/routes/faces.py` | 18 | 50%+ |
| **Total** | **~94** | |

---

## 4. Implementation Order

The recommended implementation order optimizes for quick wins and dependency chains:

```
Day 1: trainer.py tests (pure math tests first, then mocked)
    -> TripletFaceDataset tests are pure numpy (no mocking)
    -> compute_triplet_loss tests are pure math
    -> Remaining trainer tests need Qdrant mock fixture

Day 2: dual_clusterer.py tests
    -> Depends on same Qdrant mock fixture as trainer
    -> Clustering tests need sklearn (already a dependency)

Day 3-4: face_jobs.py tests (largest module, most mocking needed)
    -> Start with simple jobs (expire, cleanup)
    -> Then cluster_dual_job (wraps dual_clusterer)
    -> Then detect_faces_for_session_job (most complex)

Day 5: training_jobs.py tests
    -> _build_evidence_metadata is pure (test first)
    -> train_single_asset needs mock embedding service
    -> train_session needs everything mocked

Day 6: faces routes tests
    -> Use existing test_client fixture
    -> Mostly HTTP-level testing with DB setup

Day 7: Buffer for debugging, edge cases, coverage analysis
```

---

## 5. Shared Test Fixtures

Create a shared fixture file for face-pipeline testing:

**File**: `image-search-service/tests/faces/conftest_coverage.py` (new or merged into existing `conftest.py`)

```python
"""Shared fixtures for face pipeline coverage tests."""

import numpy as np
import uuid
from contextlib import contextmanager
from unittest.mock import MagicMock

import pytest
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, PointStruct, VectorParams

from image_search_service.db.models import FaceInstance, ImageAsset, Person
from image_search_service.vector.face_qdrant import FaceQdrantClient


@pytest.fixture
def populated_face_db(sync_db_session):
    """Create a database with persons and face instances for testing.

    Creates:
    - 3 persons (Alice, Bob, Carol) with 5 faces each
    - 5 unassigned faces (no person_id)
    - All faces have qdrant_point_ids set
    """
    persons = {}
    all_faces = []

    # Create image asset
    asset = ImageAsset(path="/test/photo.jpg")
    sync_db_session.add(asset)
    sync_db_session.flush()

    for name in ["Alice", "Bob", "Carol"]:
        person = Person(name=name)
        sync_db_session.add(person)
        sync_db_session.flush()
        persons[name] = person

        for i in range(5):
            face = FaceInstance(
                asset_id=asset.id,
                bbox_x=i * 100, bbox_y=0, bbox_w=50, bbox_h=50,
                detection_confidence=0.95,
                qdrant_point_id=uuid.uuid4(),
                person_id=person.id,
            )
            sync_db_session.add(face)
            all_faces.append(face)

    # Unassigned faces
    for i in range(5):
        face = FaceInstance(
            asset_id=asset.id,
            bbox_x=500 + i * 100, bbox_y=0, bbox_w=50, bbox_h=50,
            detection_confidence=0.90,
            qdrant_point_id=uuid.uuid4(),
            person_id=None,
        )
        sync_db_session.add(face)
        all_faces.append(face)

    sync_db_session.commit()
    return {"persons": persons, "faces": all_faces, "asset": asset}


@pytest.fixture
def face_qdrant_with_embeddings(populated_face_db):
    """Create in-memory Qdrant populated with embeddings matching the DB faces."""
    client = QdrantClient(":memory:")
    client.create_collection("faces", VectorParams(size=512, distance=Distance.COSINE))

    # Generate clustered embeddings per person
    person_centers = {}
    rng = np.random.RandomState(42)  # Reproducible

    for name, person in populated_face_db["persons"].items():
        center = rng.randn(512).astype(np.float64)
        center = center / np.linalg.norm(center)
        person_centers[name] = center

    for face in populated_face_db["faces"]:
        if face.person_id:
            # Find person name
            person_name = next(
                name for name, p in populated_face_db["persons"].items()
                if p.id == face.person_id
            )
            base = person_centers[person_name]
            emb = base + rng.randn(512) * 0.05
        else:
            emb = rng.randn(512)

        emb = emb / np.linalg.norm(emb)

        client.upsert(
            collection_name="faces",
            points=[PointStruct(
                id=str(face.qdrant_point_id),
                vector=emb.tolist(),
                payload={"person_id": str(face.person_id) if face.person_id else None},
            )],
        )

    face_qdrant = FaceQdrantClient()
    face_qdrant._client = client
    return client, face_qdrant, populated_face_db
```

---

## 6. Verification Checklist

- [ ] `tests/faces/test_dual_clusterer.py` -- 15 tests passing
- [ ] `tests/faces/test_trainer.py` -- 21 tests passing
- [ ] `tests/unit/queue/test_face_jobs_coverage.py` -- 25 tests passing
- [ ] `tests/unit/queue/test_training_jobs_coverage.py` -- 15 tests passing
- [ ] `tests/api/test_faces_routes_extended.py` -- 18 tests passing
- [ ] All existing tests still pass (`make test`)
- [ ] No real GPU models loaded during tests (no InsightFace/OpenCLIP import)
- [ ] No external services required (Qdrant in-memory, SQLite, no Redis)
- [ ] Coverage for `dual_clusterer.py` reaches 70%+
- [ ] Coverage for `trainer.py` reaches 80%+
- [ ] Coverage for `face_jobs.py` reaches 40%+
- [ ] Coverage for `training_jobs.py` reaches 40%+
- [ ] Coverage for `api/routes/faces.py` reaches 50%+

---

## 7. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Monkepatching breaks when code refactored | Use `monkeypatch.setattr` with full module path; add comment documenting patched path |
| Qdrant in-memory still uses 512-dim | Acceptable for unit tests; PostgreSQL tier (Plan 5) validates dimensions |
| face_jobs.py functions too tightly coupled | Start with higher-level tests (job entry point), add unit tests where feasible |
| hdbscan import not available in test env | Already a project dependency; if missing, test skips with `pytest.importorskip("hdbscan")` |
| Redis mock doesn't validate operations | Acceptable -- Redis is used for progress caching, not critical state |
| Tests may be slow due to numpy operations | Use small dimensions (512) and small datasets (5-20 faces per test) |

---

## 8. Success Criteria

1. Overall test coverage increases from 48% to 55%+ (estimated ~800 new covered statements)
2. No production-critical module remains at 0% coverage
3. Face clustering, training, and suggestion generation have basic happy-path coverage
4. All new tests run in under 30 seconds without external dependencies
5. Test failures surface real bugs (not mock behavior)
