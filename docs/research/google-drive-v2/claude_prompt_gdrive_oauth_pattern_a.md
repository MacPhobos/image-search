# Claude Code Prompt — Add Google Drive OAuth “Pattern A” Storage Backend (Single-User My Drive)

You are working in a Python monorepo that already contains `GoogleDriveV3Storage`, which authenticates using **service account** credentials (Workspace-friendly). We need to add a *second* backend implementation that works for **personal Google accounts / My Drive** by using **OAuth 2.0 user credentials with offline refresh tokens** (“Pattern A”).

## Goals

- Add a new storage backend implementation (parallel to `GoogleDriveV3Storage`) that:
  - Authenticates via OAuth user credentials (client_id/client_secret + refresh_token)
  - Confines all operations to a configured `root_folder_id`
  - Keeps API behavior consistent with `StorageBackend` protocol and existing exceptions
- Keep the current service-account backend unchanged.
- Provide safe defaults and clear operational steps for bootstrapping refresh tokens.

## Research Topics / Questions to Answer

1. **OAuth 2.0 Flow for Server Use**
   - What is the recommended way for a backend service to obtain a **refresh token** (offline access)?
   - For a self-hosted internal service, is the best approach:
     - a one-time local bootstrap script using `InstalledAppFlow`, OR
     - a web-server redirect flow?
   - How to ensure refresh tokens are returned (e.g., `access_type=offline`, `prompt=consent`) and how to handle cases where Google doesn’t re-issue refresh tokens.

2. **Credential Construction in Python**
   - How to build `google.oauth2.credentials.Credentials` from stored secrets (refresh token, client id/secret, token_uri, scopes).
   - Verify token auto-refresh behavior inside `google-api-python-client` calls.
   - What libraries are needed (`google-auth`, `google-auth-oauthlib`) and how to minimize new dependencies.

3. **Scope Selection**
   - Compare required scopes for current operations (upload/list/get/delete/create_folder).
   - Decide whether to keep `drive` full access or narrow to `drive.file` / other scopes.
   - Confirm how scope choice affects ability to access files not created by the app, and access to shared folders.

4. **Config & Secrets Management**
   - Define configuration inputs for the new backend:
     - client_id, client_secret, refresh_token, root_folder_id
     - optional: token_uri, scopes, application name
   - Determine where/how to store refresh tokens (env vars vs secrets manager vs DB).
   - Security considerations: rotation, least privilege, preventing accidental logging of secrets.

5. **Service Construction + Thread Safety**
   - The existing implementation lazily builds a Drive service and notes that the service object is not thread-safe.
   - Decide the safest pattern for the OAuth backend:
     - build a new Drive service per request, OR
     - keep one per instance but serialize `.execute()` / resumable chunk operations with a lock.
   - Ensure behavior is consistent under ThreadPoolExecutor concurrency.

6. **API Compatibility & Behavioral Parity**
   - Ensure the new backend matches the service-account backend semantics:
     - `supportsAllDrives` flags (still relevant for shared items)
     - `includeItemsFromAllDrives` for listing/searching
     - query escaping, folder creation idempotency, path resolver integration
   - Confirm ownership/quota behavior now attaches to the user account.

7. **Bootstrap / Onboarding Script**
   - Create a small CLI utility (checked into repo under `scripts/` or `tools/`) that:
     - reads OAuth client secrets JSON
     - runs consent flow
     - prints/stores refresh_token and related config
   - Confirm best UX for headless deployments (document how to run locally and copy token to secrets).

8. **Error Handling Mapping**
   - Validate current `HttpError` translation logic for OAuth usage:
     - 401 invalid/expired refresh token handling (force re-auth)
     - 403 permission vs rate limits
     - quota errors now mean real user quota issues
   - Ensure exceptions raised remain consistent (`StoragePermissionError`, `RateLimitError`, `StorageQuotaError`, etc.).

9. **Test Plan**
   - Add unit tests that mock Drive client calls similarly to existing tests (if any).
   - Add an integration test checklist:
     - validate root folder access
     - upload/download/list/delete
     - folder creation idempotency
     - concurrency smoke test
   - Ensure tests do not require committing secrets; use env-based injection.

## Required Deliverables

1. **New implementation file**
   - e.g. `image_search_service/storage/google_drive_oauth_v3.py`
   - Class name suggestion: `GoogleDriveOAuthV3Storage`
   - Must implement the same public methods as `GoogleDriveV3Storage` (or `StorageBackend`) used by callers.

2. **Config wiring**
   - Add a new settings/config object for OAuth Drive credentials.
   - Provide a factory/selector that picks SA backend vs OAuth backend based on config.

3. **Bootstrap script**
   - e.g. `scripts/gdrive_oauth_bootstrap.py`
   - Outputs a minimal config blob for deployment (without leaking secrets to logs).

4. **Docs**
   - A short `docs/` page explaining:
     - personal accounts require OAuth
     - how to generate refresh token
     - how to configure the service
     - security warnings (protect refresh token)

## Repo-Specific Instructions

- Search the repo for existing storage backend selection logic and `StorageBackend` protocol definitions.
- Follow existing exception translation conventions (see `INTERFACES.md` references in the current file).
- Keep code style consistent with existing module patterns (lazy initialization, logging, docstrings).
- Don’t break the existing service account backend.
- Prefer minimal, boring changes that are easy to review.

## Implementation Sketch (high-level)

- Add `GoogleDriveOAuthV3Storage._build_service()` that:
  - constructs `Credentials(refresh_token=..., client_id=..., client_secret=..., token_uri=..., scopes=...)`
  - calls `build("drive", "v3", credentials=creds)`
- Reuse the rest of the logic from `GoogleDriveV3Storage` as much as possible (upload/list/get/create_folder/delete, path resolver).
- Add a note in docs that refresh tokens may require `prompt=consent` to obtain reliably.

## Output Format

- Provide:
  1. A brief research summary (key findings)
  2. A concrete step-by-step implementation plan with file paths
  3. A checklist for testing + rollout
