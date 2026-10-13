# Step 10: Web app (Gate M3)

## Context
Phase B, step 10. This is the frontend for photographers and guests, without People yet. After this step, the product is a usable gallery on its own.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/proposals.md` P8, P12, `docs/api.md`, `docs/reports/step-09.md`.

## Decisions needed
D6 (Tailwind or CSS modules). Ask before adding each dependency, and record them in `docs/licences.md`.

## Tasks
1. `apps/web`: Vite + React + TypeScript strict + React Router + TanStack Query, using the API types from `packages/shared`. The build goes to `apps/web/dist`, which the API Worker serves as static assets with SPA fallback (configure `assets.not_found_handling`).
2. **Photographer pages:**
   - login (enter email → "check your inbox")
   - galleries list
   - create gallery
   - gallery page with tabs: Photos (grid, multi-select, hide, delete), Upload, Settings (name, expiry date, PIN set or clear, share link with copy button, QR code optional)
3. **Uploader** (Uppy Dashboard + XHR Upload):
   - one request per file, concurrency 5, retry with backoff, type and size checks before upload
   - a progress summary, and "you can close this page once uploads finish"
   - folder drop support
4. **Guest pages:**
   - `/g/:slug`: gallery cover with a PIN form if needed, then the gallery
   - justified grid (react-photo-album) with infinite scroll on the cursor API, and lazy-loaded thumbs
   - PhotoSwipe lightbox using the `web` size, with a download button (original)
   - mobile first
5. **Accessibility:** keyboard navigation in the grid and lightbox, alt text (the photo index), focus states. Dark and light theme.
6. **Errors:** expired gallery page, wrong PIN, rate limited, network retry.
7. **Playwright smoke test** (local): log in using the console link, create a gallery, upload 5 generated images, set a PIN, open the guest link in a new context, enter the PIN, open the lightbox and download.

## Constraints
- No People UI yet. Keep components small; no state library beyond TanStack Query.
- Same origin only; no CORS.

## Done when
- `pnpm -F web build` then `pnpm -F api dev` serves the app. The Playwright smoke test passes. I can browse from my phone on the LAN.

## Stop and report
`docs/reports/step-10.md` with screenshots saved under `docs/reports/img/` (UI only, no real photos; use generated images). **Gate M3:** ask me to try it with 500 of my photos and confirm. Update `docs/STATUS.md`. Commit. Stop.
