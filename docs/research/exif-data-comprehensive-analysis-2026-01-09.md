# Comprehensive EXIF Data Analysis for Image Search System

**Research Date**: 2026-01-09
**Scope**: Backend (image-search-service) + Frontend (image-search-ui)
**Status**: Gap Analysis & Implementation Roadmap
**Priority**: 🟡 IMPORTANT - Mentioned in roadmap, critical for temporal accuracy

---

## Executive Summary

The image search system currently has **minimal EXIF data extraction capabilities**. Orientation handling exists (via `ImageOps.exif_transpose()`) but comprehensive metadata extraction (dates, GPS, camera info) is not implemented. This analysis provides:

1. **Current State Assessment** - What exists today in both backend and frontend
2. **Gap Analysis** - What's missing for complete EXIF support
3. **Data Model Recommendations** - Where to store EXIF (Postgres vs Qdrant vs both)
4. **Architecture Design** - Where to extract, how to access, performance considerations
5. **Migration Strategy** - How to backfill existing images
6. **Implementation Phases** - Suggested order of work (prioritized)
7. **API Contract Changes** - New endpoints and response fields
8. **UI Integration Points** - Where EXIF data enhances user experience

**Key Finding**: System architecture already supports EXIF integration with minimal disruption. Orientation handling proves PIL/Pillow is available. Face detection pipeline provides natural injection point for metadata extraction during image processing.

---

## 1. Current State Analysis

### 1.1 Backend EXIF Handling (image-search-service)

#### ✅ **What Exists**: Orientation Handling

**Location**: `src/image_search_service/services/thumbnail_service.py`

```python
# Lines 90-92: get_image_dimensions()
with Image.open(path) as img:
    # Apply EXIF orientation to get actual dimensions
    img = ImageOps.exif_transpose(img) or img
    width, height = img.size
    return (width, height)

# Lines 153-154: generate_thumbnail()
with Image.open(original) as img:
    # Apply EXIF orientation transformation (handles rotated smartphone photos)
    img = ImageOps.exif_transpose(img) or img
```

**Capabilities**:
- ✅ Reads EXIF orientation tag (tag 0x0112)
- ✅ Applies rotation during thumbnail generation
- ✅ Prevents sideways/upside-down thumbnails
- ✅ Maintains aspect ratios correctly

**Research Reference**: `docs/research/face-thumbnail-orientation-issue-2025-12-27.md` documents orientation fix implementation.

#### ❌ **What's Missing**: Comprehensive EXIF Extraction

**No extraction of**:
- 📅 **Date/Time**:
  - `DateTimeOriginal` (when photo was taken)
  - `DateTime` (when file was modified)
  - `DateTimeDigitized` (when digitized/scanned)
- 📍 **GPS Location**:
  - `GPSLatitude`, `GPSLongitude`
  - `GPSAltitude`
  - Location name (reverse geocoding)
- 📷 **Camera Information**:
  - `Make`, `Model` (e.g., "Apple", "iPhone 14 Pro")
  - `LensModel`
  - `FocalLength`, `FNumber`, `ISO`, `ExposureTime`
- 🎨 **Image Characteristics**:
  - `Orientation` (stored but not persisted to DB)
  - `Flash`, `WhiteBalance`
  - `ColorSpace`

**Why This Matters**:
- **Temporal Accuracy**: Face age estimation currently relies on folder/filename dates (unreliable) instead of `DateTimeOriginal`
- **Search Enhancement**: Can't filter by date range, location, or camera type
- **User Experience**: Photo details missing from UI (users want to know when/where/how photo was taken)
- **Deduplication**: Can't use metadata to find duplicates (same timestamp + same camera = likely duplicate)

#### 📊 **Database Models** (`src/image_search_service/db/models.py`)

**ImageAsset Model** (Lines 137-185):

```python
class ImageAsset(Base):
    __tablename__ = "image_assets"

    id: Mapped[int]
    path: Mapped[str]
    created_at: Mapped[datetime]  # DB insert timestamp
    indexed_at: Mapped[datetime | None]

    # Technical metadata (populated)
    thumbnail_path: Mapped[str | None]
    width: Mapped[int | None]  # From EXIF-corrected dimensions
    height: Mapped[int | None]
    file_size: Mapped[int | None]
    mime_type: Mapped[str | None]
    file_modified_at: Mapped[datetime | None]  # Filesystem mtime
    training_status: Mapped[str]

    # 🔴 MISSING EXIF fields:
    # taken_at: Mapped[datetime | None]  # EXIF DateTimeOriginal
    # exif_orientation: Mapped[int | None]  # 1-8
    # camera_make: Mapped[str | None]
    # camera_model: Mapped[str | None]
    # gps_latitude: Mapped[float | None]
    # gps_longitude: Mapped[float | None]
    # focal_length: Mapped[float | None]
    # iso: Mapped[int | None]
    # exif_json: Mapped[dict | None]  # Full EXIF as JSONB
```

**Key Observations**:
- `file_modified_at` exists but stores filesystem mtime (not photo capture time)
- `created_at` is DB insertion timestamp (not photo date)
- No `taken_at` field for actual photo capture timestamp
- No EXIF metadata storage (neither individual fields nor JSON blob)

**Person Model** (Lines 390-434):

```python
class Person(Base):
    __tablename__ = "persons"

    id: Mapped[uuid.UUID]
    name: Mapped[str]
    status: Mapped[PersonStatus]
    created_at: Mapped[datetime]
    updated_at: Mapped[datetime]

    # 🔴 MISSING person metadata:
    # birth_date: Mapped[date | None]  # For accurate age calculation
    # birth_year: Mapped[int | None]  # For estimated age ranges
```

**Temporal Age Estimation**: Currently uses era buckets (infant/child/teen/adult/senior) without knowing person's birth date. With EXIF dates + DOB, could calculate exact age at photo time.

#### 🔌 **Qdrant Vector Metadata** (`src/image_search_service/scripts/bootstrap_qdrant.py`)

**Image Assets Collection** (Lines 50-56):

```python
# Create collection with cosine similarity
client.create_collection(
    collection_name=collection_name,
    vectors_config=VectorParams(size=embedding_dim, distance=Distance.COSINE),
)
# No payload indexes defined for image_assets collection
```

**Faces Collection** (Lines 88-105):

```python
# Payload indexes for efficient filtering
indexes = [
    ("person_id", PayloadSchemaType.KEYWORD),
    ("cluster_id", PayloadSchemaType.KEYWORD),
    ("is_prototype", PayloadSchemaType.BOOL),
    ("asset_id", PayloadSchemaType.KEYWORD),
    ("face_instance_id", PayloadSchemaType.KEYWORD),
]
# No date/location indexes
```

**Observation**: Qdrant supports payload filtering but no EXIF metadata is currently stored in payloads.

#### 🚀 **Image Ingestion Pipeline** (Inference)

Based on model fields and thumbnail service:

```
1. File Discovery (asset_discovery.py or training_service.py)
   ↓
2. Database Insert (ImageAsset created with path + file_modified_at)
   ↓
3. Background Job Queued (RQ)
   ↓
4. Embedding Generation (OpenCLIP)
   ↓
5. Thumbnail Generation (EXIF orientation applied, then discarded)
   ↓
6. Face Detection (Optional, via InsightFace)
   ↓
7. Vector Storage (Qdrant)
```

**❌ No EXIF Extraction Step**: Metadata is read temporarily for orientation but not persisted.

### 1.2 Frontend Metadata Display (image-search-ui)

#### 📱 **UI Components**

**PhotoPreviewModal.svelte** (`src/lib/components/faces/PhotoPreviewModal.svelte`):

```typescript
interface Props {
  photo: PersonPhotoGroup;  // Contains faces in photo
  currentPersonId?: string | null;
  currentPersonName?: string | null;
  onClose: () => void;
  // ... navigation and assignment callbacks
}
```

**Displayed Metadata**:
- ✅ Asset ID
- ✅ Image dimensions (from thumbnail service)
- ✅ Face bounding boxes
- ✅ Person assignments
- ❌ Photo capture date
- ❌ Location information
- ❌ Camera details

**TemporalTimeline.svelte** (`src/lib/components/faces/TemporalTimeline.svelte`):

```typescript
interface Props {
  prototypes: Prototype[];  // Face exemplars per age era
  coverage: TemporalCoverage;  // Age bucket coverage stats
  onPinClick?: (era: AgeEraBucket) => void;
}

const eras = [
  { key: 'infant', label: 'Infant', range: '0-3' },
  { key: 'child', label: 'Child', range: '4-12' },
  { key: 'teen', label: 'Teen', range: '13-19' },
  { key: 'young_adult', label: 'Young Adult', range: '20-35' },
  { key: 'adult', label: 'Adult', range: '36-55' },
  { key: 'senior', label: 'Senior', range: '56+' }
];
```

**Age Bucketing**: Based on facial analysis estimates, NOT photo dates. With `taken_at` + `birth_date`, could show:
- "Age 8 (2015)" instead of "Child (4-12)"
- Timeline scrubber by actual years
- Gap detection ("No photos from 2018-2020")

#### 🎯 **API Types** (`src/lib/api/generated.ts`)

Auto-generated from backend OpenAPI spec. Currently includes:

```typescript
interface Asset {
  id: number;
  path: string;
  createdAt: string;  // ISO 8601 (DB insert time)
  indexedAt?: string | null;
  url: string;  // Computed: /api/v1/images/{id}/full
  thumbnailUrl: string;  // Computed: /api/v1/images/{id}/thumbnail
  filename: string;  // Computed: path.split('/').pop()

  // 🔴 MISSING:
  // takenAt?: string | null;  // EXIF DateTimeOriginal
  // location?: { lat: number; lng: number } | null;
  // camera?: { make: string; model: string } | null;
  // exif?: ExifMetadata | null;
}
```

**Pydantic Schemas** (`image-search-service/src/image_search_service/api/schemas.py`):

```python
class Asset(BaseModel):
    model_config = ConfigDict(populate_by_name=True, from_attributes=True)

    id: int
    path: str
    created_at: datetime = Field(alias="createdAt")
    indexed_at: datetime | None = Field(None, alias="indexedAt")

    @computed_field(alias="url")
    def url(self) -> str:
        return f"/api/v1/images/{self.id}/full"

    @computed_field(alias="thumbnailUrl")
    def thumbnail_url(self) -> str:
        return f"/api/v1/images/{self.id}/thumbnail"

    @computed_field(alias="filename")
    def filename(self) -> str:
        return self.path.split("/")[-1]
```

**Gap**: No EXIF fields exposed via API.

### 1.3 Cross-Cutting Concerns

#### ✅ **Orientation Handling** (SOLVED)

**Current Flow**:
1. ✅ Thumbnail generation applies `ImageOps.exif_transpose()`
2. ✅ Dimension calculation applies orientation
3. ✅ Face detection uses OpenCV (orientation-agnostic after thumbnail fix)
4. ✅ Browser displays thumbnails correctly (orientation baked into pixels)

**Status**: ✅ COMPLETE (as of 2025-12-27 research)

#### ❌ **Date Handling** (MISSING)

**Current Situation**:
- `created_at` = Database insertion timestamp (useless for "when was photo taken?")
- `file_modified_at` = Filesystem mtime (unreliable, changes with file moves/edits)
- No `taken_at` field

**User Impact**:
- Can't sort photos by actual capture date
- Timeline features show incorrect chronology
- Age estimation uses folder dates (if available) or guesses

**Example Mismatch**:
```
Photo: IMG_1234.jpg
file_modified_at: 2025-12-01 (when copied to server)
created_at: 2025-12-15 (when ingested to DB)
EXIF DateTimeOriginal: 2018-06-15 14:32:10 (actual photo date)
                       ^^^^^^^^^^^^^^^^^ NOT STORED!
```

#### 🗺️ **Metadata Flow** (INCOMPLETE)

**Current**:
```
Upload → Path stored in DB → Background job → Embedding + Thumbnail
                                              ↓
                                    EXIF read temporarily
                                    (orientation only, then discarded)
```

**Desired**:
```
Upload → Path stored in DB → Background job → EXIF Extraction
                                              ↓
                                         Postgres (structured)
                                         Qdrant payload (filterable)
                                              ↓
                                    Embedding + Thumbnail
                                              ↓
                                         API response
                                              ↓
                                         UI display
```

---

## 2. Gap Analysis

### 2.1 Critical Gaps (High Impact, Mentioned in Roadmap)

#### 🔴 **Gap 1: No `taken_at` Field**

**Impact**:
- **Face Age Estimation**: Currently guesses age era from visual analysis. With `taken_at` + `birth_date`, could calculate exact age.
- **Photo Sorting**: Users can't sort by actual capture date.
- **Timeline View**: Can't show "2015 summer vacation" vs "2020 graduation" chronologically.
- **Search Filters**: Can't implement "photos from 2010-2015" date range filter.

**Evidence**:
- README.md roadmap: "EXIF Data Extraction - Parse taken_at timestamps from photo metadata"
- BLOG_PROJECT_HIGHLIGHTS.md: "Would add photo EXIF date extraction for better temporal accuracy"

**Technical Severity**: 🔴 HIGH - Core feature gap blocking multiple user stories.

#### 🔴 **Gap 2: No Person Birth Date**

**Impact**:
- **Age Calculation**: Can't determine "person was 5 years old in this photo" without DOB.
- **Temporal Prototypes**: Era buckets ("child 4-12") are imprecise vs. "Age 8 (2015)".
- **Lifecycle Visualization**: Can't show "All photos of John from birth to present".

**Schema Change Needed**:
```sql
ALTER TABLE persons ADD COLUMN birth_date DATE NULL;
ALTER TABLE persons ADD COLUMN birth_year INTEGER NULL;  -- For privacy (year-only estimate)
```

**Technical Severity**: 🟡 MEDIUM - Enhances existing feature (temporal prototypes).

#### 🟡 **Gap 3: No GPS Location Data**

**Impact**:
- **Location-Based Search**: Can't find "photos taken in Paris" or "within 10 miles of home".
- **Map View**: Can't show photos on a map (Google Maps / OpenStreetMap integration).
- **Privacy Concerns**: GPS data is sensitive (needs opt-in extraction).

**Schema Change Needed**:
```sql
ALTER TABLE image_assets ADD COLUMN gps_latitude FLOAT NULL;
ALTER TABLE image_assets ADD COLUMN gps_longitude FLOAT NULL;
ALTER TABLE image_assets ADD COLUMN gps_altitude FLOAT NULL;
ALTER TABLE image_assets ADD COLUMN location_name TEXT NULL;  -- Reverse geocoded
```

**Technical Severity**: 🟢 LOW - Nice-to-have feature, complex privacy implications.

#### 🟢 **Gap 4: No Camera Metadata**

**Impact**:
- **Camera Statistics**: Can't answer "which camera did I use most in 2020?"
- **Quality Filters**: Can't filter by "photos from DSLR only" vs smartphone.
- **Lens Analysis**: Photographers want to know which lens/focal length was used.

**Schema Change Needed**:
```sql
ALTER TABLE image_assets ADD COLUMN camera_make VARCHAR(100) NULL;
ALTER TABLE image_assets ADD COLUMN camera_model VARCHAR(100) NULL;
ALTER TABLE image_assets ADD COLUMN lens_model VARCHAR(100) NULL;
ALTER TABLE image_assets ADD COLUMN focal_length FLOAT NULL;
ALTER TABLE image_assets ADD COLUMN iso INTEGER NULL;
```

**Technical Severity**: 🟢 LOW - Power user feature, not critical for core workflows.

### 2.2 Extraction Gaps (Where to Extract)

**Current Extraction Points**:
1. ✅ Thumbnail generation (orientation only)
2. ❌ Image ingestion (no EXIF read)
3. ❌ Background jobs (no EXIF processing)

**Proposed Injection Point**: Background job after image ingestion, before embedding.

**Reasoning**:
- Async processing (won't block API responses)
- Batch-friendly (can process thousands of images)
- Centralized error handling (EXIF read failures don't crash ingestion)
- Resumable (can retry failed EXIF extractions)

### 2.3 Storage Gaps (Where to Store)

**Current Storage**:
- ✅ Postgres: `ImageAsset` table (path, dimensions, file metadata)
- ✅ Qdrant: `image_assets` collection (embeddings + minimal payload)
- ❌ No EXIF storage

**Proposed Storage Architecture** (see Section 3):
- **Postgres**: Queryable fields (`taken_at`, `camera_make`, `gps_latitude`)
- **Postgres**: Full EXIF JSON blob (JSONB column for extensibility)
- **Qdrant**: Filterable fields in payload (date ranges, location filters)

---

## 3. Data Model Recommendations

### 3.1 Database Schema Design

#### **Option A: Individual Columns (Recommended)**

**Pros**:
- ✅ Type-safe (DATE, FLOAT, INTEGER)
- ✅ Indexable (fast queries on `taken_at`, `gps_latitude`)
- ✅ Sortable (ORDER BY taken_at DESC)
- ✅ API schema generation (Pydantic models match DB columns)

**Cons**:
- ❌ Schema evolution (adding new EXIF field = migration)
- ❌ Verbose (many columns for comprehensive metadata)

**Recommended Fields**:

```sql
-- Migration: Add EXIF fields to image_assets table
ALTER TABLE image_assets
  -- Core temporal metadata (HIGH PRIORITY)
  ADD COLUMN taken_at TIMESTAMP WITH TIME ZONE NULL,
  ADD COLUMN exif_orientation INTEGER NULL,  -- 1-8

  -- Location metadata (MEDIUM PRIORITY)
  ADD COLUMN gps_latitude DOUBLE PRECISION NULL,
  ADD COLUMN gps_longitude DOUBLE PRECISION NULL,
  ADD COLUMN gps_altitude DOUBLE PRECISION NULL,

  -- Camera metadata (LOW PRIORITY)
  ADD COLUMN camera_make VARCHAR(100) NULL,
  ADD COLUMN camera_model VARCHAR(100) NULL,
  ADD COLUMN lens_model VARCHAR(100) NULL,
  ADD COLUMN focal_length DOUBLE PRECISION NULL,  -- mm
  ADD COLUMN f_number DOUBLE PRECISION NULL,      -- e.g., 2.8
  ADD COLUMN iso INTEGER NULL,
  ADD COLUMN exposure_time VARCHAR(20) NULL,      -- e.g., "1/250"

  -- Full EXIF blob (EXTENSIBILITY)
  ADD COLUMN exif_json JSONB NULL;

-- Indexes for common queries
CREATE INDEX idx_image_assets_taken_at ON image_assets(taken_at) WHERE taken_at IS NOT NULL;
CREATE INDEX idx_image_assets_gps ON image_assets(gps_latitude, gps_longitude) WHERE gps_latitude IS NOT NULL;
CREATE INDEX idx_image_assets_camera_model ON image_assets(camera_model) WHERE camera_model IS NOT NULL;

-- GIN index for JSONB queries (if needed)
CREATE INDEX idx_image_assets_exif_json ON image_assets USING GIN(exif_json);
```

**Why JSONB for Full Blob?**
- Extensibility: Can add obscure EXIF tags without migrations
- Debugging: Preserve all metadata for troubleshooting
- User requests: "Show me all EXIF data for this photo"

#### **Option B: JSONB Only (Not Recommended)**

**Approach**: Store all EXIF in single JSONB column.

**Pros**:
- ✅ Schema-free (no migrations for new fields)
- ✅ Simple initial implementation

**Cons**:
- ❌ Slower queries (JSONB index overhead)
- ❌ No type safety (everything is JSON)
- ❌ Complex API schemas (Pydantic must parse JSON)
- ❌ Poor sorting performance

**Verdict**: ❌ Not recommended for queryable fields like `taken_at`.

#### **Option C: Hybrid (RECOMMENDED)**

**Approach**: Individual columns for common queries + JSONB blob for comprehensive metadata.

```python
class ImageAsset(Base):
    # ... existing fields ...

    # Extracted EXIF (queryable)
    taken_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    exif_orientation: Mapped[int | None] = mapped_column(Integer, nullable=True)
    gps_latitude: Mapped[float | None] = mapped_column(Float, nullable=True)
    gps_longitude: Mapped[float | None] = mapped_column(Float, nullable=True)
    camera_make: Mapped[str | None] = mapped_column(String(100), nullable=True)
    camera_model: Mapped[str | None] = mapped_column(String(100), nullable=True)

    # Full EXIF blob (debugging/extensibility)
    exif_json: Mapped[dict[str, object] | None] = mapped_column(JSONB, nullable=True)
```

**Benefits**:
- ✅ Fast queries on common fields (`taken_at`, `camera_model`)
- ✅ Full metadata preserved in `exif_json`
- ✅ Type-safe API responses
- ✅ Graceful degradation (if EXIF missing, columns are NULL)

**Trade-off**: Slight storage overhead (columns + JSON blob), but worth it for query performance.

### 3.2 Person Model Enhancement

**Current**:
```python
class Person(Base):
    id: Mapped[uuid.UUID]
    name: Mapped[str]
    status: Mapped[PersonStatus]
    created_at: Mapped[datetime]
    updated_at: Mapped[datetime]
```

**Proposed**:
```python
class Person(Base):
    # ... existing fields ...

    # Birth date for age calculation
    birth_date: Mapped[date | None] = mapped_column(Date, nullable=True)
    birth_year: Mapped[int | None] = mapped_column(Integer, nullable=True)  # Privacy option

    # Computed age ranges (cached for performance)
    earliest_photo_date: Mapped[date | None] = mapped_column(Date, nullable=True)
    latest_photo_date: Mapped[date | None] = mapped_column(Date, nullable=True)
```

**Usage**:
```python
# Calculate age at photo time
photo_date = asset.taken_at
person_age = (photo_date - person.birth_date).days / 365.25  # Fractional years

# Temporal bucket assignment
if person_age < 3:
    era = AgeEraBucket.INFANT
elif person_age < 12:
    era = AgeEraBucket.CHILD
# etc.
```

**Privacy Consideration**: `birth_year` allows "Age 8" display without revealing exact birthdate (Dec 31 vs Jan 1).

### 3.3 Qdrant Payload Design

**Image Assets Collection**:

```python
# When upserting vector to Qdrant
payload = {
    "asset_id": str(asset.id),
    "path": asset.path,
    "taken_at": asset.taken_at.isoformat() if asset.taken_at else None,
    "year": asset.taken_at.year if asset.taken_at else None,
    "month": asset.taken_at.month if asset.taken_at else None,
    "camera_model": asset.camera_model,
    "has_gps": bool(asset.gps_latitude and asset.gps_longitude),
    "gps_lat": asset.gps_latitude,
    "gps_lng": asset.gps_longitude,
}

# Payload indexes for filtering
client.create_payload_index("image_assets", "taken_at", PayloadSchemaType.DATETIME)
client.create_payload_index("image_assets", "year", PayloadSchemaType.INTEGER)
client.create_payload_index("image_assets", "camera_model", PayloadSchemaType.KEYWORD)
```

**Search Filters**:
```python
# Date range filter
from qdrant_client.models import Filter, FieldCondition, Range

filter = Filter(
    must=[
        FieldCondition(
            key="year",
            range=Range(gte=2015, lte=2020)
        )
    ]
)

# Search "beach photos from 2015-2020"
results = qdrant.search(
    collection_name="image_assets",
    query_vector=text_embedding("beach vacation"),
    query_filter=filter,
    limit=50
)
```

**Trade-off**: Duplicates Postgres data in Qdrant, but enables fast vector+filter queries without DB joins.

---

## 4. Architecture Recommendations

### 4.1 Extraction Service Design

**New Service**: `src/image_search_service/services/exif_service.py`

```python
"""EXIF metadata extraction service."""

from datetime import datetime, timezone
from pathlib import Path
from PIL import Image
from PIL.ExifTags import TAGS, GPSTAGS
from typing import TypedDict, Optional
import logging

logger = logging.getLogger(__name__)


class ExifData(TypedDict, total=False):
    """Structured EXIF data with type hints."""
    taken_at: datetime | None
    orientation: int | None
    gps_latitude: float | None
    gps_longitude: float | None
    gps_altitude: float | None
    camera_make: str | None
    camera_model: str | None
    lens_model: str | None
    focal_length: float | None
    f_number: float | None
    iso: int | None
    exposure_time: str | None
    full_exif: dict[str, object]  # Complete EXIF for JSONB


class ExifService:
    """Service for extracting EXIF metadata from images."""

    @staticmethod
    def extract_exif(image_path: str) -> ExifData:
        """Extract EXIF metadata from an image file.

        Args:
            image_path: Path to image file

        Returns:
            Structured EXIF data dictionary

        Raises:
            FileNotFoundError: If image doesn't exist
            IOError: If image can't be read
        """
        path = Path(image_path)
        if not path.exists():
            raise FileNotFoundError(f"Image not found: {image_path}")

        try:
            with Image.open(path) as img:
                exif_dict = img.getexif()
                if not exif_dict:
                    logger.debug(f"No EXIF data in {image_path}")
                    return {"full_exif": {}}

                # Extract common fields
                result = ExifService._parse_exif_dict(exif_dict)

                # Store full EXIF for JSONB
                result["full_exif"] = {
                    TAGS.get(k, k): v for k, v in exif_dict.items()
                }

                return result

        except Exception as e:
            logger.error(f"EXIF extraction failed for {image_path}: {e}")
            return {"full_exif": {}}

    @staticmethod
    def _parse_exif_dict(exif: dict) -> ExifData:
        """Parse EXIF dictionary into structured data."""
        result: ExifData = {}

        # Date/Time (tag 36867: DateTimeOriginal)
        dt_original = exif.get(36867)  # DateTimeOriginal
        if dt_original:
            result["taken_at"] = ExifService._parse_exif_datetime(dt_original)

        # Orientation (tag 274)
        orientation = exif.get(274)
        if orientation:
            result["orientation"] = int(orientation)

        # Camera info
        result["camera_make"] = exif.get(271)  # Make
        result["camera_model"] = exif.get(272)  # Model
        result["lens_model"] = exif.get(42036)  # LensModel

        # Exposure settings
        focal_length = exif.get(37386)  # FocalLength
        if focal_length:
            result["focal_length"] = float(focal_length)

        f_number = exif.get(33437)  # FNumber
        if f_number:
            result["f_number"] = float(f_number)

        iso = exif.get(34855)  # ISOSpeedRatings
        if iso:
            result["iso"] = int(iso)

        exposure_time = exif.get(33434)  # ExposureTime
        if exposure_time:
            result["exposure_time"] = str(exposure_time)

        # GPS data (tag 34853: GPSInfo)
        gps_info = exif.get(34853)
        if gps_info:
            gps_data = ExifService._parse_gps(gps_info)
            result.update(gps_data)

        return result

    @staticmethod
    def _parse_exif_datetime(dt_str: str) -> datetime | None:
        """Parse EXIF datetime string to timezone-aware datetime.

        EXIF format: "YYYY:MM:DD HH:MM:SS"
        """
        try:
            # Replace colons in date part with dashes
            dt_str = dt_str.replace(":", "-", 2)
            naive_dt = datetime.strptime(dt_str, "%Y-%m-%d %H:%M:%S")
            # Assume UTC (photos don't store timezone)
            return naive_dt.replace(tzinfo=timezone.utc)
        except ValueError:
            logger.warning(f"Invalid EXIF datetime: {dt_str}")
            return None

    @staticmethod
    def _parse_gps(gps_info: dict) -> dict:
        """Parse GPS coordinates from EXIF GPS info.

        Returns:
            Dict with gps_latitude, gps_longitude, gps_altitude
        """
        result = {}

        # GPS Latitude
        lat = gps_info.get(2)  # GPSLatitude
        lat_ref = gps_info.get(1)  # GPSLatitudeRef ('N' or 'S')
        if lat and lat_ref:
            lat_decimal = ExifService._dms_to_decimal(lat, lat_ref)
            if lat_decimal is not None:
                result["gps_latitude"] = lat_decimal

        # GPS Longitude
        lng = gps_info.get(4)  # GPSLongitude
        lng_ref = gps_info.get(3)  # GPSLongitudeRef ('E' or 'W')
        if lng and lng_ref:
            lng_decimal = ExifService._dms_to_decimal(lng, lng_ref)
            if lng_decimal is not None:
                result["gps_longitude"] = lng_decimal

        # GPS Altitude
        alt = gps_info.get(6)  # GPSAltitude
        if alt:
            result["gps_altitude"] = float(alt)

        return result

    @staticmethod
    def _dms_to_decimal(dms: tuple, ref: str) -> float | None:
        """Convert GPS degrees/minutes/seconds to decimal degrees.

        Args:
            dms: Tuple of (degrees, minutes, seconds)
            ref: Reference ('N', 'S', 'E', 'W')

        Returns:
            Decimal degrees (negative for S/W)
        """
        try:
            degrees, minutes, seconds = dms
            decimal = float(degrees) + float(minutes) / 60 + float(seconds) / 3600

            # South and West are negative
            if ref in ('S', 'W'):
                decimal = -decimal

            return decimal
        except (ValueError, TypeError, ZeroDivisionError):
            return None
```

**Usage in Background Job**:

```python
# In queue/jobs.py or services/training_service.py
from image_search_service.services.exif_service import ExifService

async def process_image_asset(asset_id: int):
    """Process image asset: extract EXIF, generate embedding, create thumbnail."""
    async with get_db() as db:
        asset = await db.get(ImageAsset, asset_id)

        # Extract EXIF metadata
        exif_service = ExifService()
        try:
            exif_data = exif_service.extract_exif(asset.path)

            # Update asset with EXIF data
            asset.taken_at = exif_data.get("taken_at")
            asset.exif_orientation = exif_data.get("orientation")
            asset.gps_latitude = exif_data.get("gps_latitude")
            asset.gps_longitude = exif_data.get("gps_longitude")
            asset.camera_make = exif_data.get("camera_make")
            asset.camera_model = exif_data.get("camera_model")
            asset.lens_model = exif_data.get("lens_model")
            asset.focal_length = exif_data.get("focal_length")
            asset.f_number = exif_data.get("f_number")
            asset.iso = exif_data.get("iso")
            asset.exposure_time = exif_data.get("exposure_time")
            asset.exif_json = exif_data.get("full_exif", {})

            await db.commit()
            logger.info(f"Extracted EXIF for asset {asset_id}")

        except Exception as e:
            logger.error(f"EXIF extraction failed for asset {asset_id}: {e}")
            # Continue processing (EXIF is optional)

        # Continue with embedding generation, thumbnail, etc.
```

### 4.2 API Integration

**Pydantic Schema Update** (`api/schemas.py`):

```python
from datetime import datetime
from pydantic import BaseModel, Field
from typing import Optional

class ExifMetadata(BaseModel):
    """EXIF metadata for image asset."""
    taken_at: datetime | None = Field(None, alias="takenAt")
    orientation: int | None = None
    camera_make: str | None = Field(None, alias="cameraMake")
    camera_model: str | None = Field(None, alias="cameraModel")
    lens_model: str | None = Field(None, alias="lensModel")
    focal_length: float | None = Field(None, alias="focalLength")
    f_number: float | None = Field(None, alias="fNumber")
    iso: int | None = None
    exposure_time: str | None = Field(None, alias="exposureTime")

    class Config:
        populate_by_name = True


class LocationMetadata(BaseModel):
    """GPS location metadata."""
    latitude: float = Field(alias="lat")
    longitude: float = Field(alias="lng")
    altitude: float | None = Field(None, alias="alt")

    class Config:
        populate_by_name = True


class Asset(BaseModel):
    """Enhanced Asset response schema with EXIF data."""
    model_config = ConfigDict(populate_by_name=True, from_attributes=True)

    id: int
    path: str
    created_at: datetime = Field(alias="createdAt")
    indexed_at: datetime | None = Field(None, alias="indexedAt")

    # New EXIF fields
    taken_at: datetime | None = Field(None, alias="takenAt")
    exif: ExifMetadata | None = None
    location: LocationMetadata | None = None

    @computed_field(alias="url")
    def url(self) -> str:
        return f"/api/v1/images/{self.id}/full"

    @computed_field(alias="thumbnailUrl")
    def thumbnail_url(self) -> str:
        return f"/api/v1/images/{self.id}/thumbnail"

    @computed_field(alias="filename")
    def filename(self) -> str:
        return self.path.split("/")[-1]

    @classmethod
    def from_db_model(cls, asset: ImageAsset) -> "Asset":
        """Construct Asset from ImageAsset DB model."""
        exif = None
        if asset.camera_make or asset.camera_model:
            exif = ExifMetadata(
                takenAt=asset.taken_at,
                orientation=asset.exif_orientation,
                cameraMake=asset.camera_make,
                cameraModel=asset.camera_model,
                lensModel=asset.lens_model,
                focalLength=asset.focal_length,
                fNumber=asset.f_number,
                iso=asset.iso,
                exposureTime=asset.exposure_time,
            )

        location = None
        if asset.gps_latitude and asset.gps_longitude:
            location = LocationMetadata(
                lat=asset.gps_latitude,
                lng=asset.gps_longitude,
                alt=asset.gps_altitude,
            )

        return cls(
            id=asset.id,
            path=asset.path,
            createdAt=asset.created_at,
            indexedAt=asset.indexed_at,
            takenAt=asset.taken_at,
            exif=exif,
            location=location,
        )
```

**OpenAPI Spec Change**: Frontend will regenerate types with:
```typescript
interface Asset {
  id: number;
  path: string;
  createdAt: string;
  takenAt?: string | null;  // NEW
  exif?: ExifMetadata | null;  // NEW
  location?: LocationMetadata | null;  // NEW
  url: string;
  thumbnailUrl: string;
}
```

### 4.3 Search Endpoint Enhancement

**Add Date Range Filters** (`api/routes/search.py`):

```python
class SearchRequest(BaseModel):
    query: str
    limit: int = 50
    offset: int = 0
    filters: SearchFilters | None = None  # NEW

class SearchFilters(BaseModel):
    """Search filters for image assets."""
    from_date: str | None = Field(None, alias="fromDate")  # ISO 8601: "2015-01-01"
    to_date: str | None = Field(None, alias="toDate")
    camera_model: str | None = Field(None, alias="cameraModel")
    has_gps: bool | None = Field(None, alias="hasGps")
    category_id: int | None = Field(None, alias="categoryId")

@router.post("/search", response_model=SearchResponse)
async def search_images(request: SearchRequest, db: AsyncSession = Depends(get_db)):
    # Build Qdrant filter from request.filters
    filter_conditions = []

    if request.filters:
        if request.filters.from_date:
            filter_conditions.append(
                FieldCondition(
                    key="year",
                    range=Range(gte=datetime.fromisoformat(request.filters.from_date).year)
                )
            )
        if request.filters.to_date:
            filter_conditions.append(
                FieldCondition(
                    key="year",
                    range=Range(lte=datetime.fromisoformat(request.filters.to_date).year)
                )
            )
        if request.filters.camera_model:
            filter_conditions.append(
                FieldCondition(key="camera_model", match={"value": request.filters.camera_model})
            )

    query_filter = Filter(must=filter_conditions) if filter_conditions else None

    # Search Qdrant with filters
    results = await qdrant.search(
        collection_name="image_assets",
        query_vector=await embed_text(request.query),
        query_filter=query_filter,
        limit=request.limit,
        offset=request.offset,
    )

    # ... rest of search logic
```

---

## 5. Migration Strategy (Backfilling Existing Images)

### 5.1 Backfill Script Design

**New Script**: `scripts/backfill_exif.py`

```python
"""Backfill EXIF metadata for existing image assets."""

import asyncio
from sqlalchemy import select
from tqdm import tqdm

from image_search_service.db.session import get_async_session
from image_search_service.db.models import ImageAsset
from image_search_service.services.exif_service import ExifService


async def backfill_exif(batch_size: int = 100, limit: int | None = None):
    """Extract EXIF data for all assets missing taken_at timestamp.

    Args:
        batch_size: Number of assets to process per batch
        limit: Optional limit on number of assets to process
    """
    exif_service = ExifService()
    processed = 0
    updated = 0
    failed = 0

    async for db in get_async_session():
        # Query assets missing taken_at
        query = select(ImageAsset).where(ImageAsset.taken_at.is_(None))
        if limit:
            query = query.limit(limit)

        result = await db.execute(query)
        assets = result.scalars().all()

        print(f"Found {len(assets)} assets without EXIF data")

        for asset in tqdm(assets, desc="Extracting EXIF"):
            try:
                exif_data = exif_service.extract_exif(asset.path)

                # Update asset
                asset.taken_at = exif_data.get("taken_at")
                asset.exif_orientation = exif_data.get("orientation")
                asset.gps_latitude = exif_data.get("gps_latitude")
                asset.gps_longitude = exif_data.get("gps_longitude")
                asset.camera_make = exif_data.get("camera_make")
                asset.camera_model = exif_data.get("camera_model")
                asset.lens_model = exif_data.get("lens_model")
                asset.focal_length = exif_data.get("focal_length")
                asset.f_number = exif_data.get("f_number")
                asset.iso = exif_data.get("iso")
                asset.exposure_time = exif_data.get("exposure_time")
                asset.exif_json = exif_data.get("full_exif", {})

                if exif_data.get("taken_at"):
                    updated += 1

                processed += 1

                # Commit in batches
                if processed % batch_size == 0:
                    await db.commit()
                    print(f"Processed {processed}, updated {updated}, failed {failed}")

            except FileNotFoundError:
                failed += 1
                print(f"File not found: {asset.path}")
            except Exception as e:
                failed += 1
                print(f"Error processing asset {asset.id}: {e}")

        # Final commit
        await db.commit()

    print(f"\nBackfill complete:")
    print(f"  Processed: {processed}")
    print(f"  Updated: {updated}")
    print(f"  Failed: {failed}")


if __name__ == "__main__":
    asyncio.run(backfill_exif())
```

**Makefile Target**:

```makefile
# Add to image-search-service/Makefile
.PHONY: backfill-exif
backfill-exif:  ## Backfill EXIF metadata for existing images
	@echo "Backfilling EXIF data..."
	uv run python -m scripts.backfill_exif
```

**Usage**:
```bash
cd image-search-service
make backfill-exif
```

### 5.2 Qdrant Payload Update

**Separate Script**: `scripts/sync_exif_to_qdrant.py`

```python
"""Sync EXIF metadata from Postgres to Qdrant payloads."""

async def sync_exif_to_qdrant(batch_size: int = 1000):
    """Update Qdrant payloads with EXIF data from Postgres."""
    qdrant = QdrantClient(url=settings.qdrant_url)

    async for db in get_async_session():
        # Get all assets with EXIF data
        query = select(ImageAsset).where(ImageAsset.taken_at.isnot(None))
        result = await db.execute(query)
        assets = result.scalars().all()

        print(f"Syncing {len(assets)} assets to Qdrant")

        for asset in tqdm(assets):
            try:
                # Update Qdrant payload
                qdrant.set_payload(
                    collection_name="image_assets",
                    payload={
                        "taken_at": asset.taken_at.isoformat(),
                        "year": asset.taken_at.year,
                        "month": asset.taken_at.month,
                        "camera_model": asset.camera_model,
                        "has_gps": bool(asset.gps_latitude and asset.gps_longitude),
                    },
                    points=[str(asset.id)]  # Qdrant point ID = asset_id
                )
            except Exception as e:
                print(f"Failed to update asset {asset.id}: {e}")

    print("Qdrant sync complete")
```

**Two-Phase Migration**:
1. **Phase 1**: `backfill_exif.py` - Extract EXIF to Postgres
2. **Phase 2**: `sync_exif_to_qdrant.py` - Copy to Qdrant payloads

### 5.3 Incremental Backfill (Safe for Production)

**Design**: Process images in batches over multiple days.

```python
# Add to backfill_exif.py
async def backfill_exif_incremental(
    batch_size: int = 100,
    per_run_limit: int = 1000,
    checkpoint_file: str = ".exif_backfill_checkpoint"
):
    """Incremental EXIF backfill with checkpointing.

    Args:
        batch_size: Assets per transaction
        per_run_limit: Max assets per execution
        checkpoint_file: File to store last processed asset ID
    """
    # Load checkpoint
    last_id = 0
    if Path(checkpoint_file).exists():
        with open(checkpoint_file) as f:
            last_id = int(f.read().strip())

    print(f"Resuming from asset ID {last_id}")

    # Query assets after checkpoint
    query = (
        select(ImageAsset)
        .where(ImageAsset.id > last_id)
        .where(ImageAsset.taken_at.is_(None))
        .order_by(ImageAsset.id)
        .limit(per_run_limit)
    )

    # ... process batch ...

    # Save checkpoint
    with open(checkpoint_file, "w") as f:
        f.write(str(max_id))

    print(f"Checkpoint saved: {max_id}")
```

**Cron Job** (run daily until complete):
```bash
# /etc/cron.d/exif-backfill
0 2 * * * cd /app/image-search-service && make backfill-exif-incremental
```

---

## 6. Implementation Phases (Prioritized)

### **Phase 1: Core Date Support** 🔴 HIGH PRIORITY

**Objective**: Enable photo sorting by capture date and improve temporal accuracy.

**Tasks**:
1. **Database Migration**:
   - Add `taken_at TIMESTAMP WITH TIME ZONE` to `image_assets`
   - Add `exif_orientation INTEGER` (already used, just not stored)
   - Add `exif_json JSONB` for full metadata
   - Create index on `taken_at`

2. **EXIF Service**:
   - Implement `ExifService.extract_exif()`
   - Focus on `DateTimeOriginal` extraction
   - Handle missing EXIF gracefully (return NULL)

3. **Background Job Integration**:
   - Add EXIF extraction to image processing pipeline
   - Extract EXIF before embedding generation
   - Log extraction failures (don't crash job)

4. **Backfill Script**:
   - Create `backfill_exif.py`
   - Test on 100 images first
   - Run full backfill (checkpoint-based for safety)

5. **API Schema Update**:
   - Add `taken_at` to `Asset` Pydantic model
   - Update OpenAPI spec
   - Frontend regenerates types (`npm run gen:api`)

6. **Frontend Display**:
   - Show `takenAt` in PhotoPreviewModal
   - Format: "December 15, 2018 at 2:32 PM"
   - Fall back to `createdAt` if `takenAt` is NULL

**Acceptance Criteria**:
- ✅ 90%+ of photos have `taken_at` populated
- ✅ API returns `takenAt` in Asset responses
- ✅ UI displays capture date in photo details
- ✅ Photos sortable by `taken_at DESC`

**Estimated Effort**: 2-3 days

---

### **Phase 2: Person Birth Dates + Age Calculation** 🟡 MEDIUM PRIORITY

**Objective**: Enable accurate age display ("Age 8" instead of "Child 4-12").

**Tasks**:
1. **Person Model Migration**:
   - Add `birth_date DATE` to `persons`
   - Add `birth_year INTEGER` (privacy option)
   - Nullable (not all persons have known birthdays)

2. **Person API Enhancement**:
   - Add `birthDate` to PersonDetail schema
   - Create `PATCH /api/v1/people/{person_id}` endpoint
   - Allow setting/updating birth date

3. **Age Calculation Service**:
   ```python
   def calculate_age_at_photo(
       photo_date: datetime,
       birth_date: date
   ) -> float:
       """Calculate person's age at time of photo."""
       delta = photo_date.date() - birth_date
       return delta.days / 365.25  # Fractional years
   ```

4. **Temporal Prototype Enhancement**:
   - When `taken_at` and `birth_date` both available:
     - Calculate exact age
     - Store in `PersonPrototype.age_at_photo` (new field)
     - Display "Age 8 (2015)" instead of "Child (4-12)"

5. **UI Updates**:
   - Add DOB field to Person edit form
   - Show "Age X" in photo metadata when available
   - Timeline shows "2010 (Age 5)" instead of just "2010"

**Acceptance Criteria**:
- ✅ Person model supports birth dates
- ✅ Age calculation accurate to fractional years
- ✅ Temporal timeline shows exact ages when DOB known
- ✅ UI allows setting DOB for known persons

**Estimated Effort**: 2 days

---

### **Phase 3: Camera Metadata + Filtering** 🟢 LOW PRIORITY

**Objective**: Support camera-based filtering and statistics.

**Tasks**:
1. **Database Migration**:
   - Add camera fields to `image_assets`:
     - `camera_make VARCHAR(100)`
     - `camera_model VARCHAR(100)`
     - `lens_model VARCHAR(100)`
     - `focal_length FLOAT`
     - `f_number FLOAT`
     - `iso INTEGER`
     - `exposure_time VARCHAR(20)`

2. **EXIF Service Enhancement**:
   - Extract camera/lens info from tags
   - Extract exposure settings

3. **Search Filter**:
   - Add `cameraModel` to SearchFilters
   - Implement Qdrant filter by camera

4. **Statistics Endpoint**:
   ```python
   GET /api/v1/stats/cameras
   Response: {
     "cameras": [
       {"make": "Apple", "model": "iPhone 14 Pro", "count": 1234},
       {"make": "Canon", "model": "EOS 5D Mark IV", "count": 567}
     ]
   }
   ```

5. **UI Enhancements**:
   - Camera filter dropdown in search
   - Photo detail shows camera/lens info
   - Statistics dashboard: "Top Cameras"

**Acceptance Criteria**:
- ✅ Camera metadata extracted and stored
- ✅ Users can filter by camera model
- ✅ Statistics show camera usage breakdown

**Estimated Effort**: 2 days

---

### **Phase 4: GPS Location Support** 🟢 LOW PRIORITY

**Objective**: Enable location-based search and map view.

**Tasks**:
1. **Database Migration**:
   - Add GPS fields to `image_assets`:
     - `gps_latitude FLOAT`
     - `gps_longitude FLOAT`
     - `gps_altitude FLOAT`
   - Create spatial index (PostGIS if needed)

2. **EXIF Service Enhancement**:
   - Extract GPS coordinates
   - Convert DMS to decimal degrees

3. **Reverse Geocoding** (Optional):
   - Use OpenStreetMap Nominatim API
   - Convert lat/lng to "Paris, France"
   - Store in `location_name` field

4. **Location Search**:
   ```python
   GET /api/v1/search?near_lat=48.8566&near_lng=2.3522&radius_km=10
   ```

5. **Map View UI** (Big Feature):
   - New page: `/photos/map`
   - Leaflet.js map component
   - Cluster markers for nearby photos
   - Click marker → show photos at location

**Privacy Considerations**:
- ⚠️ GPS data is sensitive (home address leak risk)
- Add opt-in setting: "Extract GPS data from photos"
- Provide "Strip GPS" utility for sharing photos

**Acceptance Criteria**:
- ✅ GPS coordinates extracted and stored
- ✅ Location-based search works
- ✅ Map view shows photo locations
- ✅ Privacy controls in place

**Estimated Effort**: 4-5 days (map UI is complex)

---

### **Phase 5: Advanced Date Range Filters** 🟢 ENHANCEMENT

**Objective**: Fine-grained date filtering for search.

**Tasks**:
1. **Qdrant Payload Enhancement**:
   - Add `year`, `month`, `day` fields to payload
   - Create indexes on date fields

2. **Search Filter API**:
   ```python
   class SearchFilters(BaseModel):
       from_date: str | None  # "2015-06-01"
       to_date: str | None    # "2020-12-31"
       year: int | None       # 2018
       month: int | None      # 6 (June)
       season: str | None     # "summer" (Jun-Aug)
   ```

3. **Season Detection**:
   ```python
   def get_season(month: int) -> str:
       if month in [12, 1, 2]: return "winter"
       if month in [3, 4, 5]: return "spring"
       if month in [6, 7, 8]: return "summer"
       return "fall"
   ```

4. **UI Filter Controls**:
   - Date range picker (calendar component)
   - Year dropdown
   - Season filter buttons

**Acceptance Criteria**:
- ✅ Date range filters work in search
- ✅ Season-based filtering works
- ✅ UI provides intuitive date selection

**Estimated Effort**: 2 days

---

## 7. API Contract Changes

### 7.1 New Endpoints

**None required** - Existing endpoints enhanced with new response fields.

### 7.2 Modified Endpoints

#### **GET /api/v1/search** (Enhanced)

**Request**:
```json
{
  "query": "beach vacation",
  "limit": 50,
  "offset": 0,
  "filters": {
    "fromDate": "2015-01-01",
    "toDate": "2020-12-31",
    "cameraModel": "iPhone 14 Pro",
    "hasGps": true
  }
}
```

**Response** (Asset schema enhanced):
```json
{
  "results": [
    {
      "asset": {
        "id": 123,
        "path": "/photos/IMG_1234.jpg",
        "createdAt": "2025-12-01T00:00:00Z",
        "takenAt": "2018-06-15T14:32:10Z",  // NEW
        "exif": {                            // NEW
          "cameraMake": "Apple",
          "cameraModel": "iPhone 14 Pro",
          "lensModel": null,
          "focalLength": 6.86,
          "fNumber": 1.8,
          "iso": 64,
          "exposureTime": "1/250"
        },
        "location": {                        // NEW
          "lat": 48.8566,
          "lng": 2.3522,
          "alt": 35.0
        },
        "url": "/api/v1/images/123/full",
        "thumbnailUrl": "/api/v1/images/123/thumbnail",
        "filename": "IMG_1234.jpg"
      },
      "score": 0.89
    }
  ],
  "total": 42,
  "query": "beach vacation"
}
```

#### **GET /api/v1/people/{person_id}** (Enhanced)

**Response**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "John Doe",
  "status": "active",
  "birthDate": "2010-03-15",  // NEW
  "birthYear": 2010,          // NEW (privacy option)
  "photoCount": 234,
  "faceCount": 456,
  "createdAt": "2025-01-01T00:00:00Z",
  "updatedAt": "2025-12-15T10:30:00Z"
}
```

#### **PATCH /api/v1/people/{person_id}** (New Operation)

**Request**:
```json
{
  "name": "John Doe",
  "birthDate": "2010-03-15"  // NEW
}
```

**Response**: Same as GET (PersonDetail)

### 7.3 Frontend Type Regeneration

After backend changes deployed:

```bash
cd image-search-ui
npm run gen:api  # Regenerates src/lib/api/generated.ts from OpenAPI spec
```

**Generated Types**:
```typescript
// src/lib/api/generated.ts (auto-generated)
export interface Asset {
  id: number;
  path: string;
  createdAt: string;
  indexedAt?: string | null;
  takenAt?: string | null;  // NEW
  exif?: ExifMetadata | null;  // NEW
  location?: LocationMetadata | null;  // NEW
  url: string;
  thumbnailUrl: string;
  filename: string;
}

export interface ExifMetadata {
  takenAt?: string | null;
  orientation?: number | null;
  cameraMake?: string | null;
  cameraModel?: string | null;
  lensModel?: string | null;
  focalLength?: number | null;
  fNumber?: number | null;
  iso?: number | null;
  exposureTime?: string | null;
}

export interface LocationMetadata {
  lat: number;
  lng: number;
  alt?: number | null;
}

export interface PersonDetail {
  id: string;
  name: string;
  status: PersonStatus;
  birthDate?: string | null;  // NEW
  birthYear?: number | null;  // NEW
  photoCount: number;
  faceCount: number;
  createdAt: string;
  updatedAt: string;
}
```

---

## 8. UI Integration Points

### 8.1 PhotoPreviewModal Enhancement

**File**: `image-search-ui/src/lib/components/faces/PhotoPreviewModal.svelte`

**Current Display**:
```svelte
<div class="photo-metadata">
  <p>Asset ID: {photo.assetId}</p>
  <p>Faces: {photo.faces.length}</p>
</div>
```

**Enhanced Display**:
```svelte
<script lang="ts">
  import { formatDate, formatLocation } from '$lib/utils/format';

  let { photo, ... }: Props = $props();

  // Derived metadata
  let captureInfo = $derived(() => {
    if (!photo.asset.takenAt) return null;
    return {
      date: formatDate(photo.asset.takenAt, 'long'),  // "December 15, 2018"
      time: formatDate(photo.asset.takenAt, 'time'),  // "2:32 PM"
    };
  });

  let cameraInfo = $derived(() => {
    const exif = photo.asset.exif;
    if (!exif?.cameraModel) return null;
    return {
      camera: `${exif.cameraMake} ${exif.cameraModel}`,
      settings: `f/${exif.fNumber} • ${exif.exposureTime}s • ISO ${exif.iso}`,
    };
  });

  let locationInfo = $derived(() => {
    if (!photo.asset.location) return null;
    return {
      coords: `${photo.asset.location.lat.toFixed(4)}, ${photo.asset.location.lng.toFixed(4)}`,
      // Future: reverse geocode to "Paris, France"
    };
  });
</script>

<div class="photo-metadata">
  {#if captureInfo}
    <div class="metadata-row">
      <span class="label">📅 Taken:</span>
      <span class="value">{captureInfo.date} at {captureInfo.time}</span>
    </div>
  {/if}

  {#if cameraInfo}
    <div class="metadata-row">
      <span class="label">📷 Camera:</span>
      <span class="value">{cameraInfo.camera}</span>
    </div>
    <div class="metadata-row">
      <span class="label">⚙️ Settings:</span>
      <span class="value">{cameraInfo.settings}</span>
    </div>
  {/if}

  {#if locationInfo}
    <div class="metadata-row">
      <span class="label">📍 Location:</span>
      <span class="value">{locationInfo.coords}</span>
      <button onclick={() => openMap(photo.asset.location)}>View Map</button>
    </div>
  {/if}

  <div class="metadata-row">
    <span class="label">👤 Faces:</span>
    <span class="value">{photo.faces.length}</span>
  </div>
</div>

<style>
  .metadata-row {
    display: flex;
    gap: 0.5rem;
    padding: 0.25rem 0;
  }
  .label {
    font-weight: 600;
    min-width: 100px;
  }
  .value {
    color: #555;
  }
</style>
```

### 8.2 TemporalTimeline with Exact Ages

**File**: `image-search-ui/src/lib/components/faces/TemporalTimeline.svelte`

**Current**:
```svelte
{#each eras as era}
  <div class="era-slot">
    <span class="era-name">{era.label}</span>
    <span class="era-range">{era.range} yrs</span>
  </div>
{/each}
```

**Enhanced** (with birth date):
```svelte
<script lang="ts">
  interface Props {
    prototypes: Prototype[];
    coverage: TemporalCoverage;
    person: PersonDetail;  // NEW: Include person for DOB
  }

  let { prototypes, coverage, person }: Props = $props();

  // Calculate age ranges from prototypes
  let ageBasedTimeline = $derived.by(() => {
    if (!person.birthDate) {
      // Fallback to era buckets
      return standardEras;
    }

    // Group prototypes by actual age
    const birthDate = new Date(person.birthDate);
    const ageGroups = prototypes.map(proto => {
      const photoDate = new Date(proto.takenAt);  // Assume prototype has takenAt
      const ageYears = (photoDate - birthDate) / (365.25 * 24 * 60 * 60 * 1000);
      return {
        age: Math.floor(ageYears),
        year: photoDate.getFullYear(),
        proto,
      };
    });

    return ageGroups;
  });
</script>

<div class="temporal-timeline">
  {#if person.birthDate}
    <!-- Exact age timeline -->
    {#each ageBasedTimeline as { age, year, proto }}
      <div class="age-slot">
        <span class="age-label">Age {age}</span>
        <span class="year-label">({year})</span>
        <img src={proto.thumbnailUrl} alt="Age {age}" />
      </div>
    {/each}
  {:else}
    <!-- Era-based timeline (current) -->
    {#each eras as era}
      <div class="era-slot">
        <span class="era-name">{era.label}</span>
        <span class="era-range">{era.range} yrs</span>
      </div>
    {/each}
  {/if}
</div>
```

### 8.3 Search Page Date Filters

**File**: `image-search-ui/src/routes/+page.svelte`

**Add Filter Controls**:
```svelte
<script lang="ts">
  import { searchImages } from '$lib/api/client';

  let searchQuery = $state('');
  let dateFrom = $state<string | null>(null);
  let dateTo = $state<string | null>(null);
  let cameraModel = $state<string | null>(null);

  async function handleSearch() {
    const results = await searchImages({
      query: searchQuery,
      limit: 50,
      filters: {
        fromDate: dateFrom,
        toDate: dateTo,
        cameraModel: cameraModel,
      },
    });
    // ... render results
  }
</script>

<div class="search-filters">
  <input type="text" bind:value={searchQuery} placeholder="Search photos..." />

  <div class="date-filters">
    <label>
      From:
      <input type="date" bind:value={dateFrom} />
    </label>
    <label>
      To:
      <input type="date" bind:value={dateTo} />
    </label>
  </div>

  <select bind:value={cameraModel}>
    <option value={null}>All Cameras</option>
    <option value="iPhone 14 Pro">iPhone 14 Pro</option>
    <option value="Canon EOS 5D Mark IV">Canon EOS 5D</option>
  </select>

  <button onclick={handleSearch}>Search</button>
</div>
```

### 8.4 Person Edit Form (Birth Date)

**New Component**: `image-search-ui/src/lib/components/faces/PersonEditForm.svelte`

```svelte
<script lang="ts">
  import type { PersonDetail } from '$lib/api/generated';
  import { updatePerson } from '$lib/api/faces';

  interface Props {
    person: PersonDetail;
    onSave: (updated: PersonDetail) => void;
  }

  let { person, onSave }: Props = $props();

  let name = $state(person.name);
  let birthDate = $state(person.birthDate || '');
  let saving = $state(false);

  async function handleSave() {
    saving = true;
    try {
      const updated = await updatePerson(person.id, {
        name,
        birthDate: birthDate || null,
      });
      onSave(updated);
    } finally {
      saving = false;
    }
  }
</script>

<form onsubmit|preventDefault={handleSave}>
  <label>
    Name:
    <input type="text" bind:value={name} required />
  </label>

  <label>
    Birth Date (optional):
    <input type="date" bind:value={birthDate} />
    <small>Used for exact age calculations in timeline</small>
  </label>

  <button type="submit" disabled={saving}>
    {saving ? 'Saving...' : 'Save'}
  </button>
</form>
```

---

## 9. Performance Considerations

### 9.1 EXIF Extraction Performance

**Benchmarks** (PIL on modern hardware):
- Read EXIF: ~1-2ms per image
- Parse GPS/dates: ~0.5ms
- **Total overhead**: ~2-5ms per image

**Impact on Ingestion**:
- Current: ~200ms per image (embedding + thumbnail)
- With EXIF: ~205ms per image (+2.5% overhead)
- **Negligible impact** on background job throughput

**Optimization**: Batch extraction in parallel:
```python
from concurrent.futures import ThreadPoolExecutor

async def extract_exif_batch(asset_ids: list[int], max_workers: int = 8):
    """Extract EXIF for multiple assets in parallel."""
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = [
            executor.submit(exif_service.extract_exif, asset.path)
            for asset in assets
        ]
        results = [f.result() for f in futures]
    return results
```

### 9.2 Database Query Performance

**Index Strategy**:
```sql
-- Critical indexes for common queries
CREATE INDEX idx_image_assets_taken_at ON image_assets(taken_at) WHERE taken_at IS NOT NULL;
CREATE INDEX idx_image_assets_taken_at_desc ON image_assets(taken_at DESC) WHERE taken_at IS NOT NULL;

-- For date range queries
CREATE INDEX idx_image_assets_taken_at_year ON image_assets(EXTRACT(YEAR FROM taken_at)) WHERE taken_at IS NOT NULL;

-- For location queries (if using PostGIS)
CREATE INDEX idx_image_assets_gps ON image_assets USING GIST(ll_to_earth(gps_latitude, gps_longitude)) WHERE gps_latitude IS NOT NULL;
```

**Query Performance**:
- Date range filter: O(log N) with index (sub-second for 100K+ images)
- Sort by `taken_at`: O(N log N) worst case, but index accelerates
- GPS radius search: O(log N) with spatial index

### 9.3 Qdrant Payload Overhead

**Payload Size**:
```json
{
  "asset_id": "123",              // 10 bytes
  "path": "/photos/IMG_1234.jpg", // 50 bytes
  "taken_at": "2018-06-15T14:32:10Z", // 25 bytes
  "year": 2018,                   // 4 bytes
  "month": 6,                     // 4 bytes
  "camera_model": "iPhone 14 Pro", // 20 bytes
  "has_gps": true,                // 1 byte
  "gps_lat": 48.8566,             // 8 bytes
  "gps_lng": 2.3522               // 8 bytes
}
// Total: ~130 bytes per vector
```

**Impact**:
- 100K images × 130 bytes = 13 MB payload overhead
- Vector data: 100K × 512 dims × 4 bytes = 205 MB
- **Payload is 6% of total storage** (acceptable)

**Filter Performance**:
- Qdrant payload filters are indexed (sub-second for date ranges)
- Combining vector similarity + date filter: ~50-100ms for 100K images

### 9.4 API Response Size

**Current Asset Response**: ~200 bytes
**With EXIF**: ~400 bytes (+100% size)

**Mitigation**:
- Gzip compression (FastAPI default): ~60% size reduction
- Sparse EXIF (only include non-null fields): ~50% reduction
- Pagination (already implemented): Limits response to 50 assets

**Example**:
```
50 assets × 400 bytes = 20 KB (uncompressed)
→ 8 KB (gzipped) → ~100ms transfer on 3G
```

**Verdict**: Acceptable overhead, especially with compression.

---

## 10. Security & Privacy Considerations

### 10.1 GPS Data Privacy

**Risks**:
- 🔴 **Home Address Exposure**: GPS coordinates reveal where user lives (photos taken at home)
- 🟡 **Location History**: Can infer travel patterns and daily routines
- 🟡 **Stalking Risk**: Sharing photos with embedded GPS can enable tracking

**Mitigations**:
1. **Opt-In Extraction**:
   ```python
   # In config.py
   extract_gps: bool = Field(False, env="EXTRACT_GPS_DATA")  # Default: disabled
   ```

2. **GPS Stripping Tool**:
   ```python
   # API endpoint: POST /api/v1/images/{asset_id}/strip-gps
   async def strip_gps(asset_id: int):
       """Remove GPS coordinates from image EXIF."""
       asset = await db.get(ImageAsset, asset_id)
       asset.gps_latitude = None
       asset.gps_longitude = None
       await db.commit()
       return {"status": "gps_stripped"}
   ```

3. **UI Warning**:
   ```svelte
   {#if photo.asset.location}
     <div class="privacy-warning">
       ⚠️ This photo contains GPS coordinates. Be careful when sharing.
       <button onclick={() => stripGps(photo.assetId)}>Remove Location</button>
     </div>
   {/if}
   ```

4. **Export Option**: "Strip EXIF when exporting" checkbox

### 10.2 Birth Date Privacy

**Risks**:
- 🟡 **Identity Theft**: Birth date is used in identity verification
- 🟢 **Age Inference**: Can infer approximate age from photos

**Mitigations**:
1. **Optional Field**: Birth date is nullable (don't require it)
2. **Year-Only Option**: Store `birth_year` instead of full date
   ```python
   # Person model
   birth_date: Mapped[date | None]  # Precise (optional)
   birth_year: Mapped[int | None]   # Coarse (more privacy)
   ```
3. **Access Control**: (Future) Only show birth dates to photo owner

### 10.3 EXIF Data Leakage

**Risk**: Full EXIF blob (`exif_json`) may contain sensitive data:
- Camera serial numbers
- Software versions (security vulnerability detection)
- User comments (may contain personal notes)

**Mitigation**:
1. **Whitelist Fields**: Only expose safe fields in API
   ```python
   # In Asset.from_db_model()
   safe_exif = {
       k: v for k, v in asset.exif_json.items()
       if k in SAFE_EXIF_FIELDS
   }
   ```
2. **Admin-Only Full EXIF**: Regular users see curated metadata, admins see everything

---

## 11. Testing Strategy

### 11.1 Unit Tests

**Test File**: `tests/services/test_exif_service.py`

```python
import pytest
from datetime import datetime, timezone
from pathlib import Path
from PIL import Image

from image_search_service.services.exif_service import ExifService


@pytest.fixture
def test_image_with_exif(tmp_path: Path) -> Path:
    """Create test image with EXIF metadata."""
    img = Image.new("RGB", (800, 600), color="blue")
    img_path = tmp_path / "test_exif.jpg"

    # Set EXIF tags
    exif = img.getexif()
    exif[36867] = "2018:06:15 14:32:10"  # DateTimeOriginal
    exif[274] = 6  # Orientation (rotate 90 CW)
    exif[271] = "Apple"  # Make
    exif[272] = "iPhone 14 Pro"  # Model

    img.save(img_path, exif=exif)
    return img_path


def test_extract_exif_datetime(test_image_with_exif):
    """Test EXIF datetime extraction."""
    service = ExifService()
    exif_data = service.extract_exif(str(test_image_with_exif))

    assert exif_data["taken_at"] == datetime(2018, 6, 15, 14, 32, 10, tzinfo=timezone.utc)


def test_extract_exif_orientation(test_image_with_exif):
    """Test EXIF orientation extraction."""
    service = ExifService()
    exif_data = service.extract_exif(str(test_image_with_exif))

    assert exif_data["orientation"] == 6


def test_extract_exif_camera(test_image_with_exif):
    """Test camera info extraction."""
    service = ExifService()
    exif_data = service.extract_exif(str(test_image_with_exif))

    assert exif_data["camera_make"] == "Apple"
    assert exif_data["camera_model"] == "iPhone 14 Pro"


def test_extract_exif_missing_file():
    """Test handling of missing file."""
    service = ExifService()
    with pytest.raises(FileNotFoundError):
        service.extract_exif("/nonexistent/image.jpg")


def test_extract_exif_no_metadata(tmp_path):
    """Test handling of image without EXIF."""
    img = Image.new("RGB", (800, 600))
    img_path = tmp_path / "no_exif.jpg"
    img.save(img_path)

    service = ExifService()
    exif_data = service.extract_exif(str(img_path))

    assert exif_data["taken_at"] is None
    assert exif_data["camera_make"] is None
```

### 11.2 Integration Tests

**Test File**: `tests/api/test_asset_endpoints.py`

```python
@pytest.mark.asyncio
async def test_asset_response_includes_exif(client, test_db, test_asset_with_exif):
    """Test that asset API responses include EXIF data."""
    response = await client.get(f"/api/v1/images/{test_asset_with_exif.id}")

    assert response.status_code == 200
    data = response.json()

    assert data["takenAt"] is not None
    assert data["exif"] is not None
    assert data["exif"]["cameraModel"] == "iPhone 14 Pro"


@pytest.mark.asyncio
async def test_search_with_date_filter(client, test_db):
    """Test search with date range filter."""
    response = await client.post("/api/v1/search", json={
        "query": "beach",
        "filters": {
            "fromDate": "2015-01-01",
            "toDate": "2020-12-31"
        }
    })

    assert response.status_code == 200
    results = response.json()["results"]

    # Verify all results are within date range
    for result in results:
        taken_at = datetime.fromisoformat(result["asset"]["takenAt"])
        assert 2015 <= taken_at.year <= 2020
```

### 11.3 End-to-End Tests

**Manual Test Plan**:

1. **Upload Image with EXIF**:
   - Use real smartphone photo (with GPS, camera info, orientation)
   - Verify EXIF extracted and stored in DB
   - Check API response includes `takenAt`, `exif`, `location`

2. **Photo Preview**:
   - Open PhotoPreviewModal
   - Verify capture date displays ("December 15, 2018")
   - Verify camera info displays ("Apple iPhone 14 Pro")
   - Verify GPS coordinates display

3. **Search with Filters**:
   - Apply date range filter ("2015-2020")
   - Verify only photos from that range returned
   - Apply camera model filter
   - Verify only iPhone photos returned

4. **Person Birth Date**:
   - Create person with birth date
   - View temporal timeline
   - Verify "Age 8 (2015)" instead of "Child (4-12)"

5. **Backfill Script**:
   - Run `make backfill-exif`
   - Verify 90%+ photos have `taken_at` populated
   - Check logs for errors (missing files, corrupt EXIF)

---

## 12. Risks & Mitigation

### Risk 1: EXIF Data Missing 🟡 MEDIUM

**Risk**: Many photos lack EXIF (screenshots, scanned images, edited photos).

**Impact**: `taken_at` is NULL for 10-30% of photos.

**Mitigation**:
- Fall back to `file_modified_at` for sorting
- UI shows "Date unknown" when `taken_at` is NULL
- Allow manual date entry for photos without EXIF

### Risk 2: GPS Privacy Backlash 🔴 HIGH

**Risk**: Users discover GPS coordinates are stored, feel privacy violated.

**Impact**: User trust loss, potential legal issues (GDPR).

**Mitigation**:
- **Default: GPS extraction disabled** (opt-in only)
- Prominent UI warning when GPS data present
- "Strip GPS" button in photo preview
- Export option: "Remove location data from shared photos"

### Risk 3: Backfill Performance 🟡 MEDIUM

**Risk**: Backfilling 100K+ images takes hours, blocks other operations.

**Impact**: Database load spike, slow API responses during backfill.

**Mitigation**:
- **Incremental backfill** (1000 images/day, checkpoint-based)
- Run during low-traffic hours (cron job at 2 AM)
- Add `LIMIT` to backfill script (process in batches)
- Monitor DB CPU/memory during backfill

### Risk 4: EXIF Parsing Bugs 🟢 LOW

**Risk**: Corrupt EXIF data crashes extraction service.

**Impact**: Background jobs fail, image ingestion blocked.

**Mitigation**:
- Wrap EXIF extraction in try/catch
- Log errors but continue processing (EXIF is optional)
- Unit tests for malformed EXIF data

### Risk 5: Schema Migration Issues 🟡 MEDIUM

**Risk**: Alembic migration fails on production DB with 100K+ rows.

**Impact**: Downtime during migration, rollback required.

**Mitigation**:
- Test migration on production-like dataset (load test)
- Use `ADD COLUMN ... DEFAULT NULL` (instant in Postgres)
- Avoid `NOT NULL` constraints initially (add after backfill)
- Have rollback migration ready

---

## 13. Success Metrics

### Phase 1 Success (Date Support):
- ✅ 90%+ photos have `taken_at` populated
- ✅ API latency <200ms for date-filtered searches
- ✅ Users can sort photos by capture date
- ✅ Temporal timeline shows chronological gaps

### Phase 2 Success (Age Calculation):
- ✅ 50%+ known persons have birth dates
- ✅ Temporal timeline shows exact ages ("Age 8")
- ✅ Age estimation accuracy ±1 year (validated against known DOB)

### Phase 3 Success (Camera Metadata):
- ✅ Camera statistics dashboard shows top 10 cameras
- ✅ Users can filter by camera model
- ✅ Photo detail shows camera/lens/exposure info

### Phase 4 Success (GPS Support):
- ✅ Map view displays photo locations
- ✅ Location-based search works ("photos in Paris")
- ✅ Privacy controls prevent accidental GPS leakage

---

## 14. Recommendations

### Immediate (Phase 1 - Next Sprint):
1. ✅ Implement `ExifService` (datetime + orientation extraction)
2. ✅ Add `taken_at` to `ImageAsset` model
3. ✅ Integrate EXIF extraction into background jobs
4. ✅ Create backfill script with checkpointing
5. ✅ Update API schema to expose `takenAt`
6. ✅ Show capture date in PhotoPreviewModal

**Effort**: 2-3 days
**Impact**: 🔴 HIGH - Fixes temporal accuracy, mentioned in roadmap

### Short-Term (Phase 2 - Next Month):
1. ✅ Add `birth_date` to Person model
2. ✅ Calculate exact ages in temporal timeline
3. ✅ Allow DOB entry in person edit form

**Effort**: 2 days
**Impact**: 🟡 MEDIUM - Enhances face recognition features

### Medium-Term (Phase 3-4 - Next Quarter):
1. ⚪ Extract camera metadata
2. ⚪ Add camera-based filtering
3. ⚪ Extract GPS coordinates (opt-in)
4. ⚪ Build map view UI

**Effort**: 4-5 days
**Impact**: 🟢 LOW - Power user features

### Long-Term (Phase 5 - Future):
1. ⚪ Advanced date range filters (seasons, decades)
2. ⚪ Reverse geocoding (lat/lng → "Paris, France")
3. ⚪ Camera statistics dashboard
4. ⚪ EXIF-based duplicate detection

**Effort**: 1-2 weeks
**Impact**: 🟢 LOW - Nice-to-have enhancements

---

## 15. References

### External Documentation
- [PIL/Pillow EXIF Guide](https://pillow.readthedocs.io/en/stable/reference/ExifTags.html)
- [EXIF Specification](https://exiftool.org/TagNames/EXIF.html)
- [GPS EXIF Tags](https://exiftool.org/TagNames/GPS.html)
- [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [Qdrant Filtering](https://qdrant.tech/documentation/concepts/filtering/)

### Internal Documentation
- `docs/research/face-thumbnail-orientation-issue-2025-12-27.md` - Orientation handling implementation
- `docs/api-contract.md` - API contract (to be updated)
- `README.md` - Roadmap mentions EXIF extraction
- `BLOG_PROJECT_HIGHLIGHTS.md` - Temporal accuracy improvement request

### Code References

**Backend**:
- `src/image_search_service/db/models.py` - Database models
- `src/image_search_service/services/thumbnail_service.py` - Existing orientation handling
- `src/image_search_service/api/schemas.py` - API response schemas
- `src/image_search_service/queue/jobs.py` - Background job processing

**Frontend**:
- `src/lib/components/faces/PhotoPreviewModal.svelte` - Photo metadata display
- `src/lib/components/faces/TemporalTimeline.svelte` - Age timeline component
- `src/lib/api/generated.ts` - Auto-generated API types
- `src/lib/types.ts` - Frontend type definitions

---

## 16. Acceptance Criteria (Phase 1)

**Phase 1 Complete When**:

1. ✅ **Database Schema**:
   - `taken_at TIMESTAMP WITH TIME ZONE` column exists on `image_assets`
   - `exif_orientation INTEGER` column exists
   - `exif_json JSONB` column exists
   - Indexes created on `taken_at`

2. ✅ **EXIF Service**:
   - `ExifService.extract_exif()` successfully extracts `DateTimeOriginal`
   - Handles missing EXIF gracefully (returns NULL)
   - Unit tests pass (datetime, orientation, camera)

3. ✅ **Integration**:
   - Background jobs extract EXIF before embedding generation
   - EXIF extraction failures logged but don't crash jobs
   - Backfill script runs without errors on 100+ image sample

4. ✅ **API**:
   - Asset responses include `takenAt` field
   - `takenAt` is ISO 8601 formatted string
   - Frontend types regenerated (`npm run gen:api`)

5. ✅ **UI**:
   - PhotoPreviewModal displays "Taken: December 15, 2018 at 2:32 PM"
   - Falls back to "Date unknown" when `takenAt` is NULL
   - Search results can be sorted by `takenAt DESC`

6. ✅ **Data Quality**:
   - 90%+ of test images have `taken_at` populated
   - `taken_at` matches actual photo capture date (validated manually on 10 samples)
   - No EXIF extraction errors in logs for valid images

---

**Research Complete** - Ready for implementation.

**Recommended Priority**: 🔴 HIGH - Start with Phase 1 (Date Support) in next sprint.
