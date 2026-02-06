# UI/UX Analysis: Clustering & Suggestions System

**Date**: 2026-02-06
**Researcher**: UI/UX Researcher (Agent)
**Scope**: Frontend code analysis of face clustering, suggestions, and people management UI
**Codebase**: `image-search-ui/` (SvelteKit 2 + Svelte 5)

---

## Executive Summary

The image-search-ui frontend implements a comprehensive face clustering, suggestion, and person management system spanning approximately 25 Svelte components, 7 route pages, and a 1,650-line API client. The system handles the full lifecycle from face detection through clustering, suggestion review, and person assignment.

**Key Strengths:**
- Well-structured domain-specific API client (`faces.ts`) with clear type definitions
- Thoughtful keyboard accessibility on card and list components
- Batch operations (bulk accept/reject) reduce repetitive workflows
- Recent assignments panel with undo capability provides good error recovery
- Bidirectional face highlighting between image and sidebar in the detail modal

**Critical Findings:**
1. **Terminology inconsistency** is the single largest usability barrier -- the same concepts are labeled differently across pages ("Unknown Faces" vs "Needs Names" vs "Unidentified", "Clusters" vs "Face Groups")
2. **Navigation dead-ends** exist for noise-type faces -- they appear in the People view but are not clickable and have no actionable path forward
3. **Modal stacking reaches 3+ levels deep** (Suggestion Page > Centroid Dialog > Centroid Results > Detail Modal > Assignment Modal), creating disorientation
4. **Native browser `confirm()` dialogs** are used for destructive operations in 7 locations, breaking the UI's otherwise polished modal system
5. **Discoverability problems** with key features -- "Find More" is hidden behind a cryptic centroid icon, and unidentified persons are hidden by default on the People page
6. **Sequential API calls** for batch centroid accepts (no batch endpoint) create poor perceived performance for users accepting many suggestions
7. **Double-load risk** on the Suggestions page where `$effect` on filters may fire on mount AND on change
8. **No empty-state guidance** -- when cluster lists or suggestion lists are empty, the UI shows "no items" without explaining why or what action to take next

**Recommendation Priority**: The top 3 issues to address are terminology unification (#1), navigation coherence (#3 noise faces), and modal depth reduction. These are high-impact, moderate-effort improvements that would measurably improve the user experience for face management workflows.

---

## Section 1: Cluster Visualization & User Experience

### 1.1 Cluster Listing Page (`/faces/clusters`)

**File**: `src/routes/faces/clusters/+page.svelte` (608 lines)

The clusters page is titled "Unknown Faces" but lives at the route `/faces/clusters` and the navigation link in the layout says "Clusters" (line 62 of `+layout.svelte`). This is the first and most impactful terminology mismatch -- users clicking "Clusters" in the nav expect a clusters view, but the page header says "Unknown Faces," and neither term maps clearly to what users see (groups of similar unidentified faces).

**Confidence Score Presentation:**
The page displays cluster confidence as a percentage (e.g., "72% match") in `ClusterCard.svelte` (line 100-101), derived from the `clusterConfidence` field. However:
- The confidence input field on the clusters page (line 148-165) uses a raw decimal value (0.6) but the display shows percentage (60%). Users must mentally convert between these representations.
- The custom confidence input requires clicking an "Apply" button -- there is no indication that the value is not yet applied after typing, which could lead users to believe their change took effect when it did not.
- The confidence threshold meaning is not explained anywhere in the UI -- users may not understand whether "60% confidence" means "60% chance these faces are the same person" or something else entirely.

**Cluster Size Communication:**
`ClusterCard.svelte` (220 lines) shows up to 6 sample face thumbnails (line 129), face count (line 172), truncated cluster ID (line 154-162), and an average quality score. The representative face is prioritized to the front of the preview (lines 115-128). However:
- There is no visual tier indicator for cluster quality (high/medium/low)
- Clusters with 2 faces look nearly identical to clusters with 200 faces -- the 6-thumbnail preview cap means size differences are invisible until reading the face count number
- The truncated cluster ID (UUID) provides zero meaningful information to end users

**Pagination:**
The page uses a "Load More" button pattern (PAGE_SIZE=100, line 16) rather than traditional pagination. While infinite-scroll-style loading is acceptable, 100 items per page is a large batch that may cause noticeable rendering lag. The page persists sort order and confidence settings to localStorage via `localSettings`, which is good.

**Sort Options:**
Sort options include face count (desc/asc), cluster confidence, and average quality. These are reasonable for a power-user audience. The sort selector uses a native `<select>` element, which is functional but inconsistent with the shadcn/ui Select components used elsewhere.

### 1.2 Cluster Detail Page (`/faces/clusters/[clusterId]`)

**File**: `src/routes/faces/clusters/[clusterId]/+page.svelte` (785 lines)

This is the detail view for a single cluster. The page is well-structured with:
- Breadcrumb navigation ("Back to Clusters")
- Quality distribution stats (excellent/good/fair/poor breakdown)
- Client-side face pagination (24 per page)
- "Label as Person" and "Split Cluster" actions

**Issues Found:**

1. **Split Cluster uses native `confirm()` dialog** (line 175): `if (!confirm('This will attempt to split this cluster into smaller sub-clusters. Continue?'))`. This is jarring because every other dialog in the application uses the shadcn/ui Dialog component. The split operation is destructive and irreversible, making this a poor choice for a native browser dialog that cannot be styled or enhanced with additional information.

2. **Split button disabled threshold** (line 6 faces minimum): The UI disables the "Split Cluster" button if there are fewer than 6 faces, but provides no tooltip or explanation for why it is disabled. Users see a grayed-out button with no context.

3. **Quality distribution categories** use terms "excellent/good/fair/poor" but the thresholds are not visible to the user. A face with quality score 0.79 falls into "good" but users have no way to know the boundaries.

4. **No batch face operations**: On the cluster detail page, individual faces can be viewed in a lightbox but there is no way to select multiple faces for bulk operations (e.g., "remove these 5 bad-quality faces from the cluster").

### 1.3 Cluster Card Component

**File**: `src/lib/components/faces/ClusterCard.svelte` (220 lines)

The card component is well-implemented for accessibility:
- Keyboard navigation via Enter/Space (lines 188-195)
- Proper `role="button"` semantics
- Face count displayed clearly

**Missing UX Elements:**
- No hover preview expansion -- users must click into the detail page to see more than 6 faces
- No action indicator -- the card does not visually suggest that clicking leads to a detail page
- No visual distinction between high-confidence and low-confidence clusters beyond the numeric score

---

## Section 2: Suggestion Flow UX

### 2.1 Suggestions Page (`/faces/suggestions`)

**File**: `src/routes/faces/suggestions/+page.svelte` (682 lines)

The suggestions page groups face suggestions by person using `SuggestionGroupCard` components. It implements a two-column layout: main content area and a `RecentlyAssignedPanel` sidebar.

**End-to-End Suggestion Flow:**
1. User sees grouped suggestions (grouped by suggested person)
2. Each group shows face thumbnails with confidence scores
3. User can accept/reject individually or in bulk
4. After bulk accept, the system auto-triggers "Find More" jobs (lines 400-430)
5. Recently accepted assignments appear in the sidebar with undo capability

**Flow Issues:**

1. **Double-load risk** (lines 428-433): The `$effect` that loads suggestions fires when `statusFilter`, `currentPage`, `groupsPerPage`, or `personFilter` change. However, because `$effect` also fires on component mount, and `onMount` also calls `loadGroupedSuggestions()`, there is a risk of duplicate API calls on initial page load. The `$effect` at line 428 watches:
   ```
   $effect(() => {
     statusFilter; currentPage; groupsPerPage; personFilter;
     loadGroupedSuggestions();
   });
   ```
   This pattern causes the load function to be called once on mount (via `onMount`) AND once when the `$effect` first runs (Svelte 5 effects run on first tracking). This is a potential race condition that could result in flickering or double-rendering of results.

2. **"Find More" button discoverability** (in `SuggestionGroupCard.svelte`, line 255-260): The "Find More" feature is triggered by a button with a centroid icon (SVG of concentric circles). There is no text label on the button -- only a tooltip appears on hover. This critical feature for discovering additional face matches is essentially hidden behind an unlabeled icon that users must discover by hovering.

3. **Pagination confusion**: The page uses "Previous/Next" pagination with a configurable "groups per page" selector (10/20/50). However, the pagination controls show page N of M at the group level, while the `totalSuggestions` count in the header refers to individual suggestions, not groups. Users see "142 pending suggestions" but pagination shows "Page 1 of 3 (10 groups)" -- the relationship between these numbers is unclear.

4. **Filter state persistence**: The status filter defaults to "pending" on each page load. There is no persistence of the user's last-used filter. If a user is reviewing rejected suggestions and navigates away, they must re-select "rejected" on return. The `localSettings` store is available and used elsewhere but not employed here.

### 2.2 Suggestion Group Card

**File**: `src/lib/components/faces/SuggestionGroupCard.svelte` (386 lines)

This component groups suggestions for a single person and provides bulk accept/reject with individual selection.

**Strengths:**
- Sort order prioritizes multi-prototype matches (lines 163-175), putting higher-confidence suggestions first
- Indeterminate checkbox state when some-but-not-all items are selected
- Error message display within the card context (not a global alert)

**Issues:**
- The "Find More" button (centroid icon) only appears when the person has 2 or more labeled faces (line 249). There is no explanation of this prerequisite -- a person with 1 labeled face simply does not see the button, with no indication of why or what they could do to unlock the feature.
- The card header shows "X suggestions" but does not indicate how many are selected until the user starts selecting, at which point "Y selected" appears. A persistent selection indicator (e.g., "0 of X selected") would reduce cognitive load.
- Bulk accept/reject buttons are at the top of the card. For cards with many suggestions, the user must scroll back up to trigger the bulk action after reviewing thumbnails at the bottom.

### 2.3 Suggestion Detail Modal

**File**: `src/lib/components/faces/SuggestionDetailModal.svelte` (1008 lines)

This is the largest component in the face management system. It opens as a near-full-screen modal (98vw x 98vh) showing the face in context of the full image.

**Layout:**
- Left side: Full image with SVG bounding box overlays (`ImageWithFaceBoundingBoxes`)
- Right side: `FaceListSidebar` showing all faces in the image + primary suggestion details

**Strengths:**
- Bidirectional face highlighting -- clicking a face in the sidebar highlights it in the image, and vice versa
- Shows all faces in the image, not just the suggested one, allowing users to understand context
- Loads suggestions for other unknown faces in the same image (lines 270-316), enabling batch processing
- Copy path to clipboard functionality
- Undo assignment capability with confirmation dialog

**Issues:**

1. **Component size** (1008 lines): This component far exceeds the project's own 300-line guideline stated in CLAUDE.md. The complexity makes it difficult to reason about state management, and the component handles at least 8 distinct responsibilities: image display, face highlighting, suggestion accept/reject, face assignment, prototype pinning, undo operations, suggestion loading for other faces, and navigation between suggestions.

2. **Explicit error handling gaps** (lines 396, 526): Two locations contain the comment `// Error handling could be improved here`. At line 396, a failed face assignment logs to console but shows no user-facing feedback. At line 526, a failed undo operation similarly lacks user notification.

3. **Modal stacking depth**: From the Suggestions page, a user can reach 4+ levels of modal stacking:
   - Level 1: SuggestionDetailModal (via thumbnail click)
   - Level 2: PersonAssignmentModal (via "Assign" button in sidebar)
   - Level 3: Potentially another dialog within assignment

   From the SuggestionGroupCard's "Find More" flow:
   - Level 1: ComputeCentroidsDialog
   - Level 2: CentroidResultsDialog
   - Level 3: SuggestionDetailModal (via thumbnail click in results)
   - Level 4: PersonAssignmentModal (via "Assign" in detail sidebar)

   Each modal layer adds z-index complexity. The PersonAssignmentModal uses `z-[60]` (line 102 of `PersonAssignmentModal.svelte`) to ensure it stacks above the detail modal, but this manual z-index management is fragile.

4. **No keyboard shortcut for accept/reject**: For the primary suggestion, the user must click small buttons. Keyboard shortcuts (e.g., A for accept, R for reject, N for next) would significantly accelerate the review workflow, especially for users processing hundreds of suggestions.

### 2.4 Centroid-Based "Find More" Flow

The centroid flow spans three components:

1. **`ComputeCentroidsDialog.svelte`** (254 lines): Multi-step dialog (options > computing > searching > results)
2. **`CentroidResultsDialog.svelte`** (395 lines): Grid of results with bulk accept
3. **`SuggestionDetailModal.svelte`**: Reused for individual face detail

**Flow:**
1. User clicks centroid icon on a suggestion group card
2. ComputeCentroidsDialog opens with options (similarity threshold slider, max results, clustering toggle)
3. User clicks "Compute & Search" -- dialog transitions through computing > searching states
4. Results flow to CentroidResultsDialog (90vw x 90vh)
5. User reviews grid, selects faces, bulk accepts

**Issues:**

1. **No batch API endpoint for centroid accepts** (line 121-139 of `CentroidResultsDialog.svelte`): Accepting suggestions uses sequential `assignFaceToPerson()` calls in a for-loop. For 50 suggestions, this means 50 sequential HTTP requests, which is slow and error-prone. The component handles partial failures gracefully (success/fail counts), but the UX of watching a spinner for each sequential request is poor.

2. **"Reject" for centroid suggestions is purely cosmetic** (line 221-226 of `CentroidResultsDialog.svelte`): The handleDetailReject function shows a toast "Suggestion removed from list" but does not persist the rejection to the backend. If the user re-runs centroids, the same face will appear again. This is misleading -- users expect "reject" to be permanent.

3. **Similarity threshold conceptual overload**: The ComputeCentroidsDialog has a similarity slider (0.5-0.85) while the suggestions page has a confidence threshold. These are different measurements (cosine similarity vs prototype match confidence) but are presented similarly. Users have no way to understand the relationship between these thresholds.

4. **The "Compute Centroids" button in ComputeCentroidsDialog requires 2+ labeled faces** (line 164-166), mirroring the SuggestionGroupCard's prerequisite. But the dialog is the only place this is communicated -- via disabled state text: "Need at least 2 labeled faces."

### 2.5 Recently Assigned Panel

**File**: `src/lib/components/faces/RecentlyAssignedPanel.svelte` (271 lines)

This sticky sidebar panel on the Suggestions page shows recent face assignments with undo capability.

**Strengths:**
- Collapsible card with reasonable max of 10 items
- Shows face thumbnail, person name, photo filename, and relative time
- Per-item loading state for undo operations
- Clean empty state: "No recent assignments"

**Issues:**
- The panel is only visible on the Suggestions page. If a user assigns faces from the Person Detail page or the Cluster Detail page, those assignments do not appear here, and there is no undo path from those contexts.
- The undo operation has no confirmation step -- a single click immediately unassigns, which could be disruptive if clicked accidentally. Given that "accept" requires explicit confirmation, this asymmetry is surprising.

---

## Section 3: Critical Usability Issues

### 3.1 Navigation Inconsistencies

**Top Navigation** (`+layout.svelte`, lines 57-66):
```
Search | People | Suggestions | Clusters | Categories | Training | Queues | Vectors | Admin
```

The navigation presents "People," "Suggestions," and "Clusters" as three separate top-level concerns. However:
- "People" (`/people`) shows identified persons, unidentified clusters, AND noise faces -- it is a superset of "Clusters"
- "Clusters" (`/faces/clusters`) shows only unlabeled clusters -- it is a subset of "People"
- "Suggestions" (`/faces/suggestions`) shows suggestions grouped by person -- it relates to "People" but has its own entry

This creates confusion about where to go for face management tasks. A user wanting to "see all unidentified faces" could go to either People (toggle "Needs Names") or Clusters, and they would see overlapping but not identical content because the People page uses the unified API while Clusters uses the cluster-specific API.

**Cross-Page Navigation Breaks:**
- From People page, clicking an "unidentified" person navigates to `/faces/clusters/{id}` -- crossing from the "People" section into the "Clusters" section without any breadcrumb context
- From the Cluster detail page, "Back to Clusters" goes to `/faces/clusters`, not back to `/people` where the user may have come from
- The Person detail page (`/people/{id}`) has "Back to People" but the Cluster detail page (`/faces/clusters/{id}`) has "Back to Clusters" -- they use different navigation paradigms even though both are accessible from the People page

### 3.2 Terminology Inconsistencies

The following terms are used for the same or closely related concepts:

| Concept | Term Used in Location | Location |
|---|---|---|
| Unidentified face groups | "Unknown Faces" | Clusters page title |
| Unidentified face groups | "Clusters" | Nav link, breadcrumbs |
| Unidentified face groups | "Needs Names" | People page badge (`UnifiedPersonCard.svelte` line 84) |
| Unidentified face groups | "Unidentified" | People page section header |
| Low-quality ungrouped faces | "Noise" | Internal type name |
| Low-quality ungrouped faces | "Review" | Badge label (`UnifiedPersonCard.svelte` line 87) |
| Low-quality ungrouped faces | "Unknown Faces" | People page section header |
| Matching confidence | "X% match" | `ClusterCard.svelte` line 101 |
| Matching confidence | "X% confidence" | `UnifiedPersonCard.svelte` line 177 |
| Matching confidence | "Score" | `CentroidResultsDialog.svelte` line 173 |
| Matching confidence | "Similarity" | `ComputeCentroidsDialog.svelte` slider |
| Face suggestion | "Suggestion" | Suggestions page |
| Face suggestion | "Centroid Suggestion" | `CentroidResultsDialog.svelte` title |
| Face suggestion | "Match" | Various tooltip text |

This inconsistency means a user must learn multiple terms for the same concept depending on which page they are viewing.

### 3.3 Missing User Feedback

1. **Silent failures in SuggestionDetailModal** (lines 396, 526): Two `console.error` calls with explicit `// Error handling could be improved here` comments. Users performing face assignments or undo operations may experience failures with no visible notification.

2. **Disabled button states without explanation**: Multiple locations disable buttons without tooltip explanations:
   - Split Cluster button disabled when < 6 faces (cluster detail page)
   - "Find More" button hidden (not disabled) when < 2 labeled faces (SuggestionGroupCard)
   - Compute Centroids "Compute & Search" disabled when < 2 labeled faces (shows text but only in dialog)

3. **Loading states inconsistency**: Some operations show inline spinners (CentroidResultsDialog processing), some show skeleton loaders (clusters page), some show full-page spinners (person detail page), and some show no loading state at all (PersonAssignmentModal's person list initial load briefly shows "Loading persons...").

4. **No progress indication for sequential operations**: The CentroidResultsDialog processes accepts sequentially but shows only a generic spinner overlay on each thumbnail. There is no "3 of 50 processed" progress indicator.

### 3.4 Native `confirm()` Dialog Usage

Seven locations use the native browser `confirm()` dialog instead of the application's shadcn/ui Dialog system:

1. `routes/people/[personId]/+page.svelte:276` -- Unpin prototype
2. `routes/people/[personId]/+page.svelte:290` -- Delete prototype
3. `routes/faces/clusters/[clusterId]/+page.svelte:175` -- Split cluster
4. `components/training/TrainingSessionList.svelte:39` -- Delete training session
5. `components/faces/PhotoPreviewModal.svelte:298` -- Unassign face
6. `components/faces/FaceDetectionSessionCard.svelte:113` -- Cancel session
7. `components/faces/PersonPhotosTab.svelte:89` -- Remove photos

All of these are destructive operations where the native dialog provides:
- No styling consistency with the rest of the UI
- No ability to show contextual information (e.g., "This will affect 47 faces")
- No ability to add a "Don't ask again" option
- Browser-dependent appearance and positioning

### 3.5 Noise Face Dead-End

On the People page (`/people/+page.svelte`), noise faces appear in the "Unknown Faces" section with a "Review" badge and italic text "These faces need individual review and manual grouping." However:

- **Noise faces are not clickable** (`UnifiedPersonCard.svelte` line 26: `isClickable = person.type !== 'noise'`)
- **Noise faces have no "Assign Name" button** (line 194: `person.type !== 'noise'` guard)
- **There is no route for individual noise face review** -- no `/faces/noise/{id}` or equivalent exists

This means noise faces are displayed but provide zero actionable path forward. Users see them but cannot do anything with them. The "Review" badge is especially misleading because it implies an action is available.

### 3.6 Default Visibility Hides Key Content

On the People page, `showUnidentified` defaults to `false` (inferred from the filter toggle behavior). This means:
- New users visiting `/people` for the first time see only identified persons
- The primary workflow for face management (identifying unknown faces) requires toggling a filter that users may not discover
- The "Needs Names" count is shown in the header stats but the actual cards are hidden until the toggle is activated

This is a significant discoverability problem -- the most important action items (unidentified clusters needing names) are hidden by default.

---

## Section 4: Information Architecture

### 4.1 Conceptual Model: Images > Faces > Clusters > People

The application models a pipeline: images are processed for face detection, detected faces are grouped into clusters, and clusters are labeled as persons. The UI exposes this pipeline across multiple pages:

```
Image (asset)
  |-- Face Instance (detected face with bounding box)
       |-- Cluster (group of similar faces)
       |    |-- Person (named identity)
       |-- Suggestion (proposed person match)
       |-- Prototype (reference face for a person at an age era)
```

**How well does the UI communicate this model?**

The People page (`/people`) is the closest to a unified view, showing identified persons, unidentified clusters, and noise faces in one place. However, it does not explain the relationship between these categories. A new user would not understand that:
- "Identified" means someone manually labeled a cluster
- "Needs Name" means the clustering algorithm grouped faces that look similar
- "Unknown Faces" (noise) means faces that could not be confidently grouped

There is no onboarding, help text, or documentation within the UI explaining the pipeline or what actions to take at each stage.

### 4.2 Type System Clarity

The API client (`faces.ts`) defines clear TypeScript types that map well to the domain model:

- `FaceInstance` -- individual detected face
- `ClusterSummary` / `ClusterDetailResponse` -- face groupings
- `Person` -- named identity
- `FaceSuggestion` -- proposed match
- `CentroidSuggestion` -- centroid-based match (different type from FaceSuggestion)
- `Prototype` -- reference face with age era
- `UnifiedPersonResponse` -- combined view (identified + unidentified + noise)

The type system is well-organized but the existence of TWO different suggestion types (`FaceSuggestion` and `CentroidSuggestion`) with different shapes creates adaptation complexity. `CentroidResultsDialog.svelte` includes an `adaptCentroidToFaceSuggestion()` function (lines 180-201) to bridge the gap, creating synthetic `FaceSuggestion` objects with dummy values:
```typescript
id: 0,  // Not relevant for display-only modal
path: '',  // Not available in CentroidSuggestion
bboxX: null, bboxY: null, bboxW: null, bboxH: null,
```

This adaptation means the detail modal receives incomplete data for centroid suggestions -- bounding box data is missing, so the image bounding box overlay will not work correctly for centroid-origin suggestions.

### 4.3 Page-Level Information Architecture

| Page | Route | Purpose | Relationship to Others |
|---|---|---|---|
| People | `/people` | Unified person/cluster view | Superset of Clusters content |
| Clusters | `/faces/clusters` | Unknown face clusters | Subset of People "Needs Names" |
| Cluster Detail | `/faces/clusters/{id}` | Single cluster faces | Linked from both People and Clusters |
| Suggestions | `/faces/suggestions` | Review face matches | Independent; cross-references People |
| Person Detail | `/people/{id}` | Individual person management | Linked from People page |
| Sessions | `/faces/sessions` | Face detection jobs | Independent; feeds clusters |

The overlap between People and Clusters creates two entry points to the same underlying data. This is not inherently wrong -- it could serve as both a "power user" (Clusters) and "simplified" (People) view -- but the current UI does not frame them this way. Both pages expose similar complexity.

### 4.4 Person Detail Page Architecture

**File**: `src/routes/people/[personId]/+page.svelte` (1911 lines with styles)

The person detail page is the most feature-rich individual page in the application. It contains:
- Person header with avatar, name, stats, status badge
- Birth date editing (inline form)
- Tab navigation: Photos | Prototypes
- Merge person modal
- Compute Centroids dialog flow
- Lightbox for photo preview
- Prototype grid with pin/unpin/delete operations
- Temporal coverage visualization
- Recompute prototypes and re-scan for suggestions actions

**Issues:**

1. **Prototype count always shows 0** (line 125): The code comments note that `PersonDetailResponse` does not have `prototypeCount`, so it defaults to 0. This means the header always shows "0 prototypes" even when prototypes exist, which is incorrect and misleading.

2. **The page does NOT show suggestions for this person.** To see suggestions for a specific person, users must go to the Suggestions page and search by person name. There is no "Suggestions" tab on the Person detail page, which breaks the expectation of seeing all person-related information in one place.

3. **`fetchAllPersons` for merge targets** (line 163): When the page loads, it fetches ALL persons to populate the merge target list. For a system with thousands of persons, this creates a potentially large and slow initial load. The `fetchAllPersons` function in `faces.ts` uses a pagination loop with pageSize=1000.

4. **Tab state in URL** (line 226-231): The active tab is stored in the URL query parameter (`?tab=photos`), which is good for shareable links. However, the code includes a special redirect: `urlTab === 'faces' ? 'photos' : urlTab` (line 42), suggesting a past rename from "Faces" to "Photos" that was not fully cleaned up.

5. **Mixed styling approaches**: The person detail page uses custom CSS classes (`.person-header`, `.prototype-card`, etc.) while other pages in the application use Tailwind CSS utility classes. This inconsistency suggests the person detail page was built before the migration to Tailwind/shadcn and has not been updated.

### 4.5 The FaceListSidebar as Information Hub

**File**: `src/lib/components/faces/FaceListSidebar.svelte` (534 lines)

The FaceListSidebar serves as a critical information display in the SuggestionDetailModal. It shows:
- All faces detected in the current image
- Color-coded indicators for each face
- Person name or "Unknown" label
- Detection confidence ("IsFace: X%") and quality score
- Suggestion hints with accept button for unknown faces
- "Pin as Prototype" button for assigned faces
- Bulk accept button when multiple suggestions exist

**Terminology in the sidebar:**
- "IsFace: 85%" -- this label is unusual and may confuse users. It appears to represent detection confidence (how confident the model is that this region is a face), but users might interpret it as identity confidence.
- The sidebar shows `detectionConfidence` (is this a face?) separate from suggestion `confidence` (is this person X?). These are different measurements but both displayed as percentages, which could cause confusion.

### 4.6 API Architecture Observations

The `faces.ts` API client (1,652 lines) reveals an important architectural observation: the centroid-related API functions (`computeCentroids`, `getCentroids`, `getCentroidSuggestions`, `deleteCentroids`) use raw `fetch()` calls (lines 1563-1651) instead of the centralized `apiRequest()` helper used by all other API functions. This means:
- Centroid API errors do not go through the standard `ApiError` class
- Error messages from centroid endpoints have a different format (they throw `new Error(error.detail)` instead of `new ApiError(...)`)
- The centralized error handling in `apiRequest()` (lines 23-58) is bypassed for centroid operations

This inconsistency could lead to different error behavior for centroid operations vs all other face operations, potentially manifesting as different error display patterns in the UI.

---

## Prioritized Recommendations (Top 10)

### 1. Unify Terminology Across All Pages [HIGH IMPACT / MEDIUM EFFORT]

**Problem**: The same concepts have 3-5 different labels depending on the page.

**Recommendation**: Establish a glossary and apply consistently:
- "Person" (not "Individual" or "Identity") for named entities
- "Face Group" or "Cluster" (pick one) for unlabeled face groupings
- "Ungrouped Faces" (not "Noise", "Unknown", or "Review") for low-confidence singles
- "Match Confidence" (not "score", "similarity", "match %") for all confidence metrics

**Files affected**: `UnifiedPersonCard.svelte`, `ClusterCard.svelte`, `CentroidResultsDialog.svelte`, `ComputeCentroidsDialog.svelte`, clusters `+page.svelte`, people `+page.svelte`

### 2. Add Actionable Path for Noise Faces [HIGH IMPACT / MEDIUM EFFORT]

**Problem**: Noise faces are displayed on the People page but are dead-end items with no clickable action.

**Recommendation**: Either (a) create a dedicated noise face review page where individual faces can be assigned, or (b) make noise-type cards expandable in-place to show individual faces with assignment buttons. The current "Review" badge implies an action that does not exist.

**Files affected**: `UnifiedPersonCard.svelte`, `+page.svelte` (people route), potentially new route or component

### 3. Reduce Modal Stacking Depth [HIGH IMPACT / HIGH EFFORT]

**Problem**: Users can reach 4+ levels of stacked modals, causing disorientation and z-index management issues.

**Recommendation**: Convert the Centroid Results and Suggestion Detail views into full pages (or slide-over panels) rather than stacked modals. The 90vw/98vw modal sizes already fill the viewport -- they are effectively pages rendered as modals. A page-based approach would also enable URL-based deep linking to specific suggestions.

**Files affected**: `CentroidResultsDialog.svelte`, `SuggestionDetailModal.svelte`, `ComputeCentroidsDialog.svelte`, suggestion and person routes

### 4. Replace All Native `confirm()` Dialogs [MEDIUM IMPACT / LOW EFFORT]

**Problem**: 7 locations use native browser `confirm()` for destructive operations, breaking visual consistency.

**Recommendation**: Create a reusable `ConfirmDialog.svelte` component (or use shadcn/ui's AlertDialog) and replace all native confirm calls. Include contextual information about what will be affected (e.g., "This will split 47 faces into smaller groups").

**Files affected**: 7 files listed in Section 3.4

### 5. Make "Find More" Feature Discoverable [HIGH IMPACT / LOW EFFORT]

**Problem**: The "Find More" centroid feature is behind an unlabeled icon button that users must hover to understand.

**Recommendation**: Add a text label to the button ("Find More Matches") and show it prominently in the suggestion group header. When the feature is unavailable (< 2 labeled faces), show a disabled button with tooltip explaining the prerequisite rather than hiding it entirely.

**Files affected**: `SuggestionGroupCard.svelte`

### 6. Show Unidentified Clusters by Default on People Page [MEDIUM IMPACT / LOW EFFORT]

**Problem**: `showUnidentified` defaults to false, hiding the primary action items (unidentified face clusters) from new users.

**Recommendation**: Default to showing all categories (identified, unidentified, noise), or at minimum show identified and unidentified by default. The noise category can remain hidden by default since those items have no actions.

**Files affected**: `+page.svelte` (people route)

### 7. Add Empty State Guidance [MEDIUM IMPACT / LOW EFFORT]

**Problem**: Empty states show "No items" without explaining why or what to do next.

**Recommendation**: Each empty state should include:
- Why the list might be empty (e.g., "No suggestions found. Suggestions appear after face detection and clustering.")
- What action to take (e.g., "Run face detection on the Sessions page to generate clusters.")
- A link to the relevant next step

**Files affected**: Clusters `+page.svelte`, Suggestions `+page.svelte`, People `+page.svelte`

### 8. Fix Prototype Count Display on Person Detail [LOW IMPACT / LOW EFFORT]

**Problem**: Person detail header always shows "0 prototypes" because `PersonDetailResponse` lacks `prototypeCount`.

**Recommendation**: Either (a) add `prototypeCount` to the backend's `PersonDetailResponse`, or (b) derive the count from the loaded prototypes list (`prototypes.length`) and display that instead. The data is already loaded on the Prototypes tab.

**Files affected**: `routes/people/[personId]/+page.svelte` line 125

### 9. Add Keyboard Shortcuts for Suggestion Review [MEDIUM IMPACT / MEDIUM EFFORT]

**Problem**: Reviewing hundreds of suggestions requires repetitive clicking with no keyboard acceleration.

**Recommendation**: Implement keyboard shortcuts in the SuggestionDetailModal:
- `A` / `Enter` -- Accept primary suggestion
- `R` / `Backspace` -- Reject primary suggestion
- `N` / Right Arrow -- Next suggestion
- `P` / Left Arrow -- Previous suggestion
- `Escape` -- Close modal

**Files affected**: `SuggestionDetailModal.svelte`

### 10. Standardize Centroid API Error Handling [LOW IMPACT / LOW EFFORT]

**Problem**: Centroid API functions use raw `fetch()` instead of the centralized `apiRequest()` helper, creating inconsistent error behavior.

**Recommendation**: Refactor `computeCentroids()`, `getCentroids()`, `getCentroidSuggestions()`, and `deleteCentroids()` in `faces.ts` to use the `apiRequest()` helper, ensuring consistent error formatting via `ApiError` class.

**Files affected**: `src/lib/api/faces.ts` lines 1559-1651

---

## Appendix: Files Examined

| File | Path | Lines | Key Findings |
|---|---|---|---|
| Layout | `src/routes/+layout.svelte` | 221 | 9 nav items, health indicator |
| Clusters Page | `src/routes/faces/clusters/+page.svelte` | 608 | "Unknown Faces" title, Load More pagination |
| Cluster Detail | `src/routes/faces/clusters/[clusterId]/+page.svelte` | 785 | Native confirm() for split, quality stats |
| Suggestions Page | `src/routes/faces/suggestions/+page.svelte` | 682 | Double-load risk, auto Find More |
| People Page | `src/routes/people/+page.svelte` | 697 | 3-tier view, hidden unidentified default |
| Person Detail | `src/routes/people/[personId]/+page.svelte` | 1911 | Prototype count bug, mixed styling |
| ClusterCard | `src/lib/components/faces/ClusterCard.svelte` | 220 | Good a11y, no action indicator |
| SuggestionGroupCard | `src/lib/components/faces/SuggestionGroupCard.svelte` | 386 | Hidden Find More, bulk operations |
| SuggestionDetailModal | `src/lib/components/faces/SuggestionDetailModal.svelte` | 1008 | Exceeds 300-line guideline, error gaps |
| FindMoreResultsDialog | `src/lib/components/faces/FindMoreResultsDialog.svelte` | 202 | No error recovery |
| ComputeCentroidsDialog | `src/lib/components/faces/ComputeCentroidsDialog.svelte` | 254 | Multi-step flow, 2-face prerequisite |
| CentroidResultsDialog | `src/lib/components/faces/CentroidResultsDialog.svelte` | 395 | Sequential accepts, cosmetic reject |
| RecentlyAssignedPanel | `src/lib/components/faces/RecentlyAssignedPanel.svelte` | 271 | Undo without confirm, suggestions-only |
| UnifiedPersonCard | `src/lib/components/faces/UnifiedPersonCard.svelte` | 208 | Noise dead-end, type badges |
| FaceListSidebar | `src/lib/components/faces/FaceListSidebar.svelte` | 534 | Bidirectional highlighting, bulk accept |
| PersonAssignmentModal | `src/lib/components/faces/PersonAssignmentModal.svelte` | 205 | MRU sorting, create-and-assign |
| Faces API Client | `src/lib/api/faces.ts` | 1652 | Raw fetch for centroids, comprehensive types |
| Frontend Types | `src/lib/types.ts` | 102 | Re-exports, SearchFilters |

---

## Methodology

This analysis was conducted through systematic code review of the `image-search-ui/` frontend codebase. The approach:

1. **Discovery**: Used Glob and Grep to identify all face/cluster/suggestion-related components, routes, stores, and API modules
2. **Deep Reading**: Read 18 key files in full to understand component structure, state management, user flows, and edge cases
3. **Pattern Identification**: Identified recurring issues (terminology, confirm dialogs, error handling gaps) across the codebase
4. **Flow Tracing**: Traced user journeys through the suggestion review pipeline, from clusters page through detail modals to person assignment
5. **Cross-Reference**: Compared navigation labels, page titles, component labels, and type names to identify inconsistencies

No automated testing or visual inspection was performed -- all findings are based on source code analysis. Some findings (e.g., the double-load risk) would need runtime verification to confirm severity.
