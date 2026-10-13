# Step 03: Quality scoring and tiers (including partly covered faces)

## Context
Phase A, step 3. Not every detected face should drive clustering. Partly covered faces (hands, sunglasses, veils, hair, raised glasses, another person's shoulder), profiles, tiny background faces and blur cause wrong merges. We sort faces into tiers: **core** (builds clusters), **partial** (attach-only), **unusable** (`docs/proposals.md` P18).

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/proposals.md` P18, `docs/spec.md` §5 step 5, `docs/reports/step-02.md`.

## Tasks
1. `face_core/quality.py`, a pure function `assess(face, aligned_crop, feat_norm, gallery_stats) -> Quality`. Fields:
   - `px_size` (inter-eye distance in original pixels)
   - `det_score`
   - `blur` (variance of the Laplacian on the grey aligned crop)
   - `yaw` (estimated degrees, from the nose's horizontal position between the eyes and the eye-distance ratio)
   - `occlusion` in [0, 1], combining:
     (a) landmark geometry residual against a frontal 5-point template after a similarity fit, corrected for yaw
     (b) low detector score
     (c) low `feat_norm` relative to the gallery median
   - `tier` and `reasons` (the list of failed checks)
2. Tier rules in `config.py`, each with a comment:
   - **core:** `px_size ≥ MIN_EYE_PX_CORE`, `det_score ≥ DET_CORE_MIN`, `blur ≥ BLUR_CORE_MIN`, `|yaw| ≤ YAW_CORE_MAX` (≈ 45°), `occlusion ≤ OCC_CORE_MAX`
   - **partial:** fails core, but `px_size ≥ MIN_EYE_PX_PARTIAL`, `det_score ≥ DET_STORE_MIN`, `|yaw| ≤ YAW_PARTIAL_MAX` (≈ 70°), `occlusion ≤ OCC_PARTIAL_MAX`
   - **unusable:** everything else
   - Feature-norm normalisation uses the gallery median, so tiers are computed in a second pass after all faces are embedded. Make it a re-runnable CLI command `tier`.
3. Store every measure plus the tier and reasons on each face.
4. CLI `quality-report --db … --out data/review/quality.html`, as plain static HTML opened from disk:
   - Counts per tier and per reason.
   - A histogram of each measure.
   - A grid of 30 random display crops per tier and per top reason.
5. Tests:
   - Each measure on synthetic inputs: blurred vs sharp crop; rotated landmarks for yaw; a displaced mouth landmark for occlusion.
   - Tier boundaries.
   - The second pass is deterministic.

## Constraints
- Thresholds are starting values to be tuned in step 5; say so in the comments.
- No new dependencies.

## Done when
- `tier` and `quality-report` run on the step-2 database. The HTML shows sensible examples per tier. Tests and ruff pass.

## Stop and report
`docs/reports/step-03.md`: tier counts and %, top reasons, and 3–5 observations from the HTML (for example "sunglasses mostly partial", "veils often core, which may be a risk"). Update `docs/STATUS.md`. Commit. Stop.
