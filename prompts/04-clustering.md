# Step 04: Clustering (three algorithms, same-photo rule, locks, attach pass)

## Context
Phase A, step 4. This groups faces into people within one gallery.
- Photos often contain **several people**, so two faces in one photo can never be the same person (P17).
- **Partly covered faces** never create or bridge clusters. They can only attach to an existing person under strict rules (P18).
- We compare three algorithms (P4) and choose between them with data in step 5.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §6 (follow it step by step), `docs/proposals.md` P2, P3, P4, P14, P17, P18, `docs/reports/step-03.md`.

## Tasks
1. `face_core/cluster.py`, a pure function:
   ```
   cluster(face_ids, photo_ids, emb, tiers, *, locks, cannot_links, excluded,
           optout_centroids, prev_assign, params: ClusterParams) -> ClusterResult
   ```
   Implement spec §6 steps 1–10:
   - **k-NN graph:** exact chunked cosine, k = 30. Use sklearn `NearestNeighbors` only if it is faster with identical results. Apply the edge threshold `T_EDGE`, then remove same-photo edges.
   - **Algorithms:**
     - `agglomerative`: sklearn `AgglomerativeClustering(metric="cosine", linkage="average", distance_threshold=1 - T_EDGE)`, run per connected component of the thresholded graph to keep memory bounded.
     - `chinese_whispers`: our own implementation, with a fixed seed, sorted node order and a maximum iteration count.
     - `knn_dbscan`: a face is a core point if it has ≥ `MIN_SAMPLES` neighbours above `T_EDGE`. Clusters are connected core points; border faces join the nearest core point; the rest are outliers. Reimplemented from Immich's public docs only; do not read or copy Immich's AGPL code.
   - **Shared post-processing:** same-photo split, then locks and cannot-links, then minimum distinct photos (`MIN_PHOTOS`).
   - **Attach pass** for partial and unassigned core faces: `T_ATTACH` plus `MARGIN` over the second-best person, measured against the centroid **and** top-5 exemplars. Never attach to a person already present in that photo.
   - **Also:** the opt-out guard (`T_OPTOUT`); stable IDs against `prev_assign` with `scipy.optimize.linear_sum_assignment`; cover face selection.
   - `ClusterResult`: per-face assignment (`person_id`, `kind` = cluster | attach | lock, `confidence`), persons with stats and cover face, and diagnostics (timings, counts).
2. CLI:
   - `cluster --db … --algo agglomerative|chinese_whispers|knn_dbscan [--param KEY=VALUE …]` writes persons and assignments to the local SQLite.
   - `review --db … --out data/review/<algo>/` writes a static HTML contact sheet:
     - one section per person: cover first, then core faces, then attached partial faces with a badge
     - the number of distinct photos per person
     - unassigned faces at the end
3. Tests (pytest, synthetic embeddings drawn as Gaussian blobs on the unit sphere):
   - (a) Same-photo rule: two faces of one photo never share a person, even with identical embeddings.
   - (b) Determinism: same input and seed give identical output, for all three algorithms.
   - (c) Locks always win, even for a locked face far from its person.
   - (d) Cannot-linked persons are never merged.
   - (e) Partial faces never create a person and never connect two clusters.
   - (f) The attach margin is respected.
   - (g) Stable IDs: adding 10% new faces keeps ≥ 95% of person IDs.
   - (h) The opt-out guard excludes matching faces.
   - (i) Speed: 100k random 128-d faces cluster in < 120 s on 4 threads (mark as a slow test).

## Constraints
- `cluster()` is pure: no I/O and no globals. No new dependencies beyond the step-1 list.
- Do not tune thresholds on the real photos yet. Use the config defaults; tuning is step 5.

## Done when
- All tests pass. `review` HTML exists for all three algorithms on the event, so I can compare them side by side.

## Stop and report
`docs/reports/step-04.md`: per algorithm, the counts of persons, unassigned faces and attached partial faces, plus timings; anything clearly wrong in the contact sheets (file names only). Update `docs/STATUS.md`. Commit. Stop.
