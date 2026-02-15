# Navigation Reorganization: Move Queues & Vectors into Admin

**Date**: 2026-02-15
**Status**: Proposal
**Scope**: Frontend (image-search-ui) navigation and routing

---

## 1. Current State

### 1.1 Top-Level Navigation

The main navigation is defined inline in the root layout file:

**File**: `image-search-ui/src/routes/+layout.svelte` (lines 57-67)

```html
<nav class="nav">
    <a href="/">Search</a>
    <a href="/people">People</a>
    <a href="/faces/suggestions">Suggestions</a>
    <a href="/faces/clusters">Clusters</a>
    <a href="/categories">Categories</a>
    <a href="/training">Training</a>
    <a href="/queues">Queues</a>       <!-- TARGET: Move to Admin -->
    <a href="/vectors">Vectors</a>     <!-- TARGET: Move to Admin -->
    <a href="/admin">Admin</a>
</nav>
```

**Key observations**:
- Navigation items are **hardcoded anchor tags** (not a config array or dynamic)
- All 9 items are at the same visual weight -- flat horizontal row
- No dropdown menus, grouping indicators, or active-state highlighting
- The nav sits between the app title ("Mac'Image Search") and a health indicator
- CSS uses flexbox with 1.5rem gap between items

**Problem**: 9 top-level items is crowded. Queues (queue monitoring) and Vectors (Qdrant vector database management) are system/infrastructure concerns used infrequently by most users. They belong under Admin alongside the existing data management and settings features.

### 1.2 Current Admin Page

**File**: `image-search-ui/src/routes/admin/+page.svelte`

The Admin page uses **shadcn/ui Tabs** with two tabs:

| Tab | Component | Purpose |
|-----|-----------|---------|
| **Data Management** | `AdminDataManagement.svelte` | Person data import/export, delete all data (danger zone) |
| **Settings** | `FaceMatchingSettings.svelte` | Face matching thresholds, suggestion config, clustering config |

**Admin component inventory** (`src/lib/components/admin/`):
- `AdminDataManagement.svelte` -- Data management panel with delete-all and person data import/export
- `FaceMatchingSettings.svelte` -- Comprehensive face matching configuration (thresholds, prototypes, clustering, post-training)
- `PersonDataManagement.svelte` -- Person data import/export (embedded in AdminDataManagement)
- `DeleteAllDataModal.svelte` -- Confirmation dialog for deleting all application data
- `ExportPersonDataModal.svelte` -- Export dialog
- `ImportPersonDataModal.svelte` -- Import dialog

### 1.3 Queues Page

**File**: `image-search-ui/src/routes/queues/+page.svelte`

**Content**: Real-time background job queue monitoring dashboard
- Overview stats (total jobs, workers, busy/idle counts)
- Queue cards grid with per-queue status
- Workers panel showing individual worker status
- Auto-polling every 3 seconds
- Sub-route: `/queues/[queueName]` for individual queue detail with job listing

**Components** (`src/lib/components/queues/`):
- `QueueCard.svelte` -- Individual queue status card
- `WorkersPanel.svelte` -- Worker list with status
- `ConnectionIndicator.svelte` -- Redis connection status dot
- `JobStatusBadge.svelte` -- Job status pill
- `WorkerStatusBadge.svelte` -- Worker status pill
- `QueueJobsTable.svelte` -- Paginated job list table

### 1.4 Vectors Page

**File**: `image-search-ui/src/routes/vectors/+page.svelte`

**Content**: Qdrant vector database management
- Directory statistics table (vectors per directory)
- Delete vectors by directory
- Retrain vectors by directory
- Orphan vector cleanup
- Full collection reset (danger zone)
- Deletion history log with pagination

**Components** (`src/lib/components/vectors/`):
- `DirectoryStatsTable.svelte` -- Directory-level vector counts
- `DeleteConfirmationModal.svelte` -- Reusable delete confirmation dialog
- `RetrainModal.svelte` -- Retrain configuration dialog
- `DangerZone.svelte` -- Destructive operation buttons
- `DeletionLogsTable.svelte` -- Deletion audit log table
- `index.ts` -- Barrel export

---

## 2. Files That Would Change

### 2.1 Primary Changes (Required)

| File | Change | Impact |
|------|--------|--------|
| `src/routes/+layout.svelte` | Remove "Queues" and "Vectors" nav links | Reduces nav from 9 to 7 items |
| `src/routes/admin/+page.svelte` | Add "Queues" and "Vectors" tabs alongside existing tabs | Admin becomes the hub for all system/infra features |

### 2.2 Route Changes (Recommended)

| File | Change | Impact |
|------|--------|--------|
| `src/routes/admin/queues/+page.svelte` | **NEW** -- Move queue monitoring under admin route | URL becomes `/admin/queues` |
| `src/routes/admin/queues/[queueName]/+page.svelte` | **NEW** -- Move queue detail under admin route | URL becomes `/admin/queues/[queueName]` |
| `src/routes/admin/queues/[queueName]/+page.ts` | **NEW** -- Copy from existing queues route | Passes queueName param |
| `src/routes/admin/vectors/+page.svelte` | **NEW** -- Move vector management under admin route | URL becomes `/admin/vectors` |
| `src/routes/queues/+page.svelte` | **DELETE** or redirect to `/admin/queues` | Old URL deprecated |
| `src/routes/queues/+page.ts` | **DELETE** or redirect | Old URL deprecated |
| `src/routes/queues/[queueName]/+page.svelte` | **DELETE** or redirect | Old URL deprecated |
| `src/routes/queues/[queueName]/+page.ts` | **DELETE** or redirect | Old URL deprecated |
| `src/routes/vectors/+page.svelte` | **DELETE** or redirect to `/admin/vectors` | Old URL deprecated |

### 2.3 Internal Navigation Updates

| File | Change | Reason |
|------|--------|--------|
| `src/routes/queues/[queueName]/+page.svelte` (or its new location) | Update "Back to Queues" link from `/queues` to `/admin/queues` | Internal navigation consistency |
| `src/lib/components/admin/FaceMatchingSettings.svelte` | Update text referencing "Admin Panel" for queue status (line ~497) | Already references queues, keep accurate |

### 2.4 Test Updates

| File | Change |
|------|--------|
| `src/tests/routes/vectors.test.ts` | Update route path references if sub-routing is adopted |
| Any layout/nav tests (none currently exist) | Consider adding tests for nav structure |

### 2.5 Admin Layout (New, Recommended)

| File | Change |
|------|--------|
| `src/routes/admin/+layout.svelte` | **NEW** -- Shared admin layout with sidebar or tab navigation |

---

## 3. UX Recommendations

### 3.1 Recommended Approach: Tab-Based Navigation Within Admin

**Use the existing shadcn/ui Tabs pattern** already established in the Admin page. Extend from 2 tabs to 4 tabs.

**Proposed Admin Page Tabs**:

| Tab | Label | Content | Route |
|-----|-------|---------|-------|
| 1 | **Data Management** | Existing AdminDataManagement component | `/admin` (default) |
| 2 | **Settings** | Existing FaceMatchingSettings component | `/admin?tab=settings` |
| 3 | **Queues** | Existing queue monitoring dashboard | `/admin?tab=queues` |
| 4 | **Vectors** | Existing vector management dashboard | `/admin?tab=vectors` |

**Why tabs over alternatives**:

- **Consistency**: Admin already uses tabs (shadcn/ui Tabs component). Adding more tabs is the most natural extension.
- **Discoverability**: All admin features visible at a glance in the tab bar.
- **No new patterns**: No need to introduce sidebar navigation, nested routing, or card-based layouts.
- **Simple implementation**: Minimal routing changes -- add two tab panels to the existing page.

### 3.2 Alternative: Sub-Route Based Navigation

Instead of tabs on a single page, use SvelteKit sub-routes:

```
/admin                 -> Data Management (default)
/admin/settings        -> Settings
/admin/queues          -> Queue Monitoring
/admin/queues/[name]   -> Queue Detail
/admin/vectors         -> Vector Management
```

**Pros of sub-routes**:
- URL reflects current section (shareable, bookmarkable)
- Queue detail page (`/admin/queues/[name]`) works naturally as a nested route
- Browser back/forward navigation works between sections
- Each section loads independently (code splitting)

**Cons of sub-routes**:
- Requires a new `+layout.svelte` for admin with its own navigation
- More files to create and maintain
- Slightly more complex routing setup

### 3.3 Recommended Hybrid Approach

**Use sub-routes with an admin layout** that provides tab-like navigation. This gives the best of both approaches:

- Bookmarkable URLs per section
- The queue detail sub-page (`/admin/queues/[queueName]`) works naturally
- An `admin/+layout.svelte` provides consistent tab/section navigation across all admin pages
- Each section loads independently

This is the **recommended approach** because the Queues section already has a sub-route (`/queues/[queueName]`), which does not fit cleanly into a single-page tab model.

### 3.4 Information Architecture: Grouping Strategy

Organize the admin tabs/sections into two logical groups:

**Group 1: Configuration**
- Settings (face matching thresholds, clustering config)
- Data Management (import/export, danger zone)

**Group 2: System Monitoring**
- Queues (background job monitoring, workers)
- Vectors (Qdrant database management, cleanup)

This grouping separates "what you configure" from "what you monitor/maintain." In the tab bar, present them in this order:

```
[ Data Management ] [ Settings ] [ Queues ] [ Vectors ]
      ^--- Config group ---^      ^--- System group ---^
```

A subtle visual separator (a thin vertical line or extra spacing) between Settings and Queues can indicate the two groups without adding complexity.

### 3.5 Navigation Within Admin: Visual Treatment

The admin section navigation should:

- Use a **horizontal tab bar** at the top of the admin page (consistent with current design)
- Show the **active tab** with an underline or background highlight
- Be **sticky/fixed** within the admin content area so tabs remain accessible when scrolling long content (especially Settings and Vectors pages)
- Display a **section icon** next to each tab label for quick visual scanning:
  - Data Management: database icon
  - Settings: gear icon
  - Queues: list/stack icon
  - Vectors: grid/network icon

---

## 4. Implementation Approach

### Phase 1: Create Admin Layout with Sub-Routes (Recommended Path)

**Step 1**: Create `src/routes/admin/+layout.svelte` with tab-style navigation.

```svelte
<!-- src/routes/admin/+layout.svelte -->
<script lang="ts">
    import { page } from '$app/stores';
    import type { Snippet } from 'svelte';

    interface Props { children: Snippet; }
    let { children }: Props = $props();

    const tabs = [
        { href: '/admin', label: 'Data Management', exact: true },
        { href: '/admin/settings', label: 'Settings' },
        { href: '/admin/queues', label: 'Queues' },
        { href: '/admin/vectors', label: 'Vectors' },
    ];
</script>

<div class="admin-page">
    <header class="page-header">
        <h1>Admin Panel</h1>
        <p>System administration, configuration, and monitoring.</p>
    </header>

    <nav class="admin-tabs">
        {#each tabs as tab}
            <a
                href={tab.href}
                class="admin-tab"
                class:active={tab.exact
                    ? $page.url.pathname === tab.href
                    : $page.url.pathname.startsWith(tab.href)}
            >
                {tab.label}
            </a>
        {/each}
    </nav>

    <div class="admin-content">
        {@render children()}
    </div>
</div>
```

**Step 2**: Refactor existing admin page to remove its own tabs wrapper.

Update `src/routes/admin/+page.svelte` to render only the Data Management content (currently the default tab), removing the Tabs.Root wrapper since navigation is now handled by the layout.

**Step 3**: Create new route files.

- `src/routes/admin/settings/+page.svelte` -- Renders `<FaceMatchingSettings />`
- `src/routes/admin/queues/+page.svelte` -- Move content from `src/routes/queues/+page.svelte`
- `src/routes/admin/queues/[queueName]/+page.svelte` -- Move from `src/routes/queues/[queueName]/+page.svelte`
- `src/routes/admin/queues/[queueName]/+page.ts` -- Move from `src/routes/queues/[queueName]/+page.ts`
- `src/routes/admin/vectors/+page.svelte` -- Move content from `src/routes/vectors/+page.svelte`

**Step 4**: Update the root layout navigation.

In `src/routes/+layout.svelte`, remove the Queues and Vectors links:

```html
<!-- BEFORE (9 items) -->
<nav class="nav">
    <a href="/">Search</a>
    <a href="/people">People</a>
    <a href="/faces/suggestions">Suggestions</a>
    <a href="/faces/clusters">Clusters</a>
    <a href="/categories">Categories</a>
    <a href="/training">Training</a>
    <a href="/queues">Queues</a>
    <a href="/vectors">Vectors</a>
    <a href="/admin">Admin</a>
</nav>

<!-- AFTER (7 items) -->
<nav class="nav">
    <a href="/">Search</a>
    <a href="/people">People</a>
    <a href="/faces/suggestions">Suggestions</a>
    <a href="/faces/clusters">Clusters</a>
    <a href="/categories">Categories</a>
    <a href="/training">Training</a>
    <a href="/admin">Admin</a>
</nav>
```

**Step 5**: Add redirects for old URLs (optional but recommended).

Create redirect pages at the old routes so any bookmarks or external links continue to work:

```svelte
<!-- src/routes/queues/+page.svelte (redirect stub) -->
<script lang="ts">
    import { goto } from '$app/navigation';
    import { onMount } from 'svelte';
    onMount(() => goto('/admin/queues', { replaceState: true }));
</script>
```

Repeat for `/vectors` -> `/admin/vectors`.

**Step 6**: Update internal navigation links.

- Queue detail "Back to Queues" button: change from `goto('/queues')` to `goto('/admin/queues')`
- Any references in FaceMatchingSettings mentioning "Admin Panel" queue status are already accurate

**Step 7**: Update tests.

- Update `src/tests/routes/vectors.test.ts` route references
- Consider adding a layout navigation test

### Phase 2: (Optional) Clean Up Old Routes

Once redirects have been in place long enough:
- Delete `src/routes/queues/` directory
- Delete `src/routes/vectors/+page.svelte`

---

## 5. Wireframe Description

### 5.1 Main Navigation Bar (After Change)

```
+------------------------------------------------------------------+
| Mac'Image Search                                                  |
|                                                                    |
| [Search] [People] [Suggestions] [Clusters] [Categories]          |
| [Training] [Admin]                                    [* Backend] |
+------------------------------------------------------------------+
```

7 items instead of 9. The nav breathes easier. "Admin" becomes the gateway to all system/infra tools.

### 5.2 Admin Page with Sub-Route Navigation

```
+------------------------------------------------------------------+
| Admin Panel                                                       |
| System administration, configuration, and monitoring.             |
|                                                                    |
| [Data Management]  [Settings]  |  [Queues]  [Vectors]            |
|  ^--- active tab                  ^--- system monitoring group    |
+------------------------------------------------------------------+
|                                                                    |
|  Data Management                                                  |
|  ---------------------------------------------------------------- |
|  Manage all application data across the system.                   |
|                                                                    |
|  +-- Person Data Management --+                                   |
|  |  [Export Persons]  [Import Persons]                            |
|  +-----------------------------+                                  |
|                                                                    |
|  +-- Danger Zone (red border) --+                                 |
|  |  Delete All Application Data                                   |
|  |  [Delete All Data]                                             |
|  +-------------------------------+                                |
|                                                                    |
+------------------------------------------------------------------+
```

### 5.3 Admin > Queues View

```
+------------------------------------------------------------------+
| Admin Panel                                                       |
| System administration, configuration, and monitoring.             |
|                                                                    |
| [Data Management]  [Settings]  |  [Queues]  [Vectors]            |
|                                    ^--- active tab                |
+------------------------------------------------------------------+
|                                                                    |
|  Queue Monitoring                              [* Redis] [Refresh]|
|                                                                    |
|  +----------+  +----------+  +----------+  +----------+          |
|  | 42       |  | 3        |  | 2        |  | 1        |          |
|  | Total    |  | Workers  |  | Busy     |  | Idle     |          |
|  +----------+  +----------+  +----------+  +----------+          |
|                                                                    |
|  Queues                                                           |
|  +----------+  +----------+  +----------+  +----------+          |
|  | default  |  | high     |  | low      |  | faces    |          |
|  | 12 jobs  |  | 8 jobs   |  | 15 jobs  |  | 7 jobs   |          |
|  +----------+  +----------+  +----------+  +----------+          |
|                                                                    |
|  Workers                                                          |
|  [Worker table with status badges]                                |
|                                                                    |
+------------------------------------------------------------------+
```

### 5.4 Admin > Vectors View

```
+------------------------------------------------------------------+
| Admin Panel                                                       |
| System administration, configuration, and monitoring.             |
|                                                                    |
| [Data Management]  [Settings]  |  [Queues]  [Vectors]            |
|                                             ^--- active tab       |
+------------------------------------------------------------------+
|                                                                    |
|  Vector Management                                                |
|  Manage Qdrant vector database                                    |
|                                                                    |
|  Directory Statistics                    Total Vectors: 45,231    |
|  +----------------------------------------------------+          |
|  | Directory     | Vectors | Actions                  |          |
|  |---------------|---------|--------------------------|          |
|  | /photos/2024  | 12,340  | [Delete] [Retrain]       |          |
|  | /photos/2023  | 8,721   | [Delete] [Retrain]       |          |
|  +----------------------------------------------------+          |
|                                                                    |
|  +-- Danger Zone (red border) --+                                 |
|  |  [Cleanup Orphans]  [Reset Collection]                        |
|  +-------------------------------+                                |
|                                                                    |
|  Deletion History                                                 |
|  [Paginated log table]                                            |
|                                                                    |
+------------------------------------------------------------------+
```

---

## 6. Considerations

### 6.1 Queue Detail Sub-Route

The queue detail page (`/queues/[queueName]`) is the strongest argument for using sub-routes rather than pure tabs. With sub-routes, the queue detail becomes `/admin/queues/[queueName]` and the "Back to Queues" navigation naturally returns to `/admin/queues`. With a pure tab approach, the queue detail page would need special handling (perhaps a modal or in-page drill-down) which adds complexity.

### 6.2 Polling Behavior

Both Queues and Vectors pages have auto-refresh behavior:
- Queues polls every 3 seconds
- Vectors loads on mount

With sub-routes, each page mounts/unmounts independently, so polling starts and stops correctly as users switch tabs. With a single-page tab approach, both pages would mount simultaneously and both would poll, which wastes resources. **Sub-routes are better for polling pages**.

### 6.3 Settings Currently Uses Tabs Wrapper

The current Admin page wraps Data Management and Settings in `Tabs.Root`. Moving to sub-routes means removing this wrapper and instead using the layout-level navigation. The Settings component (`FaceMatchingSettings.svelte`) is self-contained and works standalone without the tab wrapper.

### 6.4 Test Impact

Existing tests:
- `src/tests/routes/vectors.test.ts` -- Tests the vectors page rendering; may need path updates
- `src/tests/components/vectors/*.test.ts` -- Component tests unaffected (test components in isolation)
- No existing tests for the queues page or admin page

The component tests for vectors and queues are isolated and will not need changes. Only route-level tests that reference paths may need updates.

### 6.5 Top Nav Item Count After Change

After removing Queues and Vectors, the top nav has 7 items:
- Search, People, Suggestions, Clusters, Categories, Training, Admin

This is a comfortable number. If further reduction is desired in the future, "Suggestions" and "Clusters" could similarly be grouped under a "Faces" section, reducing to 5-6 items. But that is out of scope for this proposal.

---

## 7. Summary of Recommendation

| Aspect | Decision |
|--------|----------|
| **Approach** | Sub-route based with admin layout |
| **URL scheme** | `/admin`, `/admin/settings`, `/admin/queues`, `/admin/vectors` |
| **Navigation pattern** | Horizontal tabs in admin layout (consistent with existing design) |
| **Tab grouping** | Config (Data Management, Settings) + System (Queues, Vectors) |
| **Queue detail** | `/admin/queues/[queueName]` as nested route |
| **Old URL handling** | Redirect stubs at `/queues` and `/vectors` |
| **Component changes** | None -- all existing components work as-is |
| **Estimated effort** | ~2-4 hours for an experienced developer |

This approach reduces top navigation clutter, logically groups system administration features, preserves all existing functionality, and requires no changes to the actual queue/vector components -- only their routing wrappers.
