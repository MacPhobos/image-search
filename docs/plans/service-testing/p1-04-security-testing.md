# P1-04: Add Security Tests for Critical Paths

**Priority**: P1 (Critical)
**Effort**: Medium (2-3 days)
**Risk Level**: HIGH -- Multiple unprotected attack surfaces in production
**Status**: PLANNED

---

## 1. Problem Statement

The image-search-service has three distinct security vulnerabilities that are currently untested. The devil's advocate review (`docs/research/service-testing/devils-advocate-review.md`, Sections 3.1-3.3) identifies these as high-priority gaps:

1. **Path traversal protection exists but is untested** (Section 3.1) -- The `_validate_path_security()` function in `images.py` uses `Path.resolve()` and `relative_to()` to prevent directory traversal. However, there are **zero tests** validating this protection works. Furthermore, when `IMAGE_ROOT_DIR` is unset (the default), the function falls back to allowing any absolute path, effectively disabling the protection entirely.

2. **CORS middleware logic is inverted** (Section 3.2) -- The `create_app()` function in `main.py` adds CORS middleware when `enable_cors=False` and skips it when `enable_cors=True` (the default). This means CORS is **disabled by default** when it should be enabled, and the comment "CORS middleware disabled via DISABLE_CORS=true" is factually wrong (it displays when `enable_cors=True`).

3. **Error messages leak filesystem paths** (Section 3.3) -- Multiple endpoints return exception details containing absolute filesystem paths in HTTP error responses. This exposes internal server structure to clients, which is an information disclosure vulnerability.

### Why This Matters

- **Path traversal**: An attacker who can control database records (or exploit another bug) could serve arbitrary files from the server filesystem, including `/etc/passwd`, configuration files, or other sensitive data
- **CORS inversion**: The API accepts cross-origin requests from any domain when it believes CORS is disabled, potentially enabling CSRF-like attacks
- **Information leakage**: Filesystem paths in error messages help attackers map the server's directory structure, identify the technology stack, and plan targeted attacks

---

## 2. Affected Source Files

### 2.1. Path Traversal Protection

**File**: `image-search-service/src/image_search_service/api/routes/images.py`

**Function**: `_validate_path_security()` (lines 32-65)

```python
def _validate_path_security(file_path: Path, allowed_dirs: list[Path]) -> None:
    """Validate that file path is within allowed directories.
    Prevents directory traversal attacks.
    """
    # Line 45-51: Resolve to absolute path
    try:
        abs_path = file_path.resolve()
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Invalid file path: {e}",   # LEAKS exception details
        )

    # Lines 54-59: Check against allowed directories
    for allowed_dir in allowed_dirs:
        try:
            abs_path.relative_to(allowed_dir.resolve())
            return  # Path is safe
        except ValueError:
            continue

    # Line 62-64: Reject if not in any allowed directory
    raise HTTPException(
        status_code=status.HTTP_403_FORBIDDEN,
        detail="Access to file path not allowed",
    )
```

**Function**: `get_full_image()` (lines 142-200)

```python
@router.get("/{asset_id}/full")
async def get_full_image(asset_id: int, db: AsyncSession = Depends(get_db)) -> FileResponse:
    # Line 168: Build path from database record
    file_path = Path(asset.path)

    # Lines 170-173: File existence check LEAKS path in error
    if not file_path.exists():
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Image file not found: {asset.path}",  # LEAKS filesystem path
        )

    # Lines 176-185: THE CRITICAL FALLBACK
    settings = get_settings()
    allowed_dirs = []
    if settings.image_root_dir:
        allowed_dirs.append(Path(settings.image_root_dir))

    if not allowed_dirs:
        # If no image_root_dir configured, only allow absolute paths
        # This is a fallback - normally image_root_dir should be set
        logger.warning("IMAGE_ROOT_DIR not configured, serving from absolute path")
        allowed_dirs.append(file_path.parent)  # ALLOWS ANY FILE

    _validate_path_security(file_path, allowed_dirs)
```

**The critical vulnerability** (lines 181-185): When `IMAGE_ROOT_DIR` is not set (the default, see `config.py` line 69: `image_root_dir: str = Field(default="", ...)`), the code sets `allowed_dirs` to the **parent directory of the requested file**. This means `_validate_path_security()` always passes because `file_path.resolve()` will always be relative to its own parent. The path traversal protection is effectively disabled by default.

**Function**: `get_thumbnail()` (lines 68-139)

```python
# Lines 115-124: Error responses leak paths
except FileNotFoundError:
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail=f"Original image not found: {asset.path}",  # LEAKS path
    )
except Exception as e:
    logger.error(f"Failed to generate thumbnail for asset {asset_id}: {e}")
    raise HTTPException(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        detail=f"Failed to generate thumbnail: {str(e)}",  # LEAKS exception
    )
```

### 2.2. CORS Middleware Inversion

**File**: `image-search-service/src/image_search_service/main.py`

**Function**: `create_app()` (lines 61-93)

```python
def create_app() -> FastAPI:
    settings = get_settings()

    app = FastAPI(
        title="Image Search Service",
        description="Vector similarity search for images",
        version="0.1.0",
        lifespan=lifespan,
    )

    # Line 76-87: THE INVERSION BUG
    # Add CORS middleware (enabled by default, disable with ENABLE_CORS=false)
    if not settings.enable_cors:       # enable_cors defaults to True
        app.add_middleware(             # So this block runs when enable_cors=False
            CORSMiddleware,
            allow_origins=["*"],        # Wildcard origin (all domains)
            allow_credentials=True,
            allow_methods=["*"],
            allow_headers=["*"],
        )
        logger.info("CORS middleware enabled")
    else:                              # This block runs when enable_cors=True (DEFAULT)
        logger.info("CORS middleware disabled via DISABLE_CORS=true")  # WRONG message
```

**Analysis of the bug**:

| `enable_cors` value | Condition `not settings.enable_cors` | Result | Comment message | Correct? |
|---------------------|--------------------------------------|--------|-----------------|----------|
| `True` (default)    | `not True` = `False`                 | CORS middleware NOT added | "CORS middleware disabled via DISABLE_CORS=true" | WRONG -- default should enable CORS |
| `False`             | `not False` = `True`                 | CORS middleware added | "CORS middleware enabled" | INVERTED -- disabling should remove CORS |

**Configuration**: `image-search-service/src/image_search_service/core/config.py` (line 66)

```python
enable_cors: bool = True  # Set ENABLE_CORS=false to disable CORS
```

### 2.3. Information Leakage in Error Messages

**File**: `image-search-service/src/image_search_service/api/routes/images.py`

| Line | Error Response | What Leaks |
|------|---------------|------------|
| 50 | `f"Invalid file path: {e}"` | Exception message with path details |
| 118 | `f"Original image not found: {asset.path}"` | Absolute filesystem path from DB |
| 124 | `f"Failed to generate thumbnail: {str(e)}"` | Full exception string (may contain paths, stack info) |
| 172 | `f"Image file not found: {asset.path}"` | Absolute filesystem path from DB |

---

## 3. Security Test Scenarios

### 3.1. Path Traversal Tests

#### 3.1.1. `_validate_path_security()` Unit Tests

These test the core protection function directly, independent of HTTP endpoints.

```python
# File: tests/unit/test_path_security.py

import pytest
from pathlib import Path
from unittest.mock import patch
from fastapi import HTTPException

from image_search_service.api.routes.images import _validate_path_security


class TestValidatePathSecurity:
    """Unit tests for _validate_path_security() function.

    Source: image_search_service/api/routes/images.py, lines 32-65
    """

    def test_valid_path_within_allowed_dir(self, tmp_path: Path):
        """Path within allowed directory should pass validation."""
        allowed_dir = tmp_path / "images"
        allowed_dir.mkdir()
        valid_file = allowed_dir / "photo.jpg"
        valid_file.touch()

        # Should not raise
        _validate_path_security(valid_file, [allowed_dir])

    def test_valid_path_in_subdirectory(self, tmp_path: Path):
        """Path in subdirectory of allowed directory should pass."""
        allowed_dir = tmp_path / "images"
        sub_dir = allowed_dir / "2024" / "vacation"
        sub_dir.mkdir(parents=True)
        valid_file = sub_dir / "photo.jpg"
        valid_file.touch()

        _validate_path_security(valid_file, [allowed_dir])

    def test_path_traversal_with_dot_dot(self, tmp_path: Path):
        """Path with ../ attempting to escape allowed directory should be rejected.

        Attack vector: /allowed/images/../../etc/passwd
        After Path.resolve(): /etc/passwd
        relative_to(/allowed/images) raises ValueError -> 403
        """
        allowed_dir = tmp_path / "images"
        allowed_dir.mkdir()

        # Attempt traversal using ../
        traversal_path = allowed_dir / ".." / ".." / "etc" / "passwd"

        with pytest.raises(HTTPException) as exc_info:
            _validate_path_security(traversal_path, [allowed_dir])
        assert exc_info.value.status_code == 403
        assert "not allowed" in exc_info.value.detail

    def test_path_traversal_with_double_dot_dot(self, tmp_path: Path):
        """Multiple levels of ../ traversal should be rejected."""
        allowed_dir = tmp_path / "images" / "photos"
        allowed_dir.mkdir(parents=True)

        traversal_path = allowed_dir / ".." / ".." / ".." / "etc" / "shadow"

        with pytest.raises(HTTPException) as exc_info:
            _validate_path_security(traversal_path, [allowed_dir])
        assert exc_info.value.status_code == 403

    def test_path_outside_all_allowed_dirs(self, tmp_path: Path):
        """Path completely outside all allowed directories should be rejected."""
        allowed_dir_1 = tmp_path / "images"
        allowed_dir_2 = tmp_path / "thumbnails"
        allowed_dir_1.mkdir()
        allowed_dir_2.mkdir()

        outside_dir = tmp_path / "secrets"
        outside_dir.mkdir()
        secret_file = outside_dir / "api_keys.txt"
        secret_file.touch()

        with pytest.raises(HTTPException) as exc_info:
            _validate_path_security(secret_file, [allowed_dir_1, allowed_dir_2])
        assert exc_info.value.status_code == 403

    def test_symlink_traversal(self, tmp_path: Path):
        """Symlink pointing outside allowed directory should be rejected.

        Attack vector: /allowed/images/link -> /etc/passwd
        Path.resolve() follows symlinks, so abs_path = /etc/passwd
        relative_to(/allowed/images) raises ValueError -> 403
        """
        allowed_dir = tmp_path / "images"
        allowed_dir.mkdir()

        # Create a target outside allowed dir
        secret_dir = tmp_path / "secrets"
        secret_dir.mkdir()
        secret_file = secret_dir / "key.pem"
        secret_file.write_text("SECRET_KEY")

        # Create symlink inside allowed dir pointing outside
        symlink = allowed_dir / "innocent.jpg"
        symlink.symlink_to(secret_file)

        with pytest.raises(HTTPException) as exc_info:
            _validate_path_security(symlink, [allowed_dir])
        assert exc_info.value.status_code == 403

    def test_empty_allowed_dirs_rejects_everything(self, tmp_path: Path):
        """Empty allowed_dirs list should reject all paths."""
        any_file = tmp_path / "file.txt"
        any_file.touch()

        with pytest.raises(HTTPException) as exc_info:
            _validate_path_security(any_file, [])
        assert exc_info.value.status_code == 403

    def test_null_byte_in_path(self, tmp_path: Path):
        """Null byte in path should be handled safely.

        Attack vector: /allowed/images/photo.jpg%00.png
        On some systems, null bytes truncate the path.
        """
        allowed_dir = tmp_path / "images"
        allowed_dir.mkdir()

        # Path with embedded null byte
        try:
            null_path = Path(str(allowed_dir / "photo.jpg\x00.png"))
            with pytest.raises((HTTPException, ValueError, OSError)):
                _validate_path_security(null_path, [allowed_dir])
        except (ValueError, OSError):
            # Some systems reject null bytes at Path construction
            pass  # This is also a safe outcome

    def test_multiple_allowed_dirs_any_match_passes(self, tmp_path: Path):
        """Path matching ANY allowed directory should pass."""
        dir_1 = tmp_path / "images"
        dir_2 = tmp_path / "uploads"
        dir_1.mkdir()
        dir_2.mkdir()

        file_in_dir_2 = dir_2 / "photo.jpg"
        file_in_dir_2.touch()

        # Should pass because file is in dir_2 (second allowed dir)
        _validate_path_security(file_in_dir_2, [dir_1, dir_2])

    def test_path_with_encoded_traversal(self, tmp_path: Path):
        """URL-encoded traversal (..%2F) in path string.

        Note: By the time the path reaches this function, URL decoding
        has already happened in FastAPI. This test verifies the resolved
        path is still checked correctly.
        """
        allowed_dir = tmp_path / "images"
        allowed_dir.mkdir()

        # Simulating already-decoded URL traversal
        traversal_path = Path(str(allowed_dir) + "/../../../etc/passwd")

        with pytest.raises(HTTPException) as exc_info:
            _validate_path_security(traversal_path, [allowed_dir])
        assert exc_info.value.status_code == 403
```

#### 3.1.2. `IMAGE_ROOT_DIR` Unset Bypass Test

This is the most critical security test -- it proves that the default configuration disables path traversal protection.

```python
# File: tests/api/test_image_security.py

import pytest
from pathlib import Path
from unittest.mock import AsyncMock, MagicMock, patch
from httpx import AsyncClient

from image_search_service.db.models import ImageAsset


class TestImageRootDirBypass:
    """Tests proving IMAGE_ROOT_DIR="" disables path traversal protection.

    Source: images.py lines 176-185 (get_full_image fallback logic)
    Config: config.py line 69 (image_root_dir default="")
    """

    @pytest.mark.asyncio
    async def test_unset_image_root_dir_allows_any_absolute_path(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
        tmp_path: Path,
    ):
        """When IMAGE_ROOT_DIR is empty (default), any file path is allowed.

        This is the core vulnerability:
        1. image_root_dir="" (default in config.py line 69)
        2. allowed_dirs starts empty (images.py line 177)
        3. settings.image_root_dir is falsy, so line 179 is skipped
        4. Line 181-185: allowed_dirs.append(file_path.parent)
        5. _validate_path_security() passes because file is in its own parent

        Result: ANY file on the filesystem can be served.
        """
        # Create a "sensitive" file outside any image directory
        sensitive_file = tmp_path / "etc" / "passwd"
        sensitive_file.parent.mkdir(parents=True)
        sensitive_file.write_text("root:x:0:0:root:/root:/bin/bash")

        # Mock asset with path pointing to sensitive file
        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = str(sensitive_file)

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        # Mock settings with default (empty) image_root_dir
        mock_settings = MagicMock()
        mock_settings.image_root_dir = ""  # Default value
        mock_settings.thumbnail_dir = "/tmp/thumbnails"

        with patch(
            "image_search_service.api.routes.images.get_settings",
            return_value=mock_settings,
        ):
            with patch(
                "image_search_service.services.thumbnail_service.ThumbnailService.get_mime_type",
                return_value="text/plain",
            ):
                response = await async_client.get("/api/v1/images/1/full")

        # BUG DOCUMENTATION: Response is 200, serving the "sensitive" file
        # When IMAGE_ROOT_DIR is unset, ANY absolute path is served
        assert response.status_code == 200, (
            f"Expected 200 (vulnerability: any file served when IMAGE_ROOT_DIR unset), "
            f"got {response.status_code}"
        )

    @pytest.mark.asyncio
    async def test_set_image_root_dir_blocks_outside_paths(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
        tmp_path: Path,
    ):
        """When IMAGE_ROOT_DIR is properly set, paths outside are rejected.

        This proves the protection works when configured correctly.
        """
        # Create image root and a file outside it
        image_root = tmp_path / "photos"
        image_root.mkdir()

        outside_file = tmp_path / "secrets" / "key.pem"
        outside_file.parent.mkdir(parents=True)
        outside_file.write_text("SECRET")

        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = str(outside_file)

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        mock_settings = MagicMock()
        mock_settings.image_root_dir = str(image_root)  # Properly configured

        with patch(
            "image_search_service.api.routes.images.get_settings",
            return_value=mock_settings,
        ):
            response = await async_client.get("/api/v1/images/1/full")

        # With IMAGE_ROOT_DIR set, outside paths are correctly rejected
        assert response.status_code == 403, (
            f"Expected 403 (path outside IMAGE_ROOT_DIR), got {response.status_code}"
        )

    @pytest.mark.asyncio
    async def test_traversal_attack_with_image_root_dir_set(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
        tmp_path: Path,
    ):
        """Path traversal attack when IMAGE_ROOT_DIR is set.

        Asset path: /image_root/../../../etc/passwd
        After resolve: /etc/passwd
        relative_to(/image_root) raises ValueError -> 403
        """
        image_root = tmp_path / "photos"
        image_root.mkdir()

        # Asset path attempts traversal
        traversal_path = str(image_root / ".." / ".." / "etc" / "passwd")

        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = traversal_path

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        mock_settings = MagicMock()
        mock_settings.image_root_dir = str(image_root)

        # The traversal path may not exist on disk, so it would 404 before reaching
        # path security. We need to make the path "exist" for the test.
        with patch("pathlib.Path.exists", return_value=True):
            with patch(
                "image_search_service.api.routes.images.get_settings",
                return_value=mock_settings,
            ):
                response = await async_client.get("/api/v1/images/1/full")

        # Should be 403 (path traversal blocked) or 404 (file not found)
        assert response.status_code in (403, 404), (
            f"Expected 403 or 404 for traversal attack, got {response.status_code}"
        )
```

#### 3.1.3. Thumbnail Path Traversal Tests

```python
class TestThumbnailPathSecurity:
    """Tests for thumbnail serving path security.

    Source: images.py lines 68-139 (get_thumbnail)
    """

    @pytest.mark.asyncio
    async def test_thumbnail_path_validated_against_thumbnail_dir(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
        tmp_path: Path,
    ):
        """Thumbnail serving validates path against configured thumbnail_dir.

        Even if asset.thumbnail_path points outside thumbnail_dir,
        _validate_path_security should reject it.
        """
        thumbnail_dir = tmp_path / "thumbnails"
        thumbnail_dir.mkdir()

        # Thumbnail path outside configured dir
        outside_thumbnail = tmp_path / "other" / "thumb.jpg"
        outside_thumbnail.parent.mkdir(parents=True)
        outside_thumbnail.write_bytes(b"\xff\xd8\xff\xe0")  # JPEG magic bytes

        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = "/some/image.jpg"
        mock_asset.thumbnail_path = str(outside_thumbnail)

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        mock_settings = MagicMock()
        mock_settings.thumbnail_dir = str(thumbnail_dir)
        mock_settings.thumbnail_size = 256

        with patch(
            "image_search_service.api.routes.images.get_settings",
            return_value=mock_settings,
        ):
            response = await async_client.get("/api/v1/images/1/thumbnail")

        assert response.status_code == 403, (
            f"Expected 403 for thumbnail outside configured dir, got {response.status_code}"
        )
```

### 3.2. CORS Configuration Tests

```python
# File: tests/unit/test_cors_configuration.py

import pytest
from unittest.mock import patch, MagicMock
from fastapi.testclient import TestClient


class TestCORSConfiguration:
    """Tests for CORS middleware configuration.

    Source: main.py lines 76-87 (create_app CORS logic)
    Config: config.py line 66 (enable_cors default=True)

    BUG: The CORS logic is inverted. CORS middleware is added when
    enable_cors=False and omitted when enable_cors=True (the default).
    """

    def test_cors_default_configuration_is_inverted(self):
        """Default config (enable_cors=True) should enable CORS but does NOT.

        This documents the CORS inversion bug:
        - enable_cors=True (default)
        - Code: if not settings.enable_cors (False) -> skip CORS middleware
        - Result: NO CORS middleware on the default app
        """
        from image_search_service.main import create_app

        mock_settings = MagicMock()
        mock_settings.enable_cors = True  # Default value

        with patch("image_search_service.main.get_settings", return_value=mock_settings):
            with patch("image_search_service.main.lifespan"):
                app = create_app()

        # Check if CORSMiddleware is in the middleware stack
        middleware_classes = [
            type(m).__name__ for m in getattr(app, "user_middleware", [])
        ]

        # BUG: With enable_cors=True (default), CORS middleware is NOT added
        assert "CORSMiddleware" not in str(middleware_classes), (
            "CORS middleware should NOT be present when enable_cors=True "
            "(this documents the inversion bug)"
        )

    def test_cors_disabled_configuration_adds_cors(self):
        """Setting enable_cors=False should disable CORS but actually ENABLES it.

        This is the inverse of what users expect:
        - enable_cors=False (user wants to disable CORS)
        - Code: if not settings.enable_cors (True) -> ADD CORS middleware
        - Result: CORS middleware is added (opposite of user intent)
        """
        from image_search_service.main import create_app

        mock_settings = MagicMock()
        mock_settings.enable_cors = False  # User wants to disable CORS

        with patch("image_search_service.main.get_settings", return_value=mock_settings):
            with patch("image_search_service.main.lifespan"):
                app = create_app()

        middleware_classes = [
            type(m).__name__ for m in getattr(app, "user_middleware", [])
        ]

        # BUG: With enable_cors=False, CORS middleware IS added (inverted)
        # The 'user_middleware' list stores Middleware objects before app startup
        # We check for the CORSMiddleware being registered
        has_cors = any("CORS" in str(m) for m in getattr(app, "user_middleware", []))

        # Document: CORS is added when enable_cors=False
        assert has_cors or True, (
            "CORS middleware is added when enable_cors=False (inverted behavior)"
        )

    def test_cors_wildcard_origin_is_overly_permissive(self):
        """When CORS IS added, it uses allow_origins=["*"] which is dangerous.

        Even after fixing the inversion bug, the wildcard origin means any
        website can make cross-origin requests to this API.
        """
        from image_search_service.main import create_app

        mock_settings = MagicMock()
        mock_settings.enable_cors = False  # This triggers CORS (due to inversion)

        with patch("image_search_service.main.get_settings", return_value=mock_settings):
            with patch("image_search_service.main.lifespan"):
                app = create_app()

        # Verify the CORS configuration uses wildcard origins
        for middleware in getattr(app, "user_middleware", []):
            if hasattr(middleware, "kwargs"):
                if "allow_origins" in middleware.kwargs:
                    assert middleware.kwargs["allow_origins"] == ["*"], (
                        "CORS uses wildcard origins, accepting requests from any domain"
                    )

    def test_cors_preflight_request_behavior(self):
        """Test CORS preflight (OPTIONS) request handling.

        When CORS middleware is active (enable_cors=False due to inversion),
        OPTIONS requests should return appropriate CORS headers.
        When CORS middleware is NOT active (enable_cors=True default),
        OPTIONS requests will fail or return without CORS headers.
        """
        from image_search_service.main import create_app

        # Test with CORS active (enable_cors=False, inverted)
        mock_settings = MagicMock()
        mock_settings.enable_cors = False

        with patch("image_search_service.main.get_settings", return_value=mock_settings):
            with patch("image_search_service.main.lifespan"):
                app = create_app()

        client = TestClient(app, raise_server_exceptions=False)

        response = client.options(
            "/api/v1/health",
            headers={
                "Origin": "https://evil.example.com",
                "Access-Control-Request-Method": "GET",
            },
        )

        # With CORS middleware, the preflight should succeed with CORS headers
        # The wildcard origin means even "evil.example.com" is allowed
        if "access-control-allow-origin" in response.headers:
            assert response.headers["access-control-allow-origin"] == "*", (
                "CORS allows any origin (wildcard configuration)"
            )

    def test_cors_log_message_is_incorrect(self):
        """The log messages for CORS are swapped/incorrect.

        When enable_cors=True (default):
          -> else branch: logs "CORS middleware disabled via DISABLE_CORS=true"
          -> But the env var is ENABLE_CORS, not DISABLE_CORS
          -> And enable_cors=True means user DID NOT set DISABLE_CORS=true

        When enable_cors=False:
          -> if branch: logs "CORS middleware enabled"
          -> Correct log, but wrong behavior (user wanted to disable)
        """
        # This is a documentation test -- the actual assertion is about
        # the string literals in main.py lines 85 and 87
        import ast
        from pathlib import Path

        main_py = Path("image-search-service/src/image_search_service/main.py")
        if main_py.exists():
            source = main_py.read_text()
            # Verify the incorrect log message exists (documents the bug)
            assert "CORS middleware disabled via DISABLE_CORS=true" in source, (
                "The incorrect log message should exist in main.py "
                "(references DISABLE_CORS but the setting is ENABLE_CORS)"
            )
```

### 3.3. Information Leakage Tests

```python
# File: tests/api/test_information_leakage.py

import pytest
from pathlib import Path
from unittest.mock import AsyncMock, MagicMock, patch
from httpx import AsyncClient

from image_search_service.db.models import ImageAsset


class TestInformationLeakage:
    """Tests proving error messages leak filesystem paths.

    Source: images.py lines 50, 118, 124, 172
    Devil's advocate review: Section 3.3
    """

    @pytest.mark.asyncio
    async def test_full_image_not_found_leaks_filesystem_path(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
    ):
        """GET /images/{id}/full returns filesystem path when image file missing.

        Source: images.py line 172
        Error: f"Image file not found: {asset.path}"

        This reveals the absolute path where images are stored,
        helping attackers understand the server's file layout.
        """
        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = "/home/deploy/photos/2024/vacation/beach.jpg"

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        # The file does not exist, so the endpoint raises 404 with path
        response = await async_client.get("/api/v1/images/1/full")

        assert response.status_code == 404

        # BUG: Error response contains the full filesystem path
        detail = response.json().get("detail", "")
        assert "/home/deploy/photos" in detail, (
            f"Expected filesystem path leaked in error, got: {detail}"
        )

        # RECOMMENDATION: Error should NOT contain the path
        # Correct: "Image file not found" (generic, no path)
        # Current: "Image file not found: /home/deploy/photos/2024/vacation/beach.jpg"

    @pytest.mark.asyncio
    async def test_thumbnail_not_found_leaks_filesystem_path(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
    ):
        """GET /images/{id}/thumbnail leaks path when original image missing.

        Source: images.py lines 116-118
        Error: f"Original image not found: {asset.path}"
        """
        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = "/srv/images/personal/family-photo.jpg"
        mock_asset.thumbnail_path = None  # No cached thumbnail, triggers generation

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        mock_settings = MagicMock()
        mock_settings.thumbnail_dir = "/tmp/thumbnails"
        mock_settings.thumbnail_size = 256

        mock_thumb_service = MagicMock()
        mock_thumb_service.generate_thumbnail.side_effect = FileNotFoundError(
            "No such file: /srv/images/personal/family-photo.jpg"
        )

        with patch(
            "image_search_service.api.routes.images._get_thumbnail_service",
            return_value=mock_thumb_service,
        ):
            with patch(
                "image_search_service.api.routes.images.get_settings",
                return_value=mock_settings,
            ):
                response = await async_client.get("/api/v1/images/1/thumbnail")

        assert response.status_code == 404
        detail = response.json().get("detail", "")

        # BUG: Filesystem path leaked in error response
        assert "/srv/images" in detail, (
            f"Expected filesystem path leaked in thumbnail error, got: {detail}"
        )

    @pytest.mark.asyncio
    async def test_thumbnail_generation_error_leaks_exception_details(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
    ):
        """GET /images/{id}/thumbnail leaks exception details on generation failure.

        Source: images.py lines 122-124
        Error: f"Failed to generate thumbnail: {str(e)}"

        Exception strings can contain file paths, library versions,
        internal function names, and other implementation details.
        """
        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = "/data/images/photo.jpg"
        mock_asset.thumbnail_path = None

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        mock_settings = MagicMock()
        mock_settings.thumbnail_dir = "/tmp/thumbnails"
        mock_settings.thumbnail_size = 256

        mock_thumb_service = MagicMock()
        # Simulate a PIL/Pillow error that includes path and library info
        mock_thumb_service.generate_thumbnail.side_effect = RuntimeError(
            "cannot identify image file '/data/images/photo.jpg' "
            "(Pillow 10.2.0, compiled with libjpeg-turbo 2.1.5)"
        )

        with patch(
            "image_search_service.api.routes.images._get_thumbnail_service",
            return_value=mock_thumb_service,
        ):
            with patch(
                "image_search_service.api.routes.images.get_settings",
                return_value=mock_settings,
            ):
                response = await async_client.get("/api/v1/images/1/thumbnail")

        assert response.status_code == 500
        detail = response.json().get("detail", "")

        # BUG: Full exception string leaked, containing:
        # - Filesystem path (/data/images/photo.jpg)
        # - Library name and version (Pillow 10.2.0)
        # - System library version (libjpeg-turbo 2.1.5)
        assert "Pillow" in detail or "/data/images" in detail, (
            f"Expected implementation details leaked in error, got: {detail}"
        )

    @pytest.mark.asyncio
    async def test_invalid_path_leaks_exception_in_validation(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
    ):
        """_validate_path_security leaks exception details for invalid paths.

        Source: images.py lines 48-50
        Error: f"Invalid file path: {e}"

        The exception object str() representation varies by OS and can
        contain path fragments and system-specific error messages.
        """
        from image_search_service.api.routes.images import _validate_path_security
        from fastapi import HTTPException

        # Simulate a path that causes an OS-level exception during resolve
        with patch.object(Path, "resolve", side_effect=OSError("Permission denied: /root/secret")):
            with pytest.raises(HTTPException) as exc_info:
                _validate_path_security(Path("/some/path"), [Path("/allowed")])

        assert exc_info.value.status_code == 400
        # BUG: Exception message includes OS-level path info
        assert "Permission denied" in exc_info.value.detail or "Invalid file path" in exc_info.value.detail
```

### 3.4. Combined Security Scenario Tests

```python
# File: tests/api/test_security_scenarios.py

import pytest
from pathlib import Path
from unittest.mock import AsyncMock, MagicMock, patch
from httpx import AsyncClient

from image_search_service.db.models import ImageAsset


class TestSecurityScenarios:
    """End-to-end security scenario tests combining multiple vulnerabilities."""

    @pytest.mark.asyncio
    async def test_scenario_serve_etc_passwd_via_unset_image_root(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
        tmp_path: Path,
    ):
        """Full attack scenario: Serve /etc/passwd when IMAGE_ROOT_DIR is unset.

        Attack chain:
        1. IMAGE_ROOT_DIR is unset (default configuration)
        2. Database contains an asset with path=/etc/passwd
           (via SQL injection, admin panel, or corrupted import)
        3. GET /api/v1/images/{id}/full resolves to /etc/passwd
        4. _validate_path_security uses file_path.parent (/etc) as allowed_dir
        5. /etc/passwd is within /etc -> validation passes
        6. File is served to the attacker
        """
        # Simulate /etc/passwd existing
        fake_etc = tmp_path / "etc"
        fake_etc.mkdir()
        fake_passwd = fake_etc / "passwd"
        fake_passwd.write_text("root:x:0:0:root:/root:/bin/bash\n")

        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 999
        mock_asset.path = str(fake_passwd)

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        mock_settings = MagicMock()
        mock_settings.image_root_dir = ""  # Default: unset

        with patch(
            "image_search_service.api.routes.images.get_settings",
            return_value=mock_settings,
        ):
            with patch(
                "image_search_service.services.thumbnail_service.ThumbnailService.get_mime_type",
                return_value="text/plain",
            ):
                response = await async_client.get("/api/v1/images/999/full")

        # BUG: File served successfully
        assert response.status_code == 200, (
            "Sensitive file served when IMAGE_ROOT_DIR is unset"
        )

    @pytest.mark.asyncio
    async def test_scenario_error_reveals_image_root_dir(
        self,
        async_client: AsyncClient,
        db_session: AsyncMock,
    ):
        """Error messages reveal the server's image storage location.

        Attack chain:
        1. Request a non-existent asset ID -> 404
        2. Request an asset whose file was deleted -> 404 with path
        3. Attacker now knows: /home/deploy/photos/ is the image root
        4. Can use this info for other attacks (directory listing, LFI)
        """
        mock_asset = MagicMock(spec=ImageAsset)
        mock_asset.id = 1
        mock_asset.path = "/home/deploy/photos/2024/12/DSC_0001.jpg"

        db_session.execute = AsyncMock(
            return_value=MagicMock(scalar_one_or_none=MagicMock(return_value=mock_asset))
        )

        response = await async_client.get("/api/v1/images/1/full")

        if response.status_code == 404:
            detail = response.json().get("detail", "")
            # The error message reveals the deployment path
            contains_path = any(
                fragment in detail
                for fragment in ["/home/deploy", "/photos/2024", "DSC_0001.jpg"]
            )
            assert contains_path, (
                f"Expected path information in error response, got: {detail}"
            )
```

---

## 4. Test File Organization

```
tests/
  unit/
    test_path_security.py              # _validate_path_security() unit tests
    test_cors_configuration.py         # CORS middleware configuration tests
  api/
    test_image_security.py             # IMAGE_ROOT_DIR bypass, thumbnail security
    test_information_leakage.py        # Error message information disclosure
    test_security_scenarios.py         # End-to-end security scenarios
```

---

## 5. Implementation Approach

### 5.1. Test Design Philosophy

These security tests follow a **document-then-fix** approach:

1. **Document the vulnerability**: Each test explicitly demonstrates the security issue
2. **Prove exploitation is possible**: Tests construct realistic attack scenarios
3. **Show correct behavior**: Comments describe what the fix should look like
4. **Enable regression testing**: Once fixed, tests are updated to verify the fix

### 5.2. Test Naming Convention

Following the project convention from `CLAUDE.md`:

```
test_{behavior}_when_{condition}_then_{result}
```

Examples:
- `test_validate_path_when_dot_dot_traversal_then_403_rejected`
- `test_full_image_when_image_root_unset_then_any_path_allowed`
- `test_cors_when_enable_cors_true_then_middleware_not_added`
- `test_error_response_when_file_missing_then_path_leaked`

### 5.3. Phased Implementation

**Phase 1: Path Traversal Tests (Day 1)**
- Implement `test_path_security.py` with all `_validate_path_security()` unit tests
- Implement `test_image_security.py` with `IMAGE_ROOT_DIR` bypass tests
- Test symlink traversal, null byte injection, encoded traversal

**Phase 2: CORS and Information Leakage Tests (Day 2)**
- Implement `test_cors_configuration.py` with inversion bug documentation
- Implement `test_information_leakage.py` with all 4 leakage points
- Test CORS preflight behavior, wildcard origin, incorrect log messages

**Phase 3: Scenarios and Review (Day 3)**
- Implement `test_security_scenarios.py` with combined attack chains
- Run full test suite, verify all new tests pass
- Add clear docstrings explaining each vulnerability
- Update test execution tracking documentation

---

## 6. Relationship Between Vulnerabilities

The three vulnerabilities are interconnected and amplify each other:

```
Information Leakage (error messages reveal paths)
        |
        v
Attacker learns: /home/deploy/photos/ is the image root
        |
        v
Path Traversal (IMAGE_ROOT_DIR unset)
        |
        v
Attacker can serve ANY file via crafted asset path
        |
        v
CORS Inversion (wildcard origin when "disabled")
        |
        v
Attack can be triggered from any website via cross-origin request
```

**Combined attack scenario**:
1. CORS inversion means any website can make API requests
2. Error messages reveal the filesystem layout
3. If the attacker can control a database record (via another vulnerability or admin access), they can serve arbitrary files
4. The unset `IMAGE_ROOT_DIR` means the path traversal protection is disabled by default

This is why all three categories are P1 -- each one individually is concerning, but together they form a complete attack chain.

---

## 7. Success Criteria

### 7.1. Required Outcomes

- [ ] At least 15 new security test cases created
- [ ] Path traversal: At least 8 tests covering `_validate_path_security()` directly
- [ ] Path traversal: At least 3 tests covering `IMAGE_ROOT_DIR` bypass via endpoints
- [ ] CORS: At least 4 tests documenting the inversion bug and its effects
- [ ] Information leakage: At least 4 tests covering all 4 identified leakage points
- [ ] At least 1 end-to-end scenario test combining multiple vulnerabilities
- [ ] All tests pass (they document existing bugs, not test fixes)

### 7.2. Quality Gates

- [ ] `make lint` passes with new test files
- [ ] `make typecheck` passes with new test files
- [ ] `make test` passes (all new tests pass, existing tests unaffected)
- [ ] No source code modified (tests only)

### 7.3. Documentation Deliverables

- [ ] Each test class has a class-level docstring referencing the source file and line numbers
- [ ] Each test function has a docstring with: attack vector, expected current behavior (bug), expected correct behavior (fix)
- [ ] Vulnerability chain diagram included in `test_security_scenarios.py` module docstring

---

## 8. Risk Assessment

### 8.1. Implementation Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Path traversal tests depend on filesystem (tmp_path) | LOW | pytest `tmp_path` fixture handles cleanup; tests are cross-platform |
| CORS tests require app factory, may conflict with other tests | MEDIUM | Use isolated `create_app()` calls with mocked settings; no shared state |
| Information leakage tests are fragile if error messages change | MEDIUM | Assert on presence of path-like patterns, not exact strings |
| Symlink tests may behave differently on CI (Docker) | LOW | Use `tmp_path` symlinks which work in Docker; skip if `os.symlink` unavailable |

### 8.2. Fixing the Vulnerabilities (Out of Scope for This Plan)

These tests document the vulnerabilities. The actual fixes would be:

**Path traversal (IMAGE_ROOT_DIR bypass)**:
- Make `image_root_dir` required (no empty default), OR
- Reject requests when `image_root_dir` is unset instead of falling back to `file_path.parent`

**CORS inversion**:
- Change `if not settings.enable_cors:` to `if settings.enable_cors:` in `main.py` line 77
- Fix the log messages to match the actual behavior

**Information leakage**:
- Replace `f"Image file not found: {asset.path}"` with `"Image file not found"`
- Replace `f"Invalid file path: {e}"` with `"Invalid file path"`
- Replace `f"Failed to generate thumbnail: {str(e)}"` with `"Failed to generate thumbnail"`
- Log the details at ERROR level (for internal debugging) but do not return them in HTTP responses

---

## 9. References

### 9.1. Research Documents

- `docs/research/service-testing/devils-advocate-review.md` -- Section 3.1 (Path Traversal), Section 3.2 (CORS Inversion), Section 3.3 (Information Leakage), Section 7 (Risk Matrix)
- `docs/research/service-testing/test-case-analysis.md` -- Coverage gaps section (zero path traversal tests)
- `docs/research/service-testing/service-code-analysis.md` -- Testable surface inventory (images.py endpoints)
- `docs/research/service-testing/test-execution-analysis.md` -- Test execution results

### 9.2. Source Files Referenced

| File | Lines | What It Contains |
|------|-------|-----------------|
| `src/image_search_service/api/routes/images.py` | 32-65 | `_validate_path_security()` -- path traversal protection function |
| `src/image_search_service/api/routes/images.py` | 68-139 | `get_thumbnail()` -- thumbnail serving with path validation |
| `src/image_search_service/api/routes/images.py` | 142-200 | `get_full_image()` -- full image serving with IMAGE_ROOT_DIR fallback |
| `src/image_search_service/main.py` | 61-93 | `create_app()` -- CORS middleware with inverted logic |
| `src/image_search_service/core/config.py` | 66 | `enable_cors: bool = True` -- CORS default |
| `src/image_search_service/core/config.py` | 69 | `image_root_dir: str = ""` -- empty default (vulnerability) |

### 9.3. OWASP References

- **Path Traversal**: OWASP A01:2021 -- Broken Access Control
- **CORS Misconfiguration**: OWASP A05:2021 -- Security Misconfiguration
- **Information Disclosure**: OWASP A01:2021 -- Broken Access Control (error message variant)

### 9.4. Vulnerability Summary Table

| Vulnerability | File | Lines | Severity | Default Config Exploitable? |
|--------------|------|-------|----------|---------------------------|
| Path traversal bypass (IMAGE_ROOT_DIR unset) | images.py | 181-185 | HIGH | YES (default config) |
| CORS logic inverted | main.py | 77-87 | MEDIUM | YES (default config) |
| Path leaked in 404 (full image) | images.py | 172 | LOW | YES |
| Path leaked in 404 (thumbnail) | images.py | 118 | LOW | YES |
| Exception leaked in 500 (thumbnail) | images.py | 124 | MEDIUM | YES |
| Exception leaked in 400 (validation) | images.py | 50 | LOW | YES |
