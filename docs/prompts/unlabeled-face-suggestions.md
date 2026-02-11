In the UI we currently have the "Face Suggestions" view. 
It displays suggestions for previously labeled persons.
It shows only suggestions for persons that have been created and have has some faces assigned to the person.
This is great. We do want that.
What is missing is the ability to find photos containing faces for persons that have not been "created" yet (the person reconrd does not exist yet).

We want:
  - a view that will show groupings of photos containing a face that has not been assigned to a person yet.
  - this view should have means for the user to select a similarity threshold to use for grouping photos. The default should be fairly high to ensure we show groups that we are fairly confident about. The user can then lower the threshold to see more suggestions.
  - this view should be accessible from the "Face Suggestions" view using a component that allows to flip between the two views.

The main goal here is to make it easier to find photos containing faces for persons that have not been created yet.
This requires in depth research into how to utilize data in the database and qdrant to find the best way to do this.
The "Clusters" view provides some grouping functionality but it definitely does not cover this use case.

Research results are to be written to docs/research/unknown_person_face_detection/ directory in the monorepo.
Create an agent team to explore this from different angles:
- one teammate for UI/UX
- one teammate for backend
- one teammate for database and qdrant research
- one teammate for research
- one teammate playing devil's advocate

-----
Answers to architectere questions asked in docs/research/unknown_person_face_detection/05-devils-advocate.md

### Architecture Questions

1. **Is this real-time clustering or pre-computed?** pre-computed using 0.70 default threshold. 

2. **What is the relationship between this feature and the existing Clusters view?** different data entierly. We want to show groups of photos, where each group contains photos containing the same face.   

3. **How does creating a new person from this view interact with the suggestion pipeline?** Trigger a re-computation which may have a delay to process it.  

4. **What happens when the background clustering job runs while a user is actively using the unknown persons view?** Results are refreshed automatically. 

### Data Questions

5. **How many unlabeled faces do we expect in a typical deployment?** upwards of 50,000 unlabeled faces

6. **What is the expected ratio of "true new persons" to "noise faces" (background people, one-time appearances)?**
There will always be alot of noise faces. What we want to do is to find sets of similar faces, matched by similarity threshold. We want to have the ability to show only groups with a minimum of 5 faces. That default should be configurable using the UI admin settings.   

7. **Should the system remember which groups the user has dismissed?** yes, store the user dismissed groups in the database. 

### UX Questions

8. **What does the user see when there are 500+ unknown groups?** Pagination. 50 groups per page. Groups with most faces first.

9. **Can the user partially accept a group?** (Select 6 of 8 faces as the same person, reject 2.) Yes, there should be the ability to select specific faces in a group OR all faces in the group.

10. **How does this feature interact with the "Find More" button on existing person suggestions?** Both should be available. This is simply a UX improvement.

### Scope Questions

11. **Is the threshold slider a V1 requirement or a future enhancement?** Keep the slider for V1, we want the user to be able to tweak the threshold.

12. **Must this be a separate view, or could it be a section within the existing Suggestions page?** We want a tab

13. **What is the minimum viable version of this feature?** Could we ship "Suggested New Persons" (alternative 7.3 above) first and evaluate whether users need more control before building the full dynamic grouping system? Answer: Yes, ship "Suggested New Persons" (alternative 7.3 above) first and have user evaluate.

Answers to Open Questions Requiring User Input from the docs/research/unknown_person_face_detection/00-synthesis.md document:

8.1 Naming UX (Affects Phase 1-2): A: Require name immediately

8.2 Minimum Cluster Size for Display (Affects Phase 1-2): C: 5 faces minimum, with the ability to configure the default using the Admin UI settings. 

8.3 Confidence Visibility (Affects Phase 1-2): A: Show confidence prominently (badge on each group)

8.4 Cluster Persistence Across Runs (Affects Phase 1): B: Stable via membership hash (faces in cluster determine ID, dismissed state persists)

8.5 Post-Creation Behavior (Affects Phase 1-2): A: Always auto-trigger find-more (maximize recall)

8.6 Relationship Between Clusters View and Unlabeled Groups (Affects Phase 2): C: Evolve Clusters into "Advanced" mode accessible from Unlabeled Groups (single entry point)

Use these answers to update corresponding research documents for my review. 