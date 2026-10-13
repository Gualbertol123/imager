# Step 08: API skeleton and photographer auth

## Context
Phase B, step 8. This is the start of the gallery product: a Hono API on Workers with D1, magic-link auth and a test harness. Everything is local (`wrangler dev`); no cloud resources.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/spec.md` §2, §3 (D1 part), §4, `docs/proposals.md` P1, P7, P12, `docs/cloudflare-notes.md`.

## Decisions needed
D5 (email provider). Ask before adding each npm dependency.

## Tasks
1. `apps/api`:
   - `wrangler.jsonc` with bindings D1 `DB`, R2 `BUCKET`, Durable Object `GALLERY` (class stub only), and `assets` pointing at `../web/dist` (it may be empty for now). Current `compatibility_date`.
   - Use local-only resources with `wrangler dev`; **do not create remote resources**.
2. **D1 migrations** in `apps/api/migrations/0001_init.sql`: the D1 tables and indexes from spec §3. Remember D1's 100 bound-parameter limit when writing batch helpers.
3. **Data-access layer** `src/db/`: typed functions only (no SQL elsewhere). Every function that touches gallery data takes `galleryId` as its first argument and includes it in the `WHERE` clause. Add an eslint rule or grep test that fails if `env.DB.prepare` is used outside `src/db/`.
4. **App structure:** Hono app with request id, JSON error format `{error: {code, message}}`, zod validation from `packages/shared`, CORS off (same origin) and security headers.
5. **Auth (P7):**
   - `POST /api/auth/login {email}` creates a single-use token (32 random bytes, stored as SHA-256, 15 min) and sends a link via `EmailSender`.
   - `EmailSender` is an interface: a `ConsoleEmailSender` for dev, plus the chosen provider's sender using `fetch` with an API key from `wrangler secret`.
   - `GET /api/auth/callback?token=` creates the session. The cookie is `HttpOnly; Secure; SameSite=Lax; Path=/`, it holds a random id, the D1 row stores its hash, and it lasts 30 days.
   - `POST /api/auth/logout`.
   - Middleware `requireOwner`.
   - Rate limit login by IP and by email: Workers Rate Limiting binding if available locally, otherwise a D1-based fallback. Check which in the docs.
6. `packages/shared`: zod schemas and types for every request and response so far.
7. `docs/api.md`: the endpoint list with request and response shapes (start with auth and health).
8. **Tests** (vitest + `@cloudflare/vitest-pool-workers`, migrations applied per test):
   - The login flow, token reuse rejected, token expiry, session required, logout.
   - The rate limit.
   - The data-access lint test.

## Constraints
- TypeScript strict. No ORM unless I approve one; plain SQL in `src/db/`.
- No secrets in the repo; `.dev.vars` is git-ignored, and `.dev.vars.example` is committed.

## Done when
- `pnpm -F api dev` runs; I can log in using the link printed in the console; `pnpm -F api test` and `pnpm -F api typecheck` are green.

## Stop and report
`docs/reports/step-08.md`. Update `docs/STATUS.md`. Commit. Stop.
