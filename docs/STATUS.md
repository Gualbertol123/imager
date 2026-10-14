# Status

This is the handoff file between Claude Code sessions. Each build step runs in a **fresh session** with no memory of earlier ones, so this file and `docs/reports/` are the only memory. Read it first and update it last.

## Current position

- **Last completed step:** 0 (planning docs; proposals accepted; prepared for step 01)
- **Next step:** 01, [`prompts/01-scaffold.md`](../prompts/01-scaffold.md)
- **Blocked on:** nothing for step 01. D1 (photos) is needed from step 02.
- **Gates passed:** none

## Environment (owner's machine, checked 2026-10-14)

Windows 11 Enterprise. The repo is at `C:\Users\U508169\Downloads\claude\imager\repo`.

| Tool | Status | Needed by | Install |
|---|---|---|---|
| Git | 2.56 ✓ | — | — |
| Python | 3.12.7 ✓ (`py -3.12`) | step 1 | — |
| winget | ✓ | — | — |
| WSL | present (not yet checked whether WSL2 is set up) | step 6 | `wsl --install` if needed |
| uv | 0.13.1 ✓ installed with `pip --user` (winget source is blocked on this machine); `%APPDATA%PythonPython312Scripts` added to the user PATH | step 1 | — |
| Node 22 + pnpm | missing | required from step 8 (step 1 only writes package.json files) | winget is blocked here: use the nodejs.org user installer or ask IT, then `corepack enable` |
| Docker Desktop (WSL2) | missing | step 6 | winget is blocked here, so install from docker.com or ask IT (likely needs admin rights) |
| GitHub CLI | missing | optional (CI in step 15) | `winget install GitHub.cli` |

After installing, **open a new terminal** so PATH updates, then check with `uv --version`.

## Verify-previous-step commands

These are the commands the next session runs first to confirm the repo is healthy. Each step replaces this list with the commands that prove its own "Done when".

```
git status            # must be clean
git log --oneline -3  # last commit is the planning/prep commit
uv --version          # must succeed before step 01 starts
```

## Step log

| Step | Date | Result | Report | Notes for the next session |
|---|---|---|---|---|
| 0 | 2026-10-14 | Planning docs, proposals P1–P18, 15 prompts | — | — |
| 0b | 2026-10-14 | Owner accepted P1–P18; decisions D3, D6–D8, D11–D13 recorded; tools checked; `data/event1/` created locally | — | Step 01: use plain commands + pnpm scripts (D13), no justfile. Don't install Node in step 01 |

## Known issues and loose ends

- SFace training-data provenance is unknown; it is in the legal briefing (Track L).
- Email provider (D5) is open; needed in step 08.

## Deviations from the spec

Record any place where the code intentionally differs from `docs/spec.md` or `docs/proposals.md`, with the reason. Update the spec afterwards if the deviation is permanent.

- (none)
