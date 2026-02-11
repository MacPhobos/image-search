# Devil's Advocate Analysis: Unknown Person Face Detection Feature

**Research Role**: Devil's Advocate
**Date**: 2026-02-07
**Status**: Complete
**Feature Prompt**: `docs/prompts/unlabeled-face-suggestions.md`

---

## Executive Summary

The proposed "Unknown Person Face Detection" feature aims to surface groupings of unassigned faces so users can discover new persons not yet in the system. While the intent is sound -- making it easier to bootstrap person identification -- the proposal carries significant technical risk, introduces UX complexity that may confuse users, and in several cases duplicates or conflicts with existing functionality. This analysis challenges the core assumptions, identifies failure modes grounded in the actual codebase, and proposes alternative approaches that may achieve the same goal with lower risk.

**Verdict**: The feature addresses a real gap but the proposed implementation (dynamic similarity grouping with a threshold slider in a new view) is the most complex and risky way to solve the problem. Simpler alternatives exist within the current architecture that should be exhausted first.

---

## 1. Assumption Challenges

### 1.1 "The Clusters View Doesn't Cover This Use Case"

The feature prompt states: *"The 'Clusters' view provides some grouping functionality but it definitely does not cover this use case."*

**Challenge**: The existing `FaceClusterer` in `image-search-service/src/image_search_service/faces/clusterer.py` (lines 60-99) already performs exactly what is being proposed: it uses HDBSCAN to group unlabeled faces into clusters. It uses `IsEmptyCondition(is_empty=PayloadField(key="person_id"))` to filter to unassigned faces and groups them by similarity. The `DualModeClusterer` in `image-search-service/src/image_search_service/faces/dual_clusterer.py` extends this with supervised assignment before unsupervised clustering.

The Clusters view (`image-search-ui/src/routes/faces/clusters/+page.svelte`) already displays these groupings with a detail page per cluster (`image-search-ui/src/routes/faces/clusters/[clusterId]/+page.svelte`).

**Question to resolve**: What specifically is the Clusters view missing? Is it:
- A workflow to convert a cluster into a Person? (UI gap, not a new feature)
- Different clustering parameters? (Configuration, not architecture)
- Real-time re-clustering at different thresholds? (Major new capability)
- Better visual grouping? (UX improvement to existing view)

Before building a parallel system, the team must articulate exactly why enhancing the Clusters view is insufficient.

### 1.2 "Dynamic Similarity Threshold Will Help Users"

The proposal requests a slider for users to control the similarity threshold for grouping. This assumes:

1. **Users understand what similarity threshold means in embedding space.** Cosine similarity in 512-dimensional ArcFace space is not intuitive. The difference between 0.65 and 0.70 can be the difference between "same person in different lighting" and "two siblings." Users have no reference frame for these numbers.

2. **A single threshold works uniformly across all face pairs.** Similarity thresholds behave differently for:
   - High-quality frontal photos vs. blurry side profiles
   - Adults vs. children (children's faces change dramatically)
   - Photos taken years apart vs. same-day photos
   - Groups with 3 faces vs. groups with 300 faces

3. **The threshold produces stable groupings.** Small threshold changes can cause catastrophic cluster reorganization. At threshold=0.72, you might see 5 clean groups. At threshold=0.70, those 5 groups could merge into 2 messy groups. Users expect gradual refinement; they will get unpredictable phase transitions.

### 1.3 "A Separate View Is Needed"

The proposal requests a toggle between "Face Suggestions" and "Discover New Persons." This assumes users have a clear mental model of when they want to assign faces to *known* persons vs. discover *unknown* persons. In practice:

- A user looking at face suggestions may see a suggestion for a face that belongs to a person not yet in the system. They cannot act on it from the current view.
- A user in the "new persons" view may see a group that actually matches an existing person they forgot to check.
- The toggle creates a false dichotomy: the real workflow is "I see a face, I want to identify who it is" regardless of whether the person exists yet.

### 1.4 "We Need In-Depth Research Into Database and Qdrant"

The proposal states this "requires in depth research into how to utilize data in the database and qdrant." But the system already has:
- Face embeddings in Qdrant with cosine similarity search (`image-search-service/src/image_search_service/vector/face_qdrant.py`)
- HDBSCAN clustering of unlabeled faces (`clusterer.py`)
- Person centroid computation with outlier trimming (`image-search-service/src/image_search_service/services/centroid_service.py`)
- Prototype-based similarity search (`face_suggestions.py` find-more endpoints)

The building blocks already exist. The question is not "how do we use the data" but "how do we combine existing capabilities into a coherent workflow."

---

## 2. Edge Cases and Failure Modes

### 2.1 The Singleton Problem

Most face collections follow a power-law distribution: a few persons have many photos, and many persons appear in only 1-2 photos. The existing `FaceClusterer` has `min_cluster_size=5` (default) and `min_samples=3`. The `DualModeClusterer` uses `unknown_min_cluster_size=3`.

**Failure mode**: Persons who appear in only 1-2 photos will never form a cluster. They will be classified as noise by HDBSCAN (label=-1). The current code in `dual_clusterer.py` (line 258-259) assigns these noise points unique labels (`unknown_noise_{face_id}`), which means they never appear in any meaningful group. The proposed feature, if it relies on clustering, will systematically fail for exactly the use case it claims to serve: discovering persons who appear in a small number of photos.

**Scale**: In a typical family photo collection of 10,000 photos, expect 40-60% of unique faces to be "one-timers" (wedding guests, background people, service workers in vacation photos). These will all be invisible.

### 2.2 Quality-Score Gatekeeping

The existing pipeline applies a quality threshold of 0.5 by default (`face_qdrant.py` line 581, `clusterer.py` line 54). Faces below this threshold are excluded from clustering entirely.

**Failure mode**: A person who only appears in lower-quality photos (distant shots, side profiles, partially occluded) will be filtered out before clustering even begins. The user will never see these faces in the "unknown persons" view, despite them being real persons the user might want to label.

### 2.3 The "Uncanny Valley" of Similarity

When similarity thresholds are set too low, the system will group faces that look similar but are different people:
- Family members (siblings, parent-child pairs)
- People wearing similar glasses or hats
- People with similar hairstyles or skin tone

**Failure mode**: The user lowers the threshold to find more persons, sees groups containing multiple different people, loses trust in the system, and abandons the feature. There is no mechanism in the proposal for the user to split a group that was incorrectly merged.

### 2.4 Temporal Distribution Mismatch

The codebase has `PersonPrototype` with `age_era_bucket` (line reference from models.py) and `taken_at` metadata. A person photographed over 10 years will have significantly different face embeddings at different ages.

**Failure mode**: The same person appears as 2-3 different "unknown person" groups because their photos span years. The system confidently presents "Unknown Group A" (childhood photos) and "Unknown Group B" (adult photos) as different people. The user must manually recognize these are the same person, which defeats the purpose of automated discovery.

### 2.5 Database-Qdrant Consistency Gap

The existing codebase already has a known consistency problem. In `face_suggestions.py`, Qdrant sync happens after DB commit and is non-blocking. The comment in the code acknowledges this: DB is updated but Qdrant may be out of sync.

**Failure mode**: A user assigns faces from the "unknown persons" view. The DB updates succeed but the Qdrant payload update fails. The next time the system clusters or searches, those faces still appear as "unlabeled" in Qdrant, leading to:
- Faces reappearing in the unknown persons view after being assigned
- Duplicate suggestions across both views (known persons and unknown persons)
- Gradually increasing inconsistency between DB truth and Qdrant state

---

## 3. Technical Risks

### 3.1 Real-Time Clustering vs. Pre-Computation

The proposal implies users can adjust a threshold slider and see updated groupings. This requires either:

**Option A: Real-time re-clustering on threshold change.**
- HDBSCAN on 10,000 embeddings takes ~2-5 seconds. On 50,000 embeddings, expect 30-60 seconds.
- The user adjusts the slider, waits a minute, sees different results. This is not interactive.
- Each slider change hits the Qdrant full-scan path (`get_unlabeled_faces_with_embeddings` scrolls ALL records with Python-side filtering).

**Option B: Pre-compute clusters at multiple thresholds.**
- Must decide thresholds in advance (what granularity?).
- Storage multiplies: N threshold levels x M clusters per level x metadata per cluster.
- Pre-computation runs during face pipeline, increasing already-complex background job duration.
- Stale results when new photos are added between pre-computations.

**Option C: Use Qdrant similarity search per representative face.**
- Avoids full re-clustering but produces different-shaped results (nearest neighbors, not clusters).
- Not true "grouping" -- shows "faces similar to this face" which is what the existing Find More feature already does.

None of these options are simple. The proposal does not specify which approach to use, leaving a critical architectural decision unresolved.

### 3.2 The Full-Scan Problem in `get_unlabeled_faces_with_embeddings`

This function (at `face_qdrant.py` lines 579-632) scrolls through the ENTIRE Qdrant collection, loading every record with vectors and payloads, then filters in Python for unlabeled faces. The code comment explicitly acknowledges this: *"Qdrant doesn't have a native 'is null' filter, so we scroll all records and filter in Python."*

However, this comment is incorrect. The `clusterer.py` file (line 78) already uses `IsEmptyCondition(is_empty=PayloadField(key="person_id"))` to do server-side filtering. This Qdrant-native filter exists and is already used elsewhere in the codebase.

**Risk**: If the proposed feature builds on top of `get_unlabeled_faces_with_embeddings`, it inherits an O(N) full-scan where an O(log N) indexed filter already exists. For a collection of 100,000 face vectors (realistic for a large photo library), this means loading ~100K vectors x 512 floats x 4 bytes = ~200MB into Python memory per request.

### 3.3 Memory Pressure During Real-Time Grouping

Dynamic re-grouping requires:
1. Load all unlabeled face embeddings (potentially tens of thousands of 512-dim vectors)
2. Compute pairwise distance matrix (N x N matrix -- for 10K faces, that is 100M entries)
3. Run clustering algorithm
4. Return results

For 10,000 unlabeled faces, the pairwise distance matrix alone requires ~400MB (10K x 10K x 4 bytes). For 50,000 faces (the current `max_faces` default in `clusterer.py`), it requires ~10GB. This is not feasible for a web request.

### 3.4 Interaction With Existing Clustering Pipeline

The system already runs clustering as a background job (`make faces-cluster-dual`). If the proposed feature introduces a second clustering mechanism with different parameters, the results can conflict:

- Background clustering assigns `cluster_id = "unknown_cluster_3"` to a face
- New feature dynamically groups the same face into "Group B" with different members
- User acts on the dynamic grouping, but the background job re-runs and overwrites `cluster_id`
- The face disappears from the group the user was reviewing

There is no locking mechanism or versioning on `cluster_id` to prevent this race condition.

---

## 4. UX Concerns

### 4.1 Decision Fatigue

The current Face Suggestions view presents a clear workflow: "Here is a face; it probably belongs to this person; accept or reject." The proposed feature inverts this: "Here is a group of faces; they might be the same person; create a person and assign them all."

The second workflow requires:
1. Evaluate whether the group is actually one person (inspect each face)
2. Decide on a name for the new person
3. Possibly split the group if it contains faces from multiple people
4. Handle faces that are borderline

This is significantly more cognitive load than a binary accept/reject decision.

### 4.2 The Threshold Slider is a Leaky Abstraction

Exposing the similarity threshold as a slider is exposing an implementation detail. Users think in terms of "more suggestions" or "fewer suggestions" or "more confident" vs. "less confident." They do not think in terms of cosine similarity thresholds.

Worse, the slider behavior is non-linear: moving from 0.85 to 0.80 might add 50 faces to existing groups, but moving from 0.80 to 0.75 might merge half the groups together. The user has no way to predict the effect of their adjustment.

### 4.3 Two-View Toggle Creates Workflow Fragmentation

Currently the workflow for face management is:
1. Run face detection pipeline
2. Review suggestions (auto-generated for known persons)
3. Browse clusters for unknown faces

The proposal adds a fourth step: browse "unknown person" groupings. But this raises questions:
- When should a user use Clusters vs. Unknown Persons? The difference is unclear.
- If a user creates a person from the Unknown Persons view, do existing suggestions update immediately?
- If the user is mid-review in Unknown Persons and runs the detection pipeline, do the groupings change under them?

### 4.4 No Progressive Disclosure

The proposal shows all groups at once. For a photo library with 10,000 unlabeled faces forming 500+ potential groups, displaying all groups simultaneously is overwhelming. There is no mention of:
- Sorting/ranking groups by size, confidence, or relevance
- Progressive loading or pagination
- Filtering by date range, photo location, or other metadata
- Grouping the groups (meta-categories)

---

## 5. System Integration Issues

### 5.1 Conflict With Dual-Mode Clustering Results

The `DualModeClusterer.cluster_all_faces()` method (in `dual_clusterer.py`) already performs exactly the two-phase operation that would underlie this feature:
1. Supervised: assign unlabeled faces to known persons (centroid matching at threshold 0.7)
2. Unsupervised: HDBSCAN cluster the remaining unknowns

Running a new "unknown person" grouping on top of this creates a layered clustering problem:
- Phase 1 output feeds Phase 2
- The new feature would re-group Phase 2 output with different parameters
- Results from the new feature might contradict Phase 1 assignments (e.g., a face assigned to Person A at threshold 0.7 might appear in an "unknown" group at threshold 0.65)

### 5.2 Centroid Invalidation Chain

When a user creates a new person from an unknown group:
1. Assign N faces to the new person (DB + Qdrant update)
2. Compute centroid for the new person (`centroid_service.py`)
3. Existing suggestions for other persons may now be invalid (a face previously suggested for Person X might now belong to the new Person Y)
4. The "unknown" groupings must be recomputed (faces removed from the pool change all remaining groups)

This cascade is not instantaneous. Between step 1 and step 4, the system is in an inconsistent state where:
- The Unknown Persons view shows stale groups (faces already assigned)
- The Face Suggestions view may show suggestions for the just-assigned faces
- Background jobs may be computing centroids for persons that are still being modified

### 5.3 Background Job Queue Contention

The system uses a 4-tier priority Redis queue. Face detection, clustering, and suggestion generation are already competing for worker time. Adding real-time or near-real-time grouping for the unknown persons feature means:
- More Qdrant queries per user interaction
- More background jobs for re-clustering after assignments
- Potential queue backlog during active discovery sessions

### 5.4 API Surface Area Expansion

The backend currently has face-related endpoints for:
- Face detection sessions
- Face suggestions (list, accept, reject, bulk, find-more)
- Face clusters (list, detail)
- Person management (CRUD, merge)
- Face centroids and prototypes

Adding a "dynamic grouping" endpoint means new routes, new schemas, new Qdrant query patterns, and new background jobs. Each new endpoint is a testing surface, a security surface, and a maintenance burden.

---

## 6. Privacy and Ethics Considerations

### 6.1 Automated Face Grouping of Non-Consenting Subjects

The feature automatically groups faces of people who have not been registered in the system. In a family photo collection, this includes:
- Minors who cannot consent to biometric processing
- Strangers in the background of photos (pedestrians, waitstaff)
- People at events who may not want their face tracked

While the system does not perform identification (it groups, it does not name), the grouping itself constitutes biometric data processing in many jurisdictions.

### 6.2 GDPR and Biometric Data Implications

Under GDPR Article 9, facial recognition data is a "special category" of personal data. Automatically clustering faces and presenting them to a user for labeling is a form of biometric processing. Considerations:
- Is explicit consent required from photographed individuals?
- Must the system support "right to be forgotten" for faces that appear in groups?
- Should there be a mechanism to exclude specific faces from grouping?

The current system has no consent management, no opt-out mechanism for subjects, and no audit trail for biometric processing decisions beyond `FaceAssignmentEvent`.

### 6.3 Bias in Clustering

Face embedding models (including ArcFace/InsightFace) have documented biases:
- Lower accuracy for certain demographic groups
- Higher false-match rates across some ethnic groups
- Age-related accuracy degradation (especially for children)

The proposed feature would make these biases visible to users: "Why does the system group these two different people together?" or "Why can't the system find all photos of my child?" There is no mechanism in the proposal for communicating model limitations to users or allowing them to correct systematic errors.

---

## 7. Alternative Approaches

### 7.1 Enhance the Existing Clusters View (Low Risk, Medium Effort)

Instead of a new view, add to the existing Clusters view:
- A "Create Person from Cluster" button that converts a cluster to a named person
- Cluster confidence scoring (already computed by `calculate_cluster_confidence()`)
- Sort clusters by size and confidence
- Allow splitting a cluster into sub-groups
- Filter by cluster quality, size, date range

**Advantage**: Reuses existing clustering infrastructure, no new backend work, no threshold slider complexity.

### 7.2 "Quick Assign" Mode in Clusters (Low Risk, Low Effort)

Add a workflow to the cluster detail page:
1. Show the cluster of unknown faces
2. User types a name (with autocomplete from existing persons)
3. If new name: create Person + assign all faces in cluster
4. If existing name: assign all faces to that Person
5. Automatically trigger centroid recomputation and suggestion refresh

This solves the core problem (finding and labeling unknown persons) without any new clustering mechanism.

### 7.3 "Suggested New Persons" Using Existing HDBSCAN Results (Medium Risk, Low Effort)

The dual-mode clustering pipeline already identifies unknown clusters. Instead of building a new dynamic grouping system:
1. After each clustering run, extract clusters with high confidence and minimum size
2. Present these as "Suggested New Persons" in the existing Suggestions view
3. The user can accept (creating a Person) or dismiss
4. No threshold slider needed -- the system presents its best guesses
5. Over time, as users create persons, the unknown pool shrinks naturally

**Advantage**: Entirely backend-driven, no UX complexity, no real-time computation, uses existing data.

### 7.4 "Find Similar Unlabeled" From Any Face (Medium Risk, Medium Effort)

Instead of grouping all unlabeled faces at once, allow the user to:
1. Click any unlabeled face (from any view)
2. See similar unlabeled faces (Qdrant cosine search, which already works)
3. Select faces that match
4. Create a new person from the selection

This is essentially the "Find More" feature (`find-more-suggestions` endpoint in `face_suggestions.py`) but starting from an unlabeled face instead of a known person.

**Advantage**: Uses existing Qdrant search, no clustering needed, user controls the process, no threshold slider, works for singletons.

### 7.5 Automated Triage Queue (Low Risk, Medium Effort)

Instead of presenting all unknown faces at once, create a triage workflow:
1. System selects the highest-confidence unknown cluster
2. Presents it to the user: "These 8 faces may be the same person. Who is this?"
3. User can: Name them, Skip, Split, or Dismiss
4. System moves to the next-highest-confidence cluster
5. Progressive disclosure reduces cognitive load

**Advantage**: Gamified UX, no overwhelming displays, no threshold management, prioritizes high-value discoveries.

---

## 8. Risk Mitigation Recommendations

If the team proceeds with the proposed feature despite the concerns above, the following mitigations should be applied:

### 8.1 Technical Mitigations

1. **Use `IsEmptyCondition` consistently.** Fix `get_unlabeled_faces_with_embeddings` to use the same server-side filtering that `clusterer.py` already uses. This eliminates the full-scan problem.

2. **Pre-compute, do not compute on demand.** Run clustering as a background job (extending the existing pipeline), store results, and serve them statically. Do not attempt real-time re-clustering on threshold change.

3. **Implement cluster versioning.** Add a `clustering_version` field so the UI can detect when results have been recomputed and refresh accordingly. This prevents stale-display bugs.

4. **Cap unlabeled face count.** Set a hard limit (e.g., 5,000 faces) for the unknown grouping feature. Beyond this, require the user to filter by date range or photo directory first.

5. **Add DB-Qdrant reconciliation job.** A periodic background job that detects and fixes inconsistencies between DB `person_id` values and Qdrant `person_id` payloads.

### 8.2 UX Mitigations

1. **Replace the threshold slider with semantic controls.** Instead of a numeric slider, offer: "Show only high-confidence groups" / "Show more groups (may include less certain matches)" / "Show all potential groups."

2. **Sort groups by actionability.** Present large, high-confidence groups first. Push small, low-confidence groups to the bottom or hide them behind a "Show more" control.

3. **Add a group-splitting mechanism.** Users must be able to select faces within a group and split them into separate groups. Without this, incorrect merges at lower thresholds are irrecoverable.

4. **Provide inline person creation.** When the user decides a group is a new person, allow naming and creating the person directly from the group card without navigating away.

5. **Show group stability indicators.** If a group is likely to change with small threshold adjustments, indicate this visually. Stable groups deserve higher confidence from users.

### 8.3 Architectural Mitigations

1. **Do not create a parallel clustering system.** Extend `DualModeClusterer` to support multiple threshold levels rather than building a new clusterer.

2. **Reuse the existing cluster storage model.** The `FaceInstance.cluster_id` field and related infrastructure should be the single source of truth for groupings. Do not add a parallel grouping table.

3. **Treat this as a view over existing data, not a new data pipeline.** The unknown persons feature should read from the same clustering results that the Clusters view uses, with different filtering and presentation.

---

## 9. Questions the Team Must Answer Before Implementation

These are not rhetorical. Each question represents a design decision that, if left unresolved, will lead to contradictory implementations or scope creep.

### Architecture Questions

1. **Is this real-time clustering or pre-computed?** If pre-computed, at what thresholds? If real-time, how do we handle the O(N^2) memory requirement for pairwise distances on more than 10,000 faces?

2. **What is the relationship between this feature and the existing Clusters view?** Are they showing the same data with different UX, or different data entirely? If the same data, why two views?

3. **How does creating a new person from this view interact with the suggestion pipeline?** Does it trigger immediate re-computation of all suggestions, or is there a delay?

4. **What happens when the background clustering job runs while a user is actively using the unknown persons view?** Are results refreshed? Does the user lose their place?

### Data Questions

5. **How many unlabeled faces do we expect in a typical deployment?** This determines whether dynamic computation is feasible. 1,000 faces: easy. 50,000 faces: requires fundamentally different approach.

6. **What is the expected ratio of "true new persons" to "noise faces" (background people, one-time appearances)?** If 80% of unlabeled faces are noise, the feature will be 80% clutter.

7. **Should the system remember which groups the user has dismissed?** If yes, where is this stored? If no, dismissed groups reappear on every visit.

### UX Questions

8. **What does the user see when there are 500+ unknown groups?** Infinite scroll? Pagination? Forced filtering?

9. **Can the user partially accept a group?** (Select 6 of 8 faces as the same person, reject 2.) If yes, this is a complex multi-select interaction. If no, users must accept incorrect groupings or reject the entire group.

10. **How does this feature interact with the "Find More" button on existing person suggestions?** If both features can find the same face, which takes priority?

### Scope Questions

11. **Is the threshold slider a V1 requirement or a future enhancement?** Removing the slider dramatically simplifies the feature -- present the system's best groupings and let users act on them.

12. **Must this be a separate view, or could it be a section within the existing Suggestions page?** A tab or expandable section reduces navigation complexity.

13. **What is the minimum viable version of this feature?** Could we ship "Suggested New Persons" (alternative 7.3 above) first and evaluate whether users need more control before building the full dynamic grouping system?

---

## Summary of Key Risks (Ranked by Severity)

| # | Risk | Severity | Likelihood | Mitigation |
|---|------|----------|------------|------------|
| 1 | O(N^2) memory for real-time clustering at scale | Critical | High | Pre-compute, cap face count |
| 2 | Full-scan in `get_unlabeled_faces_with_embeddings` | High | Certain | Use `IsEmptyCondition` (fix exists in codebase) |
| 3 | Singleton faces never appear in groups | High | Certain | Lower min_cluster_size or use NN search |
| 4 | DB-Qdrant consistency gaps on assignment | High | High | Add reconciliation job |
| 5 | Threshold slider causes unpredictable phase transitions | Medium | High | Replace with semantic controls |
| 6 | Duplicate/conflicting views (Clusters vs Unknown Persons) | Medium | High | Merge views or clarify distinction |
| 7 | Temporal face variation splits same person into groups | Medium | Medium | Temporal-aware clustering |
| 8 | Privacy/GDPR compliance for automated face grouping | Medium | Medium | Add consent framework, opt-out |
| 9 | Background job race conditions with live view | Medium | Medium | Cluster versioning + optimistic locking |
| 10 | User decision fatigue with large numbers of groups | Low | High | Triage queue, progressive disclosure |

---

## Appendix A: Code References

| Component | File | Key Lines | Relevance |
|-----------|------|-----------|-----------|
| HDBSCAN Clusterer | `faces/clusterer.py` | 60-99 | Existing server-side filtered clustering |
| Dual-Mode Clusterer | `faces/dual_clusterer.py` | 44-126 | Supervised + unsupervised pipeline |
| Full-Scan Bug | `vector/face_qdrant.py` | 579-632 | Python-side filtering vs. available Qdrant filter |
| IsEmptyCondition Usage | `faces/clusterer.py` | 77-78 | Correct Qdrant filter (already exists) |
| Centroid Computation | `services/centroid_service.py` | N/A | Outlier trimming, staleness detection |
| Cluster Confidence | `services/face_clustering_service.py` | N/A | Pairwise similarity with sampling |
| Suggestion Accept | `api/routes/face_suggestions.py` | N/A | Non-blocking Qdrant sync after DB commit |
| DB Models | `db/models.py` | N/A | FaceInstance.cluster_id, Person, PersonCentroid |
| Frontend Clusters | `routes/faces/clusters/+page.svelte` | N/A | Existing cluster view |
| Frontend Suggestions | `routes/faces/suggestions/+page.svelte` | N/A | Existing suggestion workflow |

## Appendix B: Decision Matrix for Implementation Approach

| Approach | Risk | Effort | Reuse of Existing Code | User Value | Recommended? |
|----------|------|--------|----------------------|------------|-------------|
| 7.1 Enhance Clusters View | Low | Medium | High | Medium | Yes (V1) |
| 7.2 Quick Assign in Clusters | Low | Low | High | High | Yes (V1) |
| 7.3 Suggested New Persons | Medium | Low | High | High | Yes (V1) |
| 7.4 Find Similar Unlabeled | Medium | Medium | Medium | Medium | Yes (V2) |
| 7.5 Triage Queue | Low | Medium | Medium | High | Yes (V2) |
| Full Proposal (slider + new view) | High | High | Low | Medium | No (defer) |

---

*This analysis is based on code review of the actual codebase as of 2026-02-07 and is intended to strengthen the final design by identifying risks early. The goal is not to block the feature but to ensure it is built on solid architectural foundations with realistic expectations about complexity and user experience.*
