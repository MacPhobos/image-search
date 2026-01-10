# Face Assignment Undo & History Feature Proposal

> **Date**: 2026-01-09
> **Status**: Ready for Review
> **Priority**: HIGH - Prevents data pollution from user mistakes

---

## Executive Summary

### Problem Statement

When users assign faces to persons in the "Face Suggestions" view, mistakes can:
1. Propagate incorrect face embeddings to that person's profile
2. Trigger erroneous suggestions based on wrong face data
3. Pollute the vector database (Qdrant) with incorrect person associations

Currently, **there is no UI mechanism** to:
- Undo a mistaken assignment from the Face Suggestion Details dialog
- View assignment history for a person
- Identify and correct past mistakes

### Key Finding: Backend is Ready

**Good news**: The backend already has complete functionality for:
- ✅ Unassigning faces (`DELETE /api/v1/faces/faces/{face_id}/person`)
- ✅ Audit trail (`FaceAssignmentEvent` table with full operation history)
- ✅ Qdrant cleanup (automatic `person_id` removal on unassign)
- ✅ Suggestion expiration (related suggestions expired on unassign)

**The gap is in the UI layer** - these capabilities are not exposed to users.

### Recommended Solution

| Component | Effort | Impact |
|-----------|--------|--------|
| **1. Undo Button in Suggestion Dialog** | 1 day | HIGH - Immediate mistake correction |
| **2. Person Assignment History View** | 2-3 days | MEDIUM - Audit and bulk corrections |
| **3. "Recently Assigned" Quick Actions** | 1 day | HIGH - Catch mistakes early |

---

## Current Architecture Analysis

### Data Flow: Face Assignment

```
User accepts suggestion
        ↓
POST /api/v1/faces/suggestions/{id}/accept
        ↓
┌──────────────────────────────────────────────────────┐
│ 1. Database: face_instances.person_id = person_id   │
│ 2. Database: suggestion.status = 'accepted'         │
│ 3. Qdrant: payload.person_id = person_id            │
│ 4. Audit: FaceAssignmentEvent logged                │
│ 5. Background: PersonPrototype created (if quality) │
│ 6. Background: propagate_person_label_job queued    │
└──────────────────────────────────────────────────────┘
        ↓
    Face now associated with person
    Similar faces get new suggestions
```

### Data Flow: Face Unassignment (EXISTING)

```
DELETE /api/v1/faces/faces/{face_id}/person
        ↓
┌──────────────────────────────────────────────────────┐
│ 1. Database: face_instances.person_id = NULL        │
│ 2. Qdrant: DELETE payload.person_id                 │
│ 3. Database: Expire related suggestions             │
│ 4. Audit: FaceAssignmentEvent (operation=UNASSIGN)  │
│ 5. Background: update_asset_person_ids_job          │
└──────────────────────────────────────────────────────┘
        ↓
    Face returns to "unassigned" state
    Vector DB is clean - no person_id in payload
    Future suggestions won't use this face for person
```

### Existing Audit Trail (FaceAssignmentEvent)

The backend already captures every assignment operation:

```python
class FaceAssignmentEvent(Base):
    __tablename__ = "face_assignment_events"

    id: UUID
    operation: str           # 'ASSIGN_TO_PERSON', 'UNASSIGN_FROM_PERSON', 'MOVE_TO_PERSON'
    from_person_id: UUID     # Previous person (if any)
    to_person_id: UUID       # New person (if any)
    face_instance_ids: list  # Affected face IDs
    asset_ids: list          # Affected photo IDs
    face_count: int          # Number of faces affected
    photo_count: int         # Number of photos affected
    created_at: datetime     # Timestamp
    actor_id: UUID           # Future: User who made change
```

**This data is sufficient for complete undo functionality.**

---

## Proposed Solutions

### Solution 1: Undo Button in Suggestion Dialog (HIGH PRIORITY)

**Location**: Face Suggestion Details modal (`SuggestionDetailModal.svelte`)

**UX Flow**:
```
┌─────────────────────────────────────────────────────┐
│ Face Suggestion Details                        [X]  │
├─────────────────────────────────────────────────────┤
│  ┌────────┐                                         │
│  │ [face] │   Suggested: John Doe                   │
│  │ image  │   Confidence: 92%                       │
│  └────────┘                                         │
│                                                     │
│  Status: ✅ Accepted (2 minutes ago)                │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │ ⚠️ Made a mistake?                          │    │
│  │ [Undo Assignment] - Removes this face from  │    │
│  │ John Doe and returns it to unassigned.      │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│            [Close]                                  │
└─────────────────────────────────────────────────────┘
```

**Implementation**:

1. **Frontend**: Add "Undo Assignment" button to `SuggestionDetailModal.svelte`
   - Button visible when: `suggestion.status === 'accepted'` and face has `person_id`
   - Show confirmation dialog: "This will remove {face_id} from {person_name}"
   - Call existing API: `DELETE /api/v1/faces/faces/{face_id}/person`

2. **API Call** (already exists):
   ```typescript
   async function undoAssignment(faceId: string): Promise<void> {
     await fetch(`/api/v1/faces/faces/${faceId}/person`, {
       method: 'DELETE'
     });
   }
   ```

3. **After Undo**:
   - Suggestion status reverts to `pending` (or new status `reverted`)
   - Toast notification: "Assignment undone. Face returned to suggestions."
   - Modal closes or refreshes to show updated state

**Effort**: 1 day (frontend only)

---

### Solution 2: Person Assignment History View (MEDIUM PRIORITY)

**Location**: New tab on Person Detail page or standalone page

**UX Flow**:
```
┌─────────────────────────────────────────────────────────────────────┐
│ John Doe - Assignment History                                   [X] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ [Filter: All ▼] [Time: Last 30 days ▼]                              │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 📥 Jan 9, 2026 10:45 AM - Face assigned                        │ │
│ │ Photo: IMG_2024.jpg  [View]  [Undo]                             │ │
│ │ Source: Suggestion accepted (92% confidence)                    │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 📤 Jan 8, 2026 3:20 PM - Face unassigned                       │ │
│ │ Photo: IMG_2018.jpg  [View]                                     │ │
│ │ Previously assigned: Manual label                               │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 📥 Jan 7, 2026 9:00 AM - 5 faces assigned (bulk)               │ │
│ │ Photos: IMG_2010.jpg, IMG_2011.jpg, +3 more  [View All]  [Undo] │ │
│ │ Source: Cluster labeling                                        │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ Showing 10 of 47 events                        [Load More]          │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation**:

1. **Backend**: New endpoint (data already exists in `face_assignment_events`)
   ```python
   @router.get("/persons/{person_id}/assignment-history")
   async def get_person_assignment_history(
       person_id: UUID,
       limit: int = Query(20, ge=1, le=100),
       offset: int = Query(0, ge=0),
       operation: str | None = Query(None),  # Filter by operation type
   ) -> AssignmentHistoryResponse:
       """Get assignment events involving this person."""
       events = await db.execute(
           select(FaceAssignmentEvent)
           .where(
               or_(
                   FaceAssignmentEvent.from_person_id == person_id,
                   FaceAssignmentEvent.to_person_id == person_id
               )
           )
           .order_by(FaceAssignmentEvent.created_at.desc())
           .offset(offset)
           .limit(limit)
       )
       return AssignmentHistoryResponse(
           events=[...],
           total=total_count
       )
   ```

2. **Frontend**: New route `/faces/persons/{id}/history` or tab
   - Display timeline of events
   - Show face thumbnails for each event
   - "Undo" button for reversible operations
   - Filtering by date range and operation type

3. **Bulk Undo**:
   ```typescript
   async function undoBulkAssignment(eventId: string): Promise<void> {
     // Get event details
     const event = await getEvent(eventId);

     // Unassign all faces from that event
     for (const faceId of event.face_instance_ids) {
       await fetch(`/api/v1/faces/faces/${faceId}/person`, {
         method: 'DELETE'
       });
     }
   }
   ```

**Effort**: 2-3 days (backend endpoint + frontend page)

---

### Solution 3: "Recently Assigned" Quick Actions (HIGH PRIORITY)

**Location**: Floating panel or sidebar in Face Suggestions page

**Purpose**: Catch mistakes quickly before they propagate

**UX Flow**:
```
┌───────────────────────────────────────────┐
│ Recently Assigned (last 10 minutes)       │
├───────────────────────────────────────────┤
│                                           │
│ ┌─────┐ → John Doe     [Undo]            │
│ │face │   IMG_2024.jpg                    │
│ └─────┘   2 min ago                       │
│                                           │
│ ┌─────┐ → Jane Smith   [Undo]            │
│ │face │   IMG_2019.jpg                    │
│ └─────┘   5 min ago                       │
│                                           │
│ ┌─────┐ → John Doe     [Undo]            │
│ │face │   IMG_2015.jpg                    │
│ └─────┘   8 min ago                       │
│                                           │
└───────────────────────────────────────────┘
```

**Implementation**:

1. **Frontend State**: Track recent assignments in session/local storage
   ```typescript
   interface RecentAssignment {
     faceId: string;
     personId: string;
     personName: string;
     thumbnailUrl: string;
     assignedAt: Date;
   }

   let recentAssignments = $state<RecentAssignment[]>([]);

   // Add when user accepts suggestion
   function trackAssignment(face: Face, person: Person) {
     recentAssignments = [
       { faceId: face.id, personId: person.id, ... },
       ...recentAssignments.slice(0, 9)  // Keep last 10
     ];
   }
   ```

2. **Quick Undo**:
   ```typescript
   async function quickUndo(assignment: RecentAssignment) {
     await fetch(`/api/v1/faces/faces/${assignment.faceId}/person`, {
       method: 'DELETE'
     });
     recentAssignments = recentAssignments.filter(a => a.faceId !== assignment.faceId);
     toast.success(`Removed face from ${assignment.personName}`);
   }
   ```

**Effort**: 1 day (frontend only)

---

## Data Integrity Guarantees

### After Undo: What Gets Cleaned Up

The existing `DELETE /api/v1/faces/faces/{face_id}/person` endpoint ensures:

| Component | Cleanup Action | Verification |
|-----------|----------------|--------------|
| **PostgreSQL** | `face_instances.person_id = NULL` | `SELECT person_id FROM face_instances WHERE id = ?` |
| **Qdrant** | `delete_payload(keys=['person_id'])` | Vector no longer matches person queries |
| **Suggestions** | Related suggestions expired | `status = 'expired'` for pending suggestions |
| **Audit Log** | Event logged | `FaceAssignmentEvent(operation='UNASSIGN')` |
| **Photo metadata** | `person_ids` array updated | Background job updates photo's `people` field |

### What About Prototypes?

**Question**: If the mistakenly assigned face became a prototype, does undo remove it?

**Current Behavior**: The `PersonPrototype` is **NOT automatically deleted** on unassign.

**Recommendation**: Add prototype cleanup to the unassign flow:

```python
# In faces.py unassign endpoint
async def unassign_face_from_person(face_id: UUID, db: AsyncSession):
    face = await get_face(db, face_id)

    # Delete prototype if this face was one
    await db.execute(
        delete(PersonPrototype)
        .where(PersonPrototype.face_instance_id == face_id)
    )

    # Rest of unassign logic...
```

### What About Propagated Suggestions?

**Question**: If the wrong face triggered suggestions for other photos, what happens?

**Current Behavior**: Suggestions are **expired** when the source face is unassigned.

**Verification**: Check `face_suggestions` table:
```sql
SELECT * FROM face_suggestions
WHERE status = 'expired'
AND source_face_id = '{unassigned_face_id}';
```

---

## Implementation Phases

### Phase 1: Quick Wins (Week 1)

| Task | Effort | Files Changed |
|------|--------|---------------|
| Add "Undo" button to SuggestionDetailModal | 4 hrs | `SuggestionDetailModal.svelte` |
| Add confirmation dialog | 2 hrs | New `UndoConfirmDialog.svelte` |
| Add "Recently Assigned" sidebar | 4 hrs | `+page.svelte` |
| Toast notifications for undo | 1 hr | Existing toast system |
| **Total** | **~1.5 days** | |

### Phase 2: History View (Week 2)

| Task | Effort | Files Changed |
|------|--------|---------------|
| Backend: Assignment history endpoint | 4 hrs | `routes/faces.py`, `schemas.py` |
| Frontend: History page | 8 hrs | New route `/faces/persons/[id]/history/` |
| Bulk undo functionality | 4 hrs | API + UI |
| Filtering and pagination | 4 hrs | Frontend + backend params |
| **Total** | **~2.5 days** | |

### Phase 3: Hardening (Week 3)

| Task | Effort | Files Changed |
|------|--------|---------------|
| Prototype cleanup on unassign | 2 hrs | `routes/faces.py` |
| Add undo tests | 4 hrs | `tests/` |
| Performance optimization | 2 hrs | Indexes, batch queries |
| Documentation | 2 hrs | API docs, user guide |
| **Total** | **~1.5 days** | |

---

## API Reference

### Existing Endpoints (No Changes Needed)

**Unassign Single Face**:
```
DELETE /api/v1/faces/faces/{face_id}/person
Response: 204 No Content
```

**Bulk Remove from Person**:
```
POST /api/v1/faces/persons/{person_id}/photos/bulk-remove
Body: { "asset_ids": ["uuid1", "uuid2"] }
Response: { "removed_count": 2 }
```

### New Endpoint Needed

**Get Assignment History**:
```
GET /api/v1/faces/persons/{person_id}/assignment-history
Query Params:
  - limit: int (default 20, max 100)
  - offset: int (default 0)
  - operation: string (optional: ASSIGN, UNASSIGN, MOVE)
  - since: datetime (optional)

Response:
{
  "events": [
    {
      "id": "uuid",
      "operation": "ASSIGN_TO_PERSON",
      "created_at": "2026-01-09T10:45:00Z",
      "face_count": 1,
      "photo_count": 1,
      "face_instance_ids": ["uuid"],
      "asset_ids": ["uuid"],
      "from_person_id": null,
      "to_person_id": "person-uuid"
    }
  ],
  "total": 47,
  "offset": 0,
  "limit": 20
}
```

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Undo button visibility** | 100% of accepted suggestions | UI audit |
| **Undo completion time** | < 2 seconds | API response time |
| **Data integrity** | 0 orphaned references | Nightly consistency check |
| **User mistake recovery** | < 30 seconds to undo | UX testing |
| **Qdrant consistency** | 100% sync after undo | Verify `person_id` removed |

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Prototype not cleaned up | Medium | Medium | Add explicit prototype deletion in unassign |
| Undo after propagation | Low | High | Expire all related suggestions automatically |
| Performance on large history | Low | Low | Pagination + indexes on `created_at` |
| Concurrent modifications | Low | Medium | Optimistic locking on face row |

---

## Conclusion

The backend infrastructure for undo functionality **already exists and is robust**:
- ✅ Unassign endpoint with proper cleanup
- ✅ Audit trail for full history
- ✅ Qdrant synchronization

**The work is primarily frontend UI**:
1. Add "Undo" button to suggestion detail modal (1 day)
2. Add "Recently Assigned" panel for quick corrections (1 day)
3. Create assignment history view per person (2-3 days)

**Total estimated effort: 4-5 days**

The data integrity guarantees are solid - the existing unassign endpoint handles:
- Database cleanup (`person_id = NULL`)
- Vector DB cleanup (`delete_payload(keys=['person_id'])`)
- Suggestion expiration (related pending suggestions)
- Audit logging (`FaceAssignmentEvent`)

**Recommendation**: Start with Phase 1 (quick wins) to immediately address user pain points, then build the history view in Phase 2.

---

## Appendix: File Locations

**Backend**:
- `image-search-service/src/image_search_service/db/models.py` - FaceAssignmentEvent model
- `image-search-service/src/image_search_service/api/routes/faces.py` - Unassign endpoint
- `image-search-service/src/image_search_service/vector/face_qdrant.py` - Qdrant operations

**Frontend**:
- `image-search-ui/src/routes/faces/suggestions/` - Suggestions page
- `image-search-ui/src/lib/components/faces/SuggestionDetailModal.svelte` - Detail modal (needs undo button)

**Tests**:
- `image-search-service/tests/api/test_face_unassignment.py` - Backend tests
- `image-search-ui/src/tests/components/SuggestionDetailModal.test.ts` - Frontend tests
