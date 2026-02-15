# Plan 5: Add PostgreSQL Integration Test Tier

**Priority**: P2 (Important)
**Effort**: Medium (2-3 days)
**Risk Level**: HIGH -- addresses structural testing gap identified in devil's advocate review
**Depends On**: None (can be implemented independently)
**Research References**:
- `docs/research/service-testing/devils-advocate-review.md` Section 1.2 "SQLite-as-PostgreSQL: A Ticking Time Bomb"
- `docs/research/service-testing/test-execution-analysis.md` Section 2 "Failure Analysis"
- `docs/research/service-testing/test-case-analysis.md` Section 1.1 "Root conftest.py"

---

## 1. Problem Statement

The entire test suite runs against SQLite in-memory (`sqlite+aiosqlite:///:memory:`) instead of PostgreSQL. The codebase uses 10+ PostgreSQL-specific features that SQLite does not enforce:

### 1.1 JSONB Columns (10+ columns across models)

From `src/image_search_service/db/models.py`:

| Model | Column | Type | Line |
|-------|--------|------|------|
| `ImageAsset` | `exif_metadata` | `JSONB` | 178 |
| `FaceInstance` | `landmarks` | `JSONB` | 478-479 |
| `FaceAssignmentEvent` | `affected_photo_ids` | `JSONB` | 609-610 |
| `FaceAssignmentEvent` | `affected_face_instance_ids` | `JSONB` | 612-613 |
| `FaceSuggestion` | `matching_prototype_ids` | `JSONB` | 765-766 |
| `FaceSuggestion` | `prototype_scores` | `JSONB` | 771-772 |
| `SystemConfig` | `allowed_values` | `JSONB` | 833 |
| `PersonCentroid` | `build_params` | `JSONB` | 918 |
| `TrainingEvidence` | `metadata_json` | `JSON` | 355 |
| `TrainingSession` | `config` | `JSON` | 219 |

The codebase defines a compatibility shim on line 35:
```python
JSONB = JSON().with_variant(PG_JSONB(astext_type=Text()), "postgresql")
```

SQLite stores these as plain text JSON. PostgreSQL JSONB supports indexing, containment operators (`@>`, `<@`), and key-existence checks (`?`). Any code relying on PostgreSQL-specific JSONB operators passes on SQLite but fails in production.

### 1.2 Functional Index on `func.lower(name)`

From `models.py` line 451:
```python
__table_args__ = (
    Index("ix_persons_name_lower", func.lower(name), unique=True),
    ...
)
```

SQLite does not enforce functional indexes. Two persons with names "John" and "john" coexist in tests but crash in production with a unique constraint violation.

### 1.3 PostgreSQL Enum Types

From `models.py` lines 418-427, 548-556, 887-896, 903-912:
- `PersonStatus` enum with `create_type=False` (created in migration)
- `PrototypeRole` enum with `create_type=False`
- `CentroidType` enum with `create_type=False`
- `CentroidStatus` enum with `create_type=False`

SQLite treats these as plain strings. PostgreSQL enforces enum values. If code writes an invalid enum value, SQLite tests pass silently.

### 1.4 UUID Primary Keys

Multiple models use `UUID(as_uuid=True)` primary keys (`Person`, `FaceInstance`, `PersonPrototype`, `FaceAssignmentEvent`, `FaceDetectionSession`, `PersonCentroid`). SQLite stores UUIDs as text blobs, while PostgreSQL stores them as native 128-bit values with different comparison semantics and indexing behavior.

### 1.5 CASCADE/SET NULL Foreign Key Behavior

Multiple foreign keys specify `ondelete="CASCADE"` and `ondelete="SET NULL"`. SQLite requires explicit `PRAGMA foreign_keys = ON` to enforce these, while PostgreSQL enforces them by default. The current test fixtures do not set this pragma.

### 1.6 Alembic Migration Validation

30 migration files exist in `src/image_search_service/db/migrations/versions/`. These migrations are never tested -- they target PostgreSQL DDL syntax and are bypassed entirely by the SQLite test setup that uses `Base.metadata.create_all()`.

---

## 2. Solution Design

### 2.1 Architecture: Two-Tier Testing

Keep the existing SQLite tier for fast unit tests (sub-second feedback loop). Add a PostgreSQL tier for integration tests that validate database-specific behavior.

```
Test Tiers:
  SQLite (default)      PostgreSQL (@pytest.mark.postgres)
  ===================== ================================
  make test             make test-postgres
  ~100s runtime         ~30-60s runtime
  No Docker needed      Docker required
  Unit + API tests      DB integration tests
  Fast CI feedback      Full confidence
```

### 2.2 PostgreSQL Test Infrastructure via Docker

Use `testcontainers-python` to spin up a PostgreSQL container per test session. This avoids requiring a persistent local PostgreSQL installation and ensures isolation between CI runs.

**Alternative considered**: `docker-compose.test.yml` -- rejected because `testcontainers` is more portable (works on GitHub Actions, local dev, any CI) and handles lifecycle automatically.

### 2.3 Pytest Marker Strategy

```
@pytest.mark.postgres     -- requires PostgreSQL container
(unmarked)                -- runs on SQLite (fast, default)
```

The `make test` command runs only unmarked tests (fast path). A new `make test-postgres` command runs only `@pytest.mark.postgres` tests. CI runs both.

---

## 3. Implementation Tasks

### Task 1: Add `testcontainers` dependency (15 minutes)

**File**: `image-search-service/pyproject.toml`

Add to the `[project.optional-dependencies]` dev section:
```toml
testcontainers = { version = ">=4.0.0", extras = ["postgres"] }
```

Then run:
```bash
cd image-search-service && uv sync --dev
```

**Verification**: `uv run python -c "from testcontainers.postgres import PostgresContainer; print('OK')"`

---

### Task 2: Create PostgreSQL conftest fixtures (1-2 hours)

**File**: `image-search-service/tests/conftest_postgres.py` (new file)

```python
"""PostgreSQL integration test fixtures.

These fixtures spin up a real PostgreSQL container via testcontainers
and provide both async and sync sessions for testing database-specific behavior.

Usage: Mark tests with @pytest.mark.postgres to use these fixtures.
"""

import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import Session as SyncSession

from image_search_service.db.models import Base


# Session-scoped container (one container per test session)
@pytest.fixture(scope="session")
def postgres_container():
    """Start PostgreSQL container for integration tests.

    The container is started once per test session and stopped after
    all tests complete. Uses PostgreSQL 16 to match production.
    """
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg


@pytest.fixture(scope="session")
def pg_connection_url(postgres_container):
    """Get async PostgreSQL connection URL.

    Returns URL in postgresql+asyncpg:// format for SQLAlchemy async.
    """
    url = postgres_container.get_connection_url()
    # testcontainers returns psycopg2 URL; convert to asyncpg
    return url.replace("postgresql+psycopg2://", "postgresql+asyncpg://")


@pytest.fixture(scope="session")
def pg_sync_connection_url(postgres_container):
    """Get sync PostgreSQL connection URL.

    Returns URL in postgresql:// format for SQLAlchemy sync.
    """
    return postgres_container.get_connection_url()


@pytest.fixture(scope="session")
def pg_engine(pg_connection_url):
    """Create async PostgreSQL engine (session-scoped).

    Creates all tables via Base.metadata.create_all.
    For migration testing, use the alembic_runner fixture instead.
    """
    import asyncio

    engine = create_async_engine(pg_connection_url, echo=False)

    async def _setup():
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)

    asyncio.get_event_loop().run_until_complete(_setup())
    yield engine

    async def _teardown():
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.drop_all)
        await engine.dispose()

    asyncio.get_event_loop().run_until_complete(_teardown())


@pytest.fixture
async def pg_session(pg_engine):
    """Create async PostgreSQL session with per-test rollback.

    Each test gets a fresh session that rolls back after the test,
    ensuring test isolation without recreating tables.
    """
    session_factory = async_sessionmaker(pg_engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
        await session.rollback()


@pytest.fixture
def pg_sync_engine(pg_sync_connection_url):
    """Create sync PostgreSQL engine for background job tests."""
    engine = create_engine(pg_sync_connection_url, echo=False)
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture
def pg_sync_session(pg_sync_engine):
    """Create sync PostgreSQL session with per-test rollback."""
    session = SyncSession(pg_sync_engine)
    yield session
    session.rollback()
    session.close()
```

**Key decisions**:
- `scope="session"` for container and engine: PostgreSQL container starts once, not per test (saves ~5s startup per test).
- Per-test rollback: Each test gets isolation through transaction rollback, not table recreation.
- Both async and sync sessions: The codebase uses sync sessions for RQ worker jobs (`face_jobs.py`, `training_jobs.py`, `dual_clusterer.py`, `trainer.py`) and async sessions for API routes.

**Verification**: Write a minimal test that imports the fixtures and creates a Person record.

---

### Task 3: Register pytest marker and configure test selection (30 minutes)

**File**: `image-search-service/pyproject.toml` (add to `[tool.pytest.ini_options]`)

```toml
[tool.pytest.ini_options]
markers = [
    "postgres: marks tests as requiring PostgreSQL (deselect with '-m \"not postgres\"')",
]
# Default: exclude postgres tests from fast runs
addopts = "-m 'not postgres'"
```

**File**: `image-search-service/Makefile` (add new targets)

```makefile
# Run PostgreSQL integration tests only
test-postgres:
	uv run pytest tests/ -m "postgres" -v --tb=short

# Run ALL tests (SQLite + PostgreSQL)
test-all:
	uv run pytest tests/ -m "" -v --tb=short
```

**Explanation**: The `addopts = "-m 'not postgres'"` in pyproject.toml means `make test` continues to run only SQLite tests (fast path). `make test-postgres` explicitly selects only the postgres-marked tests. `make test-all` clears the marker filter to run everything.

**Verification**: Run `make test` (should skip postgres tests), then `make test-postgres` (should run only postgres tests).

---

### Task 4: Write JSONB behavior tests (2-3 hours)

**File**: `image-search-service/tests/integration/test_postgres_jsonb.py` (new file)

These tests validate that JSONB columns behave correctly on real PostgreSQL, catching issues that SQLite silently ignores.

#### Test 4a: JSONB storage and retrieval for `ImageAsset.exif_metadata`

```python
@pytest.mark.postgres
async def test_exif_metadata_jsonb_round_trip(pg_session):
    """Verify JSONB stores and retrieves nested dicts correctly.

    JSONB normalizes key order and deduplicates keys, unlike JSON text.
    This test catches any code that depends on JSON text ordering.
    """
    asset = ImageAsset(
        path="/test/photo.jpg",
        exif_metadata={
            "DateTimeOriginal": "2024:01:15 10:30:00",
            "Make": "Canon",
            "GPSInfo": {"latitude": 37.7749, "longitude": -122.4194},
            "nested": {"deep": {"value": [1, 2, 3]}},
        },
    )
    pg_session.add(asset)
    await pg_session.commit()
    await pg_session.refresh(asset)

    # Verify nested structure preserved
    assert asset.exif_metadata["GPSInfo"]["latitude"] == 37.7749
    assert asset.exif_metadata["nested"]["deep"]["value"] == [1, 2, 3]
```

#### Test 4b: JSONB storage for `FaceInstance.landmarks`

```python
@pytest.mark.postgres
async def test_face_landmarks_jsonb(pg_session):
    """Verify face landmarks JSONB stores 5-point facial landmarks correctly."""
    # Create prerequisite ImageAsset
    asset = ImageAsset(path="/test/face_photo.jpg")
    pg_session.add(asset)
    await pg_session.flush()

    face = FaceInstance(
        asset_id=asset.id,
        bbox_x=100, bbox_y=200, bbox_w=50, bbox_h=50,
        detection_confidence=0.95,
        qdrant_point_id=uuid.uuid4(),
        landmarks={
            "left_eye": [120, 215],
            "right_eye": [140, 215],
            "nose": [130, 230],
            "left_mouth": [118, 240],
            "right_mouth": [142, 240],
        },
    )
    pg_session.add(face)
    await pg_session.commit()
    await pg_session.refresh(face)

    assert face.landmarks["left_eye"] == [120, 215]
    assert len(face.landmarks) == 5
```

#### Test 4c: JSONB for `FaceSuggestion.prototype_scores` (dict[str, float])

```python
@pytest.mark.postgres
async def test_suggestion_prototype_scores_jsonb(pg_session):
    """Verify prototype_scores JSONB preserves float precision."""
    # (Set up prerequisite person, face_instance, source_face)
    suggestion = FaceSuggestion(
        face_instance_id=...,
        suggested_person_id=...,
        confidence=0.85,
        source_face_id=...,
        matching_prototype_ids=["proto-1", "proto-2", "proto-3"],
        prototype_scores={
            "proto-1": 0.92341,
            "proto-2": 0.87654,
            "proto-3": 0.75123,
        },
        aggregate_confidence=0.85039,
        prototype_match_count=3,
    )
    pg_session.add(suggestion)
    await pg_session.commit()
    await pg_session.refresh(suggestion)

    # Verify float precision preserved
    assert abs(suggestion.prototype_scores["proto-1"] - 0.92341) < 1e-10
    assert suggestion.matching_prototype_ids == ["proto-1", "proto-2", "proto-3"]
    assert suggestion.prototype_match_count == 3
```

#### Test 4d: JSONB for `PersonCentroid.build_params`

```python
@pytest.mark.postgres
async def test_centroid_build_params_jsonb(pg_session):
    """Verify build_params stores algorithm configuration correctly."""
    # (Set up prerequisite Person)
    centroid = PersonCentroid(
        person_id=...,
        qdrant_point_id=uuid.uuid4(),
        model_version="arcface_r100_glint360k_v1",
        centroid_version=1,
        n_faces=25,
        build_params={
            "algorithm": "trimmed_mean",
            "trim_threshold": 0.05,
            "outlier_count": 2,
            "face_ids_hash": "abc123def456",
        },
    )
    pg_session.add(centroid)
    await pg_session.commit()
    await pg_session.refresh(centroid)

    assert centroid.build_params["algorithm"] == "trimmed_mean"
    assert centroid.build_params["trim_threshold"] == 0.05
```

#### Test 4e: JSONB for `FaceAssignmentEvent.affected_photo_ids` (array in JSONB)

```python
@pytest.mark.postgres
async def test_assignment_event_jsonb_arrays(pg_session):
    """Verify JSONB array columns store and retrieve correctly."""
    event = FaceAssignmentEvent(
        operation="MOVE_TO_PERSON",
        face_count=3,
        photo_count=2,
        affected_photo_ids=[101, 102],
        affected_face_instance_ids=["uuid-1", "uuid-2", "uuid-3"],
    )
    pg_session.add(event)
    await pg_session.commit()
    await pg_session.refresh(event)

    assert event.affected_photo_ids == [101, 102]
    assert len(event.affected_face_instance_ids) == 3
```

**Verification**: `make test-postgres` passes all JSONB tests.

---

### Task 5: Write functional index and constraint tests (1-2 hours)

**File**: `image-search-service/tests/integration/test_postgres_constraints.py` (new file)

#### Test 5a: `func.lower(name)` unique index on Person

```python
@pytest.mark.postgres
async def test_person_name_case_insensitive_uniqueness(pg_session):
    """Verify Person.name unique index enforces case-insensitive uniqueness.

    This test CANNOT pass on SQLite because SQLite does not enforce
    functional indexes. This is the primary motivation for PostgreSQL tests.
    """
    from sqlalchemy.exc import IntegrityError

    person1 = Person(name="John Smith")
    pg_session.add(person1)
    await pg_session.commit()

    person2 = Person(name="john smith")  # Different case, same name
    pg_session.add(person2)

    with pytest.raises(IntegrityError):
        await pg_session.commit()
```

#### Test 5b: FaceInstance unique constraint on location

```python
@pytest.mark.postgres
async def test_face_instance_location_uniqueness(pg_session):
    """Verify uq_face_instance_location prevents duplicate face detections."""
    asset = ImageAsset(path="/test/photo.jpg")
    pg_session.add(asset)
    await pg_session.flush()

    face1 = FaceInstance(
        asset_id=asset.id,
        bbox_x=100, bbox_y=200, bbox_w=50, bbox_h=50,
        detection_confidence=0.95,
        qdrant_point_id=uuid.uuid4(),
    )
    pg_session.add(face1)
    await pg_session.commit()

    face2 = FaceInstance(
        asset_id=asset.id,
        bbox_x=100, bbox_y=200, bbox_w=50, bbox_h=50,  # Same location
        detection_confidence=0.90,
        qdrant_point_id=uuid.uuid4(),
    )
    pg_session.add(face2)

    with pytest.raises(IntegrityError):
        await pg_session.commit()
```

#### Test 5c: CASCADE delete behavior

```python
@pytest.mark.postgres
async def test_cascade_delete_person_nullifies_face_instances(pg_session):
    """Verify ON DELETE SET NULL for FaceInstance.person_id works on PostgreSQL."""
    person = Person(name="Test Person")
    pg_session.add(person)
    await pg_session.flush()

    asset = ImageAsset(path="/test/photo.jpg")
    pg_session.add(asset)
    await pg_session.flush()

    face = FaceInstance(
        asset_id=asset.id,
        bbox_x=10, bbox_y=20, bbox_w=30, bbox_h=30,
        detection_confidence=0.9,
        qdrant_point_id=uuid.uuid4(),
        person_id=person.id,
    )
    pg_session.add(face)
    await pg_session.commit()

    # Delete person -- face_instance.person_id should become NULL
    await pg_session.delete(person)
    await pg_session.commit()
    await pg_session.refresh(face)

    assert face.person_id is None  # SET NULL behavior
```

#### Test 5d: CASCADE delete for training jobs

```python
@pytest.mark.postgres
async def test_cascade_delete_session_removes_jobs(pg_session):
    """Verify ON DELETE CASCADE for TrainingJob when session deleted."""
    session = TrainingSession(name="test-session", root_path="/test")
    pg_session.add(session)
    await pg_session.flush()

    asset = ImageAsset(path="/test/img.jpg")
    pg_session.add(asset)
    await pg_session.flush()

    job = TrainingJob(session_id=session.id, asset_id=asset.id)
    pg_session.add(job)
    await pg_session.commit()

    job_id = job.id

    # Delete session -- jobs should cascade delete
    await pg_session.delete(session)
    await pg_session.commit()

    # Verify job is gone
    result = await pg_session.get(TrainingJob, job_id)
    assert result is None
```

#### Test 5e: PostgreSQL enum validation

```python
@pytest.mark.postgres
async def test_person_status_enum_enforced(pg_session):
    """Verify PostgreSQL enum prevents invalid status values.

    SQLite stores enums as strings and accepts any value.
    PostgreSQL enforces the enum constraint.
    """
    from sqlalchemy import text

    person = Person(name="Enum Test")
    pg_session.add(person)
    await pg_session.commit()

    # Try to set invalid status via raw SQL
    with pytest.raises(Exception):  # DataError or ProgrammingError
        await pg_session.execute(
            text("UPDATE persons SET status = 'invalid_status' WHERE name = 'Enum Test'")
        )
        await pg_session.commit()
```

**Verification**: `make test-postgres` passes all constraint tests.

---

### Task 6: Write Alembic migration validation tests (2-3 hours)

**File**: `image-search-service/tests/integration/test_postgres_migrations.py` (new file)

These tests run Alembic migrations against real PostgreSQL, validating that all 30 migration files produce valid DDL.

#### Test 6a: Full upgrade from empty database

```python
@pytest.mark.postgres
def test_alembic_full_upgrade(pg_sync_connection_url, tmp_path):
    """Verify all migrations apply cleanly from empty DB to head.

    This catches:
    - Invalid PostgreSQL DDL syntax
    - Missing enum type creation
    - Incorrect column types
    - Foreign key ordering issues
    """
    from alembic.config import Config
    from alembic import command

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", pg_sync_connection_url)

    # Upgrade to head (all migrations)
    command.upgrade(alembic_cfg, "head")

    # Verify we're at head
    from alembic.script import ScriptDirectory
    script = ScriptDirectory.from_config(alembic_cfg)
    head_rev = script.get_current_head()

    from alembic.runtime.migration import MigrationContext
    from sqlalchemy import create_engine

    engine = create_engine(pg_sync_connection_url)
    with engine.connect() as conn:
        context = MigrationContext.configure(conn)
        current_rev = context.get_current_revision()

    assert current_rev == head_rev
    engine.dispose()
```

#### Test 6b: Downgrade back to base

```python
@pytest.mark.postgres
def test_alembic_downgrade_to_base(pg_sync_connection_url):
    """Verify all migrations can be cleanly rolled back.

    This catches missing downgrade implementations.
    """
    from alembic.config import Config
    from alembic import command

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", pg_sync_connection_url)

    # First upgrade to head
    command.upgrade(alembic_cfg, "head")

    # Then downgrade to base
    command.downgrade(alembic_cfg, "base")

    # Verify we're at base (no revision)
    from alembic.runtime.migration import MigrationContext
    from sqlalchemy import create_engine

    engine = create_engine(pg_sync_connection_url)
    with engine.connect() as conn:
        context = MigrationContext.configure(conn)
        current_rev = context.get_current_revision()

    assert current_rev is None
    engine.dispose()
```

#### Test 6c: Upgrade-downgrade-upgrade cycle (idempotency)

```python
@pytest.mark.postgres
def test_alembic_upgrade_downgrade_upgrade_cycle(pg_sync_connection_url):
    """Verify migrations are idempotent: up -> down -> up produces same result."""
    from alembic.config import Config
    from alembic import command

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", pg_sync_connection_url)

    # Cycle: up -> down -> up
    command.upgrade(alembic_cfg, "head")
    command.downgrade(alembic_cfg, "base")
    command.upgrade(alembic_cfg, "head")

    # Verify at head
    from alembic.script import ScriptDirectory
    script = ScriptDirectory.from_config(alembic_cfg)
    assert script.get_current_head() is not None
```

#### Test 6d: Model-migration parity check

```python
@pytest.mark.postgres
def test_no_pending_migrations(pg_sync_connection_url):
    """Verify that models.py and migrations are in sync.

    Detects when someone adds a column to models.py but forgets
    to create a migration.
    """
    from alembic.config import Config
    from alembic import command
    from alembic.autogenerate import compare_metadata
    from alembic.runtime.migration import MigrationContext
    from sqlalchemy import create_engine

    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", pg_sync_connection_url)

    # Apply all migrations
    command.upgrade(alembic_cfg, "head")

    # Compare current DB schema with models
    engine = create_engine(pg_sync_connection_url)
    with engine.connect() as conn:
        context = MigrationContext.configure(conn)
        diff = compare_metadata(context, Base.metadata)

    engine.dispose()

    # Filter out noise (index name differences, etc.)
    significant_diffs = [
        d for d in diff
        if d[0] not in ("remove_index",)  # Index naming may differ
    ]

    assert len(significant_diffs) == 0, (
        f"Models and migrations are out of sync. Diffs:\n"
        + "\n".join(str(d) for d in significant_diffs)
    )
```

**Important note on migration test isolation**: Each migration test function should use a fresh PostgreSQL database. The `pg_sync_connection_url` fixture should create a new database per test to avoid contamination between upgrade/downgrade tests. The testcontainers setup can be enhanced:

```python
@pytest.fixture
def fresh_pg_database(postgres_container):
    """Create a fresh database for each migration test."""
    import psycopg2
    db_name = f"test_{uuid.uuid4().hex[:8]}"
    url = postgres_container.get_connection_url()
    # Connect to default DB and create new one
    conn = psycopg2.connect(url)
    conn.autocommit = True
    cursor = conn.cursor()
    cursor.execute(f"CREATE DATABASE {db_name}")
    cursor.close()
    conn.close()
    # Return URL pointing to new database
    yield url.rsplit("/", 1)[0] + f"/{db_name}"
    # Cleanup: drop database
    conn = psycopg2.connect(url)
    conn.autocommit = True
    cursor = conn.cursor()
    cursor.execute(f"DROP DATABASE IF EXISTS {db_name}")
    cursor.close()
    conn.close()
```

**Verification**: `make test-postgres` passes all migration tests. Each test starts from a clean database.

---

### Task 7: Update CI/Makefile configuration (30 minutes)

**File**: `image-search-service/Makefile`

Add the following targets:

```makefile
## PostgreSQL Integration Tests
test-postgres:
	@echo "Starting PostgreSQL integration tests (requires Docker)..."
	uv run pytest tests/ -m "postgres" -v --tb=short

## All Tests (SQLite + PostgreSQL)
test-all:
	@echo "Running all tests..."
	uv run pytest tests/ -m "" -v --tb=short

## PostgreSQL tests with coverage
test-postgres-cov:
	uv run pytest tests/ -m "postgres" --cov=src/image_search_service --cov-report=term-missing -v
```

If GitHub Actions CI exists, add a job:

```yaml
# .github/workflows/test.yml (addition)
test-postgres:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:16-alpine
      env:
        POSTGRES_PASSWORD: postgres
      ports:
        - 5432:5432
      options: >-
        --health-cmd pg_isready
        --health-interval 10s
        --health-timeout 5s
        --health-retries 5
  steps:
    - uses: actions/checkout@v4
    - name: Install uv
      uses: astral-sh/setup-uv@v4
    - name: Install dependencies
      run: cd image-search-service && uv sync --dev
    - name: Run PostgreSQL tests
      run: cd image-search-service && make test-postgres
      env:
        TEST_DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/postgres
```

**Verification**: Both `make test` (fast, SQLite) and `make test-postgres` (Docker) work independently.

---

## 4. Test Count Estimates

| Test File | Tests | Focus |
|-----------|-------|-------|
| `test_postgres_jsonb.py` | 5-7 | JSONB round-trip for all column types |
| `test_postgres_constraints.py` | 5-7 | Functional indexes, CASCADE, enums |
| `test_postgres_migrations.py` | 4-5 | Alembic upgrade/downgrade/parity |
| **Total** | **14-19** | |

---

## 5. Verification Checklist

- [ ] `testcontainers` installed and importable
- [ ] `conftest_postgres.py` provides `pg_session` and `pg_sync_session` fixtures
- [ ] `@pytest.mark.postgres` marker registered in pyproject.toml
- [ ] `make test` excludes postgres tests (fast path unchanged)
- [ ] `make test-postgres` runs only postgres tests with Docker
- [ ] `make test-all` runs both tiers
- [ ] All JSONB round-trip tests pass on PostgreSQL
- [ ] `func.lower(name)` uniqueness enforced (would fail on SQLite)
- [ ] CASCADE/SET NULL behavior validated
- [ ] Alembic full upgrade succeeds from empty DB
- [ ] Alembic downgrade to base succeeds
- [ ] Model-migration parity check passes (no pending migrations)
- [ ] CI job configured (if applicable)

---

## 6. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Docker not available in CI | testcontainers skips gracefully; postgres tests marked as skipped |
| Container startup slow (~5s) | Session-scoped container starts once per test session |
| Port conflicts on developer machines | testcontainers uses random ports |
| Migration test ordering | Fresh database per migration test via `fresh_pg_database` fixture |
| Flaky tests from shared container state | Per-test transaction rollback ensures isolation |

---

## 7. Success Criteria

1. JSONB column behavior validated on real PostgreSQL for all 10+ JSONB columns
2. `func.lower(name)` uniqueness constraint catches case-insensitive duplicates
3. All 30 Alembic migrations apply cleanly from empty database to head
4. Model-migration parity check detects schema drift
5. `make test` runtime unchanged (no regression for fast path)
6. `make test-postgres` completes in under 60 seconds
