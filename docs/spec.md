# Technical spec

This is the design. The reasons for each choice are in [`proposals.md`](proposals.md) (referred to as P1–P18). Verified platform limits are in [`cloudflare-notes.md`](cloudflare-notes.md).

## 1. Product scope

**A. Gallery.** Photographers create galleries, upload photos, set a PIN and an expiry date, and share a link. Guests browse, view and download.

**B. People.** When enabled per gallery, faces are detected and grouped into people within that gallery. Guests see a strip of faces; one face shows that person's photos; several faces show photos containing all of them (with an "only these people" option). Photographers merge, split, move, hide, rename and choose cover faces.

**Later:** selfie search for open galleries, zip downloads, presigned uploads, watermarking.

## 2. Architecture

```
Browser (React app, same origin)
   │
   ▼
Worker  "api"  (Hono)  ─────────────►  R2  originals/ derivatives/ crops/       (EU jurisdiction)
   │   │   │                           D1  owners, sessions, galleries, photos  (EU)
   │   │   └─ static assets (apps/web build)
   │   │
   │   └─ POST photo ──► R2.put (streamed) ──► Queue "photo-process"
   │                                              │  batch 10
   │                                              ▼
   │                               Queue consumer (same Worker)
   │                                 stream original from R2 ──► FaceContainer  POST /process
   │                                 ◄── JSON: info, derivatives, faces(bbox, landmarks, quality, tier,
   │                                                                   embedding, crop)
   │                                 write derivatives + crops to R2, photo row to D1
   │                                 GalleryDO.addFaces(faces)
   │                                              │
   ▼                                              ▼
GalleryDO (one per gallery, SQLite storage, EU)
   faces, persons, face_locks, person_cannot_links, exclusions,
   suggestions, photo_persons, cluster_runs
   alarm (debounce 90 s after last addFaces) ──► FaceContainer POST /cluster
                                                 (binary embeddings + locks) ──► assignments
   ◄── apply locks, map to stable IDs, rebuild photo_persons

Cron (daily) ──► expire galleries: GalleryDO.deleteAll, R2 prefix delete, D1 rows, verify
```

Rules:
- The Worker is the only public entry point. The Container has no public route and no R2 credentials (P9).
- Images are always served through the Worker after an access check, using short-lived signed URLs (`/i/:token`). The bucket is never public.
- All face data for a gallery lives in its GalleryDO (P1). Nothing in the code can open two GalleryDOs in one face operation, and a lint rule plus review checklist enforce it.

## 3. Data model

### D1 (global)

```sql
owners(id TEXT PK, email TEXT UNIQUE, created_at, plan TEXT, storage_bytes INT)
login_tokens(token_hash TEXT PK, owner_id, expires_at, used_at)
sessions(id_hash TEXT PK, owner_id, created_at, expires_at, user_agent)
galleries(id TEXT PK, owner_id, name, slug UNIQUE, created_at, expires_at,
          pin_hash NULL, people_enabled INT DEFAULT 0,
          consent_attested_at NULL, consent_text_version NULL,
          status TEXT,  -- active | expiring | deleted
          photo_count INT, people_status TEXT)  -- off | processing | ready
photos(id TEXT PK, gallery_id, r2_key, thumb_key, web_key, content_hash,
       width, height, taken_at NULL, bytes INT, hidden INT DEFAULT 0,
       face_count INT NULL, process_status TEXT, process_error NULL, created_at)
audit_log(id, gallery_id, actor, action, detail_json, created_at)
```

Indexes: `photos(gallery_id, taken_at)`, `photos(gallery_id, content_hash)` (dedupe), `galleries(owner_id)`.

### GalleryDO SQLite (per gallery)

```sql
faces(id TEXT PK, photo_id, bbox TEXT, landmarks TEXT, px_size REAL,
      det_score REAL, blur REAL, yaw REAL, occlusion REAL, feat_norm REAL,
      tier TEXT,               -- core | partial | unusable
      detect_pass TEXT,        -- full | tile
      embedding BLOB NULL,     -- 128 x float32, L2-normalised; NULL after exclusion
      embedding_model TEXT,    -- e.g. sface_2021dec
      crop_key TEXT NULL,
      person_id NULL, assign_kind TEXT NULL,   -- cluster | attach | lock
      assign_confidence REAL NULL,
      excluded INT DEFAULT 0)
persons(id TEXT PK, name NULL, cover_face_id, cover_locked INT DEFAULT 0,
        hidden INT DEFAULT 0, hidden_by TEXT NULL,  -- photographer | guest_optout
        centroid BLOB, face_count, photo_count, merged_into NULL, created_at)
face_locks(face_id PK, person_id, created_at, actor)
person_cannot_links(person_a, person_b, PRIMARY KEY(person_a, person_b))
exclusions(face_id PK, reason, created_at)          -- optout | not_a_face | stranger
optout_centroids(person_id PK, centroid BLOB)       -- P14, deleted at expiry
suggestions(id PK, kind, photo_id, person_id, guest_session, created_at, status)
photo_persons(photo_id, person_id, face_id, confidence, PRIMARY KEY(photo_id, person_id))
cluster_runs(id PK, started_at, finished_at, algo, params_json, n_faces, n_persons, ms)
meta(key PK, value)   -- schema_version, last_cluster_at, dirty
```

## 4. API (outline; exact shapes in `docs/api.md`, written in step 8)

Photographer (session cookie):
`POST /api/auth/login` · `GET /api/auth/callback` · `POST /api/auth/logout` · `GET/POST /api/galleries` · `GET/PATCH/DELETE /api/galleries/:id` · `POST /api/galleries/:id/photos` · `PATCH/DELETE /api/photos/:id` · `GET /api/galleries/:id/status` · `POST /api/galleries/:id/people/enable` (with attestation) · `POST /api/galleries/:id/people/disable` · `GET /api/galleries/:id/people?include=hidden` · `POST /api/galleries/:id/people/merge|split|move|hide|rename|cover` · `GET /api/galleries/:id/suggestions` · `POST /api/suggestions/:id/accept|dismiss`

Guest (gallery session token, bound to gallery ID):
`GET /g/:slug` (meta) · `POST /g/:slug/session` (PIN → token) · `GET /g/:slug/photos?cursor` · `GET /g/:slug/people` · `GET /g/:slug/people/photos?ids=a,b&only=0|1` · `POST /g/:slug/people/:pid/hide-me` · `POST /g/:slug/photos/:id/not-me` · `GET /i/:signed` (image bytes)

Container (internal only):
`POST /process?faces=1` (body: image bytes) → JSON · `POST /cluster` (body: binary frame) → JSON · `GET /healthz`

## 5. Face pipeline (Container, `face_core`)

1. **Decode** with Pillow, apply EXIF orientation (`ImageOps.exif_transpose`), convert to RGB, then strip EXIF from derivatives.
2. **Derivatives:** thumb 400 px, web 2048 px (P6).
3. **Detection** with `cv2.FaceDetectorYN`, model `face_detection_yunet_2023mar` (OpenCV ≥ 4.8):
   - Pass 1 at 2048 px long edge, score threshold `DET_STORE_MIN` (start 0.6).
   - Adaptive tile pass (P17): if ≥ 8 faces or any face is small, run 1024 px tiles with 20% overlap on the full-resolution image, then NMS at IoU 0.4.
   - Map boxes and landmarks back to original coordinates.
4. **Alignment and embedding:** `cv2.FaceRecognizerSF.alignCrop` (5-point similarity transform to 112×112), then `feature()`. Record the raw L2 norm (`feat_norm`), then L2-normalise.
5. **Quality** (each limit in `config.py`):
   - `px_size` = inter-eye distance in original pixels
   - `det_score`
   - `blur` = variance of the Laplacian on the aligned grey crop
   - `yaw` = estimated from the nose position between the eyes
   - `occlusion` = combined score (P18): landmark geometry residual, detector score and feature norm
   - **tier**: `core` if all limits pass, `partial` if within the loose band, otherwise `unusable`
6. **Display crop:** a 256 px square around the bbox with 35% padding, WebP. Only produced if People is enabled; never the aligned crop.
7. **Output:** JSON per face. Embeddings are base64 float32. Nothing is logged except counts and timings.

## 6. Clustering (Container `/cluster`, pure function `cluster(...)`)

Input: `face_ids`, `photo_ids`, `embeddings (n×128)`, `tiers`, `locks {face→person}`, `cannot_links [(person,person)]`, `excluded`, `params`.

1. Use **core** faces only and drop excluded ones.
2. Build the k-NN graph (k = 30, cosine; exact chunked numpy up to about 200k faces). Keep edges with similarity ≥ `T_EDGE`.
3. **Same-photo rule:** remove edges between faces of the same photo.
4. Run the **algorithm** (`agglomerative` | `chinese_whispers` | `knn_dbscan`; P4) with a fixed seed and sorted node order.
5. **Post-process (shared by all algorithms):**
   1. Split any cluster that contains two faces from one photo; keep the face with the higher mean similarity.
   2. Apply locks: locked faces go to their person; clusters are labelled by the majority of their locked faces; a cluster holding locks of two cannot-linked persons is split by nearest locked centroid.
   3. Drop clusters with fewer than `MIN_PHOTOS` distinct photos (start at 3), unless they contain a lock.
6. **Attach pass** (P18): every `partial` face, and every unassigned core face, attaches to the best person if `sim ≥ T_ATTACH` (stricter than `T_EDGE`) and `sim − second_best ≥ MARGIN`. Similarity is measured to the person's centroid **and** its top-5 exemplars, and the larger value is used. The same-photo rule still applies, so a face cannot attach to a person who already appears in that photo.
7. **Opt-out guard** (P14): any face within `T_OPTOUT` of an `optout_centroid` is excluded.
8. **Stable IDs** (P2): Hungarian matching between new clusters and the previous persons on face overlap. New clusters get new IDs, and persons left with no match are retired.
9. **Cover face:** the core face with the highest combined score (size, sharpness, frontal, low occlusion, closeness to the centroid), taken from a non-hidden photo. `cover_locked` overrides.
10. Return assignments and per-person statistics. GalleryDO writes them in one transaction and rebuilds `photo_persons`.

**Guest visibility:** a person appears in the strip only if `photo_count ≥ MIN_PHOTOS` and the person is not hidden. The People view shows only if the gallery has ≥ 6 such people.

## 7. GalleryDO behaviour

- `addFaces(batch)`: insert, set `dirty`, set the alarm to now + 90 s (debounce; moved forward with each batch, capped at 10 min after the first dirty batch).
- `alarm()`: if dirty and no run in progress, start a run. Load core and partial faces, locks and opt-outs; send them to `/cluster` as a binary frame (header JSON plus a contiguous float32 array); apply the result in one transaction; clear dirty. If faces arrived during the run, schedule another.
- Corrections are DO methods. They write locks or exclusions, update `photo_persons` straight away so the UI reflects them, and mark dirty with a short alarm.
- `deleteAll()` on expiry, on People disable (faces only; photos stay), or on owner request.
- Schema versioning: `meta.schema_version` plus an ordered list of migration functions, run in `blockConcurrencyWhile` in the constructor.

## 8. Privacy requirements

| # | Requirement | Where |
|---|---|---|
| 1 | People is off by default. Turning it on requires the photographer's attestation, stored with timestamp and wording version | step 13 |
| 2 | No face detection runs for galleries with People off | steps 11, 13 |
| 3 | People view only with a valid gallery session (PIN) | step 14 |
| 4 | Guest notice shown before first use of People; acknowledgement recorded per session | step 14 |
| 5 | Hide-me: immediate exclusion, embeddings set to NULL, crops deleted, opt-out centroid kept until expiry | step 14 |
| 6 | Turning People off deletes all face data for the gallery within 24 h (immediately in practice) | step 13 |
| 7 | Expiry deletes R2 objects, D1 rows and the GalleryDO; verified, logged in `audit_log` | step 14 |
| 8 | No embeddings, crops or image bytes in logs; log redaction test | steps 6, 11 |
| 9 | No cross-gallery index; GalleryDO isolation; review checklist | all |
| 10 | EU jurisdiction for R2, D1, DO and Containers | step 15 |
| 11 | No boxes on strangers' faces in the guest UI | step 14 |
| 12 | Published retention policy and guest notice text | Track L |

## 9. Evaluation

See [proposal P15](proposals.md#p15-evaluation-metrics) and step 5 in the build guide.

- `eval/labels.csv`: `face_id, true_person, occlusion, pose, group_bucket`
- `eval/detect_labels.csv`: hand-counted faces for about 20 group photos, for detection recall
- `scripts/eval_clustering.py`: sweeps `algo × T_EDGE × MIN_PHOTOS × T_ATTACH × MARGIN`. It reports pairwise P/R, BCubed P/R/F, guest-visible photo precision, cluster purity and person recall, overall and per tag.
- Operating point: guest-visible photo precision ≥ 0.98, then the highest person recall.
- Results go to `docs/eval-log.md` with date, git SHA, data description and the full table.
- Watch for near-duplicate burst shots, which inflate scores. Report results with and without deduplication (perceptual hash).
