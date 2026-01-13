# Admin Panel Settings System Research

**Date**: 2026-01-12
**Research Focus**: Understanding existing Admin Panel settings architecture to add new configurable settings
**Status**: Complete

---

## Executive Summary

The image-search monorepo implements a **database-backed settings system** for runtime-configurable application settings, accessible via the Admin Panel UI. The system supports typed configuration values (float, int, string, boolean) with validation constraints (min/max ranges, allowed values). Settings are stored in the `system_configs` PostgreSQL table and accessed via `ConfigService` (async API) and `SyncConfigService` (background workers).

**Key Findings**:
1. ✅ Settings use **database-backed storage** (`system_configs` table)
2. ✅ Settings API endpoints follow RESTful pattern (`/api/v1/config/{category}`)
3. ✅ Frontend Admin Panel has dedicated **Settings tab** with live updates
4. ✅ Validation constraints enforced at both backend (service layer) and frontend (Svelte form validation)
5. ✅ Database migrations seed default values with descriptions
6. ⚠️ **No "all vs. top N" pattern exists currently** (new pattern required)

---

## 1. Backend Settings System

### 1.1 Database Model (`system_configs` table)

**Location**: `image-search-service/src/image_search_service/db/models.py:798-838`

```python
class SystemConfig(Base):
    """System configuration table for database-backed settings.

    Provides a flexible key-value store with built-in type safety
    and validation constraints.
    """
    __tablename__ = "system_configs"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    key: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    value: Mapped[str] = mapped_column(String(500), nullable=False)  # Stored as string, cast on retrieval
    data_type: Mapped[str] = mapped_column(String(20), nullable=False)  # "float", "int", "string", "boolean"
    description: Mapped[str | None] = mapped_column(Text, nullable=True)

    # Validation constraints (stored as strings, parsed based on data_type)
    min_value: Mapped[str | None] = mapped_column(String(50), nullable=True)
    max_value: Mapped[str | None] = mapped_column(String(50), nullable=True)
    allowed_values: Mapped[list[str] | None] = mapped_column(JSONB, nullable=True)  # For enum-like values

    # Categorization for grouping in UI
    category: Mapped[str] = mapped_column(String(50), nullable=False, default="general")

    created_at: Mapped[datetime]
    updated_at: Mapped[datetime]
```

**Key Design Decisions**:
- All values stored as **strings** and cast to appropriate type on retrieval
- `allowed_values` is JSONB array for enum-like constraints
- `category` field groups related settings (e.g., `"face_matching"`)
- Indexes on `key` (unique) and `category` for fast queries

### 1.2 ConfigService (Service Layer)

**Location**: `image-search-service/src/image_search_service/services/config_service.py`

**Dual Interface**:
- `ConfigService` - Async for API endpoints (uses `AsyncSession`)
- `SyncConfigService` - Synchronous for background workers/RQ jobs (uses `SyncSession`)

**Key Methods**:

```python
class ConfigService:
    # Default fallback values (used if DB unavailable or key missing)
    DEFAULTS: dict[str, float | int | str | bool] = {
        "face_auto_assign_threshold": 0.85,
        "face_suggestion_threshold": 0.70,
        "face_suggestion_max_results": 50,
        "face_suggestion_expiry_days": 30,
        "face_prototype_min_quality": 0.5,
        "face_prototype_max_exemplars": 5,
        "face_suggestion_groups_per_page": 10,
        "face_suggestion_items_per_group": 20,
    }

    async def get_float(self, key: str) -> float
    async def get_int(self, key: str) -> int
    async def get_string(self, key: str) -> str
    async def get_bool(self, key: str) -> bool

    async def set_value(self, key: str, value: Any) -> SystemConfig:
        """Set value with validation against constraints."""
        # Validates: data type, min/max range, allowed_values
        # Raises ValueError on validation failure

    async def get_all_by_category(self, category: str) -> list[SystemConfig]
```

**Validation Logic** (`_validate_value` method):
1. **Type checking**: Ensures value matches `data_type` (float, int, boolean, string)
2. **Range validation**: For numeric types, enforces `min_value <= value <= max_value`
3. **Allowed values**: For enums, checks `value in allowed_values`

### 1.3 Database Migrations (Seed Data)

**Example Migration**: `a1b2c3d4e5f6_add_system_configs_table.py`

```python
def upgrade() -> None:
    # Create table
    op.create_table("system_configs", ...)

    # Seed default settings
    op.execute("""
        INSERT INTO system_configs (key, value, data_type, description, min_value, max_value, category)
        VALUES
        (
            'face_auto_assign_threshold',
            '0.85',
            'float',
            'Confidence threshold for automatic face-to-person assignment...',
            '0.5',
            '1.0',
            'face_matching'
        ),
        (
            'face_suggestion_threshold',
            '0.70',
            'float',
            'Minimum confidence threshold to create face-to-person suggestions...',
            '0.3',
            '0.95',
            'face_matching'
        )
        -- More settings...
    """)
```

**Migration Pattern**:
- Each new setting added via migration (with `ON CONFLICT (key) DO NOTHING` for idempotency)
- Descriptive text explains **purpose** and **impact** of each setting
- Min/max values prevent invalid configurations
- Category groups related settings for UI display

---

## 2. Backend API Endpoints

### 2.1 Config API Routes

**Location**: `image-search-service/src/image_search_service/api/routes/config.py`

**Endpoints**:

#### GET `/api/v1/config/face-matching`
Returns aggregated face matching settings:

```python
@router.get("/face-matching", response_model=FaceMatchingConfigResponse)
async def get_face_matching_config(db: AsyncSession = Depends(get_db)):
    service = ConfigService(db)
    return FaceMatchingConfigResponse(
        auto_assign_threshold=await service.get_float("face_auto_assign_threshold"),
        suggestion_threshold=await service.get_float("face_suggestion_threshold"),
        max_suggestions=await service.get_int("face_suggestion_max_results"),
        suggestion_expiry_days=await service.get_int("face_suggestion_expiry_days"),
        prototype_min_quality=await service.get_float("face_prototype_min_quality"),
        prototype_max_exemplars=await service.get_int("face_prototype_max_exemplars"),
    )
```

**Response Schema**:
```python
class FaceMatchingConfigResponse(BaseModel):
    auto_assign_threshold: float
    suggestion_threshold: float
    max_suggestions: int
    suggestion_expiry_days: int
    prototype_min_quality: float
    prototype_max_exemplars: int
```

#### PUT `/api/v1/config/face-matching`
Updates all face matching settings atomically:

```python
class FaceMatchingConfigUpdateRequest(BaseModel):
    auto_assign_threshold: float = Field(..., ge=0.5, le=1.0)
    suggestion_threshold: float = Field(..., ge=0.3, le=0.95)
    max_suggestions: int = Field(default=50, ge=1, le=200)
    suggestion_expiry_days: int = Field(default=30, ge=1, le=365)
    prototype_min_quality: float = Field(default=0.5, ge=0.0, le=1.0)
    prototype_max_exemplars: int = Field(default=5, ge=1, le=20)

    @model_validator(mode="after")
    def validate_thresholds(self):
        """Ensure suggestion_threshold < auto_assign_threshold."""
        if self.suggestion_threshold >= self.auto_assign_threshold:
            raise ValueError("suggestion_threshold must be less than auto_assign_threshold")
        return self
```

**Design Pattern**:
- Specialized endpoints for **grouped settings** (e.g., `/face-matching`, `/face-suggestions`)
- Generic endpoints for **category-based access** (`GET /config/{category}`, `PUT /config/{key}`)
- Pydantic validation at API layer (field constraints with `ge`, `le`)
- Additional cross-field validation via `@model_validator`

#### GET `/api/v1/config/face-suggestions`
Pagination settings with camelCase transformation:

```python
class FaceSuggestionSettingsResponse(BaseModel):
    model_config = ConfigDict(
        populate_by_name=True,
        alias_generator=to_camel,  # Converts snake_case -> camelCase
    )

    groups_per_page: int = Field(..., ge=1, le=50)
    items_per_group: int = Field(..., ge=1, le=50)
```

**Snake to Camel Conversion**:
```python
def to_camel(string: str) -> str:
    """Convert snake_case to camelCase."""
    words = string.split("_")
    return words[0] + "".join(word.capitalize() for word in words[1:])
```

### 2.2 Environment-Based Settings (Legacy Pattern)

**Location**: `image-search-service/src/image_search_service/core/config.py`

Some settings are **environment variables only** (not in database):

```python
class Settings(BaseSettings):
    # Unknown face clustering display settings (ENV-only, no DB backing)
    unknown_face_cluster_min_confidence: float = Field(
        default=0.70,
        ge=0.0,
        le=1.0,
        alias="UNKNOWN_FACE_CLUSTER_MIN_CONFIDENCE",
    )
    unknown_face_cluster_min_size: int = Field(
        default=2,
        ge=1,
        le=100,
        alias="UNKNOWN_FACE_CLUSTER_MIN_SIZE",
    )
```

**Why Environment Variables?**:
- Settings loaded at **startup** (via `pydantic-settings`)
- Used by background workers that don't have DB access during initialization
- **Requires service restart** to change (not runtime-configurable)

**Migration Path**:
```python
@router.put("/face-clustering-unknown", response_model=UnknownFaceClusteringConfigResponse)
async def update_unknown_clustering_config(request, db: AsyncSession):
    # Currently: Warning logged, settings NOT persisted
    logger.warning(
        "Unknown clustering config update requested but currently requires restart. "
        f"Requested: min_confidence={request.min_confidence}, "
        f"min_cluster_size={request.min_cluster_size}"
    )

    # TODO: Implement database-backed config storage for runtime updates
    return UnknownFaceClusteringConfigResponse(
        min_confidence=request.min_confidence,
        min_cluster_size=request.min_cluster_size,
    )
```

---

## 3. Frontend Admin Panel

### 3.1 Admin Page Structure

**Location**: `image-search-ui/src/routes/admin/+page.svelte`

```svelte
<Tabs.Root value="data" class="admin-tabs-container">
    <Tabs.List class="admin-tabs-list">
        <Tabs.Trigger value="data">Data Management</Tabs.Trigger>
        <Tabs.Trigger value="settings">Settings</Tabs.Trigger>  <!-- Settings tab -->
    </Tabs.List>

    <Tabs.Content value="data">
        <AdminDataManagement />  <!-- Import/Export, Delete All Data -->
    </Tabs.Content>

    <Tabs.Content value="settings">
        <FaceMatchingSettings />  <!-- Settings UI -->
    </Tabs.Content>
</Tabs.Root>
```

### 3.2 FaceMatchingSettings Component

**Location**: `image-search-ui/src/lib/components/admin/FaceMatchingSettings.svelte`

**State Management** (Svelte 5 runes):

```typescript
let config = $state<FaceMatchingConfig>({
    autoAssignThreshold: 0.85,
    suggestionThreshold: 0.7,
    maxSuggestions: 50,
    suggestionExpiryDays: 30,
    prototypeMinQuality: 0.5,
    prototypeMaxExemplars: 5
});

let paginationSettings = $state<FaceSuggestionSettings>({
    groupsPerPage: 10,
    itemsPerGroup: 20
});

let clusteringConfig = $state<UnknownFaceClusteringConfig>({
    minConfidence: 0.5,
    minClusterSize: 2
});

// Derived validation
let isValid = $derived(
    config.suggestionThreshold < config.autoAssignThreshold &&
    paginationSettings.groupsPerPage >= 1 &&
    paginationSettings.groupsPerPage <= 50 &&
    paginationSettings.itemsPerGroup >= 1 &&
    paginationSettings.itemsPerGroup <= 50
);
```

**Form Sections**:

1. **Threshold Visualization** (lines 182-217)
   - Visual bar showing unassigned/suggestion/auto zones
   - Range sliders for both thresholds
   - Real-time validation warning if `suggestionThreshold >= autoAssignThreshold`

2. **Additional Settings** (lines 265-297)
   - Number inputs for `maxSuggestions`, `suggestionExpiryDays`

3. **Prototype Settings** (lines 300-342)
   - Slider for `prototypeMinQuality` (0.0-1.0)
   - Number input for `prototypeMaxExemplars` (1-20)

4. **Pagination Settings** (lines 345-381)
   - Number inputs for `groupsPerPage`, `itemsPerGroup`

5. **Unknown Clustering Settings** (lines 384-430)
   - Slider for `minConfidence` (0.60-0.95)
   - Number input for `minClusterSize` (2-50)

**Save Logic** (lines 102-147):

```typescript
async function saveConfig() {
    if (!isValid) {
        validationError = 'Please check all validation requirements';
        return;
    }

    saving = true;

    try {
        // Save face matching config
        const matchingResponse = await fetch(`${API_BASE_URL}/api/v1/config/face-matching`, {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                auto_assign_threshold: config.autoAssignThreshold,
                suggestion_threshold: config.suggestionThreshold,
                max_suggestions: config.maxSuggestions,
                suggestion_expiry_days: config.suggestionExpiryDays,
                prototype_min_quality: config.prototypeMinQuality,
                prototype_max_exemplars: config.prototypeMaxExemplars
            })
        });

        if (!matchingResponse.ok) {
            const data = await matchingResponse.json();
            throw new Error(data.detail || 'Failed to save configuration');
        }

        // Save pagination settings
        await updateFaceSuggestionSettings(paginationSettings);

        // Save unknown clustering config
        await updateUnknownClusteringConfig(clusteringConfig);

        successMessage = 'Settings saved successfully';
        setTimeout(() => { successMessage = null; }, 5000);
    } catch (e) {
        error = e instanceof Error ? e.message : 'Failed to save configuration';
    } finally {
        saving = false;
    }
}
```

**UI Patterns**:
- **Range sliders** for float values (0.0-1.0) with live value display
- **Number inputs** for integer values with min/max constraints
- **Visual threshold bar** for confidence thresholds (gradient colors: red/yellow/green)
- **Success toast** (bottom-right corner, 5-second auto-dismiss)
- **Inline validation errors** (red banner below threshold sliders)

### 3.3 Frontend API Client

**Location**: `image-search-ui/src/lib/api/admin.ts`

```typescript
export interface UnknownFaceClusteringConfig {
    minConfidence: number;
    minClusterSize: number;
}

export async function getUnknownClusteringConfig(): Promise<UnknownFaceClusteringConfig> {
    return apiRequest<UnknownFaceClusteringConfig>('/api/v1/config/face-clustering-unknown');
}

export async function updateUnknownClusteringConfig(
    config: UnknownFaceClusteringConfig
): Promise<UnknownFaceClusteringConfig> {
    return apiRequest<UnknownFaceClusteringConfig>('/api/v1/config/face-clustering-unknown', {
        method: 'PUT',
        body: JSON.stringify(config)
    });
}
```

**Face API** (`image-search-ui/src/lib/api/faces.ts`):

```typescript
export interface FaceSuggestionSettings {
    groupsPerPage: number;
    itemsPerGroup: number;
}

export async function getFaceSuggestionSettings(): Promise<FaceSuggestionSettings> {
    return apiRequest<FaceSuggestionSettings>('/api/v1/config/face-suggestions');
}

export async function updateFaceSuggestionSettings(
    settings: FaceSuggestionSettings
): Promise<FaceSuggestionSettings> {
    return apiRequest<FaceSuggestionSettings>('/api/v1/config/face-suggestions', {
        method: 'PUT',
        body: JSON.stringify(settings)
    });
}
```

---

## 4. Existing Settings Examples

### 4.1 Simple Integer Setting

**Example**: `face_suggestion_max_results`

**Database Record**:
```sql
INSERT INTO system_configs (key, value, data_type, description, min_value, max_value, category)
VALUES (
    'face_suggestion_max_results',
    '50',
    'int',
    'Maximum number of suggestions to create when a face is manually labeled to a person.',
    '1',
    '200',
    'face_matching'
);
```

**Backend Usage**:
```python
service = ConfigService(db)
max_suggestions = await service.get_int("face_suggestion_max_results")  # Returns 50
```

**Frontend Display**:
```svelte
<input
    id="maxSuggestions"
    type="number"
    min="1"
    max="200"
    bind:value={config.maxSuggestions}
/>
```

### 4.2 Validated Float Setting with Cross-Field Constraint

**Example**: `face_suggestion_threshold` (must be less than `face_auto_assign_threshold`)

**Pydantic Validation**:
```python
class FaceMatchingConfigUpdateRequest(BaseModel):
    auto_assign_threshold: float = Field(..., ge=0.5, le=1.0)
    suggestion_threshold: float = Field(..., ge=0.3, le=0.95)

    @model_validator(mode="after")
    def validate_thresholds(self):
        if self.suggestion_threshold >= self.auto_assign_threshold:
            raise ValueError("suggestion_threshold must be less than auto_assign_threshold")
        return self
```

**Frontend Validation**:
```typescript
let isValid = $derived(config.suggestionThreshold < config.autoAssignThreshold);

function handleThresholdChange() {
    if (config.suggestionThreshold >= config.autoAssignThreshold) {
        validationError = 'Suggestion threshold must be less than auto-assign threshold';
    } else {
        validationError = null;
    }
}
```

### 4.3 Grouped Settings (Pagination)

**Example**: `face_suggestion_groups_per_page` + `face_suggestion_items_per_group`

**Migration** (`9511886120f4_seed_face_suggestion_pagination_config.py`):
```python
def upgrade() -> None:
    op.execute("""
        INSERT INTO system_configs (key, value, data_type, description, min_value, max_value, category)
        VALUES
        (
            'face_suggestion_groups_per_page',
            '10',
            'int',
            'Number of person groups to display per page in face suggestion review UI.',
            '1',
            '50',
            'face_matching'
        ),
        (
            'face_suggestion_items_per_group',
            '20',
            'int',
            'Maximum number of face suggestions to display per person group.',
            '1',
            '50',
            'face_matching'
        )
        ON CONFLICT (key) DO NOTHING
    """)
```

**API Endpoint** (specialized aggregated endpoint):
```python
@router.get("/face-suggestions", response_model=FaceSuggestionSettingsResponse)
async def get_face_suggestion_settings(db: AsyncSession = Depends(get_db)):
    service = ConfigService(db)
    return FaceSuggestionSettingsResponse(
        groups_per_page=await service.get_int("face_suggestion_groups_per_page"),
        items_per_group=await service.get_int("face_suggestion_items_per_group"),
    )
```

---

## 5. Pattern Analysis: "All vs. Top N" Settings

### 5.1 Current Limitations

**No existing "mode" pattern**: All current settings are **simple scalar values** (int, float, boolean, string).

**Closest analog**: `face_suggestion_items_per_group` (int)
- But this is **always a number**, not an "all or limit" toggle

### 5.2 Required New Pattern

For "post-training suggestions" setting with **mode** (all | top_n) and **top_n_count**:

**Option A: Two Separate Keys** (Recommended)
```sql
-- Key 1: Mode selector
INSERT INTO system_configs (key, value, data_type, description, allowed_values, category)
VALUES (
    'post_training_suggestions_mode',
    'top_n',
    'string',
    'Mode for generating suggestions after training completion. "all" processes all unknowns, "top_n" limits to top N by confidence.',
    '["all", "top_n"]',  -- JSONB array of allowed values
    'face_matching'
);

-- Key 2: Top N count (only used when mode = "top_n")
INSERT INTO system_configs (key, value, data_type, description, min_value, max_value, category)
VALUES (
    'post_training_suggestions_top_n_count',
    '10',
    'int',
    'Number of top suggestions to generate per person when mode is "top_n".',
    '1',
    '100',
    'face_matching'
);
```

**Backend API**:
```python
class PostTrainingSuggestionsConfigResponse(BaseModel):
    mode: str = Field(..., pattern="^(all|top_n)$")  # Enum-like validation
    top_n_count: int = Field(..., ge=1, le=100)

@router.get("/post-training-suggestions", response_model=PostTrainingSuggestionsConfigResponse)
async def get_post_training_suggestions_config(db: AsyncSession = Depends(get_db)):
    service = ConfigService(db)
    return PostTrainingSuggestionsConfigResponse(
        mode=await service.get_string("post_training_suggestions_mode"),
        top_n_count=await service.get_int("post_training_suggestions_top_n_count"),
    )

@router.put("/post-training-suggestions", response_model=PostTrainingSuggestionsConfigResponse)
async def update_post_training_suggestions_config(
    request: PostTrainingSuggestionsConfigUpdateRequest,
    db: AsyncSession = Depends(get_db),
):
    service = ConfigService(db)
    await service.set_value("post_training_suggestions_mode", request.mode)
    await service.set_value("post_training_suggestions_top_n_count", request.top_n_count)
    return PostTrainingSuggestionsConfigResponse(
        mode=request.mode,
        top_n_count=request.top_n_count,
    )
```

**Frontend UI Pattern** (radio buttons + conditional number input):
```svelte
<div class="form-field">
    <label>Post-Training Suggestion Mode</label>

    <div class="radio-group">
        <label>
            <input
                type="radio"
                name="suggestionMode"
                value="all"
                bind:group={config.postTrainingSuggestionsMode}
            />
            Generate suggestions for all unknown faces
        </label>

        <label>
            <input
                type="radio"
                name="suggestionMode"
                value="top_n"
                bind:group={config.postTrainingSuggestionsMode}
            />
            Generate suggestions for top N most confident matches
        </label>
    </div>

    {#if config.postTrainingSuggestionsMode === 'top_n'}
        <div class="nested-field">
            <label for="topNCount">
                Number of Suggestions per Person
                <span class="field-hint">Top N most confident matches (1-100)</span>
            </label>
            <input
                id="topNCount"
                type="number"
                min="1"
                max="100"
                bind:value={config.postTrainingSuggestionsTopNCount}
            />
        </div>
    {/if}
</div>
```

**Option B: JSON-Encoded Compound Value** (Not Recommended)
```sql
INSERT INTO system_configs (key, value, data_type, category)
VALUES (
    'post_training_suggestions_config',
    '{"mode": "top_n", "top_n_count": 10}',
    'string',  -- Store JSON as string
    'face_matching'
);
```

**Why Not Recommended?**:
- Loses type safety (all parsing at application layer)
- Can't use `min_value`/`max_value` validation constraints
- Harder to query individual sub-fields in SQL
- Less discoverable in UI (single opaque JSON blob)

---

## 6. Settings Usage in Background Jobs

### 6.1 Accessing Settings in RQ Workers

**Pattern**: Use `SyncConfigService` (synchronous DB operations):

```python
from image_search_service.services.config_service import SyncConfigService
from image_search_service.db.session import get_sync_session

def background_job_example():
    """RQ job that reads configuration values."""
    # Get synchronous DB session
    with get_sync_session() as db:
        config_service = SyncConfigService(db)

        # Read settings
        max_suggestions = config_service.get_int("face_suggestion_max_results")
        min_quality = config_service.get_float("face_prototype_min_quality")

        # Use settings in job logic
        process_faces(max_suggestions=max_suggestions, min_quality=min_quality)
```

**Why Synchronous?**:
- RQ workers run in **separate processes** (not async event loop)
- Background jobs use synchronous `requests`, `PIL`, `numpy` libraries
- Async/await not compatible with RQ's worker model

### 6.2 Settings Defaults (Fallback Behavior)

**Graceful Degradation**:
```python
class ConfigService:
    DEFAULTS: dict[str, float | int | str | bool] = {
        "face_auto_assign_threshold": 0.85,
        "face_suggestion_threshold": 0.70,
        # ...
    }

    async def _get_value(self, key: str, cast_type: type) -> Any:
        config = await self._get_config(key)
        if config:
            return cast_type(config.value)

        # Fallback to default if DB unavailable or key missing
        default = self.DEFAULTS.get(key)
        if default is not None:
            return cast_type(default)

        raise ValueError(f"Unknown configuration key: {key}")
```

**When Defaults Used**:
1. Database connection failure (service still healthy)
2. Migration not yet run (new settings not seeded)
3. Manual deletion of config record

---

## 7. Validation Strategy

### 7.1 Backend Validation (Service Layer)

**Method**: `ConfigService._validate_value()`

```python
def _validate_value(self, config: SystemConfig, value: Any) -> None:
    # 1. Type check
    if config.data_type == ConfigDataType.FLOAT.value:
        if not isinstance(value, (int, float)):
            raise ValueError(f"Value must be a number for {config.key}")
        value = float(value)

    # 2. Range check (numeric types)
    if config.min_value is not None and config.data_type in ("float", "int"):
        min_val = float(config.min_value)
        if float(value) < min_val:
            raise ValueError(f"{config.key} must be >= {min_val}")

    if config.max_value is not None and config.data_type in ("float", "int"):
        max_val = float(config.max_value)
        if float(value) > max_val:
            raise ValueError(f"{config.key} must be <= {max_val}")

    # 3. Allowed values check (enum-like)
    if config.allowed_values:
        if str(value) not in config.allowed_values:
            raise ValueError(f"{config.key} must be one of: {config.allowed_values}")
```

**Validation Points**:
1. **Service layer** (`ConfigService.set_value()`) - Enforces constraints from DB
2. **API layer** (Pydantic models) - Field-level constraints (`ge`, `le`, `pattern`)
3. **API validators** (`@model_validator`) - Cross-field constraints

### 7.2 Frontend Validation (UI Layer)

**Reactive Validation**:
```typescript
// Svelte 5 $derived for live validation
let isValid = $derived(
    config.suggestionThreshold < config.autoAssignThreshold &&
    paginationSettings.groupsPerPage >= 1 &&
    paginationSettings.groupsPerPage <= 50
);

// Save button disabled if invalid
<button onclick={saveConfig} disabled={saving || !isValid}>
    {saving ? 'Saving...' : 'Save Settings'}
</button>
```

**Input Constraints** (HTML5 validation):
```svelte
<input
    type="number"
    min="1"
    max="200"
    bind:value={config.maxSuggestions}
/>

<input
    type="range"
    min="0.5"
    max="1.0"
    step="0.01"
    bind:value={config.autoAssignThreshold}
/>
```

**Error Display**:
```svelte
{#if validationError}
    <div class="validation-error" role="alert">
        {validationError}
    </div>
{/if}
```

---

## 8. Recommendations for Adding New Settings

### 8.1 Step-by-Step Workflow

**Backend Changes**:

1. **Create Migration** (`make makemigrations`):
   ```python
   def upgrade() -> None:
       op.execute("""
           INSERT INTO system_configs (key, value, data_type, description, min_value, max_value, allowed_values, category)
           VALUES
           (
               'new_setting_key',
               'default_value',
               'int',  -- or 'float', 'string', 'boolean'
               'Description of what this setting does and its impact.',
               '1',     -- min_value (nullable)
               '100',   -- max_value (nullable)
               NULL,    -- allowed_values (JSONB array for enums)
               'face_matching'  -- category for UI grouping
           )
           ON CONFLICT (key) DO NOTHING
       """)
   ```

2. **Add to ConfigService DEFAULTS**:
   ```python
   # image-search-service/src/image_search_service/services/config_service.py
   DEFAULTS: dict[str, float | int | str | bool] = {
       # Existing...
       "new_setting_key": 50,  # Fallback value
   }
   ```

3. **Create/Update API Endpoint**:
   ```python
   # image-search-service/src/image_search_service/api/routes/config.py

   class NewSettingsResponse(BaseModel):
       new_setting_key: int

   @router.get("/new-settings", response_model=NewSettingsResponse)
   async def get_new_settings(db: AsyncSession = Depends(get_db)):
       service = ConfigService(db)
       return NewSettingsResponse(
           new_setting_key=await service.get_int("new_setting_key")
       )

   @router.put("/new-settings", response_model=NewSettingsResponse)
   async def update_new_settings(request: NewSettingsUpdateRequest, db: AsyncSession):
       service = ConfigService(db)
       await service.set_value("new_setting_key", request.new_setting_key)
       return NewSettingsResponse(new_setting_key=request.new_setting_key)
   ```

4. **Run Migration**:
   ```bash
   cd image-search-service
   make migrate
   ```

5. **Update API Contract** (if exposing to frontend):
   ```markdown
   # docs/api-contract.md

   ### GET /api/v1/config/new-settings
   Returns new configuration settings.

   **Response 200**:
   ```json
   {
       "newSettingKey": 50
   }
   ```
   ```

**Frontend Changes**:

6. **Regenerate TypeScript Types**:
   ```bash
   cd image-search-ui
   npm run gen:api  # Generates src/lib/api/generated.ts
   ```

7. **Add API Client Functions** (`src/lib/api/admin.ts` or domain-specific):
   ```typescript
   export interface NewSettings {
       newSettingKey: number;
   }

   export async function getNewSettings(): Promise<NewSettings> {
       return apiRequest<NewSettings>('/api/v1/config/new-settings');
   }

   export async function updateNewSettings(settings: NewSettings): Promise<NewSettings> {
       return apiRequest<NewSettings>('/api/v1/config/new-settings', {
           method: 'PUT',
           body: JSON.stringify(settings)
       });
   }
   ```

8. **Update FaceMatchingSettings Component** (or create new section):
   ```svelte
   <script lang="ts">
   import { getNewSettings, updateNewSettings, type NewSettings } from '$lib/api/admin';

   let newSettings = $state<NewSettings>({ newSettingKey: 50 });

   onMount(async () => {
       newSettings = await getNewSettings();
   });

   async function saveConfig() {
       // ... existing save logic
       await updateNewSettings(newSettings);
   }
   </script>

   <div class="other-settings">
       <h3>New Settings Section</h3>

       <div class="form-field">
           <label for="newSettingKey">
               New Setting Label
               <span class="field-hint">Description of what this controls (1-100)</span>
           </label>
           <input
               id="newSettingKey"
               type="number"
               min="1"
               max="100"
               bind:value={newSettings.newSettingKey}
           />
       </div>
   </div>
   ```

### 8.2 Best Practices

**Database Design**:
- ✅ Use descriptive `key` names (`feature_setting_name` pattern)
- ✅ Always set `category` for UI grouping
- ✅ Write detailed `description` (explains purpose and impact)
- ✅ Set `min_value`/`max_value` for numeric types
- ✅ Use `allowed_values` for enum-like constraints (mode selectors)

**Migration Safety**:
- ✅ Use `ON CONFLICT (key) DO NOTHING` for idempotency
- ✅ Seed with sensible defaults (don't require manual config)
- ✅ Include descriptive migration docstring

**API Design**:
- ✅ Create specialized endpoints for **grouped settings** (e.g., `/face-matching`)
- ✅ Use Pydantic validators for cross-field constraints
- ✅ Transform snake_case (backend) to camelCase (frontend) via `alias_generator`
- ✅ Return HTTP 400 with error details on validation failure

**Frontend UX**:
- ✅ Use appropriate input types (number, range, radio, checkbox)
- ✅ Show validation errors inline (don't wait for save)
- ✅ Disable save button when form invalid
- ✅ Display success toast (5-second auto-dismiss)
- ✅ Add helpful hints (`<span class="field-hint">`) for complex settings

**Testing**:
- ✅ Test migration up/down (`make migrate`, `alembic downgrade`)
- ✅ Test API validation (invalid values return 400)
- ✅ Test frontend validation (save button disabled)
- ✅ Test cross-field constraints (e.g., threshold ordering)

---

## 9. Complete Example: "All vs. Top N" Setting

### 9.1 Migration

```python
"""add_post_training_suggestions_config

Revision ID: abc123def456
Revises: previous_migration_id
Create Date: 2026-01-12 12:00:00.000000
"""
from collections.abc import Sequence
from alembic import op

revision: str = 'abc123def456'
down_revision: str | Sequence[str] | None = 'previous_migration_id'

def upgrade() -> None:
    """Add post-training suggestions configuration."""
    op.execute("""
        INSERT INTO system_configs (key, value, data_type, description, allowed_values, category)
        VALUES
        (
            'post_training_suggestions_mode',
            'top_n',
            'string',
            'Mode for generating face suggestions after training completion. "all" processes all unknown faces, "top_n" limits to top N most confident matches per person.',
            '["all", "top_n"]',
            'face_matching'
        ),
        (
            'post_training_suggestions_top_n_count',
            '10',
            'int',
            'Number of top suggestions to generate per person when mode is "top_n". Only applies when mode is set to "top_n".',
            NULL,
            '1',
            '100',
            'face_matching'
        )
        ON CONFLICT (key) DO NOTHING
    """)

def downgrade() -> None:
    """Remove post-training suggestions configuration."""
    op.execute("""
        DELETE FROM system_configs
        WHERE key IN ('post_training_suggestions_mode', 'post_training_suggestions_top_n_count')
    """)
```

### 9.2 Backend Service

**Update ConfigService Defaults**:
```python
# image-search-service/src/image_search_service/services/config_service.py

class ConfigService:
    DEFAULTS: dict[str, float | int | str | bool] = {
        # ... existing defaults
        "post_training_suggestions_mode": "top_n",
        "post_training_suggestions_top_n_count": 10,
    }
```

**API Endpoint**:
```python
# image-search-service/src/image_search_service/api/routes/config.py

class PostTrainingSuggestionsConfigResponse(BaseModel):
    """Post-training suggestion generation configuration."""

    model_config = ConfigDict(
        populate_by_name=True,
        alias_generator=to_camel,
    )

    mode: str = Field(..., pattern="^(all|top_n)$", description="Suggestion generation mode")
    top_n_count: int = Field(..., ge=1, le=100, description="Top N count when mode is top_n")


class PostTrainingSuggestionsConfigUpdateRequest(BaseModel):
    """Request to update post-training suggestions configuration."""

    model_config = ConfigDict(
        populate_by_name=True,
        alias_generator=to_camel,
    )

    mode: str = Field(..., pattern="^(all|top_n)$")
    top_n_count: int = Field(..., ge=1, le=100)

    @model_validator(mode="after")
    def validate_top_n_required(self) -> "PostTrainingSuggestionsConfigUpdateRequest":
        """Warn if top_n_count set but mode is 'all' (not enforced, just advisory)."""
        if self.mode == "all" and self.top_n_count != 10:
            logger.warning(
                f"top_n_count={self.top_n_count} set but mode='all'. "
                "This value will be ignored."
            )
        return self


@router.get("/post-training-suggestions", response_model=PostTrainingSuggestionsConfigResponse)
async def get_post_training_suggestions_config(
    db: AsyncSession = Depends(get_db),
) -> PostTrainingSuggestionsConfigResponse:
    """Get post-training suggestions configuration.

    Returns the current configuration for generating face suggestions
    after training session completion.
    """
    service = ConfigService(db)

    return PostTrainingSuggestionsConfigResponse(
        mode=await service.get_string("post_training_suggestions_mode"),
        top_n_count=await service.get_int("post_training_suggestions_top_n_count"),
    )


@router.put("/post-training-suggestions", response_model=PostTrainingSuggestionsConfigResponse)
async def update_post_training_suggestions_config(
    request: PostTrainingSuggestionsConfigUpdateRequest,
    db: AsyncSession = Depends(get_db),
) -> PostTrainingSuggestionsConfigResponse:
    """Update post-training suggestions configuration.

    Updates the configuration for generating face suggestions after
    training session completion.
    """
    service = ConfigService(db)

    try:
        await service.set_value("post_training_suggestions_mode", request.mode)
        await service.set_value("post_training_suggestions_top_n_count", request.top_n_count)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

    logger.info(
        f"Updated post-training suggestions config: mode={request.mode}, "
        f"top_n_count={request.top_n_count}"
    )

    return PostTrainingSuggestionsConfigResponse(
        mode=request.mode,
        top_n_count=request.top_n_count,
    )
```

### 9.3 Frontend API Client

```typescript
// image-search-ui/src/lib/api/admin.ts

export interface PostTrainingSuggestionsConfig {
    mode: 'all' | 'top_n';
    topNCount: number;
}

export async function getPostTrainingSuggestionsConfig(): Promise<PostTrainingSuggestionsConfig> {
    return apiRequest<PostTrainingSuggestionsConfig>('/api/v1/config/post-training-suggestions');
}

export async function updatePostTrainingSuggestionsConfig(
    config: PostTrainingSuggestionsConfig
): Promise<PostTrainingSuggestionsConfig> {
    return apiRequest<PostTrainingSuggestionsConfig>('/api/v1/config/post-training-suggestions', {
        method: 'PUT',
        body: JSON.stringify(config)
    });
}
```

### 9.4 Frontend UI Component

```svelte
<!-- image-search-ui/src/lib/components/admin/FaceMatchingSettings.svelte -->

<script lang="ts">
import {
    getPostTrainingSuggestionsConfig,
    updatePostTrainingSuggestionsConfig,
    type PostTrainingSuggestionsConfig
} from '$lib/api/admin';

let postTrainingConfig = $state<PostTrainingSuggestionsConfig>({
    mode: 'top_n',
    topNCount: 10
});

onMount(async () => {
    // Load existing config
    postTrainingConfig = await getPostTrainingSuggestionsConfig();
});

async function saveConfig() {
    // ... existing save logic

    // Save post-training suggestions config
    await updatePostTrainingSuggestionsConfig(postTrainingConfig);
}
</script>

<div class="other-settings">
    <h3>Post-Training Suggestions</h3>
    <p class="section-description">
        Configure how face suggestions are generated after training session completion.
        "All" mode processes all unknown faces (may be slow), while "Top N" limits to the
        most confident matches for faster processing.
    </p>

    <div class="form-field">
        <label>Generation Mode</label>

        <div class="radio-group">
            <label class="radio-label">
                <input
                    type="radio"
                    name="postTrainingMode"
                    value="all"
                    bind:group={postTrainingConfig.mode}
                />
                <div class="radio-content">
                    <strong>All Unknown Faces</strong>
                    <span class="radio-hint">
                        Generate suggestions for all unknown faces (comprehensive but slower)
                    </span>
                </div>
            </label>

            <label class="radio-label">
                <input
                    type="radio"
                    name="postTrainingMode"
                    value="top_n"
                    bind:group={postTrainingConfig.mode}
                />
                <div class="radio-content">
                    <strong>Top N Most Confident</strong>
                    <span class="radio-hint">
                        Generate suggestions for top N most confident matches per person (faster)
                    </span>
                </div>
            </label>
        </div>

        {#if postTrainingConfig.mode === 'top_n'}
            <div class="nested-field">
                <label for="topNCount">
                    Number of Suggestions per Person
                    <span class="field-hint">
                        Top N most confident matches (1-100). Higher values increase accuracy
                        but take longer to process.
                    </span>
                </label>
                <input
                    id="topNCount"
                    type="number"
                    min="1"
                    max="100"
                    bind:value={postTrainingConfig.topNCount}
                />
            </div>
        {/if}
    </div>
</div>

<style>
.radio-group {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    margin-top: 0.5rem;
}

.radio-label {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    padding: 0.875rem;
    background: white;
    border: 2px solid var(--border-color, #e5e7eb);
    border-radius: 8px;
    cursor: pointer;
    transition: border-color 0.15s ease;
}

.radio-label:has(input:checked) {
    border-color: var(--primary, #3b82f6);
    background: rgba(59, 130, 246, 0.05);
}

.radio-label input[type="radio"] {
    margin-top: 0.125rem;
    cursor: pointer;
}

.radio-content {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
}

.radio-content strong {
    font-size: 0.875rem;
    color: var(--text-primary, #1f2937);
}

.radio-hint {
    font-size: 0.75rem;
    color: var(--text-muted, #6b7280);
    line-height: 1.4;
}

.nested-field {
    margin-top: 1rem;
    padding: 1rem;
    background: rgba(59, 130, 246, 0.05);
    border-left: 3px solid var(--primary, #3b82f6);
    border-radius: 6px;
}
</style>
```

### 9.5 Background Job Usage

```python
# image-search-service/src/image_search_service/queue/face_jobs.py

from image_search_service.services.config_service import SyncConfigService
from image_search_service.db.session import get_sync_session

def generate_post_training_suggestions(session_id: int):
    """Generate face suggestions after training completion."""
    with get_sync_session() as db:
        config_service = SyncConfigService(db)

        # Read settings
        mode = config_service.get_string("post_training_suggestions_mode")
        top_n_count = config_service.get_int("post_training_suggestions_top_n_count")

        logger.info(
            f"Generating post-training suggestions for session {session_id}: "
            f"mode={mode}, top_n_count={top_n_count}"
        )

        if mode == "all":
            # Process all unknown faces
            unknown_faces = get_all_unknown_faces(db, session_id)
            logger.info(f"Processing {len(unknown_faces)} unknown faces (mode=all)")
        else:  # mode == "top_n"
            # Process top N per person
            unknown_faces = get_top_n_unknown_faces(db, session_id, limit=top_n_count)
            logger.info(f"Processing top {top_n_count} faces per person (mode=top_n)")

        # Generate suggestions
        suggestions = create_suggestions_from_faces(db, unknown_faces)
        logger.info(f"Created {len(suggestions)} suggestions")
```

---

## 10. File Locations Summary

**Backend**:
- Models: `image-search-service/src/image_search_service/db/models.py:798-838`
- Service: `image-search-service/src/image_search_service/services/config_service.py`
- API Routes: `image-search-service/src/image_search_service/api/routes/config.py`
- Migrations: `image-search-service/src/image_search_service/db/migrations/versions/`

**Frontend**:
- Admin Page: `image-search-ui/src/routes/admin/+page.svelte`
- Settings Component: `image-search-ui/src/lib/components/admin/FaceMatchingSettings.svelte`
- API Client: `image-search-ui/src/lib/api/admin.ts`
- Generated Types: `image-search-ui/src/lib/api/generated.ts` (auto-generated, DO NOT EDIT)

**Documentation**:
- API Contract: `docs/api-contract.md` (both subprojects, must be identical)

---

## 11. Key Insights and Learnings

### 11.1 Architectural Strengths

✅ **Type Safety**: Database constraints + Pydantic validation + TypeScript types = multiple validation layers
✅ **Runtime Updates**: Settings changes take effect immediately (no service restart required for DB-backed settings)
✅ **Graceful Degradation**: Defaults in `ConfigService.DEFAULTS` ensure service works if DB unavailable
✅ **UI Grouping**: `category` field enables logical grouping in Admin Panel
✅ **Validation Feedback**: Frontend shows validation errors inline before API call

### 11.2 Design Patterns

**Single Responsibility**: Each setting has one purpose, stored as separate key
**Grouped Endpoints**: Related settings (e.g., face matching) have specialized endpoints for atomicity
**Snake to Camel**: Backend uses snake_case, frontend camelCase (automatic transformation via `alias_generator`)
**Progressive Enhancement**: Starts with simple types (int, float), extends with allowed_values for enums

### 11.3 Gaps and Limitations

⚠️ **No Compound Settings**: Current system lacks pattern for "mode + conditional fields"
⚠️ **Environment Variable Hybrid**: Some settings (e.g., unknown clustering) still require restart
⚠️ **Limited to Scalar Values**: No support for arrays, nested objects (JSONB requires string parsing)

### 11.4 Migration Path for Compound Settings

**Recommendation**: Use **separate keys** (not JSON-encoded compound values):
- Better type safety (individual field validation)
- Leverages existing `allowed_values` for mode selector
- Discoverable in UI (separate form fields)
- Queryable in SQL (can filter by mode)

**Trade-off**: Requires conditional UI logic (show/hide `top_n_count` based on `mode`)

---

## 12. Next Steps

### 12.1 For Implementation

1. ✅ Create migration with two keys (`mode` + `top_n_count`)
2. ✅ Add keys to `ConfigService.DEFAULTS`
3. ✅ Create specialized API endpoint (`/post-training-suggestions`)
4. ✅ Add Pydantic response/request models with validation
5. ✅ Update frontend API client (`admin.ts`)
6. ✅ Extend `FaceMatchingSettings.svelte` with radio group + conditional number input
7. ✅ Update background job to read both settings
8. ✅ Test validation (frontend + backend)
9. ✅ Update API contract documentation

### 12.2 Testing Checklist

- [ ] Migration up/down (data preserved)
- [ ] API GET returns correct defaults
- [ ] API PUT validates mode (rejects invalid values)
- [ ] API PUT validates top_n_count (range 1-100)
- [ ] Frontend loads settings on mount
- [ ] Frontend disables save when invalid
- [ ] Frontend shows/hides top_n_count based on mode
- [ ] Background job reads correct values
- [ ] Backend uses fallback if DB unavailable

---

**Research Completed**: 2026-01-12
**Total Investigation Time**: ~45 minutes
**Files Analyzed**: 12 backend files, 6 frontend files, 2 migration files
**Recommendation**: ✅ **Use separate keys pattern** for "all vs. top N" settings (validated, type-safe, UI-friendly)
