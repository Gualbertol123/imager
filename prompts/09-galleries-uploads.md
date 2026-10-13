# Step 09: Galleries, uploads, guest sessions, image serving

## Context
Phase B, step 9. These are the core gallery features with strict access control. The most important property is that **a guest of gallery A can never read anything of gallery B**. All of it is local.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §3, §4, `docs/proposals.md` P8, `docs/api.md`, `docs/reports/step-08.md`.

## Tasks
1. **Gallery CRUD** for the owner: create (name; slug = random 10 characters, URL-safe, not guessable), list, get, update (name, expiry, PIN set or clear), delete (marks the gallery `expiring`; real deletion is step 14).
2. **Upload** `POST /api/galleries/:id/photos`:
   - Owner only. Accept `image/jpeg|png|webp` (HEIC later); maximum 60 MB per photo.
   - Stream the body to `BUCKET.put("g/<gid>/o/<pid>.<ext>")` **without buffering**, computing SHA-256 while streaming (use a `TransformStream` with `crypto.subtle.digest`, or `crypto.DigestStream` if available on Workers; check).
   - Dedupe by `(gallery_id, content_hash)`: if it already exists, delete the new object and return the existing photo.
   - Write the `photos` row with `process_status='pending'`. Read width and height from the image header (a small JPEG/PNG/WebP header parser written by us, with tests).
   - Enforce the owner's storage quota from config.
3. **Photo management:** list (cursor pagination by `created_at, id`), hide or unhide, delete (R2 object and row).
4. **Guest access:**
   - `GET /g/:slug` returns public metadata only: name, whether a PIN is required, cover.
   - `POST /g/:slug/session {pin?}` checks the PIN and returns a guest token.
     - PIN hash: PBKDF2-SHA256 through WebCrypto, 100k iterations, random salt.
     - Limit attempts to 5 per 15 min per (gallery, IP) and 50 per (gallery, day). After that, return 429.
     - The token is an HMAC-signed `{gid, exp(12 h), sid}` (secret `GUEST_TOKEN_SECRET`), sent as a header or cookie scoped to `/g/:slug`.
   - `GET /g/:slug/photos?cursor` lists photos for the guest, never hidden ones.
5. **Image serving:**
   - The list returns signed image URLs `/i/:token`, where the token is an HMAC over `{key, exp(10 min)}`.
   - `GET /i/:token` verifies the token and streams from R2 with cache headers (`private, max-age=600`) and `Content-Disposition` for downloads.
   - Until derivatives exist (step 11), `thumb` and `web` fall back to the original.
6. **Cross-gallery test suite** `test/access.spec.ts`:
   - Make two galleries with two owners.
   - Assert that every owner route rejects the other owner, every guest route rejects a token for the other gallery, a tampered token fails, an expired token fails, image tokens cannot be reused for other keys, and hidden photos are never listed to guests.
7. Update `docs/api.md` and the shared schemas.

## Constraints
- All SQL goes through `src/db/`. No R2 key built from user input except IDs we generated.
- Never return R2 keys or internal IDs that are not needed.

## Done when
- The tests pass, including the access suite. I can upload with `curl` and fetch a photo through `/i/`.

## Stop and report
`docs/reports/step-09.md`. Update `docs/STATUS.md`. Commit. Stop.
