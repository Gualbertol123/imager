# CLAUDE.md

Read this at the start of every session. It holds the **rules**. The design lives in `docs/spec.md`, the order of work in `docs/plan.md`, and decisions in `docs/proposals.md` and `docs/decisions.md`. If a task conflicts with any of them, stop and ask.

## What we are building

A hosted gallery platform for event photographers on Cloudflare. Photographers upload photos and share a gallery link (optionally PIN-protected) with guests. The headline feature is a **People view**: faces are detected and grouped into people **within one gallery**, with no selfie and no names, so guests tap a face to see every photo of that person. Photographers can merge, split, move, hide and rename people.

## Non-goals (never build)

- No face index or comparison across galleries. Every face operation is scoped to one gallery.
- No real-world identification, name lookup or social-media matching.
- No inference of age, gender, ethnicity or emotion.
- No training or fine-tuning on guest faces.
- No payments, print store or marketplace.

## Principles

1. **Precision over recall.** A wrong face in someone's photos is a privacy failure; a split person costs one merge.
2. **Privacy is built with each feature, not added at the end.**
3. **Corrections are permanent.** Photographer corrections are stored as locks and survive every recluster.
4. **Measure before tuning.** No change to thresholds, models or clustering without running `eval/` and logging it in `docs/eval-log.md`.
5. **Simple first.** Use the fewest Cloudflare products that work; add one only for a measured need.

## Hard constraints

- **Models:** YuNet (detection) and SFace (embedding) from the OpenCV Zoo. **Never use InsightFace pretrained weights** (`buffalo_l`, `antelopev2`, anything the `insightface` package downloads); they are non-commercial.
- Before adding any model or dependency, check that its licence allows commercial use, and for models also the **training data** provenance. Record it in `docs/licences.md`.
- **No third-party face APIs.** Images, crops and embeddings never leave our machine (local dev) or our Cloudflare account (production).
- **Face processing only runs on galleries where the photographer has enabled People view** and attested consent. No faces are detected for other galleries.
- Per-gallery face data lives in that gallery's Durable Object storage. Nothing may read faces from two galleries at once.
- Never log embeddings, crops or image bytes. Never commit photos (`data/`, `eval/photos/` are git-ignored). Never commit photos of minors or real guests.
- Secrets only through `wrangler secret` / `.dev.vars` (git-ignored).
- **No cloud resources, deploys or spending without the owner's explicit go-ahead in the current session.**

## Repository layout

```
apps/api/          Worker (Hono API, queue consumer, GalleryDO, cron), wrangler.jsonc
apps/web/          React + Vite frontend (photographer admin, guest gallery, People view)
services/face/     Python: face_core package, FastAPI app, Dockerfile (the Container image)
packages/shared/   Shared TypeScript types and zod schemas
eval/              labels and scripts (photos untracked)
docs/              plan, spec, proposals, decisions, notes, eval log, session reports
prompts/           the 15 build prompts
```

## Conventions

- TypeScript strict; Python 3.12 with type hints; `ruff` and `ruff format`; `pytest`; `vitest` with `@cloudflare/vitest-pool-workers`.
- D1 schema changes only through migrations in `apps/api/migrations/`. Durable Object schema changes only through the versioned migration function in `GalleryDO`.
- Tunable values (thresholds, k, minimum cluster size, quality limits) live in one config module per language, each with a comment citing the eval run that set it. No magic numbers.
- Every algorithmic function has a unit test. Clustering tests must cover the same-photo rule, determinism and locks. Worker tests must cover cross-gallery access (a guest token for gallery A must never read gallery B).
- Small commits, one logical step each, conventional-commit style (`feat:`, `fix:`, `docs:` …).

## Session protocol (every session is fresh)

The owner starts a **new Claude Code session for every build prompt**. You have no memory of earlier sessions; the repo is the only memory.

**At the start:**
1. Read `docs/STATUS.md`, then the previous step's `docs/reports/step-NN.md`, then the files the prompt lists.
2. Run `git status` and `git log --oneline -10`. If the working tree is dirty or the last step is not committed, stop and ask.
3. Run the "verify-previous-step commands" from `docs/STATUS.md`. If they fail, report and fix that first (ask before a large fix).
4. Check that the decisions the prompt needs are answered in `docs/decisions.md`; if not, ask.
5. Then plan.

**At the end:**
1. Write `docs/reports/step-NN.md`: what was built, how it was verified (commands and results), open issues, deviations.
2. Update `docs/STATUS.md`: last completed step, next step, verify commands for this step, step-log row, known issues, deviations.
3. Commit, then stop. Do not start the next step.

Never rely on anything "said earlier". If information matters for later steps, write it into `docs/`.

## How to work with the owner

- Start each step in plan mode and show the plan before editing.
- Ask before: downloading a model, adding a dependency, choosing between Cloudflare products, creating cloud resources, or anything that costs money.
- Report evaluation numbers honestly; never loosen a threshold to make a table look better.
- Flag anything that touches privacy or legal risk.
