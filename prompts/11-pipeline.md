# Step 11: Processing pipeline (Queue → Container → R2 + GalleryDO)

## Context
Phase C, step 11. After upload, each photo is processed by the face service Container. It always produces derivatives; it detects and embeds faces **only if the gallery has People enabled**. Face data is stored in the gallery's Durable Object (P1). It runs locally with `wrangler dev` plus Docker.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §2, §3 (GalleryDO part), §5, §7, `docs/proposals.md` P1, P6, P9, P17, P18, `docs/reports/step-06.md`, `docs/reports/step-07.md`, `docs/reports/step-10.md`.

## Tasks
1. **Container binding** in `apps/api/wrangler.jsonc`: a `FaceContainer` class (`@cloudflare/containers`, or the current official approach from step 7), image `../../services/face/Dockerfile`, instance type per the step-7 result. Check how the Container runs under local `wrangler dev` (Docker) and document it in the README.
2. **Queue** `photo-process` (local) with a DLQ `photo-process-dlq`:
   - The upload handler sends `{gid, pid, key, faces: people_enabled}` after the D1 row is written.
   - Consumer settings: `max_batch_size 10`, `max_batch_timeout 5`, `max_retries 5`, and `limits.cpu_ms` raised only if measured necessary.
3. **Consumer** `src/queue/process.ts`. For each message, with concurrency ≤ the Container's `MAX_CONCURRENCY`:
   - Get the R2 object and stream it to the Container `POST /process?faces=…`, with no buffering in the Worker.
   - On 503, retry the message with delay.
   - On success:
     - Write `thumb` and `web` to R2 (`g/<gid>/t/<pid>.webp`, `g/<gid>/w/<pid>.webp`) and crops to `g/<gid>/f/<faceId>.webp`.
     - Update `photos` (`thumb_key`, `web_key`, width, height, `taken_at`, `face_count`, `process_status='done'`).
     - Call `GALLERY.get(idFromName(gid)).addFaces(pid, faces)` over RPC.
   - On permanent failure: set `process_status='error'` with a short reason, and ack.
   - Idempotent: re-processing a photo replaces its faces in the DO (delete by `photo_id`, then insert).
4. **GalleryDO** `src/do/gallery.ts` (SQLite storage):
   - Schema v1 from spec §3, with a migration runner using `meta.schema_version` inside `blockConcurrencyWhile`.
   - RPC methods: `addFaces`, `getStatus`, `deleteFaces(photoId)`, `deleteAllFaceData()`.
   - `addFaces` refuses (and logs a counter, not data) if People is not enabled. The Worker passes the flag, and the DO stores its own copy, set by enable/disable in step 13.
   - Embeddings are stored as BLOB float32[128]. `embedding_model` is set from the Container's `/healthz`.
   - `addFaces` sets `dirty` and the debounce alarm (the clustering itself is step 12; for now, the alarm only logs).
5. **Status:** `GET /api/galleries/:id/status` returns counts by `process_status`, faces stored, and `people_status`. The photographer UI shows a progress bar and polls every 5 s while processing.
6. **Frontend:** use derivative URLs (`thumb`, `web`) when they exist; show a placeholder while pending.
7. **Logging:** a structured line per photo (`gid`, `pid`, ms, `face_count`, status). Test that logs contain no base64 or embeddings.
8. **Tests:**
   - The consumer, with a mocked Container (a fetch mock returning a fixture JSON made from synthetic data).
   - GalleryDO: migrations, `addFaces` idempotency, refusal when People is off, `deleteAllFaceData`.
   - The end-to-end local test with a real Container, behind an env flag.

## Constraints
- **No face detection or storage for galleries with People off** (spec §8 row 2). Test it.
- No cross-gallery access to DOs: the DO id always comes from the gallery id of an already-authorised request or message.

## Done when
- Uploading 100 photos locally leads to all `done`, derivatives in local R2, faces in the DO (when enabled) and zero faces when disabled. Tests pass.

## Stop and report
`docs/reports/step-11.md` with local timing for 100 photos. Update `docs/STATUS.md`. Commit. Stop.
