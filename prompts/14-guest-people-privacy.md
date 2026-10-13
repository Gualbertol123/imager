# Step 14: Guest People view, privacy flows, expiry (Gate M4)

## Context
Phase C, step 14. This is the guest-facing People feature, with every privacy requirement from spec §8 implemented and tested. Many photos contain several people, so the multi-person filters matter (P17). After this step the feature is complete, but **it must not reach real guests until the lawyer has signed off (L1)**.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §4, §6, §8, `docs/proposals.md` P11, P13, P14, P17, `docs/reports/step-13.md`.

## Tasks
1. **Guest People API** (valid guest token for this gallery only):
   - `GET /g/:slug/people` returns visible persons with `photo_count ≥ MIN_PHOTOS`, ordered by photo count, with signed crop URLs. It returns `{enabled:false}` if the gallery has fewer than 6 such people or People is off.
   - `GET /g/:slug/people/photos?ids=a,b&only=0|1&cursor`:
     - AND semantics; `only=1` means "only these people" (P17)
     - maximum 5 ids
     - pages of photos with signed URLs
2. **Guest notice:**
   - Before the first People use per guest session, show a short notice (text in `apps/web/src/legal/guest-notice.v1.md`, draft): what it does, that it's only this gallery, deletion at expiry, how to hide yourself.
   - Store the acknowledgement on the session (`ack_notice_version`).
   - The API refuses People calls without the acknowledgement.
3. **UI:**
   - A face strip above the grid; tap to filter; multi-select chips with an AND label ("Photos with Anna + 2"); an "only these people" toggle; a clear button.
   - In the lightbox, chips for recognised visible persons in that photo, and **no face boxes**.
   - Empty and loading states.
4. **Hide me (P14):** `POST /g/:slug/people/:pid/hide-me`, with a confirm dialog explaining the effect. GalleryDO:
   - marks the person hidden (`guest_optout`) and excludes all its faces
   - sets embeddings to NULL and deletes the crops from R2
   - stores the centroid in `optout_centroids`
   - writes an audit row (no guest identity beyond the session id)

   The effect is immediate for all guests. Photographers see "hidden at a guest's request" and **cannot unhide** it.
5. **Not me (P13):** `POST /g/:slug/photos/:id/not-me {person}` creates a suggestion (rate-limited to 20 per session per day) and hides that photo from **this** guest's current filter (client-side). It shows in the photographer's Review queue (step 13): accept means an exclusion of that face; dismiss closes it.
6. **Expiry cron** (`scheduled`, daily). For galleries past `expires_at`:
   1. Set status `expiring`.
   2. Run GalleryDO `deleteAll()` (storage plus alarms).
   3. Delete R2 objects under `g/<gid>/` in batches of 1,000.
   4. Delete D1 rows (photos, then the gallery).
   5. **Verify:** R2 listing empty and DO `getStatus` empty; then mark `deleted` in `audit_log`, keeping a tombstone with id and dates only.
   - It is resumable if interrupted. The same path is used for owner-requested delete.
7. **Tests**, one or more per row of spec §8:
   - People off means no faces.
   - People requires a session and the notice acknowledgement.
   - Hide-me deletes embeddings and crops and blocks re-creation after a recluster with new photos of the same synthetic identity.
   - Disable deletes everything.
   - Expiry deletes everything and is verified.
   - Logs are redacted.
   - No guest endpoint returns face boxes.
   - A cross-gallery People access test.
8. Add an end-to-end measurement script `scripts/measure_deployed_precision.ts`. It takes `eval/labels.csv` face IDs (mapped by photo sha256 and face index) and computes guest-visible precision from the DO's assignments, to compare with M1.

## Constraints
- Never weaken a privacy requirement to finish faster. If something in spec §8 is unclear, ask.

## Done when
- All tests pass. Locally, the full flow works on my event gallery, and measured precision is within 1 point of M1.

## Stop and report
`docs/reports/step-14.md` with a privacy checklist table (spec §8 rows → test names), plus the **Gate M4** verdict. Remind me that L1 (legal sign-off) is required before real guests. Update `docs/STATUS.md`. Commit. Stop.
