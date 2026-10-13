# Step 05: Labelling and evaluation (Gate M1)

## Context
Phase A, step 5, the most important step. It answers: **are YuNet + SFace with our clustering accurate enough on real event photos?** That includes group photos and partly covered faces. We report numbers honestly; we never loosen thresholds to make the table look better.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/proposals.md` P4, P15, P17, P18, `docs/spec.md` §9, `docs/reports/step-02.md` to `step-04.md`.

## Tasks
1. **Labelling helper** `eval/label_server.py`, using only the standard library (`http.server`), bound to `127.0.0.1`, never exposed.
   - Starts from the clusters of the best-looking algorithm in step 4.
   - Shows one person at a time as a grid of display crops. I can:
     - confirm the person
     - give it a label (any short token, **not real names**; e.g. `P001`)
     - move single faces to another label or to "unknown"
     - merge two labels
     - mark a face "not a face"
   - Per face, quick tags: `occlusion` (none / partial / heavy), `pose` (frontal / three-quarter / profile).
   - Per photo: `group_bucket` (1, 2–4, 5–15, 16+), computed automatically from face count; I can correct it.
   - Saves continuously to `eval/labels.csv` (`face_id,true_person,occlusion,pose,group_bucket`). Labels are face IDs plus tokens only; nothing identifying is committed. `face_id` must be stable (the photo's sha256 plus a face index).
   - Target: 300–500 labelled faces, covering ≥ 30 people and ≥ 20 group photos.
2. **Detection recall:** `eval/detect_labels.csv`. A small page shows 20 group photos with detections drawn. I click on missed faces and mark false detections. Report recall and precision per `group_bucket`.
3. `scripts/eval_clustering.py`:
   - **Sweep:** `algo × T_EDGE (0.30–0.60 step 0.025) × MIN_PHOTOS {2,3,4} × T_ATTACH {T_EDGE+0.05, +0.10} × MARGIN {0.03, 0.06}`. The quality-tier thresholds can be swept as a second, smaller sweep.
   - **Metrics** (P15): pairwise P/R, BCubed P/R/F, guest-visible photo precision, cluster purity, person recall (people with ≥ 3 photos).
   - Reported overall **and per tag**: occlusion, pose, group_bucket, and core vs attached-partial faces.
   - **Attach gain:** person recall with vs without the partial tier, at equal precision.
   - **Burst check:** rerun with near-duplicates removed (perceptual hash, Hamming distance ≤ 6) and report both.
   - **Operating point:** guest-visible photo precision ≥ 0.98, then maximise person recall. Write it to `config.py` with a comment citing the eval-log entry.
4. **Speed:** seconds per photo (mean, p95) and peak RSS with 4 workers; `/cluster` time on the real event; projected time for a 2,000-photo and a 20,000-photo gallery.
5. **Results:** the full sweep table, chosen point and failure cases (file names only) go in `docs/eval-log.md`. Failure categories:
   - the same person split by pose, glasses or lighting
   - relatives or look-alikes merged
   - profiles, partly covered faces, sunglasses, veils, hats
   - blur and low light
   - children (if present)
   - tiny faces in group photos

## Constraints
- Do not guess identities yourself; I label. Never commit photos or crops.
- If the results look suspiciously perfect, look for burst duplicates and say so.

## Done when
- `labels.csv` has ≥ 300 faces; the sweep runs; `eval-log.md` and `config.py` are updated.

## Stop and report
Write `docs/reports/step-05.md` in this format, update `docs/STATUS.md`, commit, summarise the report to me, then stop:
```
# Step 05 report (Gate M1)
## Data used
## Results at the chosen point (overall and per tag)
## Detection recall on group photos
## Partial tier: attach gain
## Algorithm comparison
## CPU speed and memory; projections
## Failure cases (file names)
## Verdict: good enough / borderline / not good enough, and why
## Recommended next step (see plan.md M1 options)
## Assumptions and doubts
```
