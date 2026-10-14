# Step 01: Scaffold the monorepo

## Context
We are building the event gallery platform described in `CLAUDE.md`. This is the first step of Phase A (de-risk). Its only goal is a clean, tested skeleton so steps 2–5 can focus on the face core.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/plan.md` (Phase A), `docs/spec.md` §5, `docs/licences.md`, `docs/decisions.md`.

## Decisions needed
D2 (at least Python 3.12 and `uv` installed), D3 (accepted) and D13 (plain commands, no justfile) in `docs/decisions.md`. D1 (photos) is not needed until step 02. If `uv --version` fails, stop and tell me how to install it (see `docs/STATUS.md`).

## Tasks
1. Create the layout from CLAUDE.md:
   - `apps/api/`, `apps/web/` and `packages/shared/` each get only a README line ("built in step 8/10").
   - Root `package.json` and `pnpm-workspace.yaml` (pnpm workspaces, no dependencies yet). Write the files only; Node and pnpm may not be installed yet, and nothing needs running until step 08.
2. `services/face/`:
   - A `uv` project with Python 3.12 and a `pyproject.toml`. Runtime dependencies: opencv-python-headless, numpy, scipy, scikit-learn, networkx, pillow. Dev dependencies: pytest, ruff.
   - Package `face_core/` with typed stub modules `io.py`, `detect.py`, `embed.py`, `quality.py`, `cluster.py`, `store.py`, `config.py`, `cli.py`. Plus `tests/` with one smoke test.
   - Ruff config (line length 100; rules E, F, I, B, UP, N).
3. `services/face/scripts/download_models.py`:
   - Downloads YuNet `face_detection_yunet_2023mar.onnx` and SFace `face_recognition_sface_2021dec.onnx` into `services/face/models/`.
   - **Look up the current official URLs** in the OpenCV Zoo GitHub repo or Hugging Face (`opencv/face_detection_yunet`, `opencv/face_recognition_sface`). Do not guess them.
   - Pin each URL and SHA-256 checksum, verify after download, and be idempotent.
   - **Ask me before running it.**
4. Confirm the licences and record any training-data statements in `docs/licences.md`.
5. Create `eval/README.md` describing the planned files (`labels.csv`, `detect_labels.csv`, `photos/` untracked).
6. Check that `.gitignore` covers `data/`, `eval/photos/`, models, `.venv`, `node_modules`, `.dev.vars`.
7. Commands (D13): no task-runner tool. Add a "Commands" section to the root README listing every command (`uv run …` for Python, `pnpm …` for JS later). Commands must work in both PowerShell and Git Bash.

## Constraints
- Add no dependency beyond those listed without asking.
- Write no algorithm code yet, only stubs with type signatures and docstrings.
- Never commit model files.

## Done when
- `cd services/face && uv run pytest` passes.
- `uv run ruff check . && uv run ruff format --check .` are clean.
- The download script fetches and verifies both models.
- `git status` is clean of models and data.

## Stop and report
Write `docs/reports/step-01.md` (what exists, commands, anything unclear about licences). Update `docs/STATUS.md`. Commit in small commits. Stop.
