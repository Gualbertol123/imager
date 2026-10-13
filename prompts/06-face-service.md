# Step 06: Face service and Docker image

## Context
Phase A, step 6. This wraps `face_core` in a small HTTP service that will run as a Cloudflare Container. It also produces image derivatives (P6), so each photo is decoded once. It runs locally in Docker only; nothing touches Cloudflare.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §2, §4 (Container endpoints), §5, §6, `docs/proposals.md` P6, P9, `docs/reports/step-05.md`.

## Decisions needed
Gate M1 passed (see `docs/decisions.md`). If it has not passed, stop and tell me.

## Tasks
1. Ask me before adding `fastapi` and `uvicorn`, then add them and record them in `docs/licences.md`.
2. `services/face/app/main.py`:
   - `POST /process?faces=0|1&tile=auto`
     - Body: raw image bytes, streamed, with a size limit of 60 MB.
     - Steps: decode with EXIF orientation; make derivatives `thumb` (400 px WebP q80) and `web` (2048 px WebP q85), both EXIF-stripped. If `faces=1`: detect, embed, quality pre-measures and display crops (256 px WebP).
     - Response JSON:
       ```
       {width, height, taken_at?, sha256,
        derivatives: {thumb: b64, web: b64},
        faces: [{bbox, landmarks, det_score, px_size, blur, yaw, occ_geom, feat_norm,
                 detect_pass, embedding: b64(float32[128]), crop: b64}],
        timings_ms}
       ```
     - **Tiering needs gallery-level statistics** (median feature norm), so `/process` returns the raw measures and the tier is computed in `/cluster`. Refactor `quality.py` so the per-face part and the gallery part are separate functions. Keep the step-3 tests green.
   - `POST /cluster`
     - Body: a binary frame `[u32 header_len][header JSON][float32 n×128]`. The header holds face ids, photo ids, the raw quality measures, locks, cannot-links, excluded faces, opt-out centroids (b64), previous assignment and params.
     - It computes tiers, then calls `face_core.cluster()`, and returns assignments, persons and diagnostics as JSON.
   - `GET /healthz`: model loaded, version, git SHA.
   - Models load once at start-up. Set OpenCV threads from an environment variable (default = CPU count).
   - Requests are limited to `MAX_CONCURRENCY` (default = vCPUs) with a semaphore; return 503 when overloaded so the caller retries.
3. **Logging:** JSON lines with request id, endpoint, ms, face count and bytes in. **Never** log image bytes, embeddings or crops. Add a test that captures logs while processing a synthetic image and asserts that no base64 or float arrays appear.
4. `services/face/Dockerfile`:
   - Multi-stage, `python:3.12-slim`, models copied in at build time (run the download script in the build stage, verify checksums), non-root user, port 8080, `HEALTHCHECK`.
   - Target image ≤ 1 GB. Write `.dockerignore`.
5. `scripts/smoke_service.py <folder> --n 20 --url http://localhost:8080`:
   - Posts photos, then calls `/cluster` on the results.
   - Compares face counts with the step-2 database for the same photos.
   - Prints latency p50 and p95, and peak container memory (from `docker stats`).
6. Tests: FastAPI `TestClient` for each endpoint on synthetic images, a binary-frame round trip, overload returning 503, and the log redaction test.

## Constraints
- No Cloudflare code here. The service must not need any secrets or R2 access (P9).
- The same `face_core` package is used, not a copy.

## Done when
- `docker build` succeeds; `docker run` plus the smoke script work; tests pass; the image size and memory are reported.

## Stop and report
`docs/reports/step-06.md` with image size, latency, memory, and any differences from step 2. Update `docs/STATUS.md`. Commit. Stop.
