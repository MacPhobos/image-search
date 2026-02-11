# Unknown Person Face Detection - General Feature Research

**Date**: 2026-02-07
**Researcher**: Research Agent (Claude Opus 4.6)
**Scope**: Comprehensive analysis of current face recognition system, industry patterns, and design considerations for discovering new/unknown persons from unassigned face clusters

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current System Analysis](#current-system-analysis)
3. [Industry Patterns and Best Practices](#industry-patterns-and-best-practices)
4. [Data Flow Mapping](#data-flow-mapping)
5. [Key Design Decisions and Trade-offs](#key-design-decisions-and-trade-offs)
6. [Recommendations](#recommendations)
7. [Open Questions for the Team](#open-questions-for-the-team)

---

## Executive Summary

The image-search system has a mature face recognition pipeline covering detection, embedding, clustering, and person assignment. However, there is a significant **workflow gap** between the existing "Clusters" view (which groups unknown faces) and the "Face Suggestions" system (which only matches faces to **already-known** persons). The system currently lacks a unified workflow for **discovering and promoting unknown face clusters into new Person entities**.

### Key Findings

1. **Strong Foundation**: The system already has the core building blocks -- 512-dim ArcFace embeddings in Qdrant, HDBSCAN clustering, dual-mode clustering (supervised + unsupervised), a two-tier threshold system, and a rich prototype system with multiple roles (centroid, exemplar, primary, temporal, fallback).

2. **The Gap**: After clustering, unknown faces receive generic `cluster_id` labels (e.g., `unknown_cluster_3`) but are never automatically promoted to `Person` entities. The Clusters UI shows grouped faces but does not provide a streamlined "create person from this cluster" workflow. Face Suggestions only match against existing persons.

3. **Industry Standard**: Google Photos, Apple Photos, and Immich all use multi-pass clustering strategies with conservative first passes, followed by merge/refine iterations with human confirmation. The consensus approach is: high-precision auto-grouping first, then human-in-the-loop for naming and merging.

4. **Recommended Direction**: Implement an "Unknown Person Discovery" pipeline that (a) runs tighter clustering on unassigned faces, (b) surfaces high-confidence clusters as "Suggested New People" in the UI, (c) lets users confirm, name, and merge clusters into Person entities, and (d) feeds confirmed assignments back into the prototype system for incremental learning.

---

## Current System Analysis

### Architecture Overview

The face recognition subsystem spans four layers:

| Layer | Components | Role |
|-------|-----------|------|
| **Detection** | `faces/detector.py`, `faces/service.py` | InsightFace detection + 512-dim ArcFace embedding extraction |
| **Storage** | `db/models.py`, `vector/face_qdrant.py` | PostgreSQL metadata + Qdrant vector storage |
| **Clustering** | `faces/clusterer.py`, `faces/dual_clusterer.py` | HDBSCAN unsupervised + dual-mode (supervised + unsupervised) |
| **Assignment** | `faces/assigner.py`, API routes | Two-tier threshold matching, prototype-based suggestions |

### Database Models (Detailed)

**`Person`** (`db/models.py`)
- UUID primary key, `name`, `status` (active/merged/hidden)
- `birth_date`, `merged_into_id` (self-referential FK for person merging)
- One-to-many relationship with `FaceInstance`, `PersonPrototype`, `PersonCentroid`

**`FaceInstance`** (`db/models.py`)
- UUID primary key, FK to `asset_id` (the photo)
- Bounding box: `bbox_x`, `bbox_y`, `bbox_w`, `bbox_h`
- `landmarks` (JSONB), `detection_confidence`, `quality_score`
- `qdrant_point_id` (links to Qdrant vector)
- `cluster_id` (nullable string) -- assigned by clustering
- `person_id` (nullable FK to `Person`) -- assigned by user or auto-assign
- Key observation: `cluster_id` and `person_id` are independent -- a face can have a cluster_id without a person_id, representing an "unknown grouped face"

**`PersonPrototype`** (`db/models.py`)
- Links a `Person` to specific `FaceInstance` records used as reference faces
- `role` enum: `centroid` | `exemplar` | `primary` | `temporal` | `fallback`
- `age_era_bucket` for temporal diversity, `is_pinned` for user-locked prototypes
- `qdrant_point_id` for direct vector lookup during matching

**`FaceSuggestion`** (`db/models.py`)
- Maps a `face_instance_id` to a `suggested_person_id` with `confidence`
- `source_face_id` -- which prototype triggered the suggestion
- `status`: pending | accepted | rejected | expired
- Multi-prototype scoring: `matching_prototype_ids`, `prototype_scores`, `aggregate_confidence`, `prototype_match_count`
- Key observation: FaceSuggestion **requires** a `suggested_person_id` -- it cannot suggest "this is a new person"

**`PersonCentroid`** (`db/models.py`)
- Precomputed centroid embeddings per person
- `centroid_version` for tracking updates, `centroid_type` (global/cluster)
- `n_faces` count, `source_face_ids_hash` for change detection
- `model_version` for embedding model compatibility

**`FaceAssignmentEvent`** (`db/models.py`)
- Audit log: `operation`, `from_person_id`, `to_person_id`
- Tracks affected `face_ids` and `photo_ids` as JSONB arrays

### Clustering Pipeline (Detailed)

#### `FaceClusterer` (`faces/clusterer.py`)

The primary clustering engine for unlabeled faces:

1. **Input Selection**: Scrolls Qdrant for faces where `person_id` is empty AND `quality_score >= threshold` (default 0.5)
2. **Embedding Collection**: Retrieves vectors from Qdrant (handles both dict and list vector formats)
3. **HDBSCAN Execution**: Configurable `min_cluster_size` (default 5), `min_samples` (default 3), metric (default euclidean on normalized vectors)
4. **Result Storage**: Assigns `cluster_id` values like `clu_{uuid_hex[:12]}` to both PostgreSQL `FaceInstance.cluster_id` and Qdrant payload
5. **Noise Handling**: Faces labeled -1 by HDBSCAN remain unassigned (no cluster_id)

Additional capability: `recluster_within_cluster()` for splitting a cluster into sub-clusters with tighter parameters.

#### `DualModeClusterer` (`faces/dual_clusterer.py`)

Two-phase clustering combining supervised and unsupervised approaches:

**Phase 1 - Supervised Assignment**:
- Computes centroids from all labeled faces (faces with `person_id`)
- For each unlabeled face, calculates cosine similarity against every person centroid
- Assigns to the best-matching person if similarity >= `person_match_threshold` (default 0.7)
- Updates both DB (`person_id`) and Qdrant payload

**Phase 2 - Unsupervised Clustering**:
- Takes remaining unmatched faces from Phase 1
- Runs HDBSCAN, DBSCAN, or Agglomerative clustering (configurable)
- Assigns cluster labels like `unknown_cluster_{N}` or `unknown_noise_{face_id}`
- Updates both DB (`cluster_id`) and Qdrant payload

Key observation: Phase 1 assigns faces to **existing** persons. Phase 2 groups remaining faces into anonymous clusters. There is no Phase 3 that would promote anonymous clusters to new Person entities.

### Face Assignment System (Detailed)

#### `FaceAssigner` (`faces/assigner.py`)

The incremental matching engine, separate from batch clustering:

- **Two-tier threshold** system pulled from a config service:
  - `face_auto_assign_threshold` -- high confidence: automatically assign face to person
  - `face_suggestion_threshold` -- medium confidence: create `FaceSuggestion` for human review
- **Prototype-based matching**: Searches Qdrant for nearest prototype vectors, resolves to person_id
- **Centroid computation**: `compute_person_centroids()` computes mean embeddings from all of a person's assigned faces, stores as CENTROID-role prototypes

### API Routes Analysis

#### Face Suggestions API (`api/routes/face_suggestions.py`)

The suggestion system is well-developed but exclusively person-centric:

- **List suggestions**: Flat or grouped-by-person pagination
- **Accept/Reject**: Updates `face.person_id` + syncs to Qdrant on accept
- **Bulk actions**: Accept/reject multiple suggestions with optional `auto_find_more` that enqueues background jobs
- **Find More**: Prototype-based and centroid-based discovery of additional matches for **existing** persons

Key limitation: All suggestion endpoints require a `suggested_person_id`. There is no concept of "suggest that these faces might be a new person."

#### Faces API (`api/routes/faces.py`)

Comprehensive endpoint set (~104KB file) covering:
- Face detection session management
- Cluster operations (list, recluster, merge)
- Person CRUD with face assignment
- Cluster-to-person promotion exists but is manual and requires explicit user action

#### Frontend Views

The UI has relevant face management views:
- `/faces/clusters` -- Shows clustered faces with sort/filter, uses `ClusterCard` component
- `/faces/suggestions` -- Shows person-matching suggestions with `SuggestionGroupCard`
- `/people` -- Person management with face assignment capabilities
- Components: `LabelClusterModal`, `PersonPickerModal`, `FindMoreResultsDialog`, `CentroidResultsDialog`

The `LabelClusterModal` and `PersonPickerModal` suggest there IS a workflow for manually labeling clusters, but it requires the user to proactively navigate to the Clusters view, find interesting clusters, and manually create/assign persons. There is no proactive "discovery" flow.

### Qdrant Vector Configuration

**Collection**: Configurable name via settings (default: face embeddings collection)
- **Dimensions**: 512 (ArcFace/InsightFace model output)
- **Distance**: Cosine similarity
- **Payload schema**: `asset_id`, `face_instance_id`, `person_id`, `cluster_id`, `detection_confidence`, `quality_score`, `taken_at`, `bbox`, `is_prototype`
- **Indexing**: HNSW index for efficient nearest-neighbor search

### Identified Gaps

| Gap | Description | Impact |
|-----|------------|--------|
| **No "New Person" suggestions** | `FaceSuggestion` requires `suggested_person_id` -- cannot suggest "this is someone new" | Users must manually discover unknown person clusters |
| **No cluster quality scoring** | Clusters lack a confidence/cohesion metric | Users cannot prioritize which clusters to review first |
| **No cluster promotion workflow** | No automated pipeline from cluster -> Person entity | Manual, friction-heavy process for creating new persons |
| **No singleton handling** | Faces not assigned to any cluster (HDBSCAN noise) are invisible | Single-occurrence faces of real people are lost |
| **No cross-session clustering** | Each clustering run is independent | Clusters may fragment or duplicate across runs |
| **No cluster merging suggestions** | System does not suggest that two clusters might be the same person | Users must manually compare clusters |

---

## Industry Patterns and Best Practices

### Google Photos Approach

- **Aggressive auto-grouping**: Uses deep learning (FaceNet) with large-scale cloud compute
- **Multi-cue recognition**: Combines face embeddings with clothing, body shape, location, and temporal proximity (photos taken close together likely contain the same people)
- **Progressive disclosure**: Shows "People" album with auto-grouped faces; users confirm names
- **Merge suggestions**: "Are these the same person?" prompts when two clusters have high similarity
- **Continuous learning**: Each user confirmation improves the model's confidence for that person
- **Handling growth**: New faces from new photos are matched against existing groups in near real-time

### Apple Photos Approach

- **On-device processing**: All face recognition runs locally for privacy
- **Multi-pass clustering strategy**:
  - **Pass 1 (Conservative)**: Very tight threshold, creates high-purity clusters (may under-merge)
  - **Pass 2 (HAC)**: Hierarchical Agglomerative Clustering merges similar clusters
  - **Pass 3 (Refinement)**: Uses additional signals (temporal, location) to merge further
- **Minimum face count**: Requires minimum faces before suggesting a person (avoids noisy singletons)
- **User confirmation**: "Confirm Additional Photos" prompt for borderline matches
- **Key insight**: Conservative first pass ensures precision; subsequent passes improve recall

### Immich (Open Source)

- **Configurable minimum recognized faces**: Setting controls how many faces must match before creating a person group
- **Background clustering**: Runs after photo upload, groups faces automatically
- **Manual naming**: Users name auto-discovered face groups
- **Merge UI**: Allows merging multiple face groups into one person

### Academic / ML Best Practices

- **Graph-based clustering**: Build face similarity graph, then apply community detection (Louvain, Chinese Whispers)
- **Threshold strategies**: Typical ranges for ArcFace/InsightFace:
  - **High confidence (auto-assign)**: cosine similarity >= 0.70-0.75
  - **Medium confidence (suggestion)**: cosine similarity >= 0.50-0.60
  - **Cluster formation**: cosine distance <= 0.40-0.45 for edge creation in graph methods
- **Rank-order clustering**: Instead of fixed thresholds, use rank-order distance (how many shared nearest neighbors two faces have) -- more robust to varying face quality
- **Incremental clustering**: For growing collections, don't re-cluster everything; use online/streaming clustering that adds new faces to existing clusters
- **Quality-weighted centroids**: Weight face embeddings by quality score when computing cluster centroids, so high-quality faces have more influence
- **Temporal coherence**: Faces from the same photo session are more likely to be the same person -- use this as a soft signal

### UX Patterns Across Systems

| Pattern | Google | Apple | Immich | Recommendation |
|---------|--------|-------|--------|----------------|
| Auto-group discovery | Yes | Yes | Yes | Essential |
| Minimum face count | Implicit | Yes | Yes (configurable) | Implement |
| Confidence display | No | No | No | Add (differentiator) |
| Merge suggestions | Yes | Limited | Manual | High value |
| Progressive naming | Yes | Yes | Yes | Standard UX |
| Undo/revert | Limited | Limited | Limited | Implement (audit log exists) |
| Cluster splitting | No | No | No | Leverage existing `recluster_within_cluster()` |

---

## Data Flow Mapping

### Current Complete Pipeline

```
                     DETECTION PHASE
                     ===============
Photo Upload/Ingest
       |
       v
FaceProcessingService.process_asset()
       |
       +-- InsightFace detection (bbox, landmarks, confidence)
       +-- ArcFace embedding extraction (512-dim vector)
       +-- Create FaceInstance in PostgreSQL
       |     (person_id=NULL, cluster_id=NULL)
       +-- Upsert vector to Qdrant
       |     (payload: asset_id, face_instance_id, quality_score, etc.)
       v
FaceInstance stored with:
  - qdrant_point_id (links to vector)
  - detection_confidence
  - quality_score
  - person_id = NULL
  - cluster_id = NULL


                     CLUSTERING PHASE
                     ================

Option A: FaceClusterer.cluster_unlabeled_faces()
       |
       +-- Scroll Qdrant: faces where person_id IS EMPTY
       +-- Filter: quality_score >= threshold
       +-- HDBSCAN clustering on embedding matrix
       +-- Assign cluster_id to DB + Qdrant
       v
FaceInstance updated:
  - cluster_id = "clu_abc123def456"
  - person_id = NULL (still unknown)

  OR

Option B: DualModeClusterer.cluster_all_faces()
       |
       +-- Phase 1: Supervised
       |     +-- Compute centroids from labeled faces
       |     +-- Match unlabeled faces to centroids
       |     +-- If similarity >= 0.7: assign person_id
       |
       +-- Phase 2: Unsupervised
       |     +-- HDBSCAN/DBSCAN/Agglomerative on remaining
       |     +-- Assign cluster_id = "unknown_cluster_N"
       v
FaceInstance updated:
  - High-confidence: person_id = <existing_person_uuid>
  - Medium-cluster: cluster_id = "unknown_cluster_N"
  - Noise: cluster_id = "unknown_noise_<face_id>"


                     ASSIGNMENT PHASE
                     ================

FaceAssigner.assign_new_faces()
       |
       +-- Search Qdrant prototypes for nearest matches
       +-- Two-tier threshold:
       |     >= auto_assign_threshold: set person_id directly
       |     >= suggestion_threshold: create FaceSuggestion
       v
High confidence: FaceInstance.person_id = <person_uuid>
Medium confidence: FaceSuggestion created
  - face_instance_id -> suggested_person_id
  - confidence score
  - status = "pending"


                     USER INTERACTION PHASE
                     =======================

Suggestions UI (/faces/suggestions)
       |
       +-- User reviews grouped suggestions
       +-- Accept: face.person_id = suggested_person_id
       +-- Reject: suggestion.status = "rejected"
       +-- Find More: discover additional matches
       v

Clusters UI (/faces/clusters)
       |
       +-- User browses anonymous clusters
       +-- Manual: select cluster -> "Label as Person"
       +-- (Requires creating new Person first)
       v

                    *** THE GAP ***
                    ===============
No automated workflow exists to:
  1. Score cluster quality/cohesion
  2. Surface high-confidence clusters as "potential new people"
  3. Let users confirm and name in one step
  4. Feed back into prototype system automatically
```

### Data State Inventory for Unassigned Faces

At any point, a `FaceInstance` can be in one of these states:

| State | person_id | cluster_id | Count Location | User Visibility |
|-------|-----------|-----------|----------------|-----------------|
| **Fully assigned** | Set | May/may not be set | N/A | People view |
| **Clustered, unknown** | NULL | Set (e.g., `unknown_cluster_3`) | Clusters view | Visible but passive |
| **Noise (singleton)** | NULL | Set (e.g., `unknown_noise_<id>`) or NULL | Not easily visible | Effectively hidden |
| **Unprocessed** | NULL | NULL | Not visible | Hidden |
| **Suggested** | NULL | May be set | Suggestions view | Pending review |

The critical population for "unknown person discovery" is the **Clustered, unknown** state -- these are faces that HDBSCAN grouped together but that have no person assignment. The system already believes they are the same person (via clustering), but no one has confirmed it.

---

## Key Design Decisions and Trade-offs

### Decision 1: Real-Time vs. Pre-Computed Clustering

**Option A: Real-time clustering on demand**
- Run clustering when user visits the "Discover People" view
- Pro: Always uses latest data, no stale clusters
- Con: Slow for large collections (HDBSCAN is O(n^2) in worst case), poor UX

**Option B: Pre-computed clustering with periodic refresh**
- Run clustering as a background job (already implemented via RQ)
- Pro: Fast UI rendering, can schedule during off-peak
- Con: Stale results between runs, requires cache invalidation

**Option C: Incremental/streaming clustering (Hybrid)**
- New faces matched against existing clusters in near real-time
- Full re-cluster periodically for refinement
- Pro: Best UX (immediate feedback + eventual consistency)
- Con: More complex to implement, risk of cluster drift

**Recommendation**: Option C (Hybrid). The existing `FaceAssigner` already does incremental matching against known persons. Extend this to also match against cluster centroids.

### Decision 2: Threshold Selection Strategy

The system currently uses these thresholds:

| Threshold | Value | Used By | Purpose |
|-----------|-------|---------|---------|
| `person_match_threshold` | 0.7 | DualModeClusterer | Assign to known person |
| `face_auto_assign_threshold` | Configurable | FaceAssigner | Auto-assign to person |
| `face_suggestion_threshold` | Configurable | FaceAssigner | Create suggestion |
| HDBSCAN `min_cluster_size` | 5 (or 3) | FaceClusterer | Minimum faces per cluster |

For unknown person discovery, additional thresholds are needed:

| New Threshold | Suggested Range | Purpose |
|---------------|----------------|---------|
| `cluster_cohesion_min` | 0.60-0.70 | Minimum average pairwise similarity within cluster |
| `cluster_promotion_min_faces` | 3-5 | Minimum faces before suggesting as new person |
| `cluster_merge_threshold` | 0.55-0.65 | Similarity between cluster centroids for merge suggestion |
| `singleton_ignore_threshold` | 0.40 | Below this, singleton is likely noise/bad detection |

**Trade-off**: Lower thresholds = more suggestions but more false positives. Higher thresholds = fewer suggestions but misses real people with few photos. The Apple approach (conservative first, then relax) is safest.

### Decision 3: Handling the Long Tail of Singletons

HDBSCAN labels many faces as "noise" (cluster label -1). These singletons represent:
- People who appear in only 1-2 photos (visitors, background people)
- Low-quality detections (blurry, partial, side-angle)
- True noise (false positive detections, art/posters)

**Options**:

| Strategy | Description | Pros | Cons |
|----------|------------|------|------|
| **Ignore singletons** | Only surface clusters with 3+ faces | Clean UI, fewer false positives | Misses people with few photos |
| **Quality-gated singletons** | Show singletons only if quality_score > 0.8 | Reduces noise | Still many singletons |
| **Temporal grouping** | Group singletons from same photo session | Finds people from events | Requires taken_at metadata |
| **Deferred singletons** | Re-evaluate after more photos added | Natural resolution over time | Users wait for results |
| **Manual triage** | Show singletons in a separate "Review" tab | Complete coverage | Labor intensive |

**Recommendation**: Use a tiered approach:
1. Primary: Surface clusters with `min_faces >= 3` as "Suggested People"
2. Secondary: Quality-gated singletons (`quality_score > 0.8`) in a separate "Might Know" section
3. Background: Re-evaluate singletons when new photos match them above threshold

### Decision 4: Cluster-to-Person Promotion Flow

**Option A: Fully automatic**
- Create Person entity automatically for any cluster above threshold
- Pro: Zero user effort
- Con: Creates unnamed persons, hard to manage, no human validation

**Option B: Semi-automatic with confirmation**
- Surface "Suggested New People" in UI with representative faces
- User confirms and names with one click
- Pro: Balance of automation and control
- Con: Requires user engagement

**Option C: Fully manual**
- User browses clusters, manually creates persons (current state)
- Pro: Maximum control
- Con: High friction, most users won't do this

**Recommendation**: Option B. This aligns with Google/Apple/Immich patterns. The existing `FaceSuggestion` model could be extended or a new `ClusterPromotion` model could track these.

### Decision 5: Prototype System Integration

When a cluster is promoted to a Person, the prototype system should be bootstrapped:

1. **Centroid prototype**: Compute from all cluster faces (quality-weighted mean)
2. **Exemplar prototypes**: Select 2-3 highest-quality, most diverse faces from cluster
3. **Primary prototype**: The single best face (highest quality + highest centrality)

This is critical because once the Person exists, the `FaceAssigner` can immediately start finding more matches using these prototypes, creating a virtuous cycle: more faces -> better prototypes -> more matches.

### Decision 6: Cluster Merge Detection

Two clusters might represent the same person (e.g., different ages, different lighting conditions). The system should detect potential merges:

**Approach**: Compute cluster centroids, then find pairs with cosine similarity above `cluster_merge_threshold`. Surface as "Are these the same person?" suggestions.

**Complexity**: O(k^2) where k = number of clusters. For 100 clusters, that is 4,950 comparisons -- very fast. For 1,000 clusters, that is 499,500 -- still fast with vectorized numpy.

---

## Recommendations

### Recommendation 1: Implement Cluster Quality Scoring

**Priority**: High
**Effort**: Low

Add metrics to each cluster:
- **Cohesion score**: Average pairwise cosine similarity within the cluster
- **Separation score**: Distance to nearest other cluster centroid
- **Size**: Number of faces
- **Quality average**: Mean quality_score of member faces
- **Temporal spread**: Number of distinct dates/events

This can be computed during clustering and stored as cluster metadata. It enables UI sorting and filtering of clusters by "interestingness."

### Recommendation 2: Build "Suggested New People" Pipeline

**Priority**: High
**Effort**: Medium

New background job that:
1. Identifies clusters meeting promotion criteria (min faces, min cohesion, min quality)
2. Computes cluster representatives (best face, centroid)
3. Creates entries in a new `ClusterPromotion` model (or extends `FaceSuggestion`)
4. Checks for potential merges with existing clusters and persons
5. Surfaces results in a new "Discover People" UI section

### Recommendation 3: Extend FaceSuggestion for "New Person" Suggestions

**Priority**: Medium
**Effort**: Medium

Two approaches:
- **Option A**: Add `suggested_person_id = NULL` case to mean "new person" with a new `cluster_id` reference
- **Option B**: Create a separate `ClusterPromotionSuggestion` model

Option A is simpler but muddies the existing model. Option B is cleaner but adds a new entity. Given the existing complexity of `FaceSuggestion` (with multi-prototype scoring fields), Option B is likely cleaner.

### Recommendation 4: Implement Incremental Cluster Matching

**Priority**: Medium
**Effort**: Medium-High

When new faces are detected, in addition to matching against person prototypes (existing behavior), also match against cluster centroids. If a new face matches an existing cluster, add it to that cluster. This keeps clusters growing without full re-clustering.

### Recommendation 5: Add Cluster Merge Suggestions

**Priority**: Low
**Effort**: Low

Post-clustering job that computes pairwise cluster centroid similarities and suggests merges above threshold. Could reuse the existing suggestion UI pattern.

### Recommendation 6: Quality-Gated Singleton Handling

**Priority**: Low
**Effort**: Low

For faces that did not cluster (HDBSCAN noise), maintain a "watch list" of high-quality singletons. When new photos are added, check if any singleton now has a match. This handles the case of a person who appears in only one early photo but later appears in more.

---

## Open Questions for the Team

1. **Naming UX**: When promoting a cluster to a Person, should we require a name immediately, or allow "Unknown Person #N" with deferred naming? (Apple uses unnamed groups; Google prompts for names)

2. **Minimum cluster size**: What should the minimum face count be for surfacing a "Suggested New Person"? Industry range is 2-5. Lower catches more people but increases noise. Given our HDBSCAN `min_cluster_size=5`, clusters already have 5+ faces -- should the promotion threshold match or be higher?

3. **Confidence display**: Should we show the cluster cohesion score to users? Most consumer apps hide this, but power users (the likely audience for this self-hosted tool) may appreciate it.

4. **Auto-merge threshold**: How aggressively should we suggest merging clusters? A conservative approach (high threshold) means users do more manual work but fewer mistakes. An aggressive approach (lower threshold) risks merging different people.

5. **Prototype bootstrapping**: When creating a Person from a cluster, how many prototypes should we auto-create? The current system supports multiple roles (centroid, exemplar, primary, temporal, fallback). Should we auto-populate all roles or start with just centroid + primary?

6. **Re-clustering frequency**: Should full re-clustering happen on a schedule, on-demand, or triggered by events (e.g., after N new photos detected)? Current system runs clustering manually via `make faces-cluster-dual`.

7. **Cluster persistence**: Should clusters be permanent entities (surviving re-clustering) or ephemeral (recalculated each run)? Current implementation assigns new `cluster_id` values each run, which means clusters are not stable across runs.

8. **Privacy/consent**: For a self-hosted photo management tool, are there privacy considerations around automatically grouping faces? Some jurisdictions have biometric data laws (GDPR, BIPA). Should this feature be opt-in?

9. **Performance budget**: What is the acceptable latency for the "Discover People" view? If we pre-compute cluster quality metrics, the view can be fast. If we compute on-demand, it depends on collection size. Target: < 2 seconds for 10,000 faces?

10. **Integration with existing Suggestions**: Should "Suggested New People" appear alongside existing person-matching suggestions in the same UI, or in a separate view? Mixing them in one view reduces navigation but increases complexity.

---

## Appendix A: File Reference

| File | Path | Relevance |
|------|------|-----------|
| Database models | `image-search-service/src/image_search_service/db/models.py` | Person, FaceInstance, PersonPrototype, FaceSuggestion, PersonCentroid, FaceAssignmentEvent |
| Face detector | `image-search-service/src/image_search_service/faces/detector.py` | InsightFace integration |
| Face service | `image-search-service/src/image_search_service/faces/service.py` | FaceProcessingService (detect + embed + store) |
| Clusterer | `image-search-service/src/image_search_service/faces/clusterer.py` | FaceClusterer (HDBSCAN unsupervised) |
| Dual clusterer | `image-search-service/src/image_search_service/faces/dual_clusterer.py` | DualModeClusterer (supervised + unsupervised) |
| Assigner | `image-search-service/src/image_search_service/faces/assigner.py` | FaceAssigner (two-tier threshold matching) |
| Qdrant client | `image-search-service/src/image_search_service/vector/face_qdrant.py` | FaceQdrantClient (512-dim vectors) |
| Faces API | `image-search-service/src/image_search_service/api/routes/faces.py` | Face detection, clustering, person management endpoints |
| Suggestions API | `image-search-service/src/image_search_service/api/routes/face_suggestions.py` | Suggestion CRUD, find-more, bulk actions |
| Clusters UI | `image-search-ui/src/routes/faces/clusters/+page.svelte` | Cluster browsing view |
| Suggestions UI | `image-search-ui/src/routes/faces/suggestions/` | Suggestion review view |

## Appendix B: Threshold Reference

| Threshold | Current Value | Component | Notes |
|-----------|--------------|-----------|-------|
| `person_match_threshold` | 0.7 | DualModeClusterer | Cosine similarity for supervised assignment |
| `face_auto_assign_threshold` | Configurable (config service) | FaceAssigner | Auto-assign without human review |
| `face_suggestion_threshold` | Configurable (config service) | FaceAssigner | Create suggestion for human review |
| `unknown_min_cluster_size` | 3 | DualModeClusterer | Min faces for unsupervised clustering |
| `min_cluster_size` | 5 | FaceClusterer | HDBSCAN min cluster size |
| `min_samples` | 3 | FaceClusterer | HDBSCAN min samples for core point |
| `quality_threshold` | 0.5 | FaceClusterer | Min quality_score for inclusion |
| `unknown_eps` | 0.5 | DualModeClusterer | Distance threshold for DBSCAN/Agglomerative |

## Appendix C: Industry Research Sources

- Google Photos face recognition: Multi-cue approach combining face embeddings, body, clothing, and temporal signals. Aggressive auto-grouping with merge suggestions.
- Apple Photos: On-device multi-pass clustering (conservative first pass + HAC refinement + temporal signals). Privacy-first design.
- Immich: Open-source photo management with configurable minimum recognized faces threshold. Background clustering with manual naming.
- Academic: Rank-order clustering, graph-based community detection (Chinese Whispers, Louvain), quality-weighted centroids, incremental/streaming clustering approaches.
