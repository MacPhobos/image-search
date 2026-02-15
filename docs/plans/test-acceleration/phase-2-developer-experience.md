# Phase 2: Developer Experience + Code Quality

**Date**: 2026-02-15
**Status**: Ready for implementation
**Prerequisite**: Phase 1 (Quick Wins / xdist) must be completed first
**Estimated Effort**: 4-6 hours total
**Expected Impact**: ~50-62s full suite (from ~55-68s post-Phase 1), plus 5-15s iterative dev workflow

---

## 1. Overview

### Goal

Improve developer experience and test code quality through:
- Faster iterative development feedback loops (pytest-testmon, --lf/--ff targets)
- Reduced torch import overhead (lazy import saves ~1.8s per session/worker)
- Cleaner test code via parametrization (reduce ~88 repetitive tests to ~30)
- Eliminate ~50 redundant copy-pasted fixture definitions

### Expected Impact

| Metric | Post-Phase 1 | Post-Phase 2 | Improvement |
|--------|-------------|-------------|-------------|
| Full suite (local) | ~55-68s | ~50-62s | ~5-8s faster |
| Iterative dev (testmon) | N/A | ~5-15s | New capability |
| Failed-first re-run | N/A | ~10-30s | New capability |
| Test function count | 1,096 | ~1,038 | ~58 fewer (same coverage) |
| Duplicate fixture defs | ~50 | ~0 | Maintenance win |
| xdist worker startup | ~1.8s/worker | ~0s/worker | torch lazy import |

### Prerequisites

- Phase 1 complete: pytest-xdist installed and configured with `-n auto --dist worksteal`
- All 1,096 tests passing with xdist (verified via `uv run pytest`)

---

## 2. Changes

### ME1: Lazy-Import torch in `core/device.py`

**Effort**: 15 minutes | **Risk**: Low | **Impact**: ~1.8s saved per session/worker startup

#### Problem

`core/device.py` imports torch at module level (line 22). This costs ~1.8s at import time. Every xdist worker pays this cost at startup, and every test session pays it once.

#### File: `image-search-service/src/image_search_service/core/device.py`

**Before** (current, lines 1-23):
```python
"""Centralized device management for ML inference.
...
"""

import os
import platform
from functools import lru_cache
from typing import Any

import torch


def _initialize_mps_workarounds() -> None:
    ...
    if not (hasattr(torch.backends, "mps") and torch.backends.mps.is_available()):
        return
    os.environ.setdefault("PYTORCH_MPS_HIGH_WATERMARK_RATIO", "0.0")


# Initialize MPS workarounds at module import time (before any GPU operations)
_initialize_mps_workarounds()
```

**After**:
```python
"""Centralized device management for ML inference.
...
"""

import os
import platform
from functools import lru_cache
from typing import Any


def _initialize_mps_workarounds() -> None:
    """Initialize MPS workarounds for known PyTorch bugs on macOS.
    ...
    """
    import torch

    if not (hasattr(torch.backends, "mps") and torch.backends.mps.is_available()):
        return

    os.environ.setdefault("PYTORCH_MPS_HIGH_WATERMARK_RATIO", "0.0")


_mps_initialized = False


def _ensure_mps_workarounds() -> None:
    """Initialize MPS workarounds once, on first torch usage."""
    global _mps_initialized
    if not _mps_initialized:
        _initialize_mps_workarounds()
        _mps_initialized = True


@lru_cache(maxsize=1)
def get_device() -> str:
    """Get the best available PyTorch device.
    ...
    """
    import torch

    _ensure_mps_workarounds()

    # Priority 1: Explicit device override
    if device := os.getenv("DEVICE"):
        if device.lower() != "auto":
            _validate_device(device)
            return device

    # Priority 2: Force CPU mode
    if os.getenv("FORCE_CPU", "").lower() in ("true", "1", "yes"):
        return "cpu"

    # Priority 3: Auto-detect best available
    if torch.cuda.is_available():
        return "cuda"

    if hasattr(torch.backends, "mps") and torch.backends.mps.is_available():
        return "mps"

    return "cpu"


def _validate_device(device: str) -> None:
    """Validate device string and raise if invalid.
    ...
    """
    import torch

    valid_prefixes = ("cuda", "mps", "cpu")
    if not any(device.startswith(prefix) for prefix in valid_prefixes):
        raise ValueError(f"Invalid DEVICE '{device}'. Must be 'cuda', 'cuda:N', 'mps', or 'cpu'")

    if device.startswith("cuda:"):
        try:
            device_id = int(device.split(":")[1])
            if torch.cuda.is_available():
                device_count = torch.cuda.device_count()
                if device_id >= device_count:
                    raise ValueError(
                        f"CUDA device {device_id} not available. "
                        f"Only {device_count} device(s) found."
                    )
        except (ValueError, IndexError):
            raise ValueError(f"Invalid CUDA device ID in '{device}'")


def clear_device_cache() -> None:
    """Clear the cached device selection. Useful for testing."""
    get_device.cache_clear()
    get_device_info.cache_clear()


@lru_cache(maxsize=1)
def get_device_info() -> dict[str, Any]:
    """Get comprehensive device and platform information.
    ...
    """
    import torch

    _ensure_mps_workarounds()

    info: dict[str, Any] = {
        "platform": platform.system(),
        "machine": platform.machine(),
        "python_version": platform.python_version(),
        "pytorch_version": torch.__version__,
        "selected_device": get_device(),
    }

    info["cuda_available"] = torch.cuda.is_available()
    if torch.cuda.is_available():
        info["cuda_version"] = torch.version.cuda
        info["cuda_device_count"] = torch.cuda.device_count()
        info["cuda_device_name"] = torch.cuda.get_device_name(0)

    if hasattr(torch.backends, "mps"):
        info["mps_built"] = torch.backends.mps.is_built()
        info["mps_available"] = torch.backends.mps.is_available()
    else:
        info["mps_built"] = False
        info["mps_available"] = False

    return info


def get_onnx_providers() -> list[str]:
    """Get ONNX Runtime execution providers in priority order.
    ...
    """
    try:
        import onnxruntime as ort
        available = ort.get_available_providers()
    except ImportError:
        return ["CPUExecutionProvider"]

    priority = [
        "CUDAExecutionProvider",
        "CoreMLExecutionProvider",
        "CPUExecutionProvider",
    ]

    return [p for p in priority if p in available]


def is_apple_silicon() -> bool:
    """Check if running on Apple Silicon (M1/M2/M3/M4).
    ...
    """
    return platform.system() == "Darwin" and platform.machine() == "arm64"
```

#### Key Changes

1. Remove `import torch` at module level (line 22)
2. Move `import torch` into each function that uses it: `_initialize_mps_workarounds()`, `get_device()`, `_validate_device()`, `get_device_info()`
3. Replace eager `_initialize_mps_workarounds()` call at module load with lazy `_ensure_mps_workarounds()` called on first `get_device()` invocation
4. Add `_mps_initialized` flag to ensure MPS workarounds run exactly once

#### Why This Is Safe

- `get_device()` is `@lru_cache(maxsize=1)` — torch is imported once on first call, then cached
- MPS workarounds still execute before any GPU operations (triggered by `_ensure_mps_workarounds()` inside `get_device()`)
- The `_validate_device()` function already receives a device string, so torch is only needed for CUDA validation
- All existing tests use `clear_device_cache()` in `setup_method`, which works identically
- The `@patch("torch.cuda.is_available", ...)` decorators in tests still work because they patch the `torch` module, not the import statement

#### Validation

```bash
# Verify all device tests pass
uv run pytest tests/core/test_device.py -v

# Verify torch is NOT imported at module level
python -c "import time; t=time.time(); import image_search_service.core.device; print(f'{time.time()-t:.3f}s')"
# Should be <0.1s (vs ~1.8s before)

# Verify device detection still works
python -c "from image_search_service.core.device import get_device; print(get_device())"
```

---

### ME2: Parametrize `test_device.py` (31 tests -> ~10)

**Effort**: 1 hour | **Risk**: Low | **Impact**: ~21 fewer test functions, cleaner code

#### File: `image-search-service/tests/core/test_device.py`

The current file has 31 test methods across 5 classes, with 56 `@patch` decorators. Many tests follow identical patterns with different inputs.

#### Change 1: Parametrize `TestGetDevice` (14 tests -> 4)

**Before** (14 individual test methods, lines 9-155):
```python
class TestGetDevice:
    def setup_method(self) -> None:
        from image_search_service.core.device import clear_device_cache
        clear_device_cache()

    @patch.dict(os.environ, {}, clear=True)
    @patch("torch.cuda.is_available", return_value=True)
    def test_cuda_available(self, mock_cuda: MagicMock) -> None:
        ...
        assert get_device() == "cuda"

    @patch.dict(os.environ, {}, clear=True)
    @patch("torch.cuda.is_available", return_value=False)
    def test_mps_fallback(self, mock_cuda: MagicMock) -> None:
        ...
        # 12 more individual test methods
```

**After**:
```python
class TestGetDevice:
    """Tests for get_device() function."""

    def setup_method(self) -> None:
        """Clear device cache before each test."""
        from image_search_service.core.device import clear_device_cache
        clear_device_cache()

    @pytest.mark.parametrize(
        "env_vars,cuda_avail,mps_avail,expected",
        [
            # Auto-detect: CUDA > MPS > CPU
            ({}, True, False, "cuda"),
            ({}, False, True, "mps"),
            ({}, False, False, "cpu"),
            # DEVICE env var override
            ({"DEVICE": "cpu"}, True, False, "cpu"),
            ({"DEVICE": "auto"}, True, False, "cuda"),
            # FORCE_CPU variants
            ({"FORCE_CPU": "true"}, True, False, "cpu"),
            ({"FORCE_CPU": "1"}, False, False, "cpu"),
            ({"FORCE_CPU": "yes"}, False, False, "cpu"),
        ],
        ids=[
            "cuda-available",
            "mps-fallback",
            "cpu-fallback",
            "device-env-override",
            "device-env-auto",
            "force-cpu-true",
            "force-cpu-1",
            "force-cpu-yes",
        ],
    )
    def test_get_device_selection(
        self, env_vars: dict, cuda_avail: bool, mps_avail: bool, expected: str
    ) -> None:
        """Should select correct device based on env vars and hardware availability."""
        from image_search_service.core.device import clear_device_cache, get_device

        with patch.dict(os.environ, env_vars, clear=True), \
             patch("torch.cuda.is_available", return_value=cuda_avail), \
             patch("torch.backends.mps.is_available", return_value=mps_avail):
            clear_device_cache()
            assert get_device() == expected

    @patch.dict(os.environ, {"DEVICE": "cuda:0"})
    @patch("torch.cuda.is_available", return_value=True)
    @patch("torch.cuda.device_count", return_value=2)
    def test_device_env_cuda_with_id(
        self, mock_count: MagicMock, mock_cuda: MagicMock
    ) -> None:
        """Should respect DEVICE environment variable with CUDA device ID."""
        from image_search_service.core.device import clear_device_cache, get_device
        clear_device_cache()
        assert get_device() == "cuda:0"

    @pytest.mark.parametrize(
        "env_vars,cuda_avail,cuda_count,error_match",
        [
            ({"DEVICE": "invalid_device"}, False, 0, "Invalid DEVICE"),
            ({"DEVICE": "cuda:5"}, True, 2, "Invalid CUDA device ID"),
            ({"DEVICE": "cuda:abc"}, False, 0, "Invalid CUDA device ID"),
        ],
        ids=["invalid-device", "cuda-id-out-of-range", "cuda-id-non-numeric"],
    )
    def test_get_device_invalid_raises(
        self, env_vars: dict, cuda_avail: bool, cuda_count: int, error_match: str
    ) -> None:
        """Should raise ValueError for invalid device configurations."""
        from image_search_service.core.device import clear_device_cache, get_device

        with patch.dict(os.environ, env_vars, clear=True), \
             patch("torch.cuda.is_available", return_value=cuda_avail), \
             patch("torch.cuda.device_count", return_value=cuda_count):
            clear_device_cache()
            with pytest.raises(ValueError, match=error_match):
                get_device()

    @patch.dict(os.environ, {}, clear=True)
    @patch("torch.cuda.is_available", return_value=True)
    def test_device_cached(self, mock_cuda: MagicMock) -> None:
        """Should cache device selection across multiple calls."""
        from image_search_service.core.device import clear_device_cache, get_device

        clear_device_cache()
        result1 = get_device()
        call_count_1 = mock_cuda.call_count
        result2 = get_device()
        call_count_2 = mock_cuda.call_count

        assert result1 == result2 == "cuda"
        assert call_count_1 == call_count_2
```

This reduces TestGetDevice from 14 methods to 4 methods (8 + 1 + 3 + 1 parametrized cases = 13 test cases, same coverage).

#### Change 2: Parametrize `TestGetOnnxProviders` (6 tests -> 2)

**After**:
```python
class TestGetOnnxProviders:
    """Tests for get_onnx_providers() function."""

    @pytest.mark.parametrize(
        "available_providers,expected",
        [
            (
                ["CPUExecutionProvider", "CUDAExecutionProvider", "CoreMLExecutionProvider"],
                ["CUDAExecutionProvider", "CoreMLExecutionProvider", "CPUExecutionProvider"],
            ),
            (
                ["CPUExecutionProvider", "CUDAExecutionProvider"],
                ["CUDAExecutionProvider", "CPUExecutionProvider"],
            ),
            (
                ["CPUExecutionProvider", "CoreMLExecutionProvider"],
                ["CoreMLExecutionProvider", "CPUExecutionProvider"],
            ),
            (
                ["CPUExecutionProvider"],
                ["CPUExecutionProvider"],
            ),
        ],
        ids=["cuda+coreml+cpu", "cuda+cpu", "coreml+cpu", "cpu-only"],
    )
    def test_provider_priority_order(
        self, available_providers: list[str], expected: list[str]
    ) -> None:
        """Should return providers in priority order (CUDA > CoreML > CPU)."""
        from image_search_service.core.device import get_onnx_providers

        with patch("onnxruntime.get_available_providers", return_value=available_providers):
            assert get_onnx_providers() == expected

    def test_onnxruntime_not_installed(self) -> None:
        """Should return CPU provider when onnxruntime not installed."""
        from image_search_service.core.device import get_onnx_providers

        with patch(
            "builtins.__import__", side_effect=ImportError("No module named 'onnxruntime'")
        ):
            providers = get_onnx_providers()
            assert providers == ["CPUExecutionProvider"]
```

This reduces TestGetOnnxProviders from 6 methods to 2 (4 + 1 parametrized cases + 1 import error = same coverage).

#### Change 3: Parametrize `TestIsAppleSilicon` (5 tests -> 1)

**After**:
```python
class TestIsAppleSilicon:
    """Tests for is_apple_silicon() function."""

    @pytest.mark.parametrize(
        "system,machine,expected",
        [
            ("Darwin", "arm64", True),
            ("Darwin", "x86_64", False),
            ("Linux", "x86_64", False),
            ("Linux", "arm64", False),
            ("Windows", "AMD64", False),
        ],
        ids=["apple-silicon", "macos-intel", "linux-x86", "linux-arm64", "windows"],
    )
    def test_is_apple_silicon(self, system: str, machine: str, expected: bool) -> None:
        """Should correctly detect Apple Silicon."""
        from image_search_service.core.device import is_apple_silicon

        with patch("platform.system", return_value=system), \
             patch("platform.machine", return_value=machine):
            assert is_apple_silicon() is expected
```

This reduces TestIsAppleSilicon from 5 methods to 1 (5 parametrized cases).

#### `TestGetDeviceInfo` and `TestClearDeviceCache` remain unchanged

These classes test specific behaviors (dict keys, caching, CUDA info structure) that don't lend themselves well to parametrization. Keep the existing 6 tests as-is.

#### Summary for test_device.py

| Class | Before | After | Test Cases |
|-------|--------|-------|------------|
| TestGetDevice | 14 methods | 4 methods | 13 cases |
| TestGetDeviceInfo | 5 methods | 5 methods (unchanged) | 5 cases |
| TestGetOnnxProviders | 6 methods | 2 methods | 5 cases |
| TestIsAppleSilicon | 5 methods | 1 method | 5 cases |
| TestClearDeviceCache | 2 methods | 2 methods (unchanged) | 2 cases |
| **Total** | **31 methods** | **14 methods** | **30 test cases** |

#### Why This Is Safe

- Every existing test case is preserved in the parametrize matrix
- The `ids` parameter ensures descriptive test names in output
- `setup_method` continues to call `clear_device_cache()` before each parametrized case
- `@patch.dict(os.environ, ..., clear=True)` moved inside `with` blocks to work with parametrize

#### Validation

```bash
# Run parametrized tests and verify same test count
uv run pytest tests/core/test_device.py -v --co -q | tail -1
# Should show 30 tests collected (same coverage as 32 before, minus 2 redundant)

# Run and verify all pass
uv run pytest tests/core/test_device.py -v

# Run with xdist to ensure parallel safety
uv run pytest tests/core/test_device.py -v -n 2
```

---

### ME3: Parametrize `test_temporal_service.py` (57 tests -> ~20)

**Effort**: 1 hour | **Risk**: Low | **Impact**: ~37 fewer test functions

#### File: `image-search-service/tests/unit/test_temporal_service.py`

The current file has 57 test methods across 8 classes. Most classes test a single function with multiple input/output pairs that are ideal for parametrization.

#### Change 1: Parametrize `TestClassifyAgeEra` (15 tests -> 1)

**Before** (15 individual test methods):
```python
class TestClassifyAgeEra:
    def test_infant_age(self):
        assert classify_age_era(2) == AgeEraBucket.INFANT

    def test_child_age(self):
        assert classify_age_era(8) == AgeEraBucket.CHILD
    # ... 13 more methods
```

**After**:
```python
class TestClassifyAgeEra:
    """Tests for age era classification."""

    @pytest.mark.parametrize(
        "age,expected",
        [
            # Core classifications
            (2, AgeEraBucket.INFANT),
            (8, AgeEraBucket.CHILD),
            (15, AgeEraBucket.TEEN),
            (25, AgeEraBucket.YOUNG_ADULT),
            (45, AgeEraBucket.ADULT),
            (65, AgeEraBucket.SENIOR),
            (None, None),
            # Boundaries
            (0, AgeEraBucket.INFANT),
            (3, AgeEraBucket.INFANT),
            (4, AgeEraBucket.CHILD),
            (12, AgeEraBucket.CHILD),
            (13, AgeEraBucket.TEEN),
            (19, AgeEraBucket.TEEN),
            (20, AgeEraBucket.YOUNG_ADULT),
            (35, AgeEraBucket.YOUNG_ADULT),
            (36, AgeEraBucket.ADULT),
            (55, AgeEraBucket.ADULT),
            (56, AgeEraBucket.SENIOR),
            (100, AgeEraBucket.SENIOR),
        ],
        ids=[
            "infant", "child", "teen", "young-adult", "adult", "senior", "none",
            "zero-age", "infant-upper", "child-lower", "child-upper",
            "teen-lower", "teen-upper", "young-adult-lower", "young-adult-upper",
            "adult-lower", "adult-upper", "senior-lower", "very-old",
        ],
    )
    def test_classify_age_era(self, age, expected):
        assert classify_age_era(age) == expected
```

#### Change 2: Parametrize `TestGetEraAgeRange` (6 tests -> 1)

**After**:
```python
class TestGetEraAgeRange:
    """Tests for getting age range for era buckets."""

    @pytest.mark.parametrize(
        "era,expected_min,expected_max",
        [
            (AgeEraBucket.INFANT, 0, 3),
            (AgeEraBucket.CHILD, 4, 12),
            (AgeEraBucket.TEEN, 13, 19),
            (AgeEraBucket.YOUNG_ADULT, 20, 35),
            (AgeEraBucket.ADULT, 36, 55),
            (AgeEraBucket.SENIOR, 56, 120),
        ],
        ids=["infant", "child", "teen", "young-adult", "adult", "senior"],
    )
    def test_era_age_range(self, era, expected_min, expected_max):
        min_age, max_age = get_era_age_range(era)
        assert min_age == expected_min
        assert max_age == expected_max
```

#### Change 3: Parametrize `TestExtractDecade` (7 tests -> 1)

**After**:
```python
class TestExtractDecade:
    """Tests for decade extraction from timestamps."""

    @pytest.mark.parametrize(
        "timestamp,expected",
        [
            (datetime(1995, 6, 15), "1990s"),
            (datetime(2005, 1, 1), "2000s"),
            (datetime(2024, 12, 31), "2020s"),
            (None, None),
            (datetime(1999, 12, 31), "1990s"),
            (datetime(2000, 1, 1), "2000s"),
            (datetime(2009, 12, 31), "2000s"),
            (datetime(2010, 1, 1), "2010s"),
            (datetime(1980, 5, 10), "1980s"),
            (datetime(1970, 3, 25), "1970s"),
        ],
        ids=[
            "1990s", "2000s", "2020s", "none",
            "1999-boundary", "2000-boundary", "2009-boundary", "2010-boundary",
            "1980s", "1970s",
        ],
    )
    def test_extract_decade(self, timestamp, expected):
        assert extract_decade_from_timestamp(timestamp) == expected
```

#### Change 4: Parametrize `TestTemporalQualityScore` (11 tests -> 1)

**After**:
```python
class TestTemporalQualityScore:
    """Tests for temporal quality score computation."""

    @pytest.mark.parametrize(
        "base_quality,kwargs,expected",
        [
            (0.8, {}, pytest.approx(0.48)),
            (0.7, {"pose": "frontal"}, pytest.approx(0.62)),
            (0.7, {"bbox_area": 15000}, pytest.approx(0.52)),
            (0.7, {"age_confidence": 0.9}, pytest.approx(0.51)),
            (0.8, {"pose": "frontal", "bbox_area": 15000, "age_confidence": 1.0}, pytest.approx(0.88)),
            (1.0, {"pose": "frontal", "bbox_area": 20000, "age_confidence": 1.0}, 1.0),
            (None, {"pose": "frontal"}, pytest.approx(0.2)),
            (0.7, {"bbox_area": 5000}, pytest.approx(0.42)),
            (0.7, {"pose": "profile"}, pytest.approx(0.42)),
            (0.0, {}, 0.0),
        ],
        ids=[
            "base-only", "frontal-bonus", "large-bbox-bonus", "age-confidence-bonus",
            "all-bonuses", "max-score-capped", "none-base-quality",
            "small-bbox-no-bonus", "non-frontal-pose", "zero-quality",
        ],
    )
    def test_temporal_quality_score(self, base_quality, kwargs, expected):
        score = compute_temporal_quality_score(base_quality, **kwargs)
        assert score == expected
        if base_quality == 1.0:
            assert score <= 1.0
```

#### Change 5: Parametrize `TestCoverageGaps` (4 tests -> 1)

**After**:
```python
class TestCoverageGaps:
    """Tests for coverage gap detection."""

    @pytest.mark.parametrize(
        "existing,expected_count,expected_contains",
        [
            ({e.value for e in AgeEraBucket}, 0, []),
            ({"child", "teen", "young_adult", "adult", "senior"}, 1, [AgeEraBucket.INFANT]),
            (
                {"child", "adult"}, 4,
                [AgeEraBucket.INFANT, AgeEraBucket.TEEN, AgeEraBucket.YOUNG_ADULT, AgeEraBucket.SENIOR],
            ),
            (set(), 6, list(AgeEraBucket)),
        ],
        ids=["full-coverage", "missing-infant", "missing-multiple", "empty-coverage"],
    )
    def test_coverage_gaps(self, existing, expected_count, expected_contains):
        gaps = get_coverage_gaps(existing)
        assert len(gaps) == expected_count
        for era in expected_contains:
            assert era in gaps
```

#### Classes that remain unchanged

- **`TestEstimatePersonBirthYear`** (7 tests): Tests have complex setups with multi-element `faces` lists and conditional assertions. Keep as individual methods.
- **`TestExtractTemporalMetadata`** (5 tests): Tests check different dict key subsets. Keep as individual methods.
- **`TestEnrichFaceWithTemporalData`** (5 tests): Tests have complex assertions on multiple dict fields. Keep as individual methods.

#### Summary for test_temporal_service.py

| Class | Before | After | Test Cases |
|-------|--------|-------|------------|
| TestClassifyAgeEra | 15 methods | 1 method | 19 cases |
| TestGetEraAgeRange | 6 methods | 1 method | 6 cases |
| TestExtractDecade | 7 methods | 1 method | 10 cases |
| TestTemporalQualityScore | 11 methods | 1 method | 10 cases |
| TestCoverageGaps | 4 methods | 1 method | 4 cases |
| TestEstimatePersonBirthYear | 7 methods | 7 methods (unchanged) | 7 cases |
| TestExtractTemporalMetadata | 5 methods | 5 methods (unchanged) | 5 cases |
| TestEnrichFaceWithTemporalData | 5 methods | 5 methods (unchanged) | 5 cases |
| **Total** | **57 methods** | **22 methods** | **66 test cases** |

> **⚠️ REVIEW NOTE (plan-reviewer):** Corrected method count from 60 to 57 (verified via `grep -c "def test_"` on actual file). The parametrized version has *more* test cases (66 vs 57) because boundary tests that previously asserted multiple values in one method are now individual parametrized cases. This is an improvement in test granularity.

#### Validation

```bash
# Collect tests and verify case count
uv run pytest tests/unit/test_temporal_service.py -v --co -q | tail -1

# Run all and verify pass
uv run pytest tests/unit/test_temporal_service.py -v
```

---

### ME4: Consolidate Duplicate Fixtures into Shared Conftest

**Effort**: 2 hours | **Risk**: None | **Impact**: Eliminate ~50 redundant fixture definitions

#### Problem

Three fixtures are copy-pasted identically across 8+ test files each:

| Fixture | Duplicate Locations | Source of Truth |
|---------|-------------------|-----------------|
| `mock_image_asset` | 10 files | `tests/faces/conftest.py:40` |
| `mock_person` | 8 files | `tests/faces/conftest.py:81` |
| `mock_face_instance` | 6 files | `tests/faces/conftest.py:59` |

**All duplicates are byte-for-byte identical** to the definitions in `tests/faces/conftest.py`.

#### Step 1: Move fixtures to root `tests/conftest.py`

Add the following fixtures to the end of `image-search-service/tests/conftest.py` (after the existing `mock_queue` fixture, after line 663):

```python
# ============================================================
# Shared face/person/asset fixtures (consolidated from 20+ duplicates)
# ============================================================

@pytest.fixture
async def mock_image_asset(db_session):
    """Create a mock ImageAsset in the database."""
    from image_search_service.db.models import ImageAsset, TrainingStatus

    asset = ImageAsset(
        path="/test/images/photo.jpg",
        training_status=TrainingStatus.PENDING.value,
        width=640,
        height=480,
        file_size=102400,
        mime_type="image/jpeg",
    )
    db_session.add(asset)
    await db_session.commit()
    await db_session.refresh(asset)
    return asset


@pytest.fixture
async def mock_face_instance(db_session, mock_image_asset):
    """Create a mock FaceInstance in the database."""
    import uuid

    from image_search_service.db.models import FaceInstance

    face = FaceInstance(
        id=uuid.uuid4(),
        asset_id=mock_image_asset.id,
        bbox_x=100,
        bbox_y=150,
        bbox_w=80,
        bbox_h=80,
        detection_confidence=0.95,
        quality_score=0.75,
        qdrant_point_id=uuid.uuid4(),
    )
    db_session.add(face)
    await db_session.commit()
    await db_session.refresh(face)
    return face


@pytest.fixture
async def mock_person(db_session):
    """Create a mock Person in the database."""
    import uuid

    from image_search_service.db.models import Person, PersonStatus

    person = Person(
        id=uuid.uuid4(),
        name="Test Person",
        status=PersonStatus.ACTIVE.value,
    )
    db_session.add(person)
    await db_session.commit()
    await db_session.refresh(person)
    return person
```

#### Step 2: Remove duplicates from individual test files

Remove the `mock_image_asset`, `mock_person`, and `mock_face_instance` fixture definitions from these files:

**`mock_image_asset` — remove from 9 files:**
1. `tests/faces/conftest.py` (lines 39-55)
2. `tests/api/test_face_suggestions.py` (lines 11-27)
3. `tests/api/test_face_session_suggestions.py` (lines 14-27)
4. `tests/api/test_faces_routes.py` (lines 11-27)
5. `tests/api/test_prototype_endpoints.py` (lines 10-27)
6. `tests/api/test_person_name_regression.py` (lines 20-36)
7. `tests/api/test_get_faces_for_asset.py` (lines 9-26)
8. `tests/api/test_clusters_filtering.py` (lines 10-26)
9. `tests/unit/test_suggestion_qdrant_sync.py` (lines 55-71)
10. `tests/api/test_get_faces_orphaned_person.py` (lines 9-26)

**`mock_person` — remove from 7 files:**
1. `tests/faces/conftest.py` (lines 80-93)
2. `tests/api/test_face_suggestions.py` (lines 52-65)
3. `tests/api/test_face_session_suggestions.py` (lines 55-68)
4. `tests/api/test_faces_routes.py` (lines 52-65)
5. `tests/api/test_prototype_endpoints.py` (lines 29-42)
6. `tests/api/test_person_name_regression.py` (lines 39-52)
7. `tests/api/test_get_faces_for_asset.py` (lines 28-41)
8. `tests/unit/test_suggestion_qdrant_sync.py` (lines 27-40)

**`mock_face_instance` — remove from 5 files:**
1. `tests/faces/conftest.py` (lines 58-77)
2. `tests/api/test_face_suggestions.py` (lines 30-49)
3. `tests/api/test_faces_routes.py` (lines 30-49)
4. `tests/api/test_prototype_endpoints.py` (lines 45-64)
5. `tests/unit/test_suggestion_qdrant_sync.py` (lines 72-88)

**Note**: `tests/api/test_prototype_endpoints.py` has a `mock_face_instance` that takes an extra `mock_person` parameter (line 45: `async def mock_face_instance(db_session, mock_image_asset, mock_person)`). This is a variant, not a duplicate. Keep this one as a local override.

**Note**: `tests/unit/test_prototype_selection.py:35` has a sync `mock_face_instance` (not async) that returns a plain object (no DB). Keep this one as a local fixture.

#### Step 3: Clean up `tests/faces/conftest.py`

After removing the 3 fixtures moved to root conftest, `tests/faces/conftest.py` will retain:
- `mock_face_embedding` (face-specific, not duplicated)
- `mock_detected_face` (face-specific, not duplicated)
- `mock_qdrant_client` (face-specific, different from root conftest's `qdrant_client`)
- `mock_insightface` (face-specific, not duplicated)

The `import uuid` at the top of `tests/faces/conftest.py` can also be removed if it was only used by the moved fixtures.

#### Why This Is Safe

- pytest fixture resolution is hierarchical: conftest.py fixtures at any level are available to all tests in that directory and below
- Root conftest fixtures are available to ALL tests in the project
- Test files that had local duplicates will automatically use the root conftest version (same name, same behavior)
- Test files with **variant** fixtures (e.g., `mock_face_instance` with extra params) will still use their local definition (local scope overrides parent)

#### Validation

```bash
# Run all tests serially (no xdist) to catch fixture resolution issues
uv run pytest -p no:xdist -x -v 2>&1 | tail -5

# Verify same test count
uv run pytest --co -q | tail -1

# Run specifically the files that had duplicates
uv run pytest tests/api/test_face_suggestions.py tests/api/test_faces_routes.py \
  tests/api/test_prototype_endpoints.py tests/faces/ -v

# Run with xdist to verify parallel safety
uv run pytest -n auto
```

---

### ME5: Install pytest-testmon for Dev Workflow

**Effort**: 15 minutes | **Risk**: Low | **Impact**: 10-50x speedup for iterative development

#### Step 1: Install dependency

```bash
cd image-search-service
uv add --group dev "pytest-testmon>=2.1.0"
```

#### Step 2: Disable testmon by default in pytest config

Testmon should NOT run in CI or by default (it uses a local `.testmondata` SQLite database to track which tests are affected by which source files). It is a developer-only workflow tool.

**File**: `image-search-service/pyproject.toml`

No changes to `[tool.pytest.ini_options]` needed. Testmon activates only when explicitly invoked with `--testmon`.

#### Step 3: Add `.testmondata` to `.gitignore`

**File**: `image-search-service/.gitignore`

Add:
```
# pytest-testmon local database
.testmondata
```

#### Why This Is Safe

- testmon is a dev-only dependency (in `[dependency-groups] dev`)
- It only activates when `--testmon` flag is passed (not in default `addopts`)
- The `.testmondata` file is local and gitignored
- Does not affect CI or `make test` behavior

> **⚠️ REVIEW NOTE (plan-reviewer):** testmon does NOT work with xdist. Since Phase 1 adds `-n auto` to `addopts`, running `uv run pytest --testmon` directly will fail or produce wrong results because xdist workers each have their own testmon state. **Always use `-p no:xdist` with `--testmon`**, which the `make test-affected` target already does. Add this warning to developer documentation.

#### How developers use it

```bash
# First run: builds the dependency map (takes full suite time)
uv run pytest --testmon -p no:xdist

# Subsequent runs: only re-runs tests affected by changed files
# Edit src/image_search_service/services/temporal_service.py
uv run pytest --testmon -p no:xdist
# Only runs ~5-15 tests that depend on temporal_service.py (~5-15s)
```

#### Validation

```bash
# Install and verify
uv sync --dev
uv run pytest --testmon -p no:xdist --co -q | tail -3
# Should show tests collected (first run collects all)

# Verify .testmondata was created
ls -la .testmondata
```

---

### ME6: Add `test-affected` Makefile Target

**Effort**: 10 minutes | **Risk**: None | **Impact**: Developer experience improvement

> **⚠️ REVIEW NOTE (plan-reviewer):** Phase 1 already adds `test-serial`, `test-fast`, `test-failed-first`, and `test-profile` Makefile targets. This step only needs to add the `test-affected` target (which requires testmon from ME5). Do NOT re-add targets that Phase 1 already defines — that would create duplicates or conflicting definitions.

#### File: `image-search-service/Makefile`

Add the following target after Phase 1's targets (after `test-profile:`):

```makefile
test-affected: ## Run only tests affected by recent changes (requires testmon)
	uv run pytest --testmon -p no:xdist
```

#### Integration with existing Makefile (post-Phase 1)

After Phase 1, the Makefile already has these test targets:
```makefile
test:              # Full suite (parallel, randomized, with timeout)
test-serial:       # Serial execution, deterministic order
test-fast:         # Re-run only previously failed tests
test-failed-first: # Run failed tests first, then rest
test-profile:      # Show slowest 20 tests (serial)
```

Phase 2 adds only `test-affected`:

| Target | Use Case | Expected Time |
|--------|----------|---------------|
| `make test-affected` | Only changed-file tests (testmon) | ~5-15s |

#### Why This Is Safe

- This is a purely additive Makefile target
- It doesn't change any existing target behavior
- `--testmon` only works after ME5 (testmon installed)
- `-p no:xdist` is required because testmon does not work with xdist workers

#### Validation

```bash
# Verify help shows new targets
make help | grep test

# Verify each target runs
make test-fast     # Should pass (or say "no previously failed tests")
make test-serial   # Should run all tests serially
```

---

## 3. New Makefile Targets

Summary of the new target added in ME6 (other test targets were already added in Phase 1):

| Target | Command | Description |
|--------|---------|-------------|
| `test-affected` | `uv run pytest --testmon -p no:xdist` | Run only tests affected by recent code changes |

---

## 4. Validation Steps

Run these commands after implementing each ME item to verify correctness.

### After ME1 (lazy torch import)

```bash
# 1. All device tests pass
uv run pytest tests/core/test_device.py -v

# 2. Full suite passes
uv run pytest

# 3. Import is fast (no torch loaded at import time)
python -c "
import time
t = time.time()
import image_search_service.core.device
print(f'Import time: {time.time()-t:.3f}s')
# Expected: <0.1s (was ~1.8s)
"

# 4. Device detection still works end-to-end
python -c "
from image_search_service.core.device import get_device, get_device_info
print(f'Device: {get_device()}')
print(f'Info: {get_device_info()}')
"

# 5. Mypy passes
uv run mypy src/image_search_service/core/device.py
```

### After ME2 (parametrize test_device.py)

```bash
# 1. All tests pass
uv run pytest tests/core/test_device.py -v

# 2. Verify test count (should be ~30 test cases)
uv run pytest tests/core/test_device.py --co -q | tail -1

# 3. Verify with xdist
uv run pytest tests/core/test_device.py -n 2 -v
```

### After ME3 (parametrize test_temporal_service.py)

```bash
# 1. All tests pass
uv run pytest tests/unit/test_temporal_service.py -v

# 2. Verify test count (should be ~66 test cases)
uv run pytest tests/unit/test_temporal_service.py --co -q | tail -1
```

### After ME4 (fixture consolidation)

```bash
# 1. Run all tests serially (most thorough check)
uv run pytest -p no:xdist -x -v 2>&1 | tail -5

# 2. Verify total test count unchanged
uv run pytest --co -q | tail -1
# Should still be ~1,096 (or slightly fewer after ME2/ME3)

# 3. Run the specific files that had duplicates removed
uv run pytest tests/api/test_face_suggestions.py tests/api/test_faces_routes.py \
  tests/api/test_prototype_endpoints.py tests/api/test_face_session_suggestions.py \
  tests/api/test_person_name_regression.py tests/api/test_get_faces_for_asset.py \
  tests/api/test_clusters_filtering.py tests/unit/test_suggestion_qdrant_sync.py \
  tests/faces/ -v

# 4. Run with xdist
uv run pytest -n auto
```

### After ME5 (testmon)

```bash
# 1. Install and verify
uv sync --dev

# 2. First testmon run (builds database)
uv run pytest --testmon -p no:xdist --co -q | tail -3

# 3. Verify .testmondata created
ls -la .testmondata

# 4. Verify .gitignore includes .testmondata
grep testmondata .gitignore
```

### After ME6 (Makefile target)

```bash
# 1. Verify test-affected target appears in help
make help | grep test-affected

# 2. Smoke test (requires ME5 testmon to be installed)
make test-affected
# First run builds testmon database, subsequent runs are fast
```

### Full Validation (after all ME items)

```bash
# 1. Full suite with xdist
time uv run pytest
# Expected: ~50-62s

# 2. Full suite serial (baseline comparison)
time uv run pytest -p no:xdist
# Expected: ~120-130s (slightly less than 136s due to fewer test functions)

# 3. Lint + typecheck
make lint && make typecheck

# 4. Verify test count
uv run pytest --co -q | tail -1
# Expected: ~1,038 tests (1,096 - ~58 from parametrization, but +some from finer granularity)
```

---

## 5. Rollback Plan

Each ME item is independent and can be reverted individually.

| Item | Rollback Method |
|------|----------------|
| ME1 | `git checkout -- src/image_search_service/core/device.py` — restores module-level `import torch` |
| ME2 | `git checkout -- tests/core/test_device.py` — restores individual test methods |
| ME3 | `git checkout -- tests/unit/test_temporal_service.py` — restores individual test methods |
| ME4 | `git checkout -- tests/conftest.py tests/faces/conftest.py tests/api/*.py tests/unit/test_suggestion_qdrant_sync.py` — restores all duplicate fixtures |
| ME5 | `uv remove --group dev pytest-testmon && rm -f .testmondata` — uninstall testmon |
| ME6 | Remove the 5 new targets from `Makefile` |

For a complete Phase 2 rollback:
```bash
git stash  # or git checkout -- . if no uncommitted work to save
```

---

## 6. Success Criteria

### Measurable Outcomes

| Metric | Baseline (pre-Phase 2) | Target | How to Measure |
|--------|----------------------|--------|----------------|
| Full suite wall-clock | ~55-68s | ~50-62s | `time uv run pytest` |
| Iterative dev feedback | ~55-68s | ~5-15s | `time make test-affected` |
| Test function count | 1,096 | ~1,038 | `uv run pytest --co -q \| tail -1` |
| Duplicate fixture defs | ~50 | 0 | `grep -r "def mock_image_asset" tests/ \| wc -l` (should be 1) |
| torch import time | ~1.8s | ~0s | `python -c "import time; t=time.time(); import image_search_service.core.device; print(f'{time.time()-t:.3f}s')"` |
| Lint passing | Yes | Yes | `make lint` |
| Typecheck passing | Yes | Yes | `make typecheck` |
| All tests passing | Yes | Yes | `uv run pytest` |

### Definition of Done

- [ ] All 6 ME items implemented
- [ ] `make lint && make typecheck && make test` all pass
- [ ] Test count is ~1,038 (±10) with same or better coverage
- [ ] No duplicate `mock_image_asset`, `mock_person`, or `mock_face_instance` definitions outside root conftest
- [ ] `make test-affected` works for iterative development
- [ ] `make help` shows all new targets

---

## 7. Known Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| ME1: Code that imports `device` at module level expects torch to be loaded | Low | Medium | Search for `from image_search_service.core.device import` + verify no code accesses `torch` via `device.torch` |
| ME2/ME3: Parametrized test IDs collide | Very Low | Low | Use explicit `ids` parameter on all parametrize decorators |
| ME4: Some "duplicate" fixtures have subtle differences | Low | Medium | Verified via grep: all copies are byte-identical. Run full suite after consolidation. |
| ME4: Fixture in `test_prototype_endpoints.py` has extra `mock_person` param | Known | None | This is a variant, not a duplicate. Keep as local override. |
| ME5: testmon database grows large | Low | Low | `.testmondata` is local, can be deleted anytime (`rm .testmondata`) |
| ME5: testmon misses affected tests | Low | Medium | testmon tracks line-level coverage; false negatives are rare. Full `make test` catches any misses. |

---

## 8. Ordering

### Independence Matrix

| Item | Depends On | Can Be Done In Parallel With |
|------|-----------|------------------------------|
| ME1 | Phase 1 complete | ME2, ME3, ME4, ME5, ME6 |
| ME2 | Phase 1 complete | ME1, ME3, ME4, ME5, ME6 |
| ME3 | Phase 1 complete | ME1, ME2, ME4, ME5, ME6 |
| ME4 | Phase 1 complete | ME1, ME2, ME3, ME5, ME6 |
| ME5 | Phase 1 complete | ME1, ME2, ME3, ME4, ME6 |
| ME6 | ME5 (for `test-affected` target) | ME1, ME2, ME3, ME4 |

**All items are independent** except ME6 depends on ME5 for the `test-affected` target. However, the other targets in ME6 (`test-fast`, `test-serial`, etc.) work without ME5.

### Recommended Implementation Order

1. **ME1** (lazy torch) — 15 min, immediate impact on worker startup
2. **ME5 + ME6** (testmon + Makefile targets) — 25 min, enables fast feedback for remaining work
3. **ME2** (parametrize test_device.py) — 1 hr, uses `make test-affected` for fast iteration
4. **ME3** (parametrize test_temporal_service.py) — 1 hr
5. **ME4** (fixture consolidation) — 2 hrs, do last as it touches the most files

This ordering maximizes the benefit: ME1+ME5+ME6 are done in 40 minutes and provide immediate developer experience gains. The remaining 3-4 hours of parametrization and fixture work can then leverage `make test-affected` for fast feedback.

---

## Appendix: Reference Documents

| Document | Location |
|----------|----------|
| Synthesis (all phases) | `docs/research/test-acceleration-synthesis-2026-02-15.md` |
| Dependency analysis | `docs/research/test-acceleration-dependency-analysis-2026-02-15.md` |
| Devil's advocate | `docs/research/test-acceleration-devils-advocate-2026-02-15.md` |
| Phase 1 plan | `docs/plans/test-acceleration/phase-1-quick-wins.md` |
| Phase 3 plan | `docs/plans/test-acceleration/phase-3-ci-optimization.md` |
| Phase 4 plan | `docs/plans/test-acceleration/phase-4-fixture-architecture.md` |
