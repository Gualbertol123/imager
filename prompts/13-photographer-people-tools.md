# Step 13: Photographer People tools and consent toggle

## Context
Phase C, step 13. Photographers must be able to turn People on (with a consent attestation) and correct mistakes before any guest sees the view. Every correction becomes a permanent lock, exclusion or cannot-link (P3), and must survive reclustering.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §3, §6, §8, `docs/proposals.md` P3, P11, P13, P17, P18, `docs/reports/step-12.md`.

## Decisions needed
D7 (markets). The attestation wording must be marked **draft until the lawyer approves it** (Track L).

## Tasks
1. **Enable People:** `POST /api/galleries/:id/people/enable {attest: true, text_version}`.
   - The dialog shows the attestation text (stored in `apps/api/src/legal/attestation.v1.md`, draft): host agreement, guests informed, no minors' data processed without parental agreement, the retention rule.
   - Store `consent_attested_at`, `consent_text_version` and the actor in `audit_log`. Set the DO flag, then enqueue **re-processing of already uploaded photos** with `faces=1` (derivatives are skipped if present).
2. **Disable People:** `POST …/people/disable`. GalleryDO `deleteAllFaceData()` (faces, persons, locks, crops in R2 under `g/<gid>/f/`) runs **immediately**; the audit log records it. Test that nothing remains.
3. **Correction endpoints** (owner only; each is a GalleryDO RPC that writes locks or exclusions, updates `photo_persons` immediately, and marks dirty):
   - `merge {from, into}`: locks the faces of `from` to `into`, sets `merged_into`, and moves name and hidden state sensibly
   - `split {person, face_ids}`: creates a new person, locks the selected faces to it, and adds a cannot-link between the old and new person
   - `move {face_id, to_person | new}`
   - `remove-face {face_id, reason: not_a_face | stranger}`: an exclusion
   - `hide {person, hidden}`
   - `rename {person, name|null}`: private to the photographer unless `show_names` is set on the gallery (default off)
   - `cover {person, face_id}`: sets `cover_locked`
4. **People management UI** (photographer):
   - A grid of persons (cover, photo count, hidden badge), sortable.
   - Person detail: all faces grouped as core, attached and locked, with multi-select for move, split and remove.
   - Merge by selecting two persons.
   - **"Unassigned" view:** faces not in any person, partial ones marked, with "assign to…".
   - **"Review" view:** low-confidence attachments first (lowest `assign_confidence`), and suggestions from guests (step 14 adds them; build the queue UI now).
   - **Photo view for photographers only:** face boxes with person labels, so corrections in group photos are easy (click a box → move or remove).
5. **Persistence tests:** run 10 mixed corrections, recluster (with a real `cluster()` in the Container mock), and assert every correction survived. Also: a merge then a split of the same faces; cannot-link prevents re-merge; removed faces never come back.

## Constraints
- Guests cannot reach any of these endpoints (add them to the access suite).
- No face boxes in any guest-facing view.

## Done when
- I can enable People on my event gallery, correct 10 things, press recluster and see my corrections kept. Tests pass, including the access suite.

## Stop and report
`docs/reports/step-13.md`. Update `docs/STATUS.md`. Commit. Stop.
