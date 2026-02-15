# Plan 7: Fix Mock Accuracy and Fidelity

**Priority**: P3 (Recommended)
**Effort**: Medium (2-3 days)
**Risk Level**: MEDIUM -- structural issue that masks real bugs
**Depends On**: None (can be implemented independently, but benefits from Plan 6 test fixtures)
**Research References**:
- `docs/research/service-testing/devils-advocate-review.md` Section 4 "Mock Accuracy"
- `docs/research/service-testing/test-case-analysis.md` Section 1.1 "MockEmbeddingService"
- `docs/research/service-testing/test-execution-analysis.md` Category B "Mock Mismatch"

---

## 1. Problem Statement

The test mocks create a false sense of confidence by deviating from production behavior in four specific ways:

### 1.1 Embedding Dimension Mismatch (512 vs 768)

**Production** (from `src/image_search_service/services/embedding.py` line 130-133):
```python
@property
def embedding_dim(self) -> int:
    model, _, _ = _load_model()
    return int(model.visual.output_dim)  # 768 for ViT-B-32 (OpenCLIP)
```

**Production SigLIP** (from `src/image_search_service/services/siglip_embedding.py` line 1-9):
```
SigLIP: 768-dimensional embeddings
```

**Config** (from `src/image_search_service/core/config.py` lines 50-55):
```python
embedding_dim: int = Field(default=512, alias="EMBEDDING_DIM")         # CLIP (legacy)
siglip_embedding_dim: int = Field(default=768, alias="SIGLIP_EMBEDDING_DIM")  # SigLIP
```

**Mock** (from `tests/conftest.py` lines 68-74):
```python
class MockEmbeddingService:
    @property
    def embedding_dim(self) -> int:
        return 512  # WRONG: production CLIP model outputs 768-dim
```

**Qdrant test collections** (from `tests/conftest.py` lines 184-198):
```python
client.create_collection(
    collection_name=settings.qdrant_collection,
    vectors_config=VectorParams(size=512, distance=Distance.COSINE),  # Matches mock, not production
)
```

**Impact**: If any code hardcodes or validates embedding dimensions, the dimension mismatch is invisible to tests. For example, the face pipeline uses 512-dim ArcFace/InsightFace embeddings (correct for faces), while the image search pipeline uses 768-dim CLIP/SigLIP embeddings. The mock conflates these two different systems by using 512 for both.

**Clarification on face vs image dimensions**:
- **Face embeddings** (InsightFace/ArcFace): 512-dim -- this is correct and unchanged
- **Image search embeddings** (OpenCLIP/SigLIP): 768-dim -- the mock incorrectly uses 512

From `src/image_search_service/vector/face_qdrant.py` line 34:
```python
FACE_VECTOR_DIM = 512  # ArcFace/InsightFace embedding dimension (correct)
```

From `src/image_search_service/core/config.py` line 50:
```python
embedding_dim: int = Field(default=512, alias="EMBEDDING_DIM")  # Legacy CLIP config
siglip_embedding_dim: int = Field(default=768, alias="SIGLIP_EMBEDDING_DIM")  # Production
```

### 1.2 MD5-Based Embeddings Prevent Meaningful Similarity Testing

**Current mock** (from `tests/conftest.py` lines 76-96):
```python
def embed_text(self, text: str) -> list[float]:
    h = hashlib.md5(text.encode()).hexdigest()
    vector = []
    for i in range(512):
        idx = (i * 2) % len(h)
        val = int(h[idx : idx + 2], 16) / 255.0
        vector.append(val)
    return vector
```

**Problems**:
1. MD5 hashes produce vectors with **no semantic relationship**. "sunset on the beach" and "sunset" do not produce similar vectors because MD5 is designed to be unpredictable.
2. The vectors cycle through 32 hex characters repeated 16 times to fill 512 dimensions. This creates artificial periodic patterns in cosine similarity that bear no relationship to semantic similarity.
3. Vectors are **not normalized** to unit length. The values are in `[0, 1]` range but the vector is not L2-normalized. Production CLIP/SigLIP embeddings are always L2-normalized. Cosine similarity calculations on unnormalized vectors produce different results.
4. `embed_image` delegates to `embed_text(str(path))`, meaning the image content is completely ignored -- only the file path matters. Two identical images at different paths produce different embeddings.

**Impact**: Every search test that asserts "query X returns result Y" is testing Qdrant's HTTP plumbing, not search quality. As the devil's advocate review noted (Section 1.1): "The test asserts `first_result["asset"]["path"] in ["/test/sunset.jpg", "/test/mountain.jpg"]` -- it doesn't even assert the correct result is ranked first."

### 1.3 Qdrant In-Memory Client Missing Behavioral Fidelity

From `tests/conftest.py` lines 176-199:
```python
def qdrant_client() -> QdrantClient:
    client = QdrantClient(":memory:")
    client.create_collection(
        collection_name=settings.qdrant_collection,
        vectors_config=VectorParams(size=512, distance=Distance.COSINE),
    )
    client.create_collection(
        collection_name=settings.qdrant_face_collection,
        vectors_config=VectorParams(size=512, distance=Distance.COSINE),
    )
    return client
```

**Missing from test setup**:
1. **No payload indexes created**. Production Qdrant has payload indexes on `person_id`, `cluster_id`, `is_prototype`, `asset_id`, `face_instance_id` (from `face_qdrant.py` lines 122-156). Tests skip index creation, so any payload-filtering performance issues or index-dependent behavior is untested.
2. **Collection dimensions don't match production**. The image assets collection should be 768-dim (CLIP/SigLIP) in production, but tests create it as 512-dim.

### 1.4 Two Tests Load Real GPU Models

From the test execution analysis (Category B):
- `test_process_assets_batch` in `tests/faces/test_service.py`
- `test_process_assets_batch_parallel_loading` in `tests/faces/test_service.py`

These tests load the real InsightFace model (`buffalo_l`), taking ~7 seconds and requiring model files on disk. They violate the project rule "Tests must pass without Postgres/Redis/Qdrant running" and are environment-dependent.

---

## 2. Solution Design

### 2.1 Fix Embedding Dimension to Match Production

Create two mock embedding services: one for image search (768-dim to match CLIP/SigLIP) and one for faces (512-dim to match InsightFace/ArcFace). This accurately represents the production architecture where these are different embedding models with different dimensions.

### 2.2 Replace MD5 Hashing with Semantic-Aware Deterministic Embeddings

Use a pre-computed lookup table of "semantic clusters" that produce meaningful similarity relationships. This enables tests to assert search ranking correctness, not just "something was returned."

### 2.3 Add Payload Indexes to Test Qdrant Setup

Mirror the production `ensure_collection()` behavior in test fixtures.

### 2.4 Fix the Two Tests Loading Real GPU Models

Correct the mock target so InsightFace is not loaded.

---

## 3. Implementation Tasks

### Task 1: Create `SemanticMockEmbeddingService` (2-3 hours)

**File**: `image-search-service/tests/conftest.py` (modify existing `MockEmbeddingService`)

Replace the current MD5-based mock with a semantic-aware mock:

```python
class SemanticMockEmbeddingService:
    """Mock embedding service with meaningful semantic similarity.

    Uses predefined "concept clusters" to generate embeddings where:
    - Semantically similar texts produce similar vectors (high cosine similarity)
    - Unrelated texts produce dissimilar vectors (low cosine similarity)
    - Results are deterministic (same input always produces same output)
    - Vectors are L2-normalized (matches production CLIP/SigLIP behavior)

    Dimension: 768 to match production CLIP/SigLIP model output.
    """

    # Concept clusters: words that should produce similar embeddings
    CONCEPT_CLUSTERS = {
        "nature": ["sunset", "beach", "ocean", "mountain", "forest", "lake", "sky",
                    "sunrise", "landscape", "outdoors", "nature", "tree", "river"],
        "animal": ["dog", "cat", "bird", "fish", "animal", "puppy", "kitten",
                    "horse", "pet", "wildlife"],
        "food": ["pizza", "burger", "sushi", "food", "meal", "restaurant",
                 "cooking", "kitchen", "chef", "recipe"],
        "urban": ["city", "building", "street", "car", "traffic", "downtown",
                  "skyscraper", "road", "architecture", "bridge"],
        "people": ["person", "face", "portrait", "family", "group", "crowd",
                   "smile", "child", "baby", "wedding"],
    }

    EMBEDDING_DIM = 768  # Match production CLIP/SigLIP

    def __init__(self, seed: int = 42):
        """Initialize with deterministic random seed."""
        self._rng = np.random.RandomState(seed)
        self._cluster_centers = self._generate_cluster_centers()
        self._cache: dict[str, list[float]] = {}

    def _generate_cluster_centers(self) -> dict[str, np.ndarray]:
        """Generate orthogonal-ish cluster centers in embedding space."""
        centers = {}
        for cluster_name in self.CONCEPT_CLUSTERS:
            center = self._rng.randn(self.EMBEDDING_DIM)
            center = center / np.linalg.norm(center)
            centers[cluster_name] = center
        return centers

    def _find_cluster(self, text: str) -> str | None:
        """Find which concept cluster a text belongs to."""
        text_lower = text.lower()
        for cluster_name, keywords in self.CONCEPT_CLUSTERS.items():
            for keyword in keywords:
                if keyword in text_lower:
                    return cluster_name
        return None

    @property
    def embedding_dim(self) -> int:
        """Return production-matching embedding dimension."""
        return self.EMBEDDING_DIM

    def embed_text(self, text: str) -> list[float]:
        """Generate semantically-aware embedding from text.

        Texts matching the same concept cluster produce similar vectors.
        Unknown texts get random (but deterministic) vectors.
        All vectors are L2-normalized.
        """
        if text in self._cache:
            return self._cache[text]

        cluster = self._find_cluster(text)

        if cluster:
            # Generate vector near cluster center
            center = self._cluster_centers[cluster]
            # Use text hash for deterministic noise (small perturbation)
            text_hash = int(hashlib.sha256(text.encode()).hexdigest()[:8], 16)
            noise_rng = np.random.RandomState(text_hash)
            noise = noise_rng.randn(self.EMBEDDING_DIM) * 0.1
            vector = center + noise
        else:
            # Unknown text: deterministic random vector
            text_hash = int(hashlib.sha256(text.encode()).hexdigest()[:8], 16)
            text_rng = np.random.RandomState(text_hash)
            vector = text_rng.randn(self.EMBEDDING_DIM)

        # L2 normalize (matches production behavior)
        vector = vector / np.linalg.norm(vector)

        result = vector.tolist()
        self._cache[text] = result
        return result

    def embed_image(self, image_path: str | Path) -> list[float]:
        """Generate embedding from image path.

        Uses the filename (not path) to determine semantic cluster,
        allowing tests to create images with meaningful names like
        "sunset_beach.jpg" that produce nature-cluster embeddings.
        """
        # Extract filename without extension for semantic matching
        filename = Path(image_path).stem
        return self.embed_text(filename)

    def embed_image_from_pil(self, image: "Image.Image") -> list[float]:
        """Generate embedding from PIL image.

        Since we can't extract semantics from pixel data without a real model,
        use a deterministic hash of the image size as a seed.
        """
        size_key = f"pil_{image.size[0]}x{image.size[1]}_{image.mode}"
        return self.embed_text(size_key)

    def embed_images_batch(self, images: list["Image.Image"]) -> list[list[float]]:
        """Batch embed PIL images."""
        return [self.embed_image_from_pil(img) for img in images]
```

**Key properties of the new mock**:
1. **768-dim output** matches production CLIP/SigLIP.
2. **L2-normalized** vectors match production behavior.
3. **Semantic clusters** enable meaningful similarity assertions: `cosine_similarity(embed("sunset"), embed("beach")) > cosine_similarity(embed("sunset"), embed("pizza"))`.
4. **Deterministic** via seeded random generators + hash-based seeds per text.
5. **Filename-based image semantics**: `embed_image("/photos/sunset_beach.jpg")` produces a nature-cluster embedding.
6. **Cached** for performance (same input always returns same output).

**Backward compatibility**: The old `MockEmbeddingService` class should be kept temporarily (renamed to `LegacyMockEmbeddingService`) to avoid breaking existing tests. New tests should use `SemanticMockEmbeddingService`. A migration path is provided in Task 5.

---

### Task 2: Update Qdrant Test Collections to Match Production Dimensions (1 hour)

**File**: `image-search-service/tests/conftest.py` (modify `qdrant_client` fixture)

```python
@pytest.fixture
def qdrant_client() -> QdrantClient:
    """Create in-memory Qdrant client with production-matching collections.

    Collection dimensions:
    - image_assets: 768-dim (CLIP/SigLIP image search embeddings)
    - faces: 512-dim (ArcFace/InsightFace face embeddings)
    """
    client = QdrantClient(":memory:")
    settings = get_settings()

    # Main image assets collection: 768-dim for CLIP/SigLIP
    client.create_collection(
        collection_name=settings.qdrant_collection,
        vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    )

    # Face embeddings collection: 512-dim for InsightFace/ArcFace
    client.create_collection(
        collection_name=settings.qdrant_face_collection,
        vectors_config=VectorParams(size=512, distance=Distance.COSINE),
    )

    return client
```

**Why face collection stays at 512**: The `FACE_VECTOR_DIM = 512` in `face_qdrant.py` line 34 is correct -- InsightFace/ArcFace models produce 512-dim embeddings. Only the image search collection needs updating to 768.

**Impact assessment**: Changing the image assets collection from 512 to 768 requires updating all tests that upsert vectors into the image assets collection. These tests currently generate 512-dim vectors from `MockEmbeddingService`. After switching to `SemanticMockEmbeddingService` (768-dim), the dimensions will match.

---

### Task 3: Add Payload Indexes to Test Qdrant Setup (30 minutes)

**File**: `image-search-service/tests/conftest.py` (modify `qdrant_client` fixture)

Add payload index creation to mirror production `ensure_collection()` behavior:

```python
@pytest.fixture
def qdrant_client() -> QdrantClient:
    """Create in-memory Qdrant client with production-matching configuration."""
    client = QdrantClient(":memory:")
    settings = get_settings()

    # Main image assets collection
    client.create_collection(
        collection_name=settings.qdrant_collection,
        vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    )

    # Face embeddings collection
    client.create_collection(
        collection_name=settings.qdrant_face_collection,
        vectors_config=VectorParams(size=512, distance=Distance.COSINE),
    )

    # Add payload indexes (mirrors FaceQdrantClient.ensure_collection())
    for field_name, field_type in [
        ("person_id", PayloadSchemaType.KEYWORD),
        ("cluster_id", PayloadSchemaType.KEYWORD),
        ("is_prototype", PayloadSchemaType.BOOL),
        ("asset_id", PayloadSchemaType.KEYWORD),
        ("face_instance_id", PayloadSchemaType.KEYWORD),
    ]:
        client.create_payload_index(
            collection_name=settings.qdrant_face_collection,
            field_name=field_name,
            field_schema=field_type,
        )

    return client
```

**Import addition**: Add `PayloadSchemaType` to imports from `qdrant_client.models`.

---

### Task 4: Fix the Two Tests Loading Real GPU Models (1-2 hours)

**File**: `image-search-service/tests/faces/test_service.py`

**Root cause** (from test execution analysis Category B): The tests mock `image_search_service.faces.detector.detect_faces` but `process_assets_batch()` in `FaceProcessingService` acquires the detector through a different code path -- likely instantiating `FaceDetector` directly or using the class method rather than the module-level function.

**Investigation needed**: Read the actual call chain in `FaceProcessingService.process_assets_batch()` to identify where the InsightFace model is loaded. The mock must patch at the correct location in the import chain.

**Likely fix pattern**:

```python
@pytest.fixture
def mock_insightface_app(monkeypatch):
    """Prevent InsightFace model from loading in tests.

    Patches at the insightface.app level to prevent buffalo_l download.
    """
    mock_app = MagicMock()
    mock_app.get.return_value = []  # No face detections

    # Patch the actual InsightFace model initialization
    monkeypatch.setattr(
        "image_search_service.faces.detector.insightface.app.FaceAnalysis",
        lambda **kwargs: mock_app,
    )
    return mock_app
```

**Alternatively**, if the `FaceDetector` class caches the model:

```python
# Patch the FaceDetector class to use a mock model
monkeypatch.setattr(
    "image_search_service.faces.service.FaceDetector",
    lambda **kwargs: mock_detector_instance,
)
```

**Verification**: After the fix, running these two tests should:
1. Not print "Loaded InsightFace model (buffalo_l)" to stdout
2. Complete in < 1 second (instead of ~7 seconds)
3. Pass their assertions

---

### Task 5: Migrate Existing Tests to New Mock (1-2 days)

This is the largest task and should be done incrementally.

#### 5.1 Identify All Tests Using `MockEmbeddingService`

From grep results, 16 test files import or use the mock embedding service. The main entry point is the `test_client` fixture in `conftest.py` which injects `MockEmbeddingService` as the embedding service dependency.

#### 5.2 Migration Strategy: Gradual Replacement

**Phase A: Keep both mock classes (Day 1)**

```python
# In conftest.py
class LegacyMockEmbeddingService:
    """Original 512-dim MD5-based mock (deprecated)."""
    # ... (current MockEmbeddingService code, unchanged)

class SemanticMockEmbeddingService:
    """New 768-dim semantic-aware mock."""
    # ... (new code from Task 1)

# Alias for backward compatibility
MockEmbeddingService = LegacyMockEmbeddingService
```

**Phase B: Update test_client to use new mock (Day 1)**

The `test_client` fixture (conftest.py lines 271-328) needs updating:

```python
@pytest.fixture
def mock_embedding_service() -> SemanticMockEmbeddingService:
    """Create semantic mock embedding service."""
    return SemanticMockEmbeddingService()
```

**Phase C: Update tests that depend on 512-dim vectors (Day 1-2)**

Tests that upsert vectors into the image assets collection must be updated:

1. **`tests/api/test_search.py`** -- Search tests that create vectors and query them
2. **`tests/api/test_vectors.py`** -- Vector management tests
3. **`tests/unit/test_embedding_service.py`** -- Embedding dimension assertions
4. **`tests/unit/test_bootstrap_qdrant.py`** -- Collection creation with dimension checks

For each file:
- Change vector generation from 512-dim to 768-dim
- Update dimension assertions from 512 to 768
- Use `SemanticMockEmbeddingService` for embed calls

**Phase D: Enhance search tests with semantic assertions (Day 2)**

The most valuable improvement: update search tests to verify ranking correctness:

```python
# BEFORE (current test - meaningless assertion):
async def test_search_with_results(test_client):
    # ... setup: ingest sunset.jpg and pizza.jpg
    response = await test_client.get("/api/v1/search?q=sunset+on+the+beach")
    assert response.status_code == 200
    data = response.json()
    assert len(data["results"]) > 0  # Just checks something returned

# AFTER (semantic assertion - validates ranking):
async def test_search_ranks_semantically_similar_higher(test_client, db_session):
    """Verify that semantically similar images rank higher in search results."""
    # Setup: ingest images with semantic filenames
    sunset_asset = ImageAsset(path="/photos/sunset_beach.jpg")
    pizza_asset = ImageAsset(path="/photos/pizza_restaurant.jpg")
    db_session.add_all([sunset_asset, pizza_asset])
    await db_session.commit()

    # Upsert embeddings (SemanticMockEmbeddingService uses filenames)
    embed_service = SemanticMockEmbeddingService()
    sunset_vec = embed_service.embed_image("/photos/sunset_beach.jpg")  # nature cluster
    pizza_vec = embed_service.embed_image("/photos/pizza_restaurant.jpg")  # food cluster

    # ... upsert to Qdrant ...

    # Search for nature-related query
    response = await test_client.get("/api/v1/search?q=ocean+waves")
    data = response.json()

    assert len(data["results"]) >= 2
    # Sunset should rank higher than pizza for "ocean waves"
    result_paths = [r["asset"]["path"] for r in data["results"]]
    sunset_idx = result_paths.index("/photos/sunset_beach.jpg")
    pizza_idx = result_paths.index("/photos/pizza_restaurant.jpg")
    assert sunset_idx < pizza_idx, "Sunset should rank higher than pizza for 'ocean waves'"
```

#### 5.3 Tests That Should NOT Be Changed

Some tests intentionally test with specific embedding values and should keep their current behavior:

1. **Face pipeline tests** (`tests/faces/`) -- These use 512-dim InsightFace embeddings, which is correct. Do not change these.
2. **Face Qdrant tests** (`tests/unit/test_centroid_qdrant.py`, etc.) -- These use 512-dim face vectors. Correct as-is.
3. **Tests that mock embeddings directly** (e.g., providing `[0.1] * 512` as a mock return value) -- These are testing specific behaviors and should use the correct dimension for their domain.

**Rule of thumb**: If a test is about **image search** (CLIP/SigLIP), update to 768-dim. If a test is about **face recognition** (InsightFace/ArcFace), keep at 512-dim.

---

### Task 6: Add Dimension Validation Guard (30 minutes)

Add a runtime check that catches dimension mismatches early:

**File**: `image-search-service/tests/conftest.py`

```python
@pytest.fixture(autouse=True)
def validate_embedding_dimensions():
    """Validate that test embedding dimensions match expected values.

    This guard catches the most common mock accuracy issue: dimension
    mismatch between test and production embedding services.
    """
    yield

    # Post-test validation (runs after each test)
    # This is informational -- logs a warning if dimensions are wrong
    # Actual enforcement is via the collection creation dimensions
```

**File**: `image-search-service/src/image_search_service/vector/qdrant.py` (if not already present)

Add a dimension check when upserting:

```python
def upsert_with_dimension_check(self, embedding: list[float], expected_dim: int):
    """Upsert with dimension validation."""
    if len(embedding) != expected_dim:
        raise ValueError(
            f"Embedding dimension mismatch: got {len(embedding)}, "
            f"expected {expected_dim}. Check embedding service configuration."
        )
```

Note: This is a suggested production code improvement. Per the task requirements (documentation/planning only), document this as a recommendation but do not implement it.

---

## 4. Affected Test Files

The following test files will need updates when migrating to the new mock:

| File | Changes Needed | Effort |
|------|----------------|--------|
| `tests/conftest.py` | Replace MockEmbeddingService, update qdrant_client fixture | 1 hour |
| `tests/api/test_search.py` | Update vector dimensions, add semantic assertions | 2 hours |
| `tests/api/test_vectors.py` | Update vector dimensions in upsert calls | 30 min |
| `tests/api/test_clusters_filtering.py` | Update mock embedding dimensions | 30 min |
| `tests/unit/test_embedding_service.py` | Update dimension assertions | 30 min |
| `tests/unit/test_bootstrap_qdrant.py` | Update collection dimension checks | 30 min |
| `tests/unit/test_qdrant_wrapper.py` | Update vector dimensions | 30 min |
| `tests/unit/test_siglip_embedding_service.py` | Already 768-dim (verify) | 15 min |
| `tests/faces/test_service.py` | Fix mock target for InsightFace (Task 4) | 1 hour |
| `tests/api/test_face_suggestions.py` | May use mock embeddings (verify) | 30 min |
| **Total** | | **~7 hours** |

---

## 5. Implementation Order

```
Day 1:
  Morning:
    - Task 1: Create SemanticMockEmbeddingService (keep old as LegacyMock)
    - Task 2: Update qdrant_client fixture dimensions
    - Task 3: Add payload indexes
  Afternoon:
    - Task 4: Fix InsightFace model loading in test_service.py
    - Run test suite to verify nothing breaks (old mock still used)

Day 2:
  Morning:
    - Task 5 Phase A: Add both mock classes to conftest
    - Task 5 Phase B: Switch test_client to new mock
    - Fix dimension-related test failures one file at a time
  Afternoon:
    - Task 5 Phase C: Update remaining test files
    - Task 5 Phase D: Add semantic search ranking assertions
    - Task 6: Add dimension validation guard

Day 3 (buffer):
    - Debug any remaining failures
    - Verify full test suite passes
    - Remove LegacyMockEmbeddingService once all tests migrated
```

---

## 6. Verification Checklist

- [ ] `SemanticMockEmbeddingService.embedding_dim` returns 768
- [ ] `SemanticMockEmbeddingService.embed_text()` returns L2-normalized vectors
- [ ] Cosine similarity between "sunset" and "beach" > 0.5 (same cluster)
- [ ] Cosine similarity between "sunset" and "pizza" < 0.3 (different clusters)
- [ ] Qdrant image assets collection created with 768-dim
- [ ] Qdrant face collection created with 512-dim (unchanged)
- [ ] Payload indexes created on face collection (person_id, cluster_id, etc.)
- [ ] `test_process_assets_batch` does NOT load real InsightFace model
- [ ] `test_process_assets_batch` completes in < 1 second
- [ ] All existing tests pass after migration (zero regressions)
- [ ] At least one search test validates ranking order (semantic assertion)
- [ ] `make test` passes cleanly

---

## 7. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Changing mock dimension breaks many tests at once | Keep `LegacyMockEmbeddingService` as fallback; migrate one file at a time |
| Semantic clusters don't cover all test scenarios | Include a catch-all that generates deterministic random vectors for unknown texts |
| New mock is slower than MD5-based mock | Cache embeddings; numpy operations on 768-dim are still sub-millisecond |
| Tests become coupled to concept cluster definitions | Document clusters clearly; make cluster membership easy to extend |
| Face tests break from dimension change | Face tests use 512-dim ArcFace embeddings (unchanged); only image search tests affected |
| Qdrant in-memory doesn't support payload indexes | `QdrantClient(":memory:")` does support payload indexes; verified in qdrant-client docs |

---

## 8. Success Criteria

1. Mock embedding dimension matches production (768 for image search, 512 for faces)
2. Mock embeddings are L2-normalized (matches production CLIP/SigLIP behavior)
3. At least one search test validates semantic ranking correctness (not just "something returned")
4. No tests accidentally load real GPU models (InsightFace or OpenCLIP)
5. Qdrant test setup mirrors production configuration (dimensions + payload indexes)
6. All existing tests continue to pass (zero regressions)
7. Test execution time does not increase by more than 10%

---

## 9. Long-Term Recommendations

Beyond the immediate fixes in this plan, consider these improvements for future iterations:

### 9.1 MockQueue Validation

The `mock_queue` fixture (conftest.py lines 331-351) records enqueued jobs but never validates function signatures. Consider adding:

```python
class MockQueue:
    def enqueue(self, func, *args, **kwargs):
        # Validate function exists and is callable
        assert callable(func), f"Enqueued non-callable: {func}"
        jobs.append({"func": func, "args": args, "kwargs": kwargs})
```

### 9.2 Face Qdrant Client Fixture Fragility

The face_qdrant_client fixture (conftest.py lines 208-219) injects via `face_client._client = qdrant_client` (private attribute mutation). If `FaceQdrantClient` is refactored, this silently breaks. Consider:
- Adding a `FaceQdrantClient.from_client(client)` factory method for testing
- Or using dependency injection via constructor parameter

### 9.3 Separate InsightFace Mock Module

Create a dedicated `tests/mocks/insightface_mock.py` that provides a drop-in replacement for the InsightFace model, generating 512-dim face embeddings with deterministic properties (e.g., faces at similar positions produce similar embeddings).
