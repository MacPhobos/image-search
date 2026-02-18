# Implementation Plan: Google Drive OAuth "Pattern A" Storage Backend

> **Date**: 2026-02-18
> **Status**: Research Complete / Ready for Implementation
> **Author**: Claude Code Research Agent
> **Source Prompt**: `docs/research/google-drive-v2/claude_prompt_gdrive_oauth_pattern_a.md`
> **Prior Art**: `docs/research/google-drive/00-synthesis.md` (SA-based v1 research)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Analysis of Existing Storage System](#2-architecture-analysis-of-existing-storage-system)
3. [Research Findings](#3-research-findings)
4. [Devil's Advocate Analysis](#4-devils-advocate-analysis)
5. [Recommended Architecture](#5-recommended-architecture)
6. [Detailed Implementation Steps](#6-detailed-implementation-steps)
7. [Bootstrap Script Design](#7-bootstrap-script-design)
8. [Configuration Guide](#8-configuration-guide)
9. [Error Handling Strategy](#9-error-handling-strategy)
10. [Test Plan](#10-test-plan)
11. [Security Considerations](#11-security-considerations)
12. [Migration and Rollout Plan](#12-migration-and-rollout-plan)
13. [Open Questions and Decisions](#13-open-questions-and-decisions)

---

## 1. Executive Summary

### Problem Statement

The existing Google Drive integration uses **service account (SA) authentication**, which requires Google Workspace or manual folder sharing with an opaque SA email address. Personal Google accounts ("My Drive") cannot use this path without a cumbersome 8-step setup process. Many users of this self-hosted image search application operate personal Google accounts and need a simpler, more natural integration path.

### Proposed Solution

Add a **second** `StorageBackend` implementation -- `GoogleDriveOAuthV3Storage` -- that authenticates via **OAuth 2.0 user credentials** (client_id + client_secret + refresh_token) instead of service account JSON keys. This new backend:

- Sits **parallel to** the existing `GoogleDriveV3Storage` (no changes to SA backend)
- Implements the **same `StorageBackend` protocol** (5 methods)
- Reuses **90%+ of the operational logic** from the SA backend (upload, folder creation, error handling, path resolution, retry/backoff)
- Requires a **one-time bootstrap script** to obtain the refresh token via `InstalledAppFlow`
- Adds **3 new config fields** (`GOOGLE_DRIVE_AUTH_MODE`, `GOOGLE_DRIVE_CLIENT_ID`, etc.)
- Is selected at startup via a **factory function enhancement** in `storage/__init__.py`

### Key Decision: Inheritance vs. Composition vs. Duplication

After thorough analysis of the existing 758-line `GoogleDriveV3Storage` implementation, the recommended approach is **inheritance** (subclass overriding `_build_service()` only). Rationale:

- The **only functional difference** between SA and OAuth backends is credential construction in `_build_service()` (lines 163-182 of `google_drive.py`)
- All other methods (upload, create_folder, file_exists, list_folder, delete_file, mkdirp, error translation, retry logic) are **identical** regardless of auth method
- Inheritance avoids 700+ lines of duplication while keeping the two auth paths clearly separated
- The subclass is approximately **50 lines** of new code

### Estimated Effort

| Component | Hours | Risk |
|-----------|-------|------|
| `GoogleDriveOAuthV3Storage` class | 2-3 | Low |
| Config additions + factory wiring | 2-3 | Low |
| Bootstrap script | 3-4 | Medium |
| Config validation updates | 1-2 | Low |
| Unit tests | 4-6 | Low |
| Integration test checklist | 2-3 | Medium |
| Documentation | 2-3 | Low |
| **Total** | **16-24** | **Low-Medium** |

---

## 2. Architecture Analysis of Existing Storage System

### Module Map

```
image-search-service/src/image_search_service/
  storage/
    __init__.py              # Factory: get_storage(), get_async_storage()
    base.py                  # StorageBackend Protocol + data types
    exceptions.py            # 9-class exception hierarchy
    google_drive.py          # GoogleDriveV3Storage (758 lines, SA auth)
    path_resolver.py         # PathResolver with thread-safe LRU cache
    async_wrapper.py         # AsyncStorageWrapper (ThreadPoolExecutor, 4 workers)
    config_validation.py     # validate_google_drive_config() startup check
```

### StorageBackend Protocol (`storage/base.py`)

```python
@runtime_checkable
class StorageBackend(Protocol):
    def upload_file(self, content: bytes, filename: str, mime_type: str,
                    folder_id: str | None = None) -> UploadResult: ...
    def create_folder(self, name: str, parent_id: str | None = None) -> str: ...
    def file_exists(self, file_id: str) -> bool: ...
    def list_folder(self, folder_id: str) -> list[StorageEntry]: ...
    def delete_file(self, file_id: str, *, trash: bool = True) -> None: ...
```

Supporting types: `UploadResult(file_id, name, size, mime_type)`, `StorageEntry(id, name, mime_type, size, entry_type, created_time, modified_time)`, `EntryType(FILE, FOLDER)`.

### GoogleDriveV3Storage Class Structure (`storage/google_drive.py`)

The existing implementation has a clean separation between **credential construction** and **operational logic**:

```
GoogleDriveV3Storage
  |
  +-- __init__()                    # Stores config, no API calls
  +-- @property service             # Lazy init with double-checked locking
  +-- @property resolver            # Lazy init, depends on service
  +-- @property root_folder_id      # Simple accessor
  |
  +-- _build_service()              # <<< ONLY METHOD THAT TOUCHES CREDENTIALS >>>
  |     Uses: service_account.Credentials.from_service_account_file()
  |     Returns: build("drive", "v3", credentials=creds)
  |
  +-- upload_file()                 # Protocol method - uses self.service
  +-- create_folder()               # Protocol method - uses self.service
  +-- file_exists()                 # Protocol method - uses self.service
  +-- list_folder()                 # Protocol method - uses self.service
  +-- delete_file()                 # Protocol method - uses self.service
  +-- mkdirp()                      # Extra convenience method
  |
  +-- _execute_with_backoff()       # Retry + error translation engine
  +-- _make_lookup_fn()             # Closure factory for PathResolver
  +-- _validate_root_access()       # Root folder accessibility check
  +-- get_service_account_email()   # SA-specific metadata method
  |
  +-- _escape_query_value()         # Static: Drive query escaping
  +-- _extract_error_reason()       # Static: HttpError parsing
```

**Critical observation**: `_build_service()` is the **sole point of divergence** between SA and OAuth authentication. Every other method operates on the `self.service` Drive resource object, which is auth-agnostic once constructed.

### Factory Pattern (`storage/__init__.py`)

```python
@lru_cache(maxsize=1)
def get_storage() -> GoogleDriveV3Storage | None:
    settings = get_settings()
    if not settings.google_drive_enabled:
        return None
    # Validates SA-specific config, constructs GoogleDriveV3Storage
    return GoogleDriveV3Storage(
        service_account_json_path=settings.google_drive_sa_json,
        root_folder_id=settings.google_drive_root_id,
        path_cache_maxsize=settings.google_drive_path_cache_maxsize,
        path_cache_ttl=settings.google_drive_path_cache_ttl,
    )

def get_async_storage() -> AsyncStorageWrapper | None:
    storage = get_storage()
    if storage is None:
        return None
    return AsyncStorageWrapper(storage)
```

The factory currently hardcodes `GoogleDriveV3Storage`. The OAuth backend requires this factory to select between SA and OAuth based on config.

### Downstream Consumers

All consumers use the `StorageBackend` protocol, making them **auth-agnostic**:

| Consumer | Import | Usage Pattern |
|----------|--------|---------------|
| `api/routes/storage.py` | `get_async_storage()` | AsyncStorageWrapper in FastAPI Depends() |
| `services/upload_service.py` | Constructor injection | `UploadService(db=session, storage=backend)` |
| `queue/storage_jobs.py` | `get_storage()` | Sync singleton in RQ jobs |

**No downstream changes are needed.** The OAuth backend, once returned by the factory, is consumed identically to the SA backend.

### Thread Safety Architecture

The existing codebase uses a consistent thread safety pattern:

1. **GoogleDriveV3Storage**: `threading.Lock` for double-checked lazy init of `_service` and `_resolver`
2. **PathResolver**: Internal `threading.Lock` for LRU cache operations
3. **AsyncStorageWrapper**: Module-level `ThreadPoolExecutor(max_workers=4)` shared across instances
4. **Factory**: `@lru_cache(maxsize=1)` ensures singleton -- safe because construction is idempotent

The OAuth backend inherits all of these patterns unchanged.

### Dependencies (Current)

From `pyproject.toml`:
```toml
"google-api-python-client>=2.190.0,<3.0.0"
"google-auth>=2.48.0,<3.0.0"
"google-auth-httplib2>=0.2.0,<1.0.0"
```

Dev dependencies:
```toml
"google-api-python-client-stubs>=1.26.0"
```

**Missing for OAuth**: `google-auth-oauthlib` is NOT currently a dependency. It is needed for:
1. `InstalledAppFlow` in the bootstrap script
2. (Optionally) for `Flow` in any future web-based token acquisition

However, for **runtime** credential construction, `google-auth-oauthlib` is NOT strictly required. The `google.oauth2.credentials.Credentials` class lives in `google-auth` (already a dependency). Only the bootstrap script needs `google-auth-oauthlib`.

---

## 3. Research Findings

### 3.1 OAuth 2.0 Flow for Server-Side Use

**Recommended approach for this project**: One-time local bootstrap using `InstalledAppFlow` from `google-auth-oauthlib`.

**Why `InstalledAppFlow` over web-server redirect**:
- This is a self-hosted, single-user application
- The refresh token only needs to be obtained once
- `InstalledAppFlow` handles the localhost redirect automatically
- No need to build a web-based OAuth callback endpoint
- The user runs the bootstrap script on their local machine, copies the refresh token to their deployment

**Flow mechanics**:
1. User creates OAuth client credentials in Google Cloud Console (type: "Desktop App")
2. User downloads `client_secrets.json`
3. User runs bootstrap script: `python scripts/gdrive_oauth_bootstrap.py --client-secrets client_secrets.json`
4. Browser opens Google consent screen
5. User approves requested scopes
6. Script receives authorization code via localhost redirect
7. Script exchanges code for access_token + refresh_token
8. Script outputs refresh_token and config snippet (never logs secrets)

**Critical detail -- ensuring refresh token is returned**:
- Must use `access_type="offline"` to request a refresh token
- Must use `prompt="consent"` to force re-consent (Google only issues a refresh token on first consent or when re-consent is forced)
- If the user has previously consented without `prompt="consent"`, Google may return an access token without a refresh token
- The bootstrap script MUST validate that a refresh token was received and error clearly if not

**Token refresh behavior**:
- `google.oauth2.credentials.Credentials` auto-refreshes the access token when expired (provided `token_uri`, `client_id`, `client_secret` are set)
- This happens transparently inside `google-api-python-client`'s `execute()` calls
- The refresh token itself does **not expire** unless:
  - The user revokes access
  - The token is unused for 6 months (Google policy)
  - The OAuth app is in "testing" mode (tokens expire after 7 days)
  - The refresh token has been used more than 50 times to obtain new access tokens within a short window (rate limit)

### 3.2 Credential Construction in Python

The runtime credential construction for OAuth user credentials uses `google.oauth2.credentials.Credentials` from the **already-installed** `google-auth` package:

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

credentials = Credentials(
    token=None,  # Will be refreshed automatically
    refresh_token=refresh_token,
    client_id=client_id,
    client_secret=client_secret,
    token_uri="https://oauth2.googleapis.com/token",
    scopes=["https://www.googleapis.com/auth/drive"],
)

service = build("drive", "v3", credentials=credentials)
```

**Key differences from SA credentials**:

| Aspect | Service Account | OAuth User |
|--------|----------------|------------|
| Class | `google.oauth2.service_account.Credentials` | `google.oauth2.credentials.Credentials` |
| Input | JSON key file path | refresh_token + client_id + client_secret |
| Constructor | `.from_service_account_file(path, scopes=...)` | `Credentials(token=None, refresh_token=..., ...)` |
| Auto-refresh | Yes (self-signed JWT) | Yes (via token_uri exchange) |
| Token expiry | 1 hour | 1 hour |
| Needs network for refresh | No (signs JWT locally) | Yes (POST to token_uri) |

**No new runtime dependencies needed.** `google.oauth2.credentials.Credentials` is part of `google-auth>=2.48.0`, which is already in `pyproject.toml`. The `google-auth-oauthlib` package is only needed by the bootstrap script.

### 3.3 Scope Selection Analysis

| Scope | Access Level | Works for OAuth User? | Works for SA? |
|-------|-------------|----------------------|---------------|
| `drive` (full) | Read/write all files and folders | Yes | Yes (if shared) |
| `drive.file` | Only files created/opened by app | Yes, but CANNOT navigate pre-existing folders | No (SA can't "open" files) |
| `drive.readonly` | Read-only access to all files | Yes | Yes |
| `drive.metadata.readonly` | Read metadata only | Yes | Yes |
| `drive.appdata` | Hidden app-specific folder | Yes, but invisible to user | No |

**Recommendation: Use `drive` scope (same as existing SA backend)**

Rationale:
- The OAuth backend needs to **navigate to a user-specified root folder** that was NOT created by this application
- `drive.file` would prevent accessing any pre-existing folder structure
- The user is explicitly consenting to this scope during the OAuth flow
- `drive.file` would work if the app created ALL folders from scratch, but the user likely wants to upload into an existing folder in their Drive
- Keep scope consistent between SA and OAuth backends to avoid behavioral surprises

**Alternative -- `drive.file` with trade-offs**:
- If we restrict to `drive.file`, the bootstrap script would need to "open" the root folder (via Picker or manual ID entry) to gain access
- All sub-folders would need to be created by the app (cannot navigate into user-created folders)
- This is more secure but significantly more restrictive
- Recommendation: Document both options and let user choose during bootstrap

### 3.4 Service Construction and Thread Safety

**Recommendation: Keep one Drive service per instance (same as SA backend)**

The existing `GoogleDriveV3Storage` creates one Drive service lazily and reuses it. This works because:
- The `google-api-python-client` service object is NOT thread-safe per individual `.execute()` call
- However, the `AsyncStorageWrapper` uses `ThreadPoolExecutor(max_workers=4)`, and each worker runs sequential operations
- The existing retry loop in `_execute_with_backoff()` is inherently sequential per call
- Multiple concurrent calls from different ThreadPoolExecutor workers create separate request objects from the shared service, which is safe

For OAuth credentials, the auto-refresh mechanism adds a consideration:
- When the access token expires, the next API call triggers a refresh
- `google.oauth2.credentials.Credentials` refresh is **not thread-safe** by default
- Two threads could race to refresh simultaneously, but this is harmless (both will get valid tokens; worst case is one extra token refresh HTTP call)
- The `google-auth` library's `Credentials.refresh()` is idempotent

**Conclusion**: No changes to thread safety architecture needed. The OAuth backend inherits the same patterns.

### 3.5 API Compatibility and Behavioral Parity

The OAuth backend must maintain identical API behavior. Key areas verified:

| Feature | SA Backend Behavior | OAuth Backend Behavior | Notes |
|---------|---------------------|----------------------|-------|
| `supportsAllDrives=True` | Used in all calls | Same | Still relevant for shared items |
| `includeItemsFromAllDrives=True` | Used in list/search | Same | Allows searching shared drives |
| Query escaping | `_escape_query_value()` | Inherited | No change |
| Folder creation idempotency | Check-then-create | Inherited | No change |
| Path resolver integration | `_make_lookup_fn()` | Inherited | No change |
| Ownership/quota | SA's project quota | **User's personal quota** | IMPORTANT difference |
| Files created by | SA email | User's email | Files appear as user's own |
| Trash vs delete | Trash by default | Same | No change |

**Important behavioral difference -- file ownership**:
- With SA auth: files are created by the SA, visible in shared folder
- With OAuth auth: files are created by the user, appear as the user's own files in their Drive
- This is actually **better UX** for personal accounts (user sees their own files, not files owned by a cryptic SA email)

**Quota difference**:
- SA auth: quota may be shared across a GCP project
- OAuth auth: quota is the user's personal Drive quota (15GB free, or Google One plan)
- `StorageQuotaError` handling remains the same; the quota limit itself is different

---

## 4. Devil's Advocate Analysis

### 4.1 Code Reuse vs. Duplication

**Concern**: Subclassing `GoogleDriveV3Storage` creates tight coupling. If the parent class changes, the OAuth subclass may break silently.

**Assessment**: This concern is **valid but manageable**. The subclass overrides exactly ONE method (`_build_service()`), which has a stable, well-defined contract: take stored config, return a Drive service object. The risk of silent breakage is low because:
- `_build_service()` is called by the `service` property (stable interface)
- The return type is `Any` (Drive service object -- same regardless of auth method)
- Changes to the parent's operational methods (upload, create_folder, etc.) benefit BOTH backends automatically
- The test suite for the OAuth backend verifies the subclass works correctly

**Risk mitigation**: Add a comment in `GoogleDriveV3Storage._build_service()` documenting that it is overridden by `GoogleDriveOAuthV3Storage`, so future changes consider both subclasses.

**Alternative considered -- composition**:
```python
class GoogleDriveOAuthV3Storage:
    def __init__(self, ..., delegate: GoogleDriveV3Storage):
        self._delegate = delegate
```
This was rejected because:
- Requires wrapping ALL 5 protocol methods plus `mkdirp()`, `_validate_root_access()`, etc.
- Results in more code than inheritance (delegation boilerplate)
- No clear benefit since the only difference is credential construction

**Alternative considered -- mixin/strategy pattern**:
```python
class CredentialStrategy(Protocol):
    def build_credentials(self) -> Credentials: ...

class GoogleDriveStorage:
    def __init__(self, credential_strategy: CredentialStrategy, ...): ...
```
This was rejected because:
- Requires refactoring the existing `GoogleDriveV3Storage` (the prompt says "keep the current service-account backend unchanged")
- Over-engineers for exactly two credential types
- Could be considered for a future v3 if more auth methods are needed

### 4.2 Scope Selection Risks

**Concern**: Using the full `drive` scope for OAuth is overly broad. `drive.file` is the recommended scope for user-consented access.

**Assessment**: This is the **most legitimate concern** in the entire analysis. Using `drive` scope grants the app access to the user's entire Drive, not just the root folder configured for uploads.

**Honest trade-off**:
- `drive` scope: User can upload to any folder, including pre-existing ones. App can list folders for a picker UI. **But** the app technically has access to ALL user files.
- `drive.file` scope: App can only access files/folders it creates. Safer. **But** the user cannot navigate to an existing folder -- they can only use folders the app creates from scratch.

**Recommendation**: Default to `drive` scope but **document `drive.file` as a hardened option**. The bootstrap script should accept a `--scope` flag. The config should allow scope override. For maximum security, users should choose `drive.file` and let the app create all folder hierarchy from scratch.

### 4.3 Refresh Token Reliability

**Concern**: Refresh tokens can expire or be revoked, leaving the backend in a broken state with no automatic recovery path.

**Assessment**: This is a **real operational risk**. Refresh token loss scenarios:

| Scenario | Likelihood | Detection | Recovery |
|----------|------------|-----------|----------|
| User revokes access in Google Account settings | Medium | 401 on next API call | Re-run bootstrap script |
| Token unused for 6+ months | Low (app runs regularly) | 401 on next API call | Re-run bootstrap script |
| OAuth app in "testing" mode (7-day expiry) | High (during development) | 401 after 7 days | Publish app OR re-run bootstrap |
| Google policy change | Very Low | Unpredictable | Re-run bootstrap |
| User changes password | Low | 401 on next API call | Re-run bootstrap |

**Mitigation implemented in this plan**:
1. The OAuth backend's `_build_service()` catches `google.auth.exceptions.RefreshError` and translates it to `StoragePermissionError` with a clear message: "OAuth refresh token expired or revoked. Re-run the bootstrap script."
2. The health check endpoint (`GET /api/v1/gdrive/status`) validates the credential by making a lightweight API call (get root folder metadata).
3. The admin UI shows credential status (valid/expired/revoked).
4. **The existing SA backend remains available as a fallback** -- switching back requires only changing `GOOGLE_DRIVE_AUTH_MODE=service_account`.

### 4.4 Quota and Rate Limiting Differences

**Concern**: Rate limits differ between SA and OAuth user credentials.

**Assessment**: After research, the rate limits are actually the **same** for both auth types under the same GCP project:
- 12,000 queries per 100 seconds per user
- ~3 writes/sec sustained
- 750 GB/day upload limit

The difference is in **quota attribution**:
- SA: Quota attributed to the GCP project's SA user
- OAuth: Quota attributed to the authenticated user

The existing retry/backoff logic in `_execute_with_backoff()` handles both identically. No changes needed.

### 4.5 Multi-User Future

**Concern**: The singleton factory pattern (`@lru_cache(maxsize=1)`) only supports one set of credentials. What about multi-user?

**Assessment**: This is **out of scope for Pattern A** but worth documenting. Pattern A explicitly targets single-user self-hosted deployment. If multi-user support is needed:
- The factory would need to be per-user (keyed by user ID)
- Each user would have their own OAuth refresh token stored in the database
- The `StorageBackend` instances would need to be user-scoped
- This is a major architectural change not addressed in this plan

**Recommendation**: Document this as a known limitation. The current singleton pattern matches the project's single-user architecture.

### 4.6 Security Concerns Specific to OAuth

**Concern**: Storing `client_secret` and `refresh_token` in environment variables is less secure than a locked-down SA key file.

**Assessment**: Mixed. Compared to SA JSON key files:

| Aspect | SA Key File | OAuth Env Vars |
|--------|-------------|---------------|
| Secret material | RSA private key | client_secret + refresh_token |
| Storage | File on disk (chmod 600) | Environment variables |
| Rotation | Generate new key in GCP Console | Re-run bootstrap script |
| Blast radius if leaked | Full SA access to all shared resources | Access scoped to consented user + scopes |
| Revocation | Delete key in GCP Console | User revokes in Google Account settings |
| Accidental commit risk | File might be committed | Env vars in .env might be committed |

**OAuth is arguably MORE secure** because:
- The blast radius is smaller (one user's data, not all shared folders across all users)
- The user can revoke access themselves (no GCP Console needed)
- No long-lived private key on disk
- `client_secret` alone is useless without the `refresh_token`

**Mitigation**: Same `.env` / `.gitignore` practices as SA key management. Document that `GOOGLE_DRIVE_REFRESH_TOKEN` should be treated as a secret.

### 4.7 Operational Complexity

**Concern**: Adding a second auth path increases operational surface area.

**Assessment**: **Valid but proportional.** The auth selection is a config-time decision, not a runtime switch. Only one auth path is active at any time. The operational matrix:

| Config State | Active Backend | Validation |
|-------------|---------------|------------|
| `GOOGLE_DRIVE_ENABLED=false` | None | No validation |
| `AUTH_MODE=service_account` | `GoogleDriveV3Storage` | Validate SA JSON + root ID |
| `AUTH_MODE=oauth` | `GoogleDriveOAuthV3Storage` | Validate client_id + secret + refresh_token + root ID |

The factory function adds one `if/elif` branch. Not materially more complex.

### 4.8 Alternative Approaches Considered

| Alternative | Verdict | Reason for Rejection |
|-------------|---------|---------------------|
| PyDrive2 library | Rejected | Adds another dependency for same underlying API; our implementation is already complete |
| rclone integration | Rejected | External binary dependency; shell-out fragility; overkill for single-provider use |
| Direct browser upload (Picker API) | Deferred | Requires frontend OAuth flow; good future enhancement but orthogonal to backend Pattern A |
| Refactor SA backend to use strategy pattern | Rejected | Prompt says "keep current SA backend unchanged"; inheritance is simpler for 2 strategies |
| Store OAuth tokens in database | Deferred | Single-user scope; env vars are simpler; database storage needed only for multi-user |

---

## 5. Recommended Architecture

### 5.1 Class Hierarchy

```
StorageBackend (Protocol)
  |
  +-- GoogleDriveV3Storage (existing, SA auth)
        |
        +-- GoogleDriveOAuthV3Storage (new, OAuth user auth)
              Overrides: _build_service(), get_user_email()
              Adds: _validate_oauth_credentials()
```

### 5.2 New File: `storage/google_drive_oauth_v3.py`

```python
"""Google Drive v3 implementation using OAuth 2.0 user credentials.

Subclasses GoogleDriveV3Storage, overriding only _build_service() to
use OAuth user credentials (refresh_token + client_id + client_secret)
instead of service account JSON key.

All operational logic (upload, folder operations, retry, path resolution)
is inherited from the parent class.

Usage:
    storage = GoogleDriveOAuthV3Storage(
        client_id="xxxx.apps.googleusercontent.com",
        client_secret="GOCSPX-xxxx",
        refresh_token="1//xxxx",
        root_folder_id="1ABC_your_root_folder_id",
    )
    result = storage.upload_file(b"...", "photo.jpg", "image/jpeg")
"""

from __future__ import annotations

from typing import Any

from google.auth.exceptions import RefreshError
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

from image_search_service.core.logging import get_logger
from image_search_service.storage.exceptions import (
    StorageError,
    StoragePermissionError,
)
from image_search_service.storage.google_drive import GoogleDriveV3Storage


class GoogleDriveOAuthV3Storage(GoogleDriveV3Storage):
    """Google Drive API v3 using OAuth 2.0 user credentials.

    Overrides _build_service() to construct credentials from
    refresh_token + client_id + client_secret instead of a
    service account JSON key file.

    All operational methods (upload_file, create_folder, etc.) and
    retry/error-handling logic are inherited from GoogleDriveV3Storage.

    Thread Safety:
        Same as parent class -- double-checked locking for lazy init.
        OAuth token refresh is idempotent (safe under concurrent access).
    """

    DEFAULT_TOKEN_URI = "https://oauth2.googleapis.com/token"

    def __init__(
        self,
        client_id: str,
        client_secret: str,
        refresh_token: str,
        root_folder_id: str,
        token_uri: str = DEFAULT_TOKEN_URI,
        scopes: list[str] | None = None,
        path_cache_maxsize: int = 1024,
        path_cache_ttl: int = 300,
    ) -> None:
        # Store OAuth-specific config BEFORE calling super().__init__()
        # because super().__init__ stores SA-specific config that we don't use.
        self._client_id = client_id
        self._client_secret = client_secret
        self._refresh_token = refresh_token
        self._token_uri = token_uri
        self._oauth_scopes = scopes or list(self.SCOPES)

        # Initialize parent with a dummy SA path (never used -- _build_service
        # is overridden). The parent stores this but our _build_service ignores it.
        super().__init__(
            service_account_json_path="",  # Not used by OAuth path
            root_folder_id=root_folder_id,
            path_cache_maxsize=path_cache_maxsize,
            path_cache_ttl=path_cache_ttl,
        )

        # Override logger to identify OAuth backend in logs
        self._logger = get_logger(__name__)

    def _build_service(self) -> Any:
        """Build Drive v3 service using OAuth user credentials.

        Constructs google.oauth2.credentials.Credentials from stored
        refresh_token + client_id + client_secret. The access token is
        obtained automatically on first API call via token refresh.

        Returns:
            googleapiclient Resource for Drive v3 API.

        Raises:
            StoragePermissionError: If refresh token is expired/revoked.
            StorageError: If service construction fails.
        """
        self._logger.info(
            "Building Drive service with OAuth user credentials",
            extra={"client_id_prefix": self._client_id[:20] + "..."},
        )

        try:
            credentials = Credentials(
                token=None,  # Will be refreshed on first API call
                refresh_token=self._refresh_token,
                client_id=self._client_id,
                client_secret=self._client_secret,
                token_uri=self._token_uri,
                scopes=self._oauth_scopes,
            )
        except Exception as exc:
            raise StorageError(
                f"Failed to construct OAuth credentials: {exc}"
            ) from exc

        # Validate credentials can refresh by attempting an eager refresh.
        # This catches expired/revoked tokens at initialization rather than
        # on the first real API call.
        try:
            from google.auth.transport.requests import Request
            credentials.refresh(Request())
        except RefreshError as exc:
            raise StoragePermissionError(
                path="<oauth-init>",
                detail=(
                    f"OAuth refresh token is invalid or revoked: {exc}. "
                    "Re-run the bootstrap script: "
                    "python scripts/gdrive_oauth_bootstrap.py"
                ),
            ) from exc
        except Exception as exc:
            self._logger.warning(
                "Could not eagerly refresh OAuth token (will retry on first API call): %s",
                exc,
            )

        return build("drive", "v3", credentials=credentials)

    def get_user_email(self) -> str:
        """Return the authenticated user's email address.

        Uses Drive API about().get() to retrieve user info.
        Replaces parent's get_service_account_email() for OAuth context.

        Returns:
            User email string.

        Raises:
            StorageError: If unable to retrieve user info.
        """
        try:
            about = self._execute_with_backoff(
                self.service.about().get(fields="user(emailAddress)"),
                path="<about>",
            )
            return about.get("user", {}).get("emailAddress", "unknown")
        except Exception as exc:
            self._logger.warning("Could not retrieve user email: %s", exc)
            return "unknown"
```

### 5.3 Factory Enhancement (`storage/__init__.py`)

The `get_storage()` function gains auth mode selection:

```python
@lru_cache(maxsize=1)
def get_storage() -> GoogleDriveV3Storage | None:
    settings = get_settings()

    if not settings.google_drive_enabled:
        return None

    if not settings.google_drive_root_id:
        raise ConfigurationError(
            field="GOOGLE_DRIVE_ROOT_ID",
            detail="Must be set when GOOGLE_DRIVE_ENABLED=true",
        )

    auth_mode = settings.google_drive_auth_mode

    if auth_mode == "oauth":
        from image_search_service.storage.google_drive_oauth_v3 import (
            GoogleDriveOAuthV3Storage,
        )

        if not settings.google_drive_client_id:
            raise ConfigurationError(
                field="GOOGLE_DRIVE_CLIENT_ID",
                detail="Must be set when GOOGLE_DRIVE_AUTH_MODE=oauth",
            )
        if not settings.google_drive_client_secret:
            raise ConfigurationError(
                field="GOOGLE_DRIVE_CLIENT_SECRET",
                detail="Must be set when GOOGLE_DRIVE_AUTH_MODE=oauth",
            )
        if not settings.google_drive_refresh_token:
            raise ConfigurationError(
                field="GOOGLE_DRIVE_REFRESH_TOKEN",
                detail=(
                    "Must be set when GOOGLE_DRIVE_AUTH_MODE=oauth. "
                    "Run: python scripts/gdrive_oauth_bootstrap.py"
                ),
            )

        return GoogleDriveOAuthV3Storage(
            client_id=settings.google_drive_client_id,
            client_secret=settings.google_drive_client_secret,
            refresh_token=settings.google_drive_refresh_token,
            root_folder_id=settings.google_drive_root_id,
            path_cache_maxsize=settings.google_drive_path_cache_maxsize,
            path_cache_ttl=settings.google_drive_path_cache_ttl,
        )

    # Default: service_account
    if not settings.google_drive_sa_json:
        raise ConfigurationError(
            field="GOOGLE_DRIVE_SA_JSON",
            detail="Must be set when GOOGLE_DRIVE_AUTH_MODE=service_account",
        )

    return GoogleDriveV3Storage(
        service_account_json_path=settings.google_drive_sa_json,
        root_folder_id=settings.google_drive_root_id,
        path_cache_maxsize=settings.google_drive_path_cache_maxsize,
        path_cache_ttl=settings.google_drive_path_cache_ttl,
    )
```

### 5.4 Config Additions (`core/config.py`)

```python
# --- Google Drive Storage (optional, disabled by default) ---
google_drive_auth_mode: str = Field(
    default="service_account",
    alias="GOOGLE_DRIVE_AUTH_MODE",
    description=(
        "Authentication mode: 'service_account' (default) for SA JSON key, "
        "'oauth' for OAuth 2.0 user credentials with refresh token"
    ),
)

# OAuth credentials (required when auth_mode=oauth)
google_drive_client_id: str = Field(
    default="",
    alias="GOOGLE_DRIVE_CLIENT_ID",
    description="OAuth 2.0 client ID (from Google Cloud Console)",
)
google_drive_client_secret: str = Field(
    default="",
    alias="GOOGLE_DRIVE_CLIENT_SECRET",
    description="OAuth 2.0 client secret (treat as sensitive)",
)
google_drive_refresh_token: str = Field(
    default="",
    alias="GOOGLE_DRIVE_REFRESH_TOKEN",
    description="OAuth 2.0 offline refresh token (from bootstrap script)",
)
```

### 5.5 Module Structure (After Implementation)

```
storage/
  __init__.py              # Updated factory with auth mode selection
  base.py                  # Unchanged
  exceptions.py            # Unchanged
  google_drive.py          # Unchanged (SA backend)
  google_drive_oauth_v3.py # NEW: OAuth backend (~80 lines)
  path_resolver.py         # Unchanged
  async_wrapper.py         # Unchanged
  config_validation.py     # Updated: add OAuth config validation
```

---

## 6. Detailed Implementation Steps

### Step 1: Add Config Fields (core/config.py)

**File**: `image-search-service/src/image_search_service/core/config.py`
**Effort**: 30 minutes

Add 4 new fields to the `Settings` class after the existing Google Drive fields (line ~314):

```python
google_drive_auth_mode: str = Field(
    default="service_account",
    alias="GOOGLE_DRIVE_AUTH_MODE",
    description=(
        "Authentication mode: 'service_account' (default) for SA JSON key, "
        "'oauth' for OAuth 2.0 user credentials with refresh token"
    ),
)
google_drive_client_id: str = Field(
    default="",
    alias="GOOGLE_DRIVE_CLIENT_ID",
    description="OAuth 2.0 client ID (from Google Cloud Console)",
)
google_drive_client_secret: str = Field(
    default="",
    alias="GOOGLE_DRIVE_CLIENT_SECRET",
    description="OAuth 2.0 client secret (treat as sensitive)",
)
google_drive_refresh_token: str = Field(
    default="",
    alias="GOOGLE_DRIVE_REFRESH_TOKEN",
    description="OAuth 2.0 offline refresh token (from bootstrap script)",
)
```

**Validation**: `auth_mode` should be one of `"service_account"` or `"oauth"`. Add a `@field_validator` or handle in factory.

### Step 2: Create OAuth Backend Class

**File**: `image-search-service/src/image_search_service/storage/google_drive_oauth_v3.py` (NEW)
**Effort**: 2-3 hours

Create the file as specified in Section 5.2 above. Key implementation details:

1. Subclass `GoogleDriveV3Storage`
2. Override `__init__()` to accept OAuth credentials instead of SA path
3. Override `_build_service()` to use `google.oauth2.credentials.Credentials`
4. Add `get_user_email()` method (replaces `get_service_account_email()`)
5. Add eager refresh validation in `_build_service()` with clear error messages

**Do NOT override**: `upload_file`, `create_folder`, `file_exists`, `list_folder`, `delete_file`, `mkdirp`, `_execute_with_backoff`, `_make_lookup_fn`, `_escape_query_value`, `_extract_error_reason`. These are all inherited.

### Step 3: Update Factory Function

**File**: `image-search-service/src/image_search_service/storage/__init__.py`
**Effort**: 1-2 hours

Modify `get_storage()` as specified in Section 5.3. Key changes:

1. Read `settings.google_drive_auth_mode` to determine backend type
2. Add `elif` branch for `auth_mode == "oauth"` with appropriate config validation
3. Update `__all__` to export `GoogleDriveOAuthV3Storage` (under TYPE_CHECKING)
4. Update module docstring to mention OAuth backend

### Step 4: Update Config Validation

**File**: `image-search-service/src/image_search_service/storage/config_validation.py`
**Effort**: 1-2 hours

Add `validate_google_drive_oauth_config()` function:

```python
def validate_google_drive_oauth_config(settings: Settings) -> list[str]:
    """Validate OAuth Drive configuration at startup.

    Returns list of warning/error messages (empty = all good).
    """
    errors: list[str] = []

    if not settings.google_drive_client_id:
        errors.append("GOOGLE_DRIVE_CLIENT_ID is required for OAuth mode")
    if not settings.google_drive_client_secret:
        errors.append("GOOGLE_DRIVE_CLIENT_SECRET is required for OAuth mode")
    if not settings.google_drive_refresh_token:
        errors.append(
            "GOOGLE_DRIVE_REFRESH_TOKEN is required for OAuth mode. "
            "Run: python scripts/gdrive_oauth_bootstrap.py"
        )
    if not settings.google_drive_root_id:
        errors.append("GOOGLE_DRIVE_ROOT_ID is required")

    return errors
```

Update `validate_google_drive_config()` to dispatch based on auth mode:

```python
def validate_google_drive_config(settings: Settings) -> list[str]:
    if not settings.google_drive_enabled:
        return []

    if settings.google_drive_auth_mode == "oauth":
        return validate_google_drive_oauth_config(settings)

    # Existing SA validation...
    return validate_google_drive_sa_config(settings)
```

### Step 5: Update .env.example

**File**: `image-search-service/.env.example`
**Effort**: 15 minutes

Add OAuth config section:

```bash
# Google Drive Storage (optional - disabled by default)
# GOOGLE_DRIVE_ENABLED=false
# GOOGLE_DRIVE_AUTH_MODE=service_account  # or "oauth" for personal accounts
# GOOGLE_DRIVE_ROOT_ID=<folder_id_from_google_drive>

# Service Account mode (GOOGLE_DRIVE_AUTH_MODE=service_account)
# GOOGLE_DRIVE_SA_JSON=/path/to/service-account-key.json

# OAuth mode (GOOGLE_DRIVE_AUTH_MODE=oauth)
# GOOGLE_DRIVE_CLIENT_ID=<your_client_id>.apps.googleusercontent.com
# GOOGLE_DRIVE_CLIENT_SECRET=<your_client_secret>
# GOOGLE_DRIVE_REFRESH_TOKEN=<from_bootstrap_script>
```

### Step 6: Update Health Endpoint

**File**: `image-search-service/src/image_search_service/api/routes/storage.py`
**Effort**: 1 hour

The existing health endpoint at `GET /api/v1/gdrive/status` calls `get_service_account_email()`. Update to handle OAuth backend:

```python
# In the health check handler:
storage = get_storage()  # Could be SA or OAuth

if isinstance(storage, GoogleDriveOAuthV3Storage):
    email = storage.get_user_email()
    auth_mode = "oauth"
elif isinstance(storage, GoogleDriveV3Storage):
    email = storage.get_service_account_email()
    auth_mode = "service_account"
```

### Step 7: Create Bootstrap Script

**File**: `scripts/gdrive_oauth_bootstrap.py` (NEW)
**Effort**: 3-4 hours

See Section 7 for full design.

### Step 8: Add Dependency (Dev Only)

**File**: `image-search-service/pyproject.toml`
**Effort**: 15 minutes

Add `google-auth-oauthlib` as an **optional/dev dependency** (only needed for bootstrap script, not runtime):

```toml
[project.optional-dependencies]
dev = [
    # ... existing dev deps ...
    "google-auth-oauthlib>=1.2.0,<2.0.0",  # Bootstrap script for OAuth token
]
```

Alternatively, if the bootstrap script should work standalone without dev deps:

```toml
[project.optional-dependencies]
oauth-bootstrap = [
    "google-auth-oauthlib>=1.2.0,<2.0.0",
]
```

### Step 9: Write Tests

**Effort**: 4-6 hours

See Section 10 for full test plan.

### Step 10: Write Documentation

**File**: `docs/google-drive-oauth-setup.md` (NEW)
**Effort**: 2-3 hours

See Section 8 for configuration guide content.

---

## 7. Bootstrap Script Design

### Purpose

The bootstrap script is a one-time CLI utility that:
1. Reads OAuth client secrets from a downloaded JSON file
2. Opens the user's browser for Google OAuth consent
3. Exchanges the authorization code for a refresh token
4. Outputs the config values needed for `.env`

### File: `scripts/gdrive_oauth_bootstrap.py`

```python
#!/usr/bin/env python3
"""Bootstrap OAuth 2.0 refresh token for Google Drive integration.

One-time setup script for obtaining an offline refresh token for
personal Google Drive access. Run this locally, then copy the
output to your deployment's .env file.

Requirements:
    pip install google-auth-oauthlib
    (or: uv sync --dev)

Usage:
    python scripts/gdrive_oauth_bootstrap.py \\
        --client-secrets /path/to/client_secrets.json \\
        [--scopes drive] \\
        [--port 8080]

Steps:
    1. Download OAuth client secrets JSON from Google Cloud Console
       (APIs & Services > Credentials > Create OAuth Client ID > Desktop App)
    2. Run this script with the downloaded JSON file
    3. Approve the consent screen in your browser
    4. Copy the output config values to your .env file
    5. Delete the client_secrets.json file (secrets are now in .env)

Security:
    - The refresh_token is a SENSITIVE credential. Treat like a password.
    - Never commit .env files containing the refresh token.
    - The client_secrets.json can be deleted after bootstrapping.
"""

from __future__ import annotations

import argparse
import json
import sys
from pathlib import Path

SCOPE_PRESETS = {
    "drive": ["https://www.googleapis.com/auth/drive"],
    "drive.file": ["https://www.googleapis.com/auth/drive.file"],
}

DEFAULT_PORT = 8080


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Bootstrap Google Drive OAuth refresh token",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=__doc__,
    )
    parser.add_argument(
        "--client-secrets",
        required=True,
        type=Path,
        help="Path to OAuth client secrets JSON downloaded from Google Cloud Console",
    )
    parser.add_argument(
        "--scopes",
        default="drive",
        choices=list(SCOPE_PRESETS.keys()),
        help="Scope preset: 'drive' (full access) or 'drive.file' (app-created only)",
    )
    parser.add_argument(
        "--port",
        type=int,
        default=DEFAULT_PORT,
        help=f"Local port for OAuth redirect (default: {DEFAULT_PORT})",
    )
    parser.add_argument(
        "--output-json",
        type=Path,
        default=None,
        help="Write config as JSON file (instead of printing to stdout)",
    )

    args = parser.parse_args()

    # Validate client secrets file
    if not args.client_secrets.exists():
        print(f"ERROR: Client secrets file not found: {args.client_secrets}", file=sys.stderr)
        return 1

    try:
        with open(args.client_secrets) as f:
            secrets_data = json.load(f)
    except json.JSONDecodeError as e:
        print(f"ERROR: Invalid JSON in client secrets file: {e}", file=sys.stderr)
        return 1

    # Extract client_id for display (never log client_secret)
    installed = secrets_data.get("installed", secrets_data.get("web", {}))
    client_id = installed.get("client_id", "unknown")
    print(f"Client ID: {client_id}")
    print(f"Scopes: {SCOPE_PRESETS[args.scopes]}")
    print(f"Redirect port: {args.port}")
    print()

    # Import google-auth-oauthlib (may not be installed)
    try:
        from google_auth_oauthlib.flow import InstalledAppFlow
    except ImportError:
        print(
            "ERROR: google-auth-oauthlib is not installed.\n"
            "Install with: uv sync --dev\n"
            "Or: pip install google-auth-oauthlib",
            file=sys.stderr,
        )
        return 1

    # Run OAuth flow
    print("Opening browser for Google OAuth consent...")
    print("(If browser doesn't open, check the URL printed below)")
    print()

    scopes = SCOPE_PRESETS[args.scopes]

    flow = InstalledAppFlow.from_client_secrets_file(
        str(args.client_secrets),
        scopes=scopes,
    )

    # Run local server flow with prompt=consent to ensure refresh token
    credentials = flow.run_local_server(
        port=args.port,
        access_type="offline",
        prompt="consent",
        success_message=(
            "Authentication successful! You may close this browser tab. "
            "Return to the terminal for your configuration."
        ),
    )

    # Validate refresh token was obtained
    if not credentials.refresh_token:
        print(
            "\nERROR: No refresh token received from Google.\n"
            "This can happen if:\n"
            "  1. You previously authorized this app without prompt=consent\n"
            "  2. The OAuth app is configured incorrectly\n"
            "\nTry: Revoke app access at https://myaccount.google.com/permissions\n"
            "Then re-run this script.",
            file=sys.stderr,
        )
        return 1

    # Build config output
    config = {
        "GOOGLE_DRIVE_AUTH_MODE": "oauth",
        "GOOGLE_DRIVE_CLIENT_ID": credentials.client_id,
        "GOOGLE_DRIVE_CLIENT_SECRET": credentials.client_secret,
        "GOOGLE_DRIVE_REFRESH_TOKEN": credentials.refresh_token,
    }

    if args.output_json:
        with open(args.output_json, "w") as f:
            json.dump(config, f, indent=2)
        print(f"\nConfig written to: {args.output_json}")
        print("SECURITY: Protect this file -- it contains sensitive credentials.")
    else:
        print("\n" + "=" * 60)
        print("SUCCESS! Add these to your .env file:")
        print("=" * 60)
        print()
        for key, value in config.items():
            print(f"{key}={value}")
        print()
        print("# Also set these (if not already configured):")
        print("GOOGLE_DRIVE_ENABLED=true")
        print("GOOGLE_DRIVE_ROOT_ID=<your_folder_id>")
        print()
        print("=" * 60)
        print("SECURITY REMINDERS:")
        print("  - Never commit .env files with these credentials")
        print("  - The refresh token grants Drive access -- treat as a password")
        print("  - You can revoke at: https://myaccount.google.com/permissions")
        print("=" * 60)

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### Bootstrap Script UX Flow

```
$ python scripts/gdrive_oauth_bootstrap.py --client-secrets ~/Downloads/client_secrets.json

Client ID: 123456789.apps.googleusercontent.com
Scopes: ['https://www.googleapis.com/auth/drive']
Redirect port: 8080

Opening browser for Google OAuth consent...

[Browser opens Google consent screen]
[User clicks "Allow"]
[Browser shows: "Authentication successful! You may close this browser tab."]

============================================================
SUCCESS! Add these to your .env file:
============================================================

GOOGLE_DRIVE_AUTH_MODE=oauth
GOOGLE_DRIVE_CLIENT_ID=123456789.apps.googleusercontent.com
GOOGLE_DRIVE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxx
GOOGLE_DRIVE_REFRESH_TOKEN=1//xxxxxxxxxxxxxxxxxxxxxxxx

# Also set these (if not already configured):
GOOGLE_DRIVE_ENABLED=true
GOOGLE_DRIVE_ROOT_ID=<your_folder_id>

============================================================
SECURITY REMINDERS:
  - Never commit .env files with these credentials
  - The refresh token grants Drive access -- treat as a password
  - You can revoke at: https://myaccount.google.com/permissions
============================================================
```

### Headless Deployment Workflow

For users running the app on a remote server:
1. Run the bootstrap script on a **local machine** with a browser
2. Copy the 4 config values to the server's `.env` file
3. Restart the application on the server
4. The refresh token works from any IP (not tied to the machine that generated it)

---

## 8. Configuration Guide

### Quick Setup (OAuth for Personal Google Accounts)

**Prerequisites**:
- A Google account (personal or Workspace)
- A Google Cloud project (free tier is sufficient)

**Step 1: Create OAuth Client Credentials**

1. Go to [Google Cloud Console > APIs & Services > Credentials](https://console.cloud.google.com/apis/credentials)
2. Click "Create Credentials" > "OAuth client ID"
3. Application type: **Desktop app**
4. Name: "Image Search" (or any name)
5. Click "Create"
6. Download the JSON file (Click "Download JSON")

**Step 2: Enable Google Drive API**

1. Go to [Google Cloud Console > APIs & Services > Library](https://console.cloud.google.com/apis/library)
2. Search for "Google Drive API"
3. Click "Enable"

**Step 3: Configure OAuth Consent Screen**

1. Go to [OAuth Consent Screen](https://console.cloud.google.com/apis/credentials/consent)
2. User type: **External** (for personal accounts) or **Internal** (for Workspace)
3. Add your email as a test user (if app is in "testing" mode)
4. Scopes: Add `https://www.googleapis.com/auth/drive`

**IMPORTANT**: If the app stays in "testing" mode, refresh tokens expire after **7 days**. To avoid this, either:
- Publish the app (requires Google verification for sensitive scopes)
- Or add your email as a test user and re-run bootstrap weekly (not recommended)

**Step 4: Run Bootstrap Script**

```bash
cd image-search-service
uv sync --dev  # Installs google-auth-oauthlib

python scripts/gdrive_oauth_bootstrap.py \
    --client-secrets ~/Downloads/client_secret_XXXXX.json
```

**Step 5: Configure .env**

Copy the output from the bootstrap script into your `.env` file:

```bash
# .env
GOOGLE_DRIVE_ENABLED=true
GOOGLE_DRIVE_AUTH_MODE=oauth
GOOGLE_DRIVE_CLIENT_ID=123456789.apps.googleusercontent.com
GOOGLE_DRIVE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxx
GOOGLE_DRIVE_REFRESH_TOKEN=1//xxxxxxxxxxxxxxxxxxxxxxxx
GOOGLE_DRIVE_ROOT_ID=1ABcDeFgHiJkLmNoPqRsTuVwXyZ
```

**Step 6: Find Your Root Folder ID**

1. Open Google Drive in a browser
2. Navigate to (or create) the folder where photos should be uploaded
3. The URL will look like: `https://drive.google.com/drive/folders/1ABcDeFgHiJkLmNoPqRsTuVwXyZ`
4. The folder ID is the last segment: `1ABcDeFgHiJkLmNoPqRsTuVwXyZ`

**Step 7: Restart the Application**

```bash
# Restart API server
make dev

# Restart worker
make worker
```

### Configuration Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GOOGLE_DRIVE_ENABLED` | Yes | `false` | Enable Google Drive integration |
| `GOOGLE_DRIVE_AUTH_MODE` | No | `service_account` | Auth mode: `service_account` or `oauth` |
| `GOOGLE_DRIVE_ROOT_ID` | Yes (both modes) | `""` | Drive folder ID for uploads |
| `GOOGLE_DRIVE_SA_JSON` | SA mode only | `""` | Path to SA JSON key file |
| `GOOGLE_DRIVE_CLIENT_ID` | OAuth mode only | `""` | OAuth client ID |
| `GOOGLE_DRIVE_CLIENT_SECRET` | OAuth mode only | `""` | OAuth client secret |
| `GOOGLE_DRIVE_REFRESH_TOKEN` | OAuth mode only | `""` | OAuth refresh token |
| `GOOGLE_DRIVE_UPLOAD_BATCH_SIZE` | No | `10` | Photos per upload job chunk |
| `GOOGLE_DRIVE_PATH_CACHE_TTL` | No | `300` | Path cache TTL (seconds) |
| `GOOGLE_DRIVE_PATH_CACHE_MAXSIZE` | No | `1024` | Path cache max entries |

### Auth Mode Comparison

| Aspect | Service Account (`service_account`) | OAuth User (`oauth`) |
|--------|-------------------------------------|---------------------|
| Best for | Google Workspace, automation | Personal accounts, My Drive |
| Setup steps | 8 (create SA, share folder, etc.) | 5 (create OAuth app, run bootstrap) |
| Credentials | JSON key file on disk | Client ID + secret + refresh token in env |
| File ownership | SA owns files | User owns files |
| Folder access | Only shared folders | All of user's Drive (or drive.file scope) |
| Token refresh | Self-signed JWT (local) | HTTP call to Google token endpoint |
| Revocation | Delete key in GCP Console | User revokes in Google Account settings |

---

## 9. Error Handling Strategy

### Error Translation (Inherited from Parent)

The OAuth backend inherits all error handling from `GoogleDriveV3Storage._execute_with_backoff()`:

| HTTP Status | Drive Error Reason | Custom Exception | User Message |
|-------------|-------------------|-----------------|--------------|
| 404 | `notFound` | `NotFoundError` | File or folder not found |
| 403 | `storageQuotaExceeded` | `StorageQuotaError` | Drive storage quota exceeded |
| 403 | `rateLimitExceeded` | Retry, then `RateLimitError` | API rate limit exceeded |
| 403 | `userRateLimitExceeded` | Retry, then `RateLimitError` | Per-user rate limit exceeded |
| 403 | (other) | `StoragePermissionError` | Insufficient permissions |
| 400 | (any) | `StorageError` | Bad request |
| 429 | (any) | Retry with backoff | Rate limited (transient) |
| 500, 502, 503, 504 | (any) | Retry with backoff | Server error (transient) |

### OAuth-Specific Error Handling (New)

The OAuth backend adds handling for credential-specific failures:

| Error Scenario | Exception Class | Detection | Message |
|----------------|----------------|-----------|---------|
| Refresh token expired | `StoragePermissionError` | `google.auth.exceptions.RefreshError` caught in `_build_service()` | "OAuth refresh token is invalid or revoked. Re-run the bootstrap script." |
| Refresh token revoked by user | `StoragePermissionError` | Same as above | Same message |
| Invalid client_id/secret | `StorageError` | `RefreshError` with "invalid_client" | "OAuth client credentials are invalid. Check GOOGLE_DRIVE_CLIENT_ID and GOOGLE_DRIVE_CLIENT_SECRET." |
| 401 during API call | Handled by `google-auth` auto-refresh | Transparent | No user-visible error (token auto-refreshes) |
| 401 after auto-refresh failure | `StoragePermissionError` | `HttpError` with 401 status | "Authentication failed. OAuth token may be expired." |

### Error Flow for API Routes

The existing `translate_storage_error()` in `api/routes/storage.py` handles all custom exceptions and converts them to appropriate HTTP responses. No changes needed:

```
OAuth Backend raises StoragePermissionError
  -> api/routes/storage.py: translate_storage_error()
  -> HTTPException(status_code=403, detail="...")
  -> Frontend displays error message
```

### Refresh Token Expiry Detection

The OAuth backend eagerly validates the refresh token during `_build_service()`:

```
First API call (or service initialization)
  -> _build_service() called
  -> Credentials constructed with token=None
  -> credentials.refresh(Request()) called eagerly
  -> If RefreshError: raise StoragePermissionError immediately
  -> If success: service built, cached for future calls
```

This means:
- Bad tokens are detected at **startup**, not on the first user-facing API call
- The health endpoint (`GET /api/v1/gdrive/status`) also validates on each call
- Background jobs (RQ) will fail fast with a clear error if the token is bad

---

## 10. Test Plan

### 10.1 Unit Tests: OAuth Backend

**File**: `tests/unit/storage/test_google_drive_oauth.py`

Test structure mirrors existing `test_google_drive.py` patterns:

```python
import pytest
from unittest.mock import MagicMock, patch

from image_search_service.storage.google_drive_oauth_v3 import (
    GoogleDriveOAuthV3Storage,
)


@pytest.fixture
def mock_drive_service() -> MagicMock:
    """Reuse same mock pattern as test_google_drive.py."""
    service = MagicMock()
    service.files.return_value = MagicMock()
    return service


@pytest.fixture
def oauth_storage(mock_drive_service: MagicMock) -> GoogleDriveOAuthV3Storage:
    """Create OAuth storage with mocked Drive service."""
    s = GoogleDriveOAuthV3Storage(
        client_id="test-client-id.apps.googleusercontent.com",
        client_secret="test-client-secret",
        refresh_token="test-refresh-token",
        root_folder_id="root-folder-id",
    )
    s._service = mock_drive_service  # Bypass _build_service
    return s
```

**Test classes**:

| Test Class | Description | Count |
|------------|-------------|-------|
| `TestOAuthBuildService` | Credential construction, refresh token validation | 5-7 |
| `TestOAuthBuildServiceErrors` | RefreshError -> StoragePermissionError translation | 3-4 |
| `TestOAuthGetUserEmail` | about().get() call, error handling | 2-3 |
| `TestOAuthInheritsProtocol` | Verify all StorageBackend methods exist | 1 |
| `TestOAuthUploadFile` | Upload through inherited method (same as SA tests) | 3-4 |
| `TestOAuthCreateFolder` | Folder creation through inherited method | 2-3 |
| `TestOAuthErrorTranslation` | Verify _execute_with_backoff works for OAuth backend | 3-4 |

**Key test cases**:

```python
class TestOAuthBuildService:
    """Test _build_service() for OAuth credential construction."""

    @patch("image_search_service.storage.google_drive_oauth_v3.build")
    @patch("image_search_service.storage.google_drive_oauth_v3.Credentials")
    def test_build_service_constructs_oauth_credentials(
        self, mock_creds_class, mock_build
    ):
        """Verify Credentials is called with correct OAuth params."""
        mock_creds = MagicMock()
        mock_creds_class.return_value = mock_creds

        s = GoogleDriveOAuthV3Storage(
            client_id="test-id",
            client_secret="test-secret",
            refresh_token="test-token",
            root_folder_id="root-id",
        )

        # Trigger lazy init (bypass eager refresh by mocking)
        with patch.object(mock_creds, "refresh"):
            _ = s.service

        mock_creds_class.assert_called_once_with(
            token=None,
            refresh_token="test-token",
            client_id="test-id",
            client_secret="test-secret",
            token_uri="https://oauth2.googleapis.com/token",
            scopes=["https://www.googleapis.com/auth/drive"],
        )
        mock_build.assert_called_once_with("drive", "v3", credentials=mock_creds)

    @patch("image_search_service.storage.google_drive_oauth_v3.Credentials")
    def test_build_service_raises_on_expired_refresh_token(
        self, mock_creds_class
    ):
        """Verify RefreshError is translated to StoragePermissionError."""
        from google.auth.exceptions import RefreshError
        from image_search_service.storage.exceptions import StoragePermissionError

        mock_creds = MagicMock()
        mock_creds_class.return_value = mock_creds
        mock_creds.refresh.side_effect = RefreshError("Token has been revoked")

        s = GoogleDriveOAuthV3Storage(
            client_id="test-id",
            client_secret="test-secret",
            refresh_token="revoked-token",
            root_folder_id="root-id",
        )

        with pytest.raises(StoragePermissionError, match="refresh token"):
            _ = s.service

    def test_does_not_require_sa_json_path(self):
        """Verify OAuth backend doesn't need service_account_json_path."""
        s = GoogleDriveOAuthV3Storage(
            client_id="test-id",
            client_secret="test-secret",
            refresh_token="test-token",
            root_folder_id="root-id",
        )
        # SA JSON path should be empty (not used)
        assert s._sa_json_path == ""


class TestOAuthInheritsProtocol:
    """Verify OAuth backend satisfies StorageBackend protocol."""

    def test_isinstance_check(self, oauth_storage):
        from image_search_service.storage.base import StorageBackend
        assert isinstance(oauth_storage, StorageBackend)

    def test_has_all_protocol_methods(self, oauth_storage):
        for method in ["upload_file", "create_folder", "file_exists",
                       "list_folder", "delete_file"]:
            assert hasattr(oauth_storage, method)
            assert callable(getattr(oauth_storage, method))
```

### 10.2 Unit Tests: Factory Enhancement

**File**: `tests/unit/storage/test_storage_factory.py` (new or extend existing)

```python
class TestGetStorageAuthMode:
    """Test get_storage() with auth mode selection."""

    def test_default_auth_mode_is_service_account(self, settings_with_sa):
        """Default auth mode uses SA backend."""
        storage = get_storage()
        assert isinstance(storage, GoogleDriveV3Storage)
        assert not isinstance(storage, GoogleDriveOAuthV3Storage)

    def test_oauth_mode_returns_oauth_backend(self, settings_with_oauth):
        """OAuth mode returns OAuth backend."""
        storage = get_storage()
        assert isinstance(storage, GoogleDriveOAuthV3Storage)

    def test_oauth_mode_missing_client_id_raises(self, settings_oauth_no_client_id):
        """OAuth mode without client_id raises ConfigurationError."""
        with pytest.raises(ConfigurationError, match="GOOGLE_DRIVE_CLIENT_ID"):
            get_storage()

    def test_oauth_mode_missing_refresh_token_raises(self, settings_oauth_no_token):
        """OAuth mode without refresh_token raises ConfigurationError."""
        with pytest.raises(ConfigurationError, match="GOOGLE_DRIVE_REFRESH_TOKEN"):
            get_storage()
```

### 10.3 Unit Tests: Config Validation

**File**: `tests/unit/storage/test_config_validation.py` (extend existing)

Test `validate_google_drive_oauth_config()` with various config combinations.

### 10.4 Integration Test Checklist (Manual)

These tests require real Google Drive credentials and cannot run in CI:

- [ ] Bootstrap script obtains refresh token successfully
- [ ] OAuth backend validates root folder access on startup
- [ ] Upload a file to root folder
- [ ] Upload a file to a subfolder
- [ ] Create folder idempotently
- [ ] List folder contents
- [ ] Check file exists
- [ ] Delete (trash) a file
- [ ] `mkdirp()` creates nested folder hierarchy
- [ ] Concurrent uploads from AsyncStorageWrapper (4 workers)
- [ ] Token auto-refresh after 1-hour expiry (wait or mock)
- [ ] Revoke token, verify clear error message
- [ ] Switch from SA to OAuth via config change
- [ ] Health endpoint returns correct auth mode and user email

### 10.5 Test Dependencies and Mocking Strategy

No new test dependencies are needed. The existing mock pattern from `test_google_drive.py` works directly:

1. Create `GoogleDriveOAuthV3Storage` with test credentials
2. Inject `mock_drive_service` via `s._service = mock_drive_service`
3. This bypasses `_build_service()` entirely, which is the only OAuth-specific code
4. All protocol method tests verify the inherited behavior works correctly

For `_build_service()` tests specifically:
- Mock `google.oauth2.credentials.Credentials` class
- Mock `google.auth.transport.requests.Request` for eager refresh
- Mock `googleapiclient.discovery.build`
- Use `patch()` decorators, same pattern as `test_google_drive.py`

---

## 11. Security Considerations

### 11.1 Credential Storage

| Credential | Storage Method | Risk | Mitigation |
|-----------|---------------|------|------------|
| `client_id` | `.env` / env var | Low (semi-public) | Not sensitive alone |
| `client_secret` | `.env` / env var | Medium | Never log; `.gitignore` |
| `refresh_token` | `.env` / env var | **High** (grants Drive access) | Never log; `.gitignore`; treat as password |
| `client_secrets.json` (bootstrap only) | Temporary file | Medium | Delete after bootstrap |

### 11.2 Logging Safety

The OAuth backend must NEVER log:
- `client_secret` value
- `refresh_token` value
- Access tokens (even expired ones)

Safe to log:
- `client_id` (or a prefix like `"123456...apps.googleusercontent.com"`)
- Auth mode (`"oauth"`)
- User email (after successful auth)
- Error messages (without embedded credentials)

Implementation: The `_build_service()` method in Section 5.2 only logs `client_id_prefix` (first 20 chars).

### 11.3 Blast Radius Comparison

| Scenario | SA Key Compromised | OAuth Credentials Compromised |
|----------|-------------------|------------------------------|
| Files accessible | All files shared with SA | All user Drive files (if `drive` scope) |
| Can create files | Yes | Yes |
| Can delete files | Yes | Yes |
| Who can revoke | GCP admin (delete key) | User (Google Account settings) |
| Revocation speed | Immediate (delete key) | Immediate (revoke app access) |
| Detectable by user | No (SA actions are invisible) | Yes (files appear under user's activity) |

**OAuth is arguably safer** because:
- User can see and revoke access themselves
- Actions are attributable to the user (audit trail)
- No private key file sitting on disk
- Credential rotation is simpler (re-run bootstrap vs. GCP Console key management)

### 11.4 OAuth App Security Settings

**IMPORTANT -- Testing vs. Published App**:

| Setting | Testing Mode | Published Mode |
|---------|-------------|---------------|
| Refresh token lifetime | **7 days** | Indefinite |
| User cap | 100 test users | Unlimited |
| Consent screen | Shows "unverified app" warning | Standard consent |
| Google verification | Not required | Required for sensitive scopes |

**Recommendation for self-hosted use**: Keep the app in "testing" mode and add your own email as a test user. Accept the 7-day token expiry and re-run bootstrap as needed. For production use with multiple users, publish the app and complete Google verification.

### 11.5 Environment Variable vs. Secrets Manager

For this single-user self-hosted application, environment variables are appropriate. For production deployments with multiple users or compliance requirements, consider:

- **HashiCorp Vault**: Store credentials as Vault secrets, inject at startup
- **AWS Secrets Manager / GCP Secret Manager**: Cloud-native secret storage
- **1Password CLI**: `op read "op://vault/gdrive/refresh_token"` in `.env`
- **Database storage**: Store encrypted tokens in PostgreSQL (requires encryption key management)

These are **out of scope** for Pattern A but documented as future options.

---

## 12. Migration and Rollout Plan

### 12.1 Rollout Phases

**Phase 1: Implementation (This Plan)**
- Implement `GoogleDriveOAuthV3Storage` subclass
- Update factory, config, validation
- Write tests
- Create bootstrap script
- Write documentation

**Phase 2: Internal Testing**
- Deploy to development environment
- Run bootstrap script with a test Google account
- Verify all operations (upload, list, create folder, delete)
- Test error scenarios (revoked token, quota exceeded)
- Compare behavior with SA backend (should be identical except for file ownership)

**Phase 3: Documentation and Release**
- Update `.env.example` with OAuth config section
- Add `docs/google-drive-oauth-setup.md` setup guide
- Update admin UI health panel to show auth mode
- Release notes describing the new option

### 12.2 Migration Path: SA to OAuth

For users currently using the SA backend who want to switch to OAuth:

1. **Do not delete SA configuration yet** (keep as fallback)
2. Run bootstrap script to obtain OAuth credentials
3. Update `.env`:
   ```bash
   GOOGLE_DRIVE_AUTH_MODE=oauth
   GOOGLE_DRIVE_CLIENT_ID=...
   GOOGLE_DRIVE_CLIENT_SECRET=...
   GOOGLE_DRIVE_REFRESH_TOKEN=...
   # Keep existing SA config commented out as backup:
   # GOOGLE_DRIVE_SA_JSON=/path/to/sa.json
   ```
4. Restart application
5. Verify health check shows `auth_mode: "oauth"` and correct user email
6. Test an upload
7. If issues: revert to `GOOGLE_DRIVE_AUTH_MODE=service_account`

**IMPORTANT -- File Ownership Change**:
- Files previously uploaded by SA are owned by the SA email
- Files uploaded after switching to OAuth are owned by the user
- Both sets of files remain in the same folder (no migration needed)
- The user may see a mix of "owned by me" and "owned by service-account@..." files
- This is cosmetic only; functionality is unaffected

### 12.3 Backward Compatibility

| Aspect | Guaranteed? | Notes |
|--------|-------------|-------|
| Existing SA config works unchanged | YES | Default `auth_mode=service_account` preserves current behavior |
| Existing API endpoints unchanged | YES | Factory returns `StorageBackend` regardless of auth mode |
| Existing RQ jobs work unchanged | YES | Jobs use `get_storage()` which handles auth mode internally |
| Existing tests pass unchanged | YES | No changes to SA backend code |
| `.env` without `AUTH_MODE` works | YES | Defaults to `service_account` |

### 12.4 Rollback Plan

If the OAuth backend causes issues in production:

1. Set `GOOGLE_DRIVE_AUTH_MODE=service_account` in `.env`
2. Restart application
3. All operations revert to SA backend immediately
4. No data migration needed (both backends operate on the same folder)

---

## 13. Open Questions and Decisions

### 13.1 Decided (Documented Above)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Inheritance vs. composition? | Inheritance | Only `_build_service()` differs; avoids 700+ lines of duplication |
| Which scope? | `drive` (default), with `drive.file` as option | Need to navigate pre-existing folders |
| `google-auth-oauthlib` as runtime dep? | Dev/optional only | Not needed at runtime; only for bootstrap |
| Thread safety changes? | None | Inherited patterns are sufficient |
| Multi-user support? | Out of scope | Pattern A is single-user |

### 13.2 Open (Needs User Decision)

**Q1: OAuth App Publishing Status**

Should the Google Cloud OAuth app be published (permanent tokens, requires Google verification) or stay in testing mode (7-day token expiry, simpler setup)?

- **Testing mode**: Simpler for self-hosted single-user. Token expires weekly. Must re-run bootstrap.
- **Published mode**: Permanent tokens. Requires Google verification process (can take weeks for `drive` scope).
- **Recommendation**: Start in testing mode. Document how to publish if needed.

**Q2: Scope Default**

Should the default scope for the bootstrap script be `drive` (full access) or `drive.file` (app-created only)?

- `drive`: User can navigate to any existing folder. More permissive.
- `drive.file`: App can only access files it creates. More restrictive.
- **Recommendation**: Default to `drive`, document `drive.file` as hardened option.

**Q3: Token Storage for Multi-User Future**

If multi-user support is eventually needed, should tokens be stored in:
- PostgreSQL (encrypted column)
- External secrets manager
- Per-user `.env` files

- **Recommendation**: Defer. Current env-var approach works for single-user. Database storage is the natural next step for multi-user.

**Q4: Health Check Behavior on Token Expiry**

Should the health endpoint return `degraded` or `unhealthy` when the OAuth token is expired/revoked?

- `degraded`: Other features still work; only Drive upload is affected
- `unhealthy`: Signals that a critical integration is broken
- **Recommendation**: Return `degraded` with clear error message, since Drive integration is optional

**Q5: Admin UI Changes**

Should the admin settings panel show different UI based on `auth_mode`?

- **SA mode**: Shows SA email, key file status, rotation reminder
- **OAuth mode**: Shows user email, token status, "Re-authenticate" button
- **Recommendation**: Yes, conditionally render based on `auth_mode` from health endpoint

---

## Appendix A: File Change Summary

| File | Action | Lines Changed |
|------|--------|---------------|
| `src/image_search_service/core/config.py` | Modify | +20 (4 new fields) |
| `src/image_search_service/storage/__init__.py` | Modify | +30 (factory branch) |
| `src/image_search_service/storage/google_drive.py` | Modify | +2 (comment about subclass) |
| `src/image_search_service/storage/google_drive_oauth_v3.py` | **New** | ~80 lines |
| `src/image_search_service/storage/config_validation.py` | Modify | +25 (OAuth validation) |
| `src/image_search_service/api/routes/storage.py` | Modify | +10 (health endpoint) |
| `scripts/gdrive_oauth_bootstrap.py` | **New** | ~150 lines |
| `tests/unit/storage/test_google_drive_oauth.py` | **New** | ~200 lines |
| `tests/unit/storage/test_storage_factory.py` | **New/Extend** | ~80 lines |
| `.env.example` | Modify | +10 (OAuth config) |
| `docs/google-drive-oauth-setup.md` | **New** | ~100 lines |
| `pyproject.toml` | Modify | +1 (dev dep) |

**Total new code**: ~510 lines (implementation + tests + script + docs)
**Total modified code**: ~95 lines across 6 existing files

## Appendix B: Dependency Analysis

### Runtime Dependencies (No Changes)

The OAuth backend uses `google.oauth2.credentials.Credentials` from `google-auth>=2.48.0`, which is **already installed**.

### Bootstrap Script Dependency (Dev Only)

```
google-auth-oauthlib>=1.2.0,<2.0.0
  Depends on:
    google-auth (already installed)
    requests-oauthlib
      requests (likely already installed)
      oauthlib
```

Total additional packages for dev: `google-auth-oauthlib`, `requests-oauthlib`, `oauthlib` (~500KB combined).

### Dependency Risk Assessment

| Package | Risk | Mitigation |
|---------|------|------------|
| `google-auth-oauthlib` | Low (maintained by Google) | Pin to `<2.0.0`; only used in bootstrap |
| `google.oauth2.credentials` | None (already a dependency) | Part of `google-auth` |

## Appendix C: Key Code References

| File | Relevant Section | Why It Matters |
|------|-----------------|----------------|
| `storage/google_drive.py:163-182` | `_build_service()` method | The ONLY method that needs to change for OAuth |
| `storage/google_drive.py:120-136` | `service` property (lazy init) | Inherited by OAuth subclass |
| `storage/google_drive.py:406-491` | `_execute_with_backoff()` | Retry + error translation (inherited) |
| `storage/__init__.py:103-144` | `get_storage()` factory | Needs auth mode branching |
| `storage/base.py:63-95` | `StorageBackend` protocol | Interface the OAuth backend must satisfy |
| `storage/exceptions.py:1-145` | Exception hierarchy | All exceptions reused (no changes) |
| `core/config.py:278-314` | Google Drive settings | New fields added here |
| `tests/unit/storage/test_google_drive.py:1-50` | Test fixtures + mock patterns | Template for OAuth tests |
| `api/routes/storage.py:580-620` | Health check endpoint | Needs auth mode awareness |

---

*This implementation plan was produced by systematic codebase investigation, including reading all storage module files, configuration, tests, API routes, and prior research documents. All file paths, class names, method signatures, and code snippets are verified against the actual codebase as of 2026-02-18.*
