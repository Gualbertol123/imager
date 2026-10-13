# Build guide

This takes you step by step from an empty machine to a private beta. Each step matches one prompt in [`prompts/`](../prompts/). For each step, the guide says what you (the owner) do, what Claude Code does, how to check the result, and when to stop.

## Part 0. One-time setup

### 0.1 Install tools (Windows 11)

| Tool | Version | Install | Check |
|---|---|---|---|
| Git | latest | `winget install Git.Git` | `git --version` |
| GitHub CLI | latest | `winget install GitHub.cli`, then `gh auth login` | `gh auth status` |
| Python | 3.12 | `winget install Python.Python.3.12` | `py -3.12 --version` |
| uv (Python env and package manager) | latest | `winget install astral-sh.uv` | `uv --version` |
| Node.js | 22 LTS | `winget install OpenJS.NodeJS.LTS` | `node -v` |
| pnpm | 9+ | `corepack enable` then `corepack prepare pnpm@latest --activate` | `pnpm -v` |
| Docker Desktop | latest, **WSL2 backend** | `winget install Docker.DockerDesktop`; enable WSL2 in settings | `docker run hello-world` |
| Claude Code | latest | already installed | `claude --version` |

Wrangler is installed per project by pnpm, so you don't need a global install.

### 0.2 Repository

```powershell
cd C:\Users\<you>\Downloads\claude\imager\repo
git pull
```

### 0.3 Before step 1, answer in `docs/decisions.md`

D1 (consented photos), D2 (tools), D3 (accept proposals).

**Photos for evaluation.** Pick one real event you have consent to use: 500–2,000 photos, with portraits, couples, **group photos with many people**, and **partly covered faces** (sunglasses, veils, hands, raised glasses, profiles). Copy them to `repo\data\event1\`. This folder is git-ignored. Do not use photos of children unless the parents explicitly agreed; if there are children, tell Claude so it reports them as a separate category.

### 0.4 Start Track L (legal), in parallel

Book a privacy lawyer and send them [`proposals.md` P11 and P14](proposals.md), [`spec.md` §8](spec.md#8-privacy-requirements), and the open questions from [`plan.md` Track L](plan.md#track-l-legal-owner-in-parallel-starting-now).

## Part 1. How to run each step with Claude Code

Repeat this loop for every step:

1. **Start fresh.** Open a new Claude Code session in the repo folder. CLAUDE.md loads automatically, and its session protocol makes Claude read `docs/STATUS.md` (the handoff file), check the previous step still passes, and only then plan.
2. **Plan mode.** Press `Shift+Tab` until "plan mode" shows, then paste the prompt from `prompts/NN-*.md`.
3. **Review the plan.** Check that it matches the step's "Done when" list and doesn't touch later steps. Ask for changes until you agree, then approve.
4. **Let it build.** Answer the questions it asks (dependencies, downloads, cloud resources). Never approve a cloud action you didn't expect.
5. **Verify yourself** with the "You check" list for the step below.
6. **Report.** Claude writes `docs/reports/step-NN.md` and updates `docs/STATUS.md`. Read both; the next session knows only what is written there.
7. **Commit and push.** Ask Claude to commit (small commits) and push, or do it yourself.
8. **Stop at gates.** After steps 5, 7, 10, 14 and 15, decide go/no-go before continuing.

Tips:
- If a session gets long or confused, stop, ask for the report so far, and start a new session with the same prompt plus "continue from docs/reports/step-NN.md".
- `/code-review` after each step catches bugs; `/security-review` after steps 9, 11 and 14.
- Keep `docs/decisions.md` up to date. Claude reads it.

## Part 2. The steps

### Phase A: De-risk

#### Step 1. Scaffold the monorepo · [prompt 01](../prompts/01-scaffold.md)
- **Claude builds:** the folder layout; a `pnpm` workspace (empty apps); `services/face` with uv, `pyproject.toml`, ruff and pytest; `scripts/download_models.py` (pinned URLs and SHA-256 checksums for YuNet and SFace); `eval/` skeleton; `docs/reports/`.
- **You check:** `cd services/face; uv run pytest` passes; `uv run ruff check .` is clean; `uv run python scripts/download_models.py` downloads both models and verifies their checksums; `git status` shows no `.onnx` files.

#### Step 2. Detect, align, embed · [prompt 02](../prompts/02-detect-embed.md)
- **Claude builds:** `face_core.detect` (YuNet; full pass plus adaptive tile pass with NMS, P17), `face_core.embed` (SFace align and feature, raw norm, L2-normalise), `face_core.io` (EXIF-safe loading), local SQLite store, and the CLI `process` command.
- **You run:** `uv run python -m face_core.cli process ..\..\data\event1 --db ..\..\data\faces.db`
- **You check:**
  - The summary shows photos, faces, faces per photo, photos with zero faces, and **faces found only by the tile pass**.
  - Open `data/crops/` and spot-check that crops are upright, including photos shot in portrait orientation.
  - Open 3 big group photos and count faces by hand against the reported number.
  - A re-run skips finished photos.

#### Step 3. Quality and tiers · [prompt 03](../prompts/03-quality.md)
- **Claude builds:** quality measures (size, detector score, blur, yaw, occlusion score from landmark geometry + detector score + feature norm), the tiers core / partial / unusable (P18), a CLI `quality-report` with an HTML page of example crops per tier and reason.
- **You check:** open the report. Are sunglasses, veils, hands-over-face and profiles mostly in `partial`? Are tiny background faces `unusable`? Note any face you think is in the wrong tier, because this feeds step 5.

#### Step 4. Clustering · [prompt 04](../prompts/04-clustering.md)
- **Claude builds:** `face_core.cluster` (pure function; three algorithms; same-photo rule; locks; attach pass for partial faces; cover face), CLI `cluster --algo …`, and `review` contact sheets (HTML, one section per person, with partial faces marked).
- **You check:** open `data/review/index.html`. Look for (a) two different people in one person (bad), (b) one person split in several (acceptable), and (c) whether partial faces were attached to the right person. Tests pass for the same-photo rule, determinism and locks.

#### Step 5. Labelling and evaluation → **Gate M1** · [prompt 05](../prompts/05-eval.md)
- **Claude builds:**
  - A local labelling page (`eval/label_server.py`, localhost only). It starts from the clusters: you confirm, rename, move faces and tag occlusion, pose and group size.
  - `eval/labels.csv`, and `scripts/eval_clustering.py` sweeping algorithm × thresholds.
  - The metrics from P15 per tag, detection recall on 20 group photos, and the burst-duplicate check.
  - CPU time per photo and peak memory, measured with **4 threads** to mimic the Container.
- **You do:** label 300–500 faces (about 2–3 h). Count the faces in 20 group photos by hand for detection recall.
- **You check:** `docs/eval-log.md` and `docs/reports/step-05.md` (session-1 report format).
- **Gate M1:** is guest-visible precision ≥ 0.98 with person recall ≥ 0.70? Write the decision in `decisions.md` (D11 if borderline).

#### Step 6. Face service and Docker image · [prompt 06](../prompts/06-face-service.md)
- **Claude builds:** FastAPI app (`/process`, `/cluster`, `/healthz`), derivatives (thumb, web, display crops, P6), the binary frame format for `/cluster`, a log-redaction test, and a slim Dockerfile with the models baked in.
- **You run:** `docker build -t imager-face services/face`, then `docker run -p 8080:8080 imager-face`, then `uv run python scripts/smoke_service.py data/event1 --n 20`.
- **You check:** the smoke script shows the same faces as step 2 on the same photos; image size ≤ 1 GB; memory during the 20-photo run ≤ 2 GB.

#### Step 7. Cloudflare Containers spike → **Gate M2** · [prompt 07](../prompts/07-container-spike.md)
- **You do first:** create or upgrade the Cloudflare account to Workers Paid (D4); `pnpm dlx wrangler login`.
- **Claude builds:** `spikes/container/`, a throwaway Worker plus Container (image from step 6) with `jurisdiction: "eu"`, and a load script using **synthetic or public-dataset images only**, never guest photos.
- **Measures:** cold start, warm latency per photo, throughput at 1/2/4 vCPU, `/cluster` time for 10k/50k/100k random embeddings, cost per 1,000 photos from the dashboard, EU placement confirmed.
- **Then:** `wrangler delete` the spike, and confirm in the dashboard that nothing is left.
- **Gate M2:** cold start ≤ 20 s, ≥ 1 photo/s on 4 vCPU, EU confirmed, cost acceptable.

### Phase B: Gallery product

#### Step 8. API skeleton · [prompt 08](../prompts/08-api-skeleton.md)
- **You decide:** D5 (email provider).
- **Claude builds:** `apps/api` (Hono, wrangler.jsonc, D1 migrations from spec §3, data-access layer, error handling, request IDs), magic-link auth with the email provider behind an interface (console sender in dev), session middleware, rate limiting, `packages/shared` zod schemas, `docs/api.md`, and vitest with the Workers pool.
- **You check:** `pnpm -F api dev`, then log in using the link printed in the console; `pnpm -F api test` is green.

#### Step 9. Galleries, uploads, guest sessions · [prompt 09](../prompts/09-galleries-uploads.md)
- **Claude builds:**
  - Gallery CRUD and upload streaming to R2, with dedupe by content hash and a per-owner quota.
  - The photo list with cursor pagination, PIN set/verify (PBKDF2 via WebCrypto, attempt limit), and guest tokens bound to the gallery.
  - Signed image URLs (`/i/:token`, HMAC, 10 min).
  - The cross-gallery access test suite.
  - Until step 11, a simple Worker placeholder that serves originals for thumbnails.
- **You check:** with `curl` or the tests, gallery A's guest token gets a 403 or 404 on every gallery-B route.

#### Step 10. Web app → **Gate M3** · [prompt 10](../prompts/10-web-app.md)
- **You decide:** D6 (CSS).
- **Claude builds:** `apps/web` (Vite, React, Router, TanStack Query).
  - **Photographer side:** login, gallery list, create, settings (PIN, expiry), Uppy uploader, photo grid with hide and delete.
  - **Guest side:** PIN page, justified grid, PhotoSwipe lightbox, download.
  - Served through Workers static assets; Playwright smoke test.
- **Gate M3:** upload 500 photos, share, and browse as a guest on your phone (`wrangler dev --ip 0.0.0.0` on the LAN).

### Phase C: People

#### Step 11. Processing pipeline · [prompt 11](../prompts/11-pipeline.md)
- **Claude builds:**
  - The queue producer on upload and the consumer (batch 10, DLQ), with the Container binding (local dev via Docker).
  - Derivatives written to R2; `photos.process_status`.
  - GalleryDO with SQLite schema v1 and `addFaces`, with faces stored **only if `people_enabled`**.
  - Status endpoint and progress UI.
- **You check:** upload 100 photos and watch the status reach 100%. Thumbnails come from the derivatives. With People off, GalleryDO has zero faces.

#### Step 12. Clustering orchestration · [prompt 12](../prompts/12-clustering-orchestration.md)
- **Claude builds:** the GalleryDO alarm with debounce, the binary frame to `/cluster`, a transactional apply, stable IDs (Hungarian), `photo_persons`, cover faces, `cluster_runs`, and a "recluster now" admin endpoint.
- **You check:** upload the step-5 event. The persons match the step-4 contact sheets within reason. Uploading 50 more photos keeps person IDs stable.

#### Step 13. Photographer People tools · [prompt 13](../prompts/13-photographer-people-tools.md)
- **Claude builds:**
  - Enable/disable People with the attestation dialog (stored with wording version); disable deletes face data.
  - People management page: grid of persons; merge by drag or select; split by selecting faces; move a face; hide; rename (private); set cover.
  - "Unassigned faces" review and the suggestions queue.
  - Each action becomes a lock, exclusion or cannot-link (P3).
- **You check:** make 10 corrections, trigger a recluster, and confirm every correction survived.

#### Step 14. Guest People view and privacy → **Gate M4** · [prompt 14](../prompts/14-guest-people-privacy.md)
- **Claude builds:**
  - Guest notice before first use; the face strip (≥ 6 people, ordered by photo count).
  - Single and multi-person AND filter, plus "only these people".
  - People chips in the lightbox (no boxes on faces).
  - Hide-me (P14) and "not me" suggestions (P13).
  - The expiry cron with verified deletion and `audit_log`.
  - Tests for every row of spec §8.
- **Gate M4:** precision on the deployed pipeline matches M1 ±1 point; every privacy row is tested; **L1 legal sign-off received**.

### Phase D: Beta

#### Step 15. Staging, production, beta → **Gate M5** · [prompt 15](../prompts/15-staging-beta.md)
- **You decide:** D8 (EU) and D10 (domain); explicit go-ahead to create resources.
- **Claude builds:**
  - `wrangler.jsonc` environments (staging, production) with EU jurisdiction resources.
  - Secrets set by you through `wrangler secret put`.
  - Custom domain; observability (Workers Logs, a tail of errors, an alert on DLQ depth).
  - A cost dashboard note, backup notes (D1 Time Travel), runbooks (deletion request, incident), and the beta checklist.
- **Gate M5:** 2–3 photographers, real galleries, measured against plan §1.

## Part 3. Checklists

**Before any commit:** tests green · ruff, tsc and eslint clean · no photos, models or secrets in `git status` · report updated.

**Before any cloud action:** owner said yes in this session · resources named `imager-<env>-<thing>` · EU jurisdiction set · cost estimate written down.

**Privacy review (steps 11–14):** no embeddings or bytes in logs · every face query goes through GalleryDO · guest tokens checked against the gallery ID · deletion paths tested.
