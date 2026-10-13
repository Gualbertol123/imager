# Step 12: Clustering orchestration in GalleryDO

## Context
Phase C, step 12. The GalleryDO decides when to cluster (debounce), sends the gallery's faces to the Container's `/cluster`, and applies the result with stable person IDs, so names, hides and corrections persist (P2, P3). Group photos (P17) and partial faces (P18) are handled by `face_core.cluster()`. This step does the plumbing and the `photo_persons` index.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §6, §7, `docs/proposals.md` P2, P3, P14, P17, P18, `docs/reports/step-11.md`.

## Tasks
1. **Alarm logic** (spec §7): debounce 90 s after the last `addFaces`, capped at 10 min after the first dirty batch. Only one run at a time; if dirty again after a run, schedule another. Store run state in `meta`, so it survives DO restarts.
2. **Building the request:**
   - Read core and partial faces that are not excluded, plus locks, cannot-links, opt-out centroids and previous assignments.
   - Build the binary frame (step 6 format) in chunks, so memory stays bounded. At 100k faces the embedding array is about 51 MB, so stream it with a `ReadableStream` rather than one big buffer if the DO memory limit (128 MB) requires it; measure.
   - Send it to `FaceContainer` `/cluster` with the gallery's params from config.
3. **Applying the result** in one `transactionSync`:
   - Upsert persons with stable IDs; retire persons with no faces (keep the `merged_into` redirect).
   - Set `faces.person_id`, `assign_kind` and `assign_confidence`; update centroids, counts and cover faces (respect `cover_locked`).
   - Rebuild `photo_persons`.
   - Write a `cluster_runs` row (algo, params, counts, ms).
   - Set `galleries.people_status='ready'` in D1 afterwards, through the Worker or a direct D1 binding in the DO.
4. **Read API** (RPC on GalleryDO, used by the Worker routes in steps 13 and 14):
   - `listPersons({includeHidden, minPhotos})`
   - `photosForPersons(ids[], {only: boolean}, cursor)`:
     - AND semantics: `GROUP BY photo_id HAVING COUNT(DISTINCT person_id)=n`
     - `only=true` additionally excludes photos containing any **other visible** person
     - returns `photo_id` list pages
   - `personFaces(personId)` for the photographer.
5. **Admin:** `POST /api/galleries/:id/people/recluster` (owner) forces a run. `GET …/people/runs` returns recent `cluster_runs`.
6. **Tests:**
   - Debounce and coalescing with fake timers or alarms.
   - One run at a time.
   - Apply is atomic (an injected failure leaves the old state).
   - Stable IDs across a re-run with 10% new faces.
   - AND and "only" queries on a fixture with group photos.
   - Excluded faces are never sent.
   - Use a Container mock that calls a deterministic fake clusterer.
7. **Local end to end:** upload the step-5 event (from `data/event1`, local only) and compare persons with the step-4 contact sheets. Report agreement as the share of faces with the same grouping (adjusted Rand index).

## Constraints
- All face reads and writes happen inside GalleryDO. The Worker never sees embeddings.
- Never send data from two galleries in one `/cluster` call.

## Done when
- The local end-to-end run produces persons. Adding 50 photos keeps IDs stable. Tests pass.

## Stop and report
`docs/reports/step-12.md` (timings for clustering the full event, ARI against step 4). Update `docs/STATUS.md`. Commit. Stop.
