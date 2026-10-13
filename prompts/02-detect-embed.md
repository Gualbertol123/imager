# Step 02: Detect, align, embed

## Context
Phase A, step 2. This turns a folder of event photos into stored faces with embeddings, locally. Event photos often have **many people per photo** (group shots with small faces) and **partly covered faces**, so detection must handle both (see `docs/proposals.md` P17, P18).

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §5 (face pipeline), `docs/proposals.md` P17 and P18, `docs/reports/step-01.md`.

## Decisions needed
D1 (consented photo folder at `data/event1/`).

## Tasks
1. `face_core/io.py`: `load_image(path) -> (rgb ndarray, meta)`.
   - Use Pillow with `ImageOps.exif_transpose`, and convert to RGB.
   - Supported: JPEG, PNG, WebP. HEIC only if `pillow-heif` is approved (ask first).
   - Skip other files with a warning; never crash on a bad file.
2. `face_core/detect.py` using `cv2.FaceDetectorYN`:
   - Pass 1 at `DETECT_LONG_EDGE` (2048) with score threshold `DET_STORE_MIN` (0.6).
   - **Adaptive tile pass** (P17) when pass 1 finds ≥ `TILE_TRIGGER_FACES` (8) faces, or any face has inter-eye distance < 2 × `MIN_EYE_PX_CORE`. Run it on the full-resolution image: tiles of `TILE_SIZE` (1024) with overlap `TILE_OVERLAP` (0.2).
   - Map boxes and landmarks back to original coordinates and merge both passes with NMS (`NMS_IOU` 0.4), keeping the higher score.
   - Return a `Face` dataclass: bbox, 5 landmarks, score, `detect_pass`, `px_size` (inter-eye distance in original pixels).
3. `face_core/embed.py` using `cv2.FaceRecognizerSF`:
   - `alignCrop` on the **original-resolution** image with original-coordinate landmarks.
   - Call `feature`, record the raw L2 norm as `feat_norm`, then L2-normalise to float32[128].
4. `face_core/store.py`: local SQLite with tables `photos(path, sha256, width, height, status, n_faces, ms)` and `faces(...)`, mirroring the GalleryDO `faces` table in spec §3 (relevant subset). Idempotent by sha256.
5. Crops:
   - Aligned 112×112 crops to `data/crops/aligned/`, for inspection only.
   - Display crops, 256 px square with 35% padding, to `data/crops/display/`.
6. `face_core/cli.py process <folder> --db <path> [--limit N] [--workers N]`:
   - Run on a process pool, defaulting to 4 workers to mimic the Container.
   - Print a summary: photos, faces, a faces-per-photo histogram (0, 1, 2–4, 5–15, 16+), photos with zero faces, faces found **only** by the tile pass, and seconds per photo (mean, p95).
7. All numbers go in `config.py`, each with a comment saying it is a starting value to be tuned in step 5.
8. Tests, using synthetic images generated in the test (**no real photos**):
   - EXIF orientation (a synthetic image with an orientation tag).
   - NMS merge.
   - Tile coordinate mapping (a box found in a tile maps back to the right place).
   - Embed output is unit-length float32.

## Constraints
- No network calls. No photos committed. Never print or log embeddings.
- If OpenCV's API differs from what you expect, check the OpenCV docs and tell me; don't guess.

## Done when
- `uv run python -m face_core.cli process ../../data/event1 --db ../../data/faces.db` runs and prints the summary.
- Crops exist on disk; a re-run skips finished photos; tests and ruff pass.

## Stop and report
`docs/reports/step-02.md`: summary numbers, the 5 photos with the most faces (file names only), how many faces only the tile pass found, timing. Update `docs/STATUS.md`. Commit. Stop.
