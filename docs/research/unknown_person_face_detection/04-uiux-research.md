# UI/UX Research: Unlabeled Face Groups View

> **Research Agent**: UI/UX Specialist
> **Date**: 2026-02-07
> **Status**: Complete
> **Output Location**: `docs/research/unknown_person_face_detection/04-uiux-research.md`

---

## Executive Summary

This document presents a comprehensive UI/UX analysis for the proposed "Unlabeled Face Groups" feature -- a new view that surfaces groupings of unassigned faces that likely belong to the same (yet uncreated) person. The existing Face Suggestions view serves known persons exclusively; the existing Clusters view groups unknown faces but lacks the workflow for discovering and creating new person identities. The proposed solution bridges this gap by introducing a tabbed sub-view within the Suggestions page, leveraging the project's existing shadcn/ui Tabs component, Svelte 5 runes-based state management, and established UI patterns (group cards, confidence controls, detail modals, sidebar panels).

**Key Design Decisions:**
1. **View Switching**: Use shadcn/ui `Tabs` component at the top of the Face Suggestions page to toggle between "Known Persons" (existing) and "Unlabeled Groups" (new).
2. **Similarity Threshold Control**: Adapt the Clusters view's confidence preset+custom pattern. Default at 0.85 (high confidence) with user-adjustable slider.
3. **Group Display**: Reuse `SuggestionGroupCard` layout patterns with adaptations for unnamed groups (auto-generated labels like "Group 1", "Group 2").
4. **Group Actions**: Enable "Create Person" workflow from each group, batch accept/reject, and merge groups.
5. **Persistence**: Use `localSettings` store for threshold, sort order, and active tab preferences.

---

## 1. Current UI Analysis: Face Suggestions View

### 1.1 Page Structure

**File**: `src/routes/faces/suggestions/+page.svelte` (688 lines)

The Face Suggestions page follows a two-column grid layout:

```
+------------------------------------------------------+----------+
|  Header: "Face Suggestions"       Total count badge  |          |
+------------------------------------------------------+          |
|  Filters Bar:                                        |  Sidebar |
|  [Status] [Person Search] [Show N] [Select] [Bulk]  | Recently |
+------------------------------------------------------+ Assigned |
|  SuggestionGroupCard (Person A)                      |  Panel   |
|    [Checkbox] Person Name | Accept All | Reject All  | (320px)  |
|    [thumb] [thumb] [thumb] [thumb]                   |          |
+------------------------------------------------------+          |
|  SuggestionGroupCard (Person B)                      |          |
|    ...                                               |          |
+------------------------------------------------------+          |
|  Pagination: [Prev] Page X of Y [Next]              |          |
+------------------------------------------------------+----------+
```

**Layout CSS:**
```css
.page-layout {
  display: grid;
  grid-template-columns: 1fr 320px;
  gap: 1.5rem;
  align-items: start;
}
```

At breakpoints <= 1024px, the grid collapses to a single column with the sidebar moved above the content (`order: -1`).

### 1.2 State Management

The page manages extensive state using Svelte 5 runes:

| State Variable | Type | Purpose |
|---|---|---|
| `groupedResponse` | `GroupedSuggestionsResponse \| null` | API response with grouped suggestions |
| `settings` | `FaceSuggestionSettings` | Backend pagination settings |
| `groupsPerPage` | `10 \| 20 \| 50` | Groups shown per page (persisted via `localSettings`) |
| `page` | `number` | Current page number |
| `statusFilter` | `string` | Filter by suggestion status (pending/accepted/rejected) |
| `personFilter` | `string \| null` | Filter by specific person |
| `selectedIds` | `Set<number>` | IDs of selected suggestions for bulk actions |
| `selectedSuggestion` | `FaceSuggestion \| null` | Currently focused suggestion for detail modal |
| `persons` | `Person[]` | Full list of active persons (fetched on mount) |
| `recentAssignments` | `RecentAssignment[]` | Tracked assignments for undo sidebar |

**Key derived values:**
- `totalPages`: Computed from `groupedResponse.totalGroups / groupsPerPage`
- `totalPendingCount`: Total pending suggestions (varies by filter mode)
- `displayedPendingCount`: Pending count on current page

### 1.3 Data Flow

1. On mount: Fetch settings, load all persons, load grouped suggestions
2. `$effect` watches `statusFilter`, `page`, `personFilter`, `groupsPerPage` to reload
3. Second `$effect` batch-loads thumbnails via `thumbnailCache` when response changes
4. Group cards emit callbacks: `onSelect`, `onAcceptAll`, `onRejectAll`, `onThumbnailClick`, `onFindMoreComplete`
5. Detail modal opens on thumbnail click with full image + face bounding boxes
6. Recent assignments tracked in sidebar with undo capability

### 1.4 Component Composition

```
+page.svelte
  +-- PersonSearchBar (filter by person)
  +-- SuggestionGroupCard (per person group)
  |     +-- SuggestionThumbnail (per face suggestion)
  |     +-- ComputeCentroidsDialog (Find More)
  |     +-- CentroidResultsDialog (results)
  +-- SuggestionDetailModal (full-screen detail)
  |     +-- ImageWithFaceBoundingBoxes
  |     +-- FaceListSidebar
  |     +-- PersonAssignmentModal
  +-- RecentlyAssignedPanel (sidebar)
```

### 1.5 Key UX Patterns

- **Grouped by person**: Each card represents one person with all their pending suggestions
- **Bulk operations**: Select all within a group or across all groups, then batch accept/reject
- **Progressive disclosure**: Thumbnail grid -> click -> full detail modal
- **Undo safety net**: Recently assigned panel with undo buttons
- **Find More**: Centroid-based search to discover additional matches for a person
- **Auto-refresh**: After bulk actions or Find More completion, suggestions reload
- **Persistent preferences**: `localSettings` preserves `groupsPerPage` across sessions

### 1.6 Strengths for Reuse

- The `SuggestionGroupCard` pattern (header with metadata + actions + thumbnail grid) is directly applicable to unlabeled groups
- The `SuggestionDetailModal` pattern (full-screen image with face bounding boxes) can be reused for reviewing unlabeled faces
- The `RecentlyAssignedPanel` sidebar pattern provides undo safety
- The filter bar pattern with dropdowns and search is a proven interaction model
- The pagination pattern (page-based) handles large datasets well

### 1.7 Gaps for Unlabeled Use Case

- **Person-centric**: Everything revolves around existing `Person` records; no concept of "potential person groups"
- **No similarity threshold control**: Suggestions use backend-computed confidence; no user-adjustable grouping threshold
- **No group creation workflow**: No UI flow for "these faces look like the same person, create a new person from them"
- **No group merge/split**: No way to combine or separate face groups

---

## 2. Current UI Analysis: Clusters View

### 2.1 Page Structure

**File**: `src/routes/faces/clusters/+page.svelte` (608 lines)

The Clusters page uses a simpler single-column layout:

```
+--------------------------------------------------+
|  Header: "Unknown Faces"                         |
|  Subtitle: "Showing clusters with N faces..."    |
+--------------------------------------------------+
|  Controls Bar:                                   |
|  [Sort: Face Count v Quality] [Threshold: 0.6]   |
|  [Custom Threshold Input]                        |
+--------------------------------------------------+
|  Clusters Grid (auto-fill, min 280px)            |
|  +----------+  +----------+  +----------+        |
|  | Cluster A |  | Cluster B |  | Cluster C |     |
|  | 12 faces  |  | 8 faces   |  | 5 faces   |    |
|  | [thumbs]  |  | [thumbs]  |  | [thumbs]  |    |
|  | avg qual  |  | avg qual  |  | avg qual  |    |
|  +----------+  +----------+  +----------+        |
+--------------------------------------------------+
|  [Load More] or empty state                      |
+--------------------------------------------------+
```

### 2.2 Confidence Threshold Control

This is the most relevant pattern for the new view:

```svelte
<!-- Preset dropdown -->
<select onchange={handleConfidenceChange}>
  {#each CONFIDENCE_PRESETS as preset}
    <option value={preset}>{(preset * 100).toFixed(0)}%</option>
  {/each}
  <option value="custom">Custom...</option>
</select>

<!-- Custom input (shown when "Custom" selected) -->
{#if isCustomConfidence}
  <input type="number" min="0.01" max="1.0" step="0.01"
    bind:value={customConfidenceInput}
    onkeydown={(e) => e.key === 'Enter' && handleCustomConfidenceSubmit()} />
{/if}
```

**Presets**: `[0.9, 0.8, 0.7, 0.6, 0.5, 0.4, 0.3, 0.2, 0.1]`
**Default**: `0.6`
**Persistence**: `localSettings.set('clusters.minConfidence', value)`

### 2.3 Sort Control

Two sort options persisted via `localSettings`:
- `faceCount` (default) - Largest clusters first
- `avgQuality` - Highest quality faces first

### 2.4 Navigation Pattern

Cluster cards are clickable, navigating to `/faces/clusters/[clusterId]` for detail view:
- Breadcrumb navigation back to clusters list
- Full face grid with quality stats
- "Label as Person" action to create a person from the cluster
- "Split Cluster" action for refinement

### 2.5 Strengths for Reuse

- **Confidence threshold control**: Exact pattern needed for similarity threshold in unlabeled groups
- **Card-based grid layout**: `auto-fill, minmax(280px, 1fr)` works well for browsing groups
- **Load More pagination**: Good for discovery-oriented browsing
- **Cluster summary cards**: Show face count, preview thumbnails, quality metrics
- **ClusterCard component**: Reusable card with face count, confidence badge, thumbnail previews

### 2.6 Gaps for Unlabeled Use Case

- **No batch operations**: Cannot select multiple clusters for bulk actions
- **No inline assignment**: Must navigate to detail page, then label; no inline "Create Person" flow
- **No comparison**: Cannot compare two clusters side-by-side to decide if they should merge
- **HDBSCAN-dependent**: Clustering happens as a backend batch job; not interactive
- **No similarity control for grouping**: The threshold controls filtering (what to show), not how groups are formed
- **Single-purpose**: Designed for the "label unknown clusters" workflow, not "discover potential persons"

---

## 3. Existing Component Inventory

### 3.1 Reusable Face Components

| Component | Location | Reuse Potential | Notes |
|---|---|---|---|
| `FaceThumbnail` | `faces/FaceThumbnail.svelte` | **HIGH** | Circle/square face crops with lazy loading |
| `SuggestionGroupCard` | `faces/SuggestionGroupCard.svelte` | **HIGH** | Card pattern with header+actions+thumbnail grid |
| `SuggestionThumbnail` | `faces/SuggestionThumbnail.svelte` | **HIGH** | Thumbnail with selection checkbox overlay |
| `SuggestionDetailModal` | `faces/SuggestionDetailModal.svelte` | **MEDIUM** | Full-screen modal; needs adaptation for unlabeled context |
| `ClusterCard` | `faces/ClusterCard.svelte` | **MEDIUM** | Card with face count + previews; navigation-oriented |
| `ClusterFaceCard` | `faces/ClusterFaceCard.svelte` | **LOW** | Full image + face detail; more suited for detail pages |
| `PersonSearchBar` | `faces/PersonSearchBar.svelte` | **MEDIUM** | For "assign to existing person" in group actions |
| `RecentlyAssignedPanel` | `faces/RecentlyAssignedPanel.svelte` | **HIGH** | Sidebar undo panel; directly applicable |
| `ComputeCentroidsDialog` | `faces/ComputeCentroidsDialog.svelte` | **LOW** | Centroid computation; may be useful for post-creation refinement |
| `ImageWithFaceBoundingBoxes` | `faces/ImageWithFaceBoundingBoxes.svelte` | **HIGH** | Essential for detail modal face highlighting |
| `FindMoreResultsDialog` | `faces/FindMoreResultsDialog.svelte` | **LOW** | Post-creation enhancement; not needed for initial discovery |

### 3.2 UI Library Components (shadcn/ui)

| Component | Location | Usage in New View |
|---|---|---|
| **Tabs** | `ui/tabs/` | **PRIMARY** - View switching (Known Persons / Unlabeled Groups) |
| **Card** | `ui/card/` | Group card containers |
| **Button** | `ui/button/` | Actions (Create Person, Accept, Reject, Merge) |
| **Badge** | `ui/badge/` | Confidence scores, face counts, group labels |
| **Dialog** | `ui/dialog/` | Create Person dialog, merge confirmation |
| **Checkbox** | `ui/checkbox/` | Face selection within groups |
| **Select** | `ui/select/` | Threshold presets, sort options |
| **Input** | `ui/input/` | Custom threshold, person name input |
| **Skeleton** | `ui/skeleton/` | Loading states |
| **Tooltip** | `ui/tooltip/` | Confidence explanations, action hints |
| **Switch** | `ui/switch/` | Optional toggle for auto-merge similar groups |
| **Separator** | `ui/separator/` | Visual dividers between controls |
| **Progress** | `ui/progress/` | Processing indicator for grouping computation |

### 3.3 Stores and Utilities

| Store/Utility | Purpose | Reuse in New View |
|---|---|---|
| `localSettings` | Persist UI preferences in localStorage | Threshold, sort, active tab, groups per page |
| `thumbnailCache` | Batch thumbnail loading with cache | Face thumbnail loading in groups |
| `jobProgressStore` | Track background job progress | Grouping computation progress |
| `tid()` | Test ID generation | All new components |

---

## 4. Proposed UI Design

### 4.1 View Switching: Tabbed Interface

The new "Unlabeled Face Groups" view will be integrated into the existing Suggestions page using the shadcn/ui `Tabs` component. This approach:
- Keeps related functionality together (all face suggestions in one place)
- Provides clear navigation between known and unknown person suggestions
- Matches the user's requirement: "accessible from the Face Suggestions view using a component that allows to flip between the two views"

**Tab Configuration:**

```
+------------------------------------------------------------------+
| [ Known Persons (247) ] [ Unlabeled Groups (34) ]                |
+------------------------------------------------------------------+
|                                                                    |
|  (Tab content for active view)                                    |
|                                                                    |
+------------------------------------------------------------------+
```

- **Tab 1: "Known Persons"** -- The existing Face Suggestions content (current `+page.svelte` body)
- **Tab 2: "Unlabeled Groups"** -- The new unlabeled face groups view

The active tab is persisted via `localSettings` using key `suggestions.activeTab` so users return to their last-used view.

**Implementation Approach:**

```svelte
<script lang="ts">
  import * as Tabs from '$lib/components/ui/tabs';
  import { localSettings } from '$lib/stores/localSettings.svelte';

  const TAB_KEY = 'suggestions.activeTab';
  let activeTab = $state(localSettings.get<string>(TAB_KEY, 'known'));

  function handleTabChange(value: string) {
    activeTab = value;
    localSettings.set(TAB_KEY, value);
  }
</script>

<Tabs.Root value={activeTab} onValueChange={handleTabChange}>
  <Tabs.List>
    <Tabs.Trigger value="known">Known Persons (247)</Tabs.Trigger>
    <Tabs.Trigger value="unlabeled">Unlabeled Groups (34)</Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="known">
    <!-- Existing Face Suggestions content -->
  </Tabs.Content>

  <Tabs.Content value="unlabeled">
    <!-- New Unlabeled Groups content -->
  </Tabs.Content>
</Tabs.Root>
```

### 4.2 Similarity Threshold Control

Adapted from the Clusters view pattern, the threshold control provides both preset buttons and a custom input:

```
+------------------------------------------------------------------+
| Similarity Threshold:                                              |
| [95%] [90%] [85%*] [80%] [75%] [70%] [Custom: ___]             |
|                                                                    |
| Higher = fewer, more confident groups                             |
| Lower = more groups, may include false matches                    |
+------------------------------------------------------------------+
```

**Design Details:**
- **Default**: 0.85 (85%) -- high confidence to start, showing only groups we are fairly confident about
- **Preset buttons**: `[0.95, 0.90, 0.85, 0.80, 0.75, 0.70]` -- narrower range than clusters since we want higher confidence by default
- **Custom input**: Number input with 0.50-1.00 range, step 0.01
- **Persistence**: `localSettings` key `unlabeledGroups.threshold`
- **Behavior**: Changing threshold triggers re-computation of groups from the backend
- **Help text**: Tooltip or subtitle explaining the threshold's effect on grouping

**Rationale for 0.85 default**: The user spec says "the default should be fairly high to ensure we show groups that we are fairly confident about." A threshold of 0.85 means only faces with >= 85% similarity to each other will be grouped, minimizing false positives.

### 4.3 Group Display: Unlabeled Face Group Cards

Each group card represents a set of faces that the system believes belong to the same (uncreated) person. The card design adapts the `SuggestionGroupCard` pattern:

```
+------------------------------------------------------------------+
| [ ] Group 1 (12 faces)              Avg Similarity: 0.91         |
|     Auto-label: "Brown hair, male"  [Create Person] [Dismiss All]|
+------------------------------------------------------------------+
| [face] [face] [face] [face] [face] [face] [+6]                  |
+------------------------------------------------------------------+
```

**Card Header Elements:**
- **Checkbox**: For selecting the entire group for bulk operations
- **Group label**: Auto-generated sequential label ("Group 1", "Group 2", etc.)
- **Face count**: Number of faces in the group
- **Average similarity badge**: Confidence metric for the group's cohesion
- **Create Person button** (primary action): Opens dialog to name and create the person
- **Dismiss All button** (secondary action): Marks all faces as "not a group" / noise

**Thumbnail Grid:**
- Shows up to 8 face thumbnails with "+N more" indicator
- Each thumbnail is selectable (checkbox overlay)
- Thumbnails sorted by similarity score (highest first)
- Click opens detail view for the face

**Visual Differentiation from Known Person Groups:**
- Dashed border instead of solid (indicates provisional/unconfirmed status)
- Slightly different background tint (e.g., light blue-gray vs white)
- No person avatar/icon in header (since person does not exist yet)

### 4.4 Group Actions

**Primary Actions (per group):**

1. **Create Person**: Opens a dialog to:
   - Enter person name (required)
   - Preview all faces in the group
   - Optionally deselect specific faces before creation
   - Submit creates the person record AND assigns all selected faces
   - After creation, group disappears from unlabeled view and the person appears in Known Persons tab

2. **Assign to Existing Person**: Opens `PersonSearchBar` to find and assign all faces to an existing person. Useful when the system created a separate group for faces that actually belong to a known person.

3. **Dismiss Group**: Marks the group as dismissed/noise. The faces remain unassigned but are excluded from future grouping at this threshold.

**Bulk Actions (multi-group):**

4. **Merge Groups**: Select two or more groups and merge them into one. Useful when the system over-separated faces of the same person.

5. **Bulk Create**: Select multiple groups and create persons for each in one operation (each group becomes a separate person).

6. **Bulk Dismiss**: Select multiple groups and dismiss them all.

### 4.5 Controls Bar

```
+------------------------------------------------------------------+
| Similarity: [95%] [90%] [85%] [80%] [75%] [70%] [Custom]       |
| Sort: [Group Size v] [Similarity v]                              |
| Show: [10] [20] [50] groups per page                            |
|                                                     [Select All] |
+------------------------------------------------------------------+
```

**Sort Options:**
- **Group Size** (default): Largest groups first (most likely to be real persons)
- **Average Similarity**: Highest confidence groups first
- **Face Count in Photos**: Groups spanning the most unique photos first

**Filtering:**
- **Min Group Size**: Filter groups with minimum N faces (default: 3)
- **Date Range**: Optional -- show only groups from faces detected in a date range

### 4.6 Edge Cases and Empty States

**No Groups Found (threshold too high):**
```
+--------------------------------------------------+
|  No unlabeled face groups found at 95% threshold  |
|                                                    |
|  Try lowering the similarity threshold to see     |
|  more potential person groups.                     |
|                                                    |
|  [Lower to 85%]  [Lower to 75%]                  |
+--------------------------------------------------+
```

**All Faces Assigned:**
```
+--------------------------------------------------+
|  All detected faces have been assigned!            |
|                                                    |
|  Run face detection on new photos to discover     |
|  more faces.                                       |
|                                                    |
|  [Go to Face Detection]                           |
+--------------------------------------------------+
```

**Grouping In Progress:**
```
+--------------------------------------------------+
|  Computing face groups...                          |
|  [=============>          ] 67%                    |
|                                                    |
|  Analyzing 1,247 unassigned faces at 85% threshold|
+--------------------------------------------------+
```

**Single-Face Groups (Singletons):**
- Groups with only 1 face are hidden by default (likely noise or unique visitors)
- Toggle: "Show singletons" switch to reveal them
- Singletons displayed in a compact list at the bottom, not as full cards

---

## 5. Interaction Flow / User Journey

### 5.1 Primary Discovery Flow

```
User navigates to /faces/suggestions
       |
       v
Tabs: [Known Persons] [Unlabeled Groups*]
       |
       v (clicks "Unlabeled Groups" tab)
       |
       v
System loads unlabeled face groups at default threshold (0.85)
       |
       v
Groups displayed as cards, sorted by size
       |
       +---> User reviews Group 1 (12 faces, 91% similarity)
       |        |
       |        +---> Clicks face thumbnail for detail view
       |        |       (sees full photo with face bounding box)
       |        |
       |        +---> Recognizes the person
       |        |       |
       |        |       +---> Clicks "Create Person"
       |        |       |       |
       |        |       |       v
       |        |       |     Enter name: "Jane Smith"
       |        |       |     Optionally deselect wrong faces
       |        |       |       |
       |        |       |       v
       |        |       |     Person created, faces assigned
       |        |       |     Group disappears from view
       |        |       |     Toast: "Created Jane Smith (12 faces)"
       |        |       |
       |        |       +---> OR clicks "Assign to Existing"
       |        |               |
       |        |               v
       |        |             PersonSearchBar opens
       |        |             Select existing person
       |        |             Faces assigned, group disappears
       |        |
       |        +---> Does not recognize the person
       |                |
       |                +---> Clicks "Dismiss"
       |                        |
       |                        v
       |                      Group marked as dismissed
       |                      (Excluded from future grouping)
       |
       +---> User adjusts threshold to 0.75 to see more groups
       |        |
       |        v
       |     More groups appear (lower confidence)
       |     Previously dismissed groups stay hidden
       |
       +---> User selects multiple groups and bulk creates
                |
                v
              Each group becomes a new person
              All groups disappear from view
```

### 5.2 Merge Flow

```
User sees Group 3 (5 faces) and Group 7 (3 faces)
       |
       v
Recognizes both groups are the same person
       |
       v
Selects checkbox on Group 3 and Group 7
       |
       v
Clicks "Merge Selected" in bulk actions bar
       |
       v
Confirmation dialog:
  "Merge 2 groups (8 faces total) into one group?"
  [Cancel] [Merge]
       |
       v
Groups merged into single group (8 faces)
User can now "Create Person" from merged group
```

### 5.3 Review Detail Flow

```
User clicks face thumbnail in a group card
       |
       v
Detail modal opens (adapted SuggestionDetailModal):
  +-------------------------------------------+
  |  [Full photo with face bounding boxes]    |
  |                                           |
  |  Sidebar:                                 |
  |  - Face #1: Unassigned                    |
  |  - Face #2: Assigned to "Bob"             |
  |  - Face #3: Unassigned (this face)        |
  |                                           |
  |  Group Info:                              |
  |  - Group 1 (12 faces, 91% avg)           |
  |  - Similarity to group centroid: 0.93     |
  |                                           |
  |  Actions:                                 |
  |  [Remove from Group] [Next Face ->]       |
  +-------------------------------------------+
```

---

## 6. Wireframe Descriptions

### 6.1 Wireframe: Full Page Layout with Tabs

```
+========================================================================+
|  Mac'Image Search  | Search People [Suggestions] Clusters ... | [*]   |
+========================================================================+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  | [ Known Persons (247) ] [ Unlabeled Groups (34) ]                 | |
|  +-------------------------------------------------------------------+ |
|  |                                                                   | |
|  | +---------------------------------------------+ +--------------+ | |
|  | | CONTROLS BAR                                | |              | | |
|  | | Similarity: [95] [90] [85*] [80] [75] [70]  | |  Recently    | | |
|  | | Sort: [Group Size v]  Show: [20 groups v]   | |  Created     | | |
|  | | Min faces: [3]        [Select All] [None]   | |  Panel       | | |
|  | +---------------------------------------------+ |              | | |
|  | |                                             | | Jane Smith   | | |
|  | | +--Group 1 (12 faces)--------+-----------+  | |  12 faces    | | |
|  | | | [x] Group 1    | Sim: 91% |           |  | |  [Undo]      | | |
|  | | |   12 faces     | [Create] [Assign] [X]|  | |              | | |
|  | | +----------------+-----------+-----------+  | | Bob Jones    | | |
|  | | | [o][o][o][o][o][o][o][o] +4             |  | |  3 faces     | | |
|  | | +--------------------------------------------+ |  [Undo]      | | |
|  | |                                             | |              | | |
|  | | +--Group 2 (8 faces)---------+-----------+  | |              | | |
|  | | | [ ] Group 2    | Sim: 88% |           |  | |              | | |
|  | | |   8 faces      | [Create] [Assign] [X]|  | |              | | |
|  | | +----------------+-----------+-----------+  | |              | | |
|  | | | [o][o][o][o][o][o][o][o]                |  | |              | | |
|  | | +--------------------------------------------+ |              | | |
|  | |                                             | |              | | |
|  | | +--Group 3 (5 faces)---------+-----------+  | |              | | |
|  | | | [ ] Group 3    | Sim: 86% |           |  | |              | | |
|  | | |   5 faces      | [Create] [Assign] [X]|  | |              | | |
|  | | +----------------+-----------+-----------+  | |              | | |
|  | | | [o][o][o][o][o]                         |  | |              | | |
|  | | +--------------------------------------------+ |              | | |
|  | |                                             | +--------------+ | |
|  | | Page 1 of 2  [Prev] [Next]                 |                  | |
|  | +---------------------------------------------+                  | |
|  +-------------------------------------------------------------------+ |
+=========================================================================+
```

### 6.2 Wireframe: Create Person Dialog

```
+--------------------------------------------+
|  Create Person from Group 1                 |
|  ----------------------------------------- |
|                                             |
|  Person Name: [_________________________]   |
|                                             |
|  Faces to assign (12 selected):            |
|  +------+------+------+------+------+      |
|  | [x]  | [x]  | [x]  | [x]  | [x]  |    |
|  | face | face | face | face | face  |     |
|  | 0.95 | 0.93 | 0.92 | 0.91 | 0.90 |    |
|  +------+------+------+------+------+      |
|  +------+------+------+------+------+      |
|  | [x]  | [x]  | [x]  | [x]  | [x]  |    |
|  | face | face | face | face | face  |     |
|  | 0.89 | 0.88 | 0.87 | 0.86 | 0.86 |    |
|  +------+------+------+------+------+      |
|  +------+------+                           |
|  | [x]  | [ ]  |  <-- user deselected     |
|  | face | face |      this one             |
|  | 0.85 | 0.72 |                           |
|  +------+------+                           |
|                                             |
|  11 of 12 faces will be assigned           |
|                                             |
|  [Cancel]            [Create & Assign]     |
+--------------------------------------------+
```

### 6.3 Wireframe: Merge Groups Confirmation

```
+--------------------------------------------+
|  Merge 2 Groups                             |
|  ----------------------------------------- |
|                                             |
|  Group 3 (5 faces, 86% avg similarity)     |
|  [face][face][face][face][face]            |
|                                             |
|  Group 7 (3 faces, 88% avg similarity)     |
|  [face][face][face]                        |
|                                             |
|  ----------------------------------------- |
|  Result: 1 group with 8 faces              |
|  Estimated similarity: 84%                 |
|                                             |
|  [Cancel]                     [Merge]      |
+--------------------------------------------+
```

### 6.4 Wireframe: Empty State (Threshold Too High)

```
+--------------------------------------------------+
|                                                    |
|        (magnifying glass icon)                    |
|                                                    |
|  No unlabeled face groups found                   |
|  at 95% similarity threshold                      |
|                                                    |
|  Try lowering the threshold to discover           |
|  more potential person groups.                     |
|                                                    |
|  [Try 85%]  [Try 75%]  [Try 65%]                |
|                                                    |
+--------------------------------------------------+
```

### 6.5 Wireframe: Detail Modal (Adapted)

```
+=====================================================================+
|  [X Close]                                                           |
|  +-----------------------------------------------+  +-----------+  |
|  |                                               |  |           |  |
|  |            Full Photo                         |  | Faces in  |  |
|  |       (with bounding boxes                    |  | this photo|  |
|  |        highlighting the                       |  |           |  |
|  |        relevant face)                         |  | #1: Bob   |  |
|  |                                               |  | #2: ---   |  |
|  |                                               |  | #3: --- * |  |
|  |                                               |  |           |  |
|  |                                               |  |-----------|  |
|  |                                               |  |           |  |
|  |                                               |  | Group Info|  |
|  |                                               |  | Group 1   |  |
|  +-----------------------------------------------+  | 12 faces  |  |
|                                                      | Sim: 0.93 |  |
|                                                      |           |  |
|                                                      | [Remove   |  |
|                                                      |  from     |  |
|                                                      |  Group]   |  |
|                                                      |           |  |
|                                                      | [< Prev]  |  |
|                                                      | [Next >]  |  |
|                                                      +-----------+  |
+=====================================================================+
```

---

## 7. Component Architecture

### 7.1 New Components Required

Based on the analysis, the following new Svelte components are recommended:

#### 7.1.1 `UnlabeledGroupsView.svelte` (Container Component)

**Purpose**: The main content component for the "Unlabeled Groups" tab. Contains all state management, data fetching, and orchestration for the unlabeled groups workflow.

**File**: `src/lib/components/faces/UnlabeledGroupsView.svelte`

**Props**:
```typescript
interface Props {
  // No required props -- self-contained view component
  onPersonCreated?: (personId: string, personName: string, faceCount: number) => void;
  onGroupAssigned?: (groupId: string, personId: string) => void;
}
```

**Internal State**:
```typescript
let groups = $state<UnlabeledFaceGroup[]>([]);
let threshold = $state(localSettings.get('unlabeledGroups.threshold', 0.85));
let sortBy = $state<'groupSize' | 'similarity'>('groupSize');
let groupsPerPage = $state(localSettings.get('unlabeledGroups.groupsPerPage', 20));
let page = $state(1);
let selectedGroupIds = $state<Set<string>>(new Set());
let isLoading = $state(false);
let isComputing = $state(false);
let error = $state<string | null>(null);
```

**Responsibilities**:
- Fetch unlabeled face groups from API
- Handle threshold changes and re-fetching
- Manage group selection for bulk operations
- Coordinate with Create Person and Merge dialogs
- Track recently created persons for sidebar

**Estimated Size**: ~250-300 lines (within the 300-line guideline)

#### 7.1.2 `UnlabeledGroupCard.svelte` (Presentation Component)

**Purpose**: Displays a single unlabeled face group as a card with header, thumbnails, and actions.

**File**: `src/lib/components/faces/UnlabeledGroupCard.svelte`

**Props**:
```typescript
interface Props {
  group: UnlabeledFaceGroup;
  selected: boolean;
  onSelect: (groupId: string, selected: boolean) => void;
  onCreatePerson: (groupId: string) => void;
  onAssignToExisting: (groupId: string) => void;
  onDismiss: (groupId: string) => void;
  onFaceClick: (face: UnlabeledFace, groupId: string) => void;
}
```

**Visual Design**:
- Dashed border (2px dashed #d0d0d0) to differentiate from known person cards
- Header: checkbox, auto-label ("Group N"), face count, average similarity badge
- Thumbnail grid: up to 8 faces with "+N" overflow indicator
- Actions: Create Person (primary green), Assign to Existing (outline), Dismiss (destructive)

**Estimated Size**: ~150-180 lines

#### 7.1.3 `CreatePersonFromGroupDialog.svelte` (Dialog Component)

**Purpose**: Dialog for creating a new person from an unlabeled face group. Shows all faces in the group with individual selection, person name input, and create+assign action.

**File**: `src/lib/components/faces/CreatePersonFromGroupDialog.svelte`

**Props**:
```typescript
interface Props {
  open: boolean;
  group: UnlabeledFaceGroup;
  onClose: () => void;
  onCreated: (personId: string, personName: string, assignedFaceIds: string[]) => void;
}
```

**Internal State**:
```typescript
let personName = $state('');
let selectedFaceIds = $state<Set<string>>(new Set()); // Initialized with all face IDs
let isCreating = $state(false);
let nameError = $state<string | null>(null);
```

**Behavior**:
- Pre-selects all faces in the group
- User can deselect individual faces (e.g., mismatched faces)
- Validates person name (non-empty, unique check)
- On submit: Creates person record, assigns selected faces, closes dialog
- Shows success toast with undo option

**Estimated Size**: ~150-200 lines

#### 7.1.4 `MergeGroupsDialog.svelte` (Dialog Component)

**Purpose**: Confirmation dialog for merging two or more unlabeled groups into one.

**File**: `src/lib/components/faces/MergeGroupsDialog.svelte`

**Props**:
```typescript
interface Props {
  open: boolean;
  groups: UnlabeledFaceGroup[];
  onClose: () => void;
  onMerged: (mergedGroupId: string) => void;
}
```

**Estimated Size**: ~100-120 lines

#### 7.1.5 `SimilarityThresholdControl.svelte` (Control Component)

**Purpose**: Reusable threshold control with preset buttons and custom input. Extracted as a shared component since both the Clusters view and Unlabeled Groups view need this pattern.

**File**: `src/lib/components/faces/SimilarityThresholdControl.svelte`

**Props**:
```typescript
interface Props {
  value: number;
  onChange: (threshold: number) => void;
  presets?: number[];   // Default: [0.95, 0.90, 0.85, 0.80, 0.75, 0.70]
  min?: number;         // Default: 0.50
  max?: number;         // Default: 1.00
  step?: number;        // Default: 0.01
  label?: string;       // Default: "Similarity Threshold"
  helpText?: string;
}
```

**Estimated Size**: ~80-100 lines

### 7.2 Modified Components

#### 7.2.1 `+page.svelte` (Suggestions Page)

The existing suggestions page needs to be refactored to add tabs:

**Changes**:
1. Add `Tabs` import from shadcn/ui
2. Wrap existing content in `Tabs.Content value="known"`
3. Add `UnlabeledGroupsView` in `Tabs.Content value="unlabeled"`
4. Move tab state persistence to `localSettings`
5. Update page title based on active tab

**The existing content stays intact** -- the refactoring wraps it, not replaces it. This minimizes risk and preserves all current functionality.

**Estimated additional lines**: ~30-40 lines for tab wrapper

#### 7.2.2 `RecentlyAssignedPanel.svelte` (Sidebar)

Minor enhancement to support "recently created persons" entries in addition to "recently assigned faces":

**Changes**:
- Accept optional `recentCreations` prop for newly created persons
- Display creation entries with different visual style (person icon instead of face thumbnail)
- Undo for creations removes the person and unassigns all faces

### 7.3 New Type Definitions

Add to `src/lib/types.ts` or `src/lib/api/faces.ts`:

```typescript
export interface UnlabeledFaceGroup {
  groupId: string;
  faces: UnlabeledFace[];
  faceCount: number;
  avgSimilarity: number;
  representativeFaceId: string;  // Best face for preview
  representativeThumbnailUrl: string;
}

export interface UnlabeledFace {
  faceInstanceId: string;
  assetId: number;
  thumbnailUrl: string;
  similarityToCenter: number;  // Similarity to group centroid
  quality: number;
  bbox: BoundingBox;
}

export interface UnlabeledGroupsResponse {
  groups: UnlabeledFaceGroup[];
  totalGroups: number;
  totalFaces: number;
  threshold: number;
  computedAt: string;  // ISO timestamp
}
```

### 7.4 New API Functions

Add to `src/lib/api/faces.ts`:

```typescript
// Fetch unlabeled face groups
export async function listUnlabeledGroups(
  threshold: number,
  page: number,
  pageSize: number,
  sortBy: 'groupSize' | 'similarity',
  minGroupSize: number
): Promise<UnlabeledGroupsResponse> { ... }

// Create person from group
export async function createPersonFromGroup(
  groupId: string,
  personName: string,
  faceIds: string[]
): Promise<{ personId: string; assignedCount: number }> { ... }

// Dismiss a group
export async function dismissGroup(groupId: string): Promise<void> { ... }

// Merge groups
export async function mergeGroups(groupIds: string[]): Promise<{
  mergedGroupId: string;
  totalFaces: number;
}> { ... }

// Assign group to existing person
export async function assignGroupToPerson(
  groupId: string,
  personId: string,
  faceIds: string[]
): Promise<{ assignedCount: number }> { ... }
```

### 7.5 Component Hierarchy

```
+page.svelte (Suggestions Page - Modified)
  +-- Tabs.Root
  |     +-- Tabs.List
  |     |     +-- Tabs.Trigger (Known Persons)
  |     |     +-- Tabs.Trigger (Unlabeled Groups)
  |     +-- Tabs.Content (known)
  |     |     +-- [Existing suggestions content unchanged]
  |     +-- Tabs.Content (unlabeled)
  |           +-- UnlabeledGroupsView
  |                 +-- SimilarityThresholdControl
  |                 +-- UnlabeledGroupCard (per group)
  |                 |     +-- FaceThumbnail (per face, up to 8)
  |                 |     +-- Checkbox
  |                 |     +-- Badge (similarity)
  |                 +-- CreatePersonFromGroupDialog
  |                 +-- MergeGroupsDialog
  |                 +-- SuggestionDetailModal (adapted)
  +-- RecentlyAssignedPanel (sidebar, shared across tabs)
```

---

## 8. UX Patterns from Similar Applications

### 8.1 Google Photos: Face Grouping

Google Photos uses a discovery-first approach:
- Shows face clusters automatically without user intervention
- Prompts "Who is this?" with a text input directly on the group
- Allows merging by selecting "Same person" between two groups
- Uses a horizontal carousel for browsing face groups
- Confidence-based ordering: most confident groups shown first

**Applicable patterns:**
- Inline "Who is this?" prompt on each group card (instead of opening a dialog)
- Confidence-based default sort order
- Smooth transitions when groups are created/dismissed

### 8.2 Apple Photos: People & Pets

Apple Photos uses a two-phase approach:
- Initial grouping happens automatically in the background
- "People" album shows all recognized groups (named and unnamed)
- Unnamed groups have "Add Name" prompt
- Users can drag faces between groups to correct mistakes
- "Review Suggested Photos" for each person to accept/reject

**Applicable patterns:**
- Two-phase workflow: browse groups, then refine
- Drag-and-drop between groups for manual correction (future enhancement)
- Inline naming directly on the group card

### 8.3 Amazon Photos

Amazon Photos provides:
- Grid of face group thumbnails with face count
- Click to see all faces in a group
- "Add Name" button prominently displayed
- "Not the same person" option to remove incorrect faces

**Applicable patterns:**
- Clean grid layout with representative face as main thumbnail
- Quick "Not the same person" action for individual faces within a group

### 8.4 Synthesis: Best Practices for Our Implementation

1. **Default to high confidence**: Show confident groups first, let users explore lower thresholds
2. **Inline naming**: Consider a lightweight inline input on each card as an alternative to a full dialog
3. **Progressive refinement**: Users start with confident groups, then lower threshold for less certain ones
4. **Quick actions**: Create Person and Dismiss should be one-click operations
5. **Visual feedback**: Smooth animations when groups are created/merged/dismissed
6. **Undo everywhere**: Every destructive action (dismiss, create) should be undoable
7. **Representative face**: Each group should show its best/most representative face prominently

---

## 9. Recommendations

### 9.1 Implementation Priority

**Phase 1: Core View (MVP)**
1. Refactor `+page.svelte` to add Tabs wrapper
2. Create `UnlabeledGroupsView.svelte` with basic group listing
3. Create `UnlabeledGroupCard.svelte` with thumbnail grid
4. Create `SimilarityThresholdControl.svelte` (reusable)
5. Implement "Create Person" flow with `CreatePersonFromGroupDialog.svelte`
6. Add `localSettings` persistence for threshold and tab state

**Phase 2: Enhanced Interactions**
7. Add "Assign to Existing Person" flow using `PersonSearchBar`
8. Add "Dismiss Group" with undo capability
9. Add bulk selection and bulk create/dismiss
10. Adapt `SuggestionDetailModal` for unlabeled face context

**Phase 3: Advanced Features**
11. Add group merge functionality
12. Add drag-and-drop face movement between groups (future)
13. Add "Show Singletons" toggle
14. Refactor Clusters view threshold control to use shared `SimilarityThresholdControl`

### 9.2 Technical Recommendations

1. **Extract existing suggestions content**: Move the current suggestions body into a separate `KnownPersonSuggestionsView.svelte` component to keep `+page.svelte` under the 300-line limit after adding tabs.

2. **Lazy load tab content**: Only fetch unlabeled groups when the user switches to that tab. This prevents unnecessary API calls and keeps initial page load fast.

3. **Debounce threshold changes**: When user adjusts threshold via custom input, debounce the API call by 500ms to prevent rapid re-fetching.

4. **Use `$state.raw()` for groups array**: Since group arrays can be large and are replaced entirely on fetch, use `$state.raw()` for better performance (no deep reactivity needed).

5. **Batch thumbnail loading**: Leverage the existing `thumbnailCache` store for batch-loading face thumbnails when groups are displayed.

6. **Keyboard navigation**: Ensure tab switching works with keyboard (Tab, Enter, Arrow keys), and group cards are navigable with arrow keys.

7. **Responsive design**: Follow the existing pattern of collapsing to single column at 1024px breakpoint. The sidebar (RecentlyAssignedPanel) should be shared across both tabs.

### 9.3 API Contract Considerations

The frontend design assumes the backend will provide:
- An endpoint for listing unlabeled face groups at a given threshold
- An endpoint for creating a person from a group (atomic: create person + assign faces)
- An endpoint for dismissing groups
- An endpoint for merging groups
- Real-time or polling-based progress for group computation

These endpoints should be specified in `docs/api-contract.md` before frontend implementation begins. The backend research team should coordinate on the exact request/response schemas.

### 9.4 Accessibility Requirements

- All interactive elements must have accessible labels
- Tab switching must work with keyboard (standard ARIA tabs pattern handled by shadcn)
- Group cards must be navigable with screen readers
- Threshold slider must announce current value
- Bulk action buttons must indicate the count of affected items
- Error states must use `role="alert"`
- Loading states must use `aria-live="polite"`
- Modal dialogs must trap focus appropriately (handled by shadcn Dialog)

### 9.5 Testing Strategy

| Component | Test Focus | Approach |
|---|---|---|
| `UnlabeledGroupsView` | Data loading, threshold changes, pagination | Route-level integration test |
| `UnlabeledGroupCard` | Rendering, selection, action callbacks | Component unit test |
| `CreatePersonFromGroupDialog` | Form validation, face selection, submit | Component unit test |
| `MergeGroupsDialog` | Rendering groups, confirmation | Component unit test |
| `SimilarityThresholdControl` | Preset selection, custom input, edge values | Component unit test |
| Tab switching | Active tab persistence, content loading | Route-level integration test |

Use fixtures from `tests/helpers/fixtures.ts` -- new fixtures will be needed:
- `createUnlabeledFaceGroup()`: Factory for test group data
- `createUnlabeledGroupsResponse()`: Factory for API response data

---

## Appendix A: File References

| File | Purpose | Lines |
|---|---|---|
| `src/routes/faces/suggestions/+page.svelte` | Current suggestions page (to be modified) | 688 |
| `src/routes/faces/clusters/+page.svelte` | Clusters view (reference for threshold UI) | 608 |
| `src/routes/faces/clusters/[clusterId]/+page.svelte` | Cluster detail page (reference) | 785 |
| `src/routes/+layout.svelte` | App layout with navigation | 221 |
| `src/lib/components/faces/SuggestionGroupCard.svelte` | Group card pattern (to adapt) | 386 |
| `src/lib/components/faces/SuggestionDetailModal.svelte` | Detail modal (to adapt) | 1008 |
| `src/lib/components/faces/ClusterCard.svelte` | Cluster card (reference) | 220 |
| `src/lib/components/faces/FaceThumbnail.svelte` | Reusable thumbnail | 174 |
| `src/lib/components/faces/RecentlyAssignedPanel.svelte` | Sidebar panel (to enhance) | 271 |
| `src/lib/components/faces/FindMoreResultsDialog.svelte` | Find more dialog (reference) | 202 |
| `src/lib/components/ui/tabs/index.ts` | shadcn Tabs component | 17 |
| `src/lib/stores/localSettings.svelte.ts` | localStorage persistence store | -- |
| `src/lib/stores/thumbnailCache.svelte.ts` | Batch thumbnail cache | -- |
| `docs/prompts/unlabeled-face-suggestions.md` | Original feature request | 16 |

## Appendix B: localStorage Keys (New)

| Key | Type | Default | Purpose |
|---|---|---|---|
| `suggestions.activeTab` | `'known' \| 'unlabeled'` | `'known'` | Active tab in suggestions page |
| `unlabeledGroups.threshold` | `number` | `0.85` | Similarity threshold for grouping |
| `unlabeledGroups.sortBy` | `'groupSize' \| 'similarity'` | `'groupSize'` | Sort order for groups |
| `unlabeledGroups.groupsPerPage` | `10 \| 20 \| 50` | `20` | Groups displayed per page |
| `unlabeledGroups.showSingletons` | `boolean` | `false` | Show single-face groups |

## Appendix C: Related Research Documents

- `docs/research/unknown_person_face_detection/01-*` -- Backend/API research
- `docs/research/unknown_person_face_detection/02-*` -- Database/Qdrant research
- `docs/research/unknown_person_face_detection/03-*` -- General research
- `docs/research/unknown_person_face_detection/05-*` -- Devil's advocate review
