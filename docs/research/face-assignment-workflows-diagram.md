# Face Assignment Workflows - Visual Comparison

## Approach 1: Manual Single Face Assignment

```
┌─────────────────────────────────────────────────────────────────────┐
│ USER ACTION: Click "Assign to Person" in FaceListSidebar           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ POST /api/v1/faces/faces/{face_id}/assign                          │
│                                                                      │
│  1. face.person_id = person_id                                     │
│  2. Update Qdrant: qdrant.update_person_ids()                      │
│  3. Create/update prototypes from verified label                   │
│  4. Queue propagate_person_label_job ✅                            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ BACKGROUND JOB: propagate_person_label_job                         │
│                                                                      │
│  1. Get source face embedding from Qdrant                          │
│  2. Search for similar faces (score >= 0.7)                        │
│  3. Filter to UNASSIGNED faces only                                │
│  4. Create up to 50 FaceSuggestion records                         │
│                                                                      │
│  Result: 0-50 NEW suggestions for user review                      │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ USER ACCEPTS SUGGESTION → Triggers ANOTHER propagate job           │
│                                                                      │
│  SNOWBALL EFFECT: 1 manual label → 50 suggestions →                │
│                   Accept 10 → 500 more suggestions →                │
│                   Accept 50 → 2500 more suggestions...             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Approach 2: Cluster Labeling (Batch Assignment)

```
┌─────────────────────────────────────────────────────────────────────┐
│ USER ACTION: Click "Label as Person" on cluster page               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ POST /api/v1/faces/clusters/{cluster_id}/label                     │
│                                                                      │
│  1. Get ALL faces in cluster (e.g., 47 faces)                      │
│  2. Assign ALL faces: face.person_id = person_id                   │
│  3. Update Qdrant for all faces                                    │
│  4. Create prototypes (top 3 by quality)                           │
│  5. Queue propagate_person_label_job using BEST face ✅            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ BACKGROUND JOB: propagate_person_label_job                         │
│                                                                      │
│  Uses highest-quality face from cluster as source                  │
│  Same logic as approach 1:                                         │
│    - Search for similar faces                                      │
│    - Create up to 50 suggestions                                   │
│                                                                      │
│  Result: 0-50 NEW suggestions (beyond the cluster faces)           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ UI CLARITY ISSUE:                                                  │
│   - UI showed 5 sample faces                                       │
│   - User clicked "Label as Person"                                 │
│   - System labeled ALL 47 faces in cluster                         │
│   - User may not realize 42 other faces were also labeled         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Approach 3: Training Pipeline (Automated)

```
┌─────────────────────────────────────────────────────────────────────┐
│ USER ACTION: Start training session / face detection               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ BACKGROUND JOB: detect_faces_for_session_job                       │
│                                                                      │
│  PHASE 1: Face Detection                                           │
│    - Process images in batches (default: 8)                        │
│    - Detect faces using InsightFace                                │
│    - Generate embeddings using Buffalo-L                           │
│    - Store in Qdrant + PostgreSQL                                  │
│                                                                      │
│  Result: 1000 new FaceInstance records with embeddings             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 2: Auto-Assignment (FaceAssigner.assign_new_faces)          │
│                                                                      │
│  For each unassigned face:                                         │
│    1. Search against ALL existing prototypes                       │
│    2. Get best match score                                         │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ TWO-TIER THRESHOLD DECISION:                                │  │
│  │                                                              │  │
│  │  if score >= 0.85:  # AUTO_ASSIGN_THRESHOLD                │  │
│  │    ✅ Assign face.person_id = person_id                     │  │
│  │    ❌ NO propagate_person_label_job queued                  │  │
│  │    Result: Face auto-assigned (high confidence)            │  │
│  │                                                              │  │
│  │  elif score >= 0.70:  # SUGGESTION_THRESHOLD                │  │
│  │    ✅ Create FaceSuggestion (status: pending)               │  │
│  │    ❌ NO propagate_person_label_job queued                  │  │
│  │    Result: User review required                             │  │
│  │                                                              │  │
│  │  else:  # score < 0.70                                      │  │
│  │    ❌ Leave unassigned                                       │  │
│  │    → Send to clustering (next phase)                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Result: 300 auto-assigned, 150 suggestions, 550 unassigned        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 3: Clustering (HDBSCAN)                                     │
│                                                                      │
│  For remaining 550 unassigned faces:                               │
│    - Run HDBSCAN clustering (min_cluster_size=3)                   │
│    - Create unknown person clusters                                │
│    - Assign faces to cluster_id (NOT person_id)                    │
│                                                                      │
│  Result: 50 new clusters (10 faces each), 50 noise (outliers)     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ END OF TRAINING SESSION                                            │
│                                                                      │
│  ❌ NO propagate_person_label_job queued for auto-assigned faces   │
│  ❌ NO recursive suggestion generation                             │
│                                                                      │
│  User sees:                                                         │
│    - 300 faces auto-assigned to known persons                      │
│    - 150 suggestions pending review                                │
│    - 50 new unknown person clusters                                │
│                                                                      │
│  To get MORE suggestions:                                          │
│    - User must manually click "Find More Suggestions"              │
│    - OR wait for next training run                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Side-by-Side Comparison

| Feature | Manual Assignment | Cluster Labeling | Training Pipeline |
|---------|------------------|------------------|-------------------|
| **Assigns faces?** | ✅ 1 face | ✅ N faces (cluster) | ✅ Auto-assign + suggestions |
| **Queues propagation job?** | ✅ Always | ✅ Always | ❌ Never |
| **Creates suggestions?** | ✅ 0-50 per assignment | ✅ 0-50 per cluster | ⚠️ Only for 0.70-0.84 scores |
| **Snowball effect?** | ✅ Yes (accept → more suggestions) | ✅ Yes (from best face) | ❌ No (one-pass only) |
| **User feedback?** | ✅ Immediate | ✅ Immediate | ⏱️ Delayed (batch complete) |
| **Use case** | Interactive labeling | Batch labeling | Bulk processing |
| **Performance impact** | Low (1 job) | Medium (1 job per cluster) | Low (no propagation) |

---

## Threshold System Visual

```
Similarity Score Scale:
0.0                    0.70                   0.85                    1.0
│─────────────────────│──────────────────────│──────────────────────│
│   LEAVE UNASSIGNED  │  CREATE SUGGESTION   │     AUTO-ASSIGN      │
│   (Send to cluster) │  (User review req'd) │  (High confidence)   │
│                     │                      │                      │
│  Example: Different │  Example: Same person│  Example: Exact match│
│  person, low qual.  │  diff lighting/angle │  or twin/sibling     │
└─────────────────────┴──────────────────────┴──────────────────────┘

Manual Assignment:
  ┌─────────────────────────────────────────────────────────────┐
  │ User assigns face → Queue propagate job                     │
  │ Propagate job searches at threshold 0.70                    │
  │ Creates suggestions for ALL matches >= 0.70                 │
  └─────────────────────────────────────────────────────────────┘

Training Pipeline:
  ┌─────────────────────────────────────────────────────────────┐
  │ FaceAssigner searches at threshold 0.70                     │
  │   ├─ Score >= 0.85 → Auto-assign (no propagate)            │
  │   ├─ Score 0.70-0.84 → Create suggestion (no propagate)    │
  │   └─ Score < 0.70 → Leave unassigned → Cluster             │
  └─────────────────────────────────────────────────────────────┘
```

---

## Suggestion Generation Workflow (propagate_person_label_job)

```
┌─────────────────────────────────────────────────────────────────────┐
│ INPUT: source_face_id, person_id                                   │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Retrieve source face embedding from Qdrant                      │
│                                                                      │
│    points = qdrant.retrieve(                                       │
│        collection_name="faces",                                    │
│        ids=[source_face.qdrant_point_id],                          │
│        with_vectors=True                                           │
│    )                                                               │
│                                                                      │
│    source_vector = points[0].vector  # 512-dim embedding          │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Search Qdrant for similar faces                                 │
│                                                                      │
│    search_results = qdrant.query_points(                           │
│        query=source_vector,                                        │
│        limit=60,  # max_suggestions + 10 buffer                    │
│        score_threshold=0.7  # min_confidence                       │
│    )                                                               │
│                                                                      │
│    Returns: List of (face_id, score) sorted by similarity         │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Filter candidates                                               │
│                                                                      │
│    for result in search_results:                                   │
│        face = get_face_by_qdrant_point_id(result.id)              │
│                                                                      │
│        ✅ Skip if face.id == source_face_id (self)                 │
│        ✅ Skip if face.person_id is not None (already assigned)    │
│        ✅ Skip if pending suggestion already exists                │
│                                                                      │
│    Remaining: Unassigned faces with no existing suggestions        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 4. Create FaceSuggestion records                                   │
│                                                                      │
│    for face in filtered_candidates[:max_suggestions]:              │
│        suggestion = FaceSuggestion(                                │
│            face_instance_id=face.id,                               │
│            suggested_person_id=person_id,                          │
│            confidence=result.score,                                │
│            source_face_id=source_face_id,                          │
│            status="pending"                                        │
│        )                                                            │
│        db.add(suggestion)                                          │
│                                                                      │
│    db.commit()                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ OUTPUT: Job result                                                 │
│                                                                      │
│    {                                                               │
│        "status": "completed",                                      │
│        "faces_checked": 60,                                        │
│        "suggestions_created": 23  # (after filtering)             │
│    }                                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Insight: The Snowball Effect

### Manual Assignment Chain Reaction:

```
User assigns 1 face → propagate_person_label_job
    ↓
Creates 50 suggestions
    ↓
User accepts 10 suggestions → 10 × propagate_person_label_job
    ↓
Creates 500 more suggestions (10 × 50)
    ↓
User accepts 50 suggestions → 50 × propagate_person_label_job
    ↓
Creates 2500 more suggestions (50 × 50)
    ↓
...continues until no more unassigned faces with score >= 0.7
```

### Training Pipeline (No Chain Reaction):

```
Training detects 1000 faces
    ↓
FaceAssigner processes all 1000:
    - 300 auto-assigned (score >= 0.85)
    - 150 suggestions created (score 0.70-0.84)
    - 550 sent to clustering (score < 0.70)
    ↓
END (no propagation jobs queued)
    ↓
To get more suggestions:
    - User must manually trigger "Find More Suggestions"
    - OR run another training session (which will use updated prototypes)
```

---

## Why Training Doesn't Propagate (Architectural Reasons)

### Performance Impact:

```
Manual Assignment:
  1 face × 1 propagate job = 1 job (acceptable)

Cluster Labeling:
  47 faces × 1 propagate job = 1 job (acceptable, uses best face)

Training Pipeline (if propagate was enabled):
  300 auto-assigned faces × 1 propagate job = 300 jobs
  Each job searches 50 faces = 15,000 searches
  Each search queries Qdrant = 15,000 Qdrant queries
  Total time: ~30 minutes (blocking other work)

  Result: Queue overload, system slowdown ❌
```

### Circular Logic:

```
Training auto-assigns face to Person A because it matched prototype #1
    ↓
If propagate job queued:
    ↓
Search Qdrant using newly assigned face as source
    ↓
Results: Same faces already in Person A's cluster
    ↓
Redundant suggestions (already discovered by prototypes) ❌
```

### Quality Concerns:

```
Manual Assignment:
  User verifies face → High quality label → Good search source ✅

Training Auto-Assignment:
  Machine predicts (0.85+ confidence) → May contain errors → Risky search source ⚠️

  Example:
    - Training auto-assigns twin sibling faces to Person A
    - If propagate enabled: Would suggest ALL twin B faces for Person A
    - Result: Error propagation across dataset ❌
```

---

**Conclusion**: The three approaches are working as designed. Manual assignment prioritizes completeness (snowball effect), while training prioritizes speed and accuracy (one-pass assignment).
