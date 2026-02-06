# Plan 7: Unify Terminology Across UI

**Issue ID**: H2 (High Severity)
**Priority**: High
**Estimated Effort**: 6-8 hours
**Risk Level**: Medium (wide surface area across many components)
**Date**: 2026-02-06

---

## Problem Statement

The face recognition UI uses inconsistent terminology across different pages and components to describe the same underlying concepts. A user navigating from the People page to the Suggestions page to the Clusters page encounters different words for the same things: "Identified" vs "Known", "Unidentified" vs "Unknown" vs "Needs Name", "Noise" vs "Review" vs "Unknown Faces", "Suggestions" vs "Centroid Suggestions" vs "Matches", "Cluster" vs "Face Group" vs "Face Cluster". This creates cognitive overhead, confuses new users, and makes the system feel like it was built by different teams who never talked to each other.

## Root Cause Analysis

The UI was built incrementally over time. The People page was built with one set of terms ("identified", "unidentified", "noise"), the Clusters page used different terms ("Unknown Faces" as page title), the Suggestions page introduced "suggestions" and "pending/accepted/rejected", and the centroid feature later added "centroid suggestions" and "centroid matching" as distinct concepts. No terminology standard was established upfront, so each component author chose terms independently.

The backend API is also inconsistent: it uses `type: 'identified' | 'unidentified' | 'noise'` in the unified people response, `status: 'pending' | 'accepted' | 'rejected'` for suggestions, and `cluster_id` for face groupings. The UI further transforms some of these (e.g., `noise` becomes "Review" badge label in UnifiedPersonCard).

---

## Complete Terminology Inventory

### Concept: Person Classification Status

How the system describes whether a face group has been assigned to a named person.

| Current Term | Location | File | Line/Context |
|-------------|----------|------|-------------|
| `'identified'` | API type value | Backend API response, `type` field | UnifiedPersonResponse |
| "Identified" | Badge label | `UnifiedPersonCard.svelte` | Badge for `type === 'identified'` |
| "This person has been assigned a name" | Badge tooltip | `UnifiedPersonCard.svelte` | Tooltip for identified badge |
| "Show Identified" | Filter checkbox label | `people/+page.svelte` | Line 255 |
| "Identified" | Stats label | `people/+page.svelte` | Line 194 |
| `showIdentified` | State variable | `people/+page.svelte` | Line 29 |
| `identifiedPeople` | Derived variable | `people/+page.svelte` | Line 66 |
| `identifiedCount` | API response field | `people/+page.svelte` | Line 195 |
| `'unidentified'` | API type value | Backend API response, `type` field | UnifiedPersonResponse |
| "Needs Name" | Badge label | `UnifiedPersonCard.svelte` | Badge for `type === 'unidentified'` |
| "This face cluster needs to be identified with a name" | Badge tooltip | `UnifiedPersonCard.svelte` | Tooltip for unidentified badge |
| "Show Unidentified" | Filter checkbox label | `people/+page.svelte` | Line 259 |
| "Unidentified" | Stats label | `people/+page.svelte` | Line 198 |
| `showUnidentified` | State variable | `people/+page.svelte` | Line 30 |
| `unidentifiedPeople` | Derived variable | `people/+page.svelte` | Line 69 |
| `unidentifiedCount` | API response field | `people/+page.svelte` | Line 199 |
| `'noise'` | API type value | Backend API response, `type` field | UnifiedPersonResponse |
| "Review" | Badge label | `UnifiedPersonCard.svelte` | Badge for `type === 'noise'` |
| "Low-confidence faces that need individual review" | Badge tooltip | `UnifiedPersonCard.svelte` | Tooltip for noise badge |
| "Show Unknown Faces" | Filter checkbox label | `people/+page.svelte` | Line 263 |
| "Unknown" | Stats label | `people/+page.svelte` | Line 203 |
| `showNoise` | State variable | `people/+page.svelte` | Line 31 |
| `noisePeople` | Derived variable | `people/+page.svelte` | Line 72 |
| `noiseCount` | API response field | `people/+page.svelte` | Line 204 |
| "Unknown Faces" | Page title | `faces/clusters/+page.svelte` | Line 186, `<title>` and `<h1>` |

**Conflict Summary**: The noise category alone has FOUR different labels: "noise" (API), "Review" (badge), "Show Unknown Faces" (filter), "Unknown" (stats). The unidentified category has THREE: "unidentified" (API), "Needs Name" (badge), "Unidentified" (filter/stats).

---

### Concept: Face Grouping

How the system describes a group of similar faces.

| Current Term | Location | File | Line/Context |
|-------------|----------|------|-------------|
| "cluster" | API field name | Backend API, `cluster_id` field | ClusterSummary type |
| "Clusters" | Navigation link | `+layout.svelte` | Line 61 |
| "Face cluster with N faces" | Aria label | `ClusterCard.svelte` | Accessibility label |
| "face clusters" | Subtitle text | `faces/clusters/+page.svelte` | Line 188: "Review and label face clusters" |
| "face groups" | Page subtitle | `people/+page.svelte` | Line 188: "face groups in your photos" |
| "Face Count" | Sort option | `people/+page.svelte` | Line 276 |
| "Face Count" | Sort option | `faces/clusters/+page.svelte` | Line 39: `SortOption = 'faceCount'` |
| "cluster" | Type description | `ClusterCard.svelte` | Throughout component |

**Conflict Summary**: "clusters" (Clusters page, navigation) vs "face groups" (People page subtitle). The People page says "Browse identified people and face groups" but clicking on a group takes you to `/faces/clusters/`.

---

### Concept: Face Assignment Recommendation

How the system describes an automated recommendation to assign a face to a person.

| Current Term | Location | File | Line/Context |
|-------------|----------|------|-------------|
| "Suggestions" | Navigation link | `+layout.svelte` | Line 60 |
| "Face Suggestions" | Page title | `faces/suggestions/+page.svelte` | Line 476 and 484 |
| "suggestion" / "suggestions" | Throughout | `SuggestionGroupCard.svelte` | Lines 142, 178 |
| "Centroid Suggestions" | Dialog title | `CentroidResultsDialog.svelte` | Title: "Centroid Suggestions for {personName}" |
| "centroid matching" | Dialog description | `CentroidResultsDialog.svelte` | "Review and accept face suggestions found using centroid matching" |
| "Find More" | Button label | `SuggestionGroupCard.svelte` | Line 166: button triggering centroid computation |
| "Compute Centroids" | Dialog title | `ComputeCentroidsDialog.svelte` | Title: "Compute Centroids for {personName}" |
| "match discovery" | Dialog description | `ComputeCentroidsDialog.svelte` | "compute cluster centroids for match discovery" |
| "Finding more suggestions" | Toast message | `faces/suggestions/+page.svelte` | Line 384 |
| "Found N new suggestions" | Toast message | `faces/suggestions/+page.svelte` | Line 370 |
| "multi-prototype match" | Sort logic | `SuggestionGroupCard.svelte` | Line 49: `isMultiPrototypeMatch` |

**Conflict Summary**: "Suggestions" (main concept) vs "Centroid Suggestions" (a specific type) vs "Find More" (the action) vs "match discovery" (the process). Users see "Compute Centroids" when they clicked "Find More" -- the button label and dialog title describe different things.

---

### Concept: Confidence/Similarity Score

How the system describes how well a face matches.

| Current Term | Location | File | Line/Context |
|-------------|----------|------|-------------|
| "confidence" | API field | `FaceSuggestion.confidence` | Backend model |
| "Confidence" | Column/label | `SuggestionThumbnail.svelte` | Displayed on suggestion cards |
| "similarity" | Page description | `faces/clusters/+page.svelte` | Line 189: "N% similarity" |
| "Minimum Similarity" | Setting label | `ComputeCentroidsDialog.svelte` | Slider label |
| "match" | Badge tooltip | `ClusterCard.svelte` | Confidence shown as "match" percentage |
| "score" | Column header | `CentroidResultsDialog.svelte` | Score column for centroid results |
| "aggregate confidence" | Sort logic | `SuggestionGroupCard.svelte` | Line 54: `aggregateConfidence` |
| "quality" | Sort option | `faces/clusters/+page.svelte` | Line 39: `'avgQuality'` sort option |

**Conflict Summary**: "confidence" vs "similarity" vs "match" vs "score" vs "quality" -- five different words for related but overlapping concepts.

---

### Concept: Representative Face / Prototype

How the system describes the exemplar face(s) used to represent a person or cluster.

| Current Term | Location | File | Line/Context |
|-------------|----------|------|-------------|
| "representative face" | Component logic | `ClusterCard.svelte` | `representativeFaceId` prop |
| "prototype" | Sort logic | `SuggestionGroupCard.svelte` | `isMultiPrototypeMatch` |
| "centroids" | Dialog | `ComputeCentroidsDialog.svelte` | "Compute Centroids" |
| "embeddings" | Dialog description | `ComputeCentroidsDialog.svelte` | "centroid embeddings" |

**Conflict Summary**: "representative face" vs "prototype" vs "centroid" -- three terms for conceptually related things (the face or vector that represents a person/cluster).

---

## Canonical Terminology

The following table defines the single canonical term for each concept. All UI text should be updated to use these terms consistently.

| Concept | Canonical Term | Rationale |
|---------|---------------|-----------|
| Named person | **"Named"** | Clear, simple, matches what the user did (gave a name) |
| Unnamed face group | **"Unnamed"** | Direct opposite of "Named", intuitive |
| Low-confidence singles | **"Unclustered"** | Describes what happened technically without jargon |
| Face grouping | **"Group"** (user-facing) / `cluster` (code/API) | "Group" is friendlier; keep `cluster` in code for backward compat |
| Assignment recommendation | **"Suggestion"** (keep current) | Already well-established across the app |
| Finding more suggestions | **"Find More"** (keep current) | The button label is clear; just unify dialog titles |
| Confidence score | **"Match"** (user-facing %) / `confidence` (code) | Users understand "85% match" better than "0.85 confidence" |
| Exemplar face | **"Prototype"** (code) / hidden from user | Users don't need to see "centroid" or "prototype" -- just "Find More" |
| Score threshold | **"Minimum Match"** | Unifies "similarity", "confidence", "score" in user-facing text |

### Alternative Considered: Keep "Identified"/"Unidentified"

The existing "Identified"/"Unidentified" pair is clear and already widely used. If the team prefers to keep these terms, only the noise category and inconsistent duplicates need to change. The key requirement is CONSISTENCY -- pick one and use it everywhere.

---

## Implementation Steps

### Step 1: Create a UI constants file for terminology (15 minutes)

**File**: `image-search-ui/src/lib/constants/terminology.ts` (new file)

```typescript
/**
 * Canonical UI terminology for face recognition concepts.
 *
 * All user-facing text should reference these constants to ensure
 * consistency across the application. Internal API field names
 * (like 'identified', 'unidentified', 'noise') remain unchanged.
 */

export const PERSON_STATUS = {
    /** Person has been given a name */
    identified: {
        label: 'Named',
        filterLabel: 'Show Named',
        statsLabel: 'Named',
        tooltip: 'This person has been given a name',
        badge: 'Named',
    },
    /** Face group has not been assigned to a named person */
    unidentified: {
        label: 'Unnamed',
        filterLabel: 'Show Unnamed',
        statsLabel: 'Unnamed',
        tooltip: 'This face group has not been assigned to a person yet',
        badge: 'Unnamed',
    },
    /** Low-confidence faces not clustered into a group */
    noise: {
        label: 'Unclustered',
        filterLabel: 'Show Unclustered',
        statsLabel: 'Unclustered',
        tooltip: 'Low-confidence faces that could not be grouped together',
        badge: 'Unclustered',
    },
} as const;

export const FACE_CONCEPTS = {
    /** A group of similar faces */
    group: {
        singular: 'group',
        plural: 'groups',
        pageTitle: 'Unnamed Face Groups',
        pageSubtitle: 'Review and name face groups to identify people in your photos.',
    },
    /** An automated recommendation to assign a face to a person */
    suggestion: {
        singular: 'suggestion',
        plural: 'suggestions',
        pageTitle: 'Face Suggestions',
        pageSubtitle: 'Review automated face-to-person match recommendations.',
    },
    /** Confidence/similarity score */
    match: {
        label: 'Match',
        thresholdLabel: 'Minimum Match',
    },
} as const;
```

### Step 2: Update UnifiedPersonCard.svelte (20 minutes)

**File**: `image-search-ui/src/lib/components/faces/UnifiedPersonCard.svelte`

```svelte
<!-- CURRENT badge logic (approximate): -->
{#if type === 'identified'}
    <Badge variant="success">Identified</Badge>
{:else if type === 'unidentified'}
    <Badge variant="warning">Needs Name</Badge>
{:else if type === 'noise'}
    <Badge variant="destructive">Review</Badge>
{/if}
```

```svelte
<!-- AFTER: -->
<script lang="ts">
    import { PERSON_STATUS } from '$lib/constants/terminology';
</script>

{#if type === 'identified'}
    <Badge variant="success" title={PERSON_STATUS.identified.tooltip}>
        {PERSON_STATUS.identified.badge}
    </Badge>
{:else if type === 'unidentified'}
    <Badge variant="warning" title={PERSON_STATUS.unidentified.tooltip}>
        {PERSON_STATUS.unidentified.badge}
    </Badge>
{:else if type === 'noise'}
    <Badge variant="destructive" title={PERSON_STATUS.noise.tooltip}>
        {PERSON_STATUS.noise.badge}
    </Badge>
{/if}
```

### Step 3: Update People page (`people/+page.svelte`) (30 minutes)

**File**: `image-search-ui/src/routes/people/+page.svelte`

Changes needed:

1. **Page subtitle** (line 188): "Browse identified people and face groups in your photos." --> "Browse named people and face groups in your photos."

2. **Stats labels** (lines 194-204):
   - "Identified" --> `PERSON_STATUS.identified.statsLabel`
   - "Unidentified" --> `PERSON_STATUS.unidentified.statsLabel`
   - "Unknown" --> `PERSON_STATUS.noise.statsLabel`

3. **Filter checkbox labels** (lines 254-264):
   - "Show Identified" --> `PERSON_STATUS.identified.filterLabel`
   - "Show Unidentified" --> `PERSON_STATUS.unidentified.filterLabel`
   - "Show Unknown Faces" --> `PERSON_STATUS.noise.filterLabel`

Note: The state variable names (`showIdentified`, `showUnidentified`, `showNoise`) and API parameter names (`includeIdentified`, `includeUnidentified`, `includeNoise`) should NOT change -- these are internal code names that match the backend API contract. Only the user-visible labels change.

### Step 4: Update Clusters page (`faces/clusters/+page.svelte`) (20 minutes)

**File**: `image-search-ui/src/routes/faces/clusters/+page.svelte`

Changes needed:

1. **Page title** (line 180-186):
   - `<title>Unknown Faces | Image Search</title>` --> `<title>Unnamed Face Groups | Image Search</title>`
   - `<h1>Unknown Faces</h1>` --> `<h1>Unnamed Face Groups</h1>`

2. **Page subtitle** (line 187-189):
   - "Review and label face clusters to identify people in your photos." --> "Review and name face groups to identify people in your photos."
   - "N% similarity" --> "N% match"

### Step 5: Update navigation links (`+layout.svelte`) (10 minutes)

**File**: `image-search-ui/src/routes/+layout.svelte`

The navigation links (line 60-61) currently say:
```html
<a href="/faces/suggestions">Suggestions</a>
<a href="/faces/clusters">Clusters</a>
```

Change to:
```html
<a href="/faces/suggestions">Suggestions</a>
<a href="/faces/clusters">Face Groups</a>
```

This aligns the navigation with the new "groups" terminology. The URL path `/faces/clusters` remains unchanged (URL stability).

### Step 6: Update CentroidResultsDialog and ComputeCentroidsDialog (20 minutes)

**File**: `image-search-ui/src/lib/components/faces/CentroidResultsDialog.svelte`

- Title: "Centroid Suggestions for {personName}" --> "Additional Suggestions for {personName}"
- Description: "Review and accept face suggestions found using centroid matching" --> "Review and accept additional face match suggestions"
- Remove "centroid" from any user-visible text (keep in code variable names)

**File**: `image-search-ui/src/lib/components/faces/ComputeCentroidsDialog.svelte`

- Title: "Compute Centroids for {personName}" --> "Find More Matches for {personName}"
- Description: "compute cluster centroids for match discovery" --> "search for additional faces that match this person"
- "Minimum Similarity" --> "Minimum Match"
- Remove "centroids", "embeddings" from user-visible text

### Step 7: Update ClusterCard.svelte (15 minutes)

**File**: `image-search-ui/src/lib/components/faces/ClusterCard.svelte`

- Aria label: "Face cluster with N faces" --> "Face group with N faces"
- Any visible "match" percentage label -- keep as "match" (already canonical)
- Any visible "cluster" text --> "group"

### Step 8: Update SuggestionGroupCard.svelte (15 minutes)

**File**: `image-search-ui/src/lib/components/faces/SuggestionGroupCard.svelte`

- "Unknown Person" fallback (line 140) --> "Unknown Person" (keep -- this is appropriate for unnamed)
- "suggestion" / "suggestions" count text (line 142) --> keep (already canonical)
- "Find More" button (line 166) --> keep (already good)
- "Compute centroids for optimized face matching" title (line 156) --> "Find additional face matches"

### Step 9: Update Suggestions page (`faces/suggestions/+page.svelte`) (15 minutes)

**File**: `image-search-ui/src/routes/faces/suggestions/+page.svelte`

- Title: "Face Suggestions" (line 476, 484) --> keep (already canonical)
- Toast: "Finding more suggestions for {personName}..." (line 384) --> keep
- Toast: "Found N new suggestions for {personName}" (line 370) --> keep
- "persons" in pagination label (line 533) --> "people" (line 533: `{option} persons` --> `{option} people`)

### Step 10: Test all affected components (1-2 hours)

Run the full test suite to catch any broken assertions that reference old text:

```bash
cd image-search-ui
make test
```

Update any test assertions that match on old label text (e.g., tests checking for "Needs Name" should now check for "Unnamed", tests checking for "Review" should now check for "Unclustered").

Search for old terms in tests:

```bash
grep -rn "Needs Name\|Review\|Unknown Faces\|Identified\|Unidentified\|Show Unknown" src/tests/
```

---

## Terminology Mapping Reference

This table maps every old term to its new canonical term for find-and-replace operations:

| Old Term (user-visible) | New Term | Where Used |
|------------------------|----------|------------|
| "Identified" (badge) | "Named" | UnifiedPersonCard |
| "Needs Name" (badge) | "Unnamed" | UnifiedPersonCard |
| "Review" (badge) | "Unclustered" | UnifiedPersonCard |
| "Show Identified" | "Show Named" | People page filter |
| "Show Unidentified" | "Show Unnamed" | People page filter |
| "Show Unknown Faces" | "Show Unclustered" | People page filter |
| "Identified" (stats) | "Named" | People page stats |
| "Unidentified" (stats) | "Unnamed" | People page stats |
| "Unknown" (stats) | "Unclustered" | People page stats |
| "Unknown Faces" (page title) | "Unnamed Face Groups" | Clusters page |
| "Clusters" (nav) | "Face Groups" | Layout navigation |
| "face clusters" (subtitle) | "face groups" | Clusters page |
| "face groups" (subtitle) | "face groups" | People page (already correct) |
| "Centroid Suggestions" (dialog) | "Additional Suggestions" | CentroidResultsDialog |
| "centroid matching" (dialog) | "face match suggestions" | CentroidResultsDialog |
| "Compute Centroids" (dialog) | "Find More Matches" | ComputeCentroidsDialog |
| "match discovery" (dialog) | "additional faces" | ComputeCentroidsDialog |
| "Minimum Similarity" | "Minimum Match" | ComputeCentroidsDialog |
| "Face cluster with N faces" (aria) | "Face group with N faces" | ClusterCard |
| "N persons" (pagination) | "N people" | Suggestions page |

### Terms NOT changed (already canonical or internal-only):

| Term | Reason for keeping |
|------|-------------------|
| `cluster_id`, `clusterId` | Internal API field -- changing would be a breaking API change |
| `showIdentified`, `showUnidentified`, `showNoise` | Internal state variable names that match API params |
| `identifiedPeople`, `unidentifiedPeople`, `noisePeople` | Internal derived variable names |
| `type: 'identified' \| 'unidentified' \| 'noise'` | Backend API enum values -- changing requires API migration |
| `confidence` (API field) | Internal API field name |
| `prototype`, `centroid` (code variables) | Internal code concepts |
| `isMultiPrototypeMatch` | Internal API field |
| "suggestions" (page) | Already canonical |
| "Find More" (button) | Already canonical |

---

## i18n Considerations

The terminology constants file (`constants/terminology.ts`) is structured to support future internationalization:

1. **All user-visible strings in one place**: The `PERSON_STATUS` and `FACE_CONCEPTS` objects contain all translatable face-related strings
2. **Keyed by concept, not translation**: Each entry is keyed by the API value (`identified`, `unidentified`, `noise`) making it easy to create locale-specific versions
3. **Future i18n migration path**: Replace the constants file with a translation function:

```typescript
// Future: import { t } from '$lib/i18n';
// PERSON_STATUS.identified.label --> t('faces.status.identified.label')
```

For now, the constants file provides a single source of truth without the overhead of a full i18n framework.

---

## Verification

### Visual Verification Checklist

After applying all changes, verify each page displays consistent terminology:

- [ ] **People page**: Stats show "Named", "Unnamed", "Unclustered"
- [ ] **People page**: Filters show "Show Named", "Show Unnamed", "Show Unclustered"
- [ ] **People page**: Person cards show "Named", "Unnamed", "Unclustered" badges
- [ ] **Navigation**: Shows "Face Groups" instead of "Clusters"
- [ ] **Face Groups page** (was Clusters): Title says "Unnamed Face Groups"
- [ ] **Face Groups page**: Subtitle says "face groups" and "N% match"
- [ ] **Suggestions page**: Title says "Face Suggestions" (unchanged)
- [ ] **Suggestions page**: "N people" in pagination (not "N persons")
- [ ] **Find More dialog**: Says "Find More Matches for {name}" (not "Compute Centroids")
- [ ] **Centroid results dialog**: Says "Additional Suggestions" (not "Centroid Suggestions")

### Automated Testing

```bash
cd image-search-ui

# Run full test suite
make test

# Check for any remaining old terminology in source (excluding test files and types)
grep -rn "Needs Name\|\"Review\"\|Unknown Faces\|Compute Centroids\|Centroid Suggestions\|match discovery\|Minimum Similarity" \
    src/lib/components/ src/routes/ \
    --include="*.svelte" --include="*.ts" \
    | grep -v "test" | grep -v "node_modules"

# Expected: ZERO results (all old terms replaced)
```

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Tests break due to changed text | High | Low | Update test assertions in same PR; search for old terms in test files |
| Users confused by terminology change | Medium | Low | The new terms are more intuitive; existing users will adapt quickly |
| Screenshots in docs become outdated | Low | Very Low | Update screenshots separately if needed |
| Backend API enum values conflict with new terms | Low | None | API values (`identified`, `unidentified`, `noise`) are NOT changed; only UI display labels change |
| Missing an occurrence of old terminology | Medium | Low | Grep verification catches most; visual review catches the rest |
| Accessibility labels become inconsistent | Low | Medium | Update aria-labels at the same time as visible text |

## Dependencies

- None. This is a UI-only change that does not affect the backend API, database, or any external contracts.
- All API field names and enum values remain unchanged.
- URL paths remain unchanged (e.g., `/faces/clusters` stays the same even though nav says "Face Groups").

## Files Modified (Summary)

| File | Changes |
|------|---------|
| `src/lib/constants/terminology.ts` | NEW: Canonical terminology constants |
| `src/lib/components/faces/UnifiedPersonCard.svelte` | Badge labels and tooltips |
| `src/routes/people/+page.svelte` | Stats labels, filter labels, subtitle |
| `src/routes/faces/clusters/+page.svelte` | Page title, subtitle, similarity text |
| `src/routes/+layout.svelte` | Navigation link text |
| `src/lib/components/faces/CentroidResultsDialog.svelte` | Dialog title and description |
| `src/lib/components/faces/ComputeCentroidsDialog.svelte` | Dialog title, description, slider label |
| `src/lib/components/faces/ClusterCard.svelte` | Aria label, visible cluster text |
| `src/lib/components/faces/SuggestionGroupCard.svelte` | Button title attribute |
| `src/routes/faces/suggestions/+page.svelte` | Pagination label |
| Test files (multiple) | Updated assertions to match new terms |

**Total**: 1 new file, ~10 modified files, ~40 string replacements.
