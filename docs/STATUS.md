# Status

This is the handoff file between Claude Code sessions. Each build step runs in a **fresh session** with no memory of earlier ones, so this file and `docs/reports/` are the only memory. Read it first and update it last.

## Current position

- **Last completed step:** 0 (planning docs written)
- **Next step:** 01, [`prompts/01-scaffold.md`](../prompts/01-scaffold.md)
- **Blocked on:** owner answers D1–D3 in [`decisions.md`](decisions.md)
- **Gates passed:** none

## Verify-previous-step commands

These are the commands the next session runs first to confirm the repo is healthy. Each step replaces this list with the commands that prove its own "Done when".

```
(none yet; step 01 adds: cd services/face && uv run pytest && uv run ruff check .)
```

## Step log

| Step | Date | Result | Report | Notes for the next session |
|---|---|---|---|---|
| 0 | 2026-10-14 | Planning docs, proposals P1–P18, 15 prompts | — | Start with prompt 01 once D1–D3 are answered |

## Known issues and loose ends

- (none)

## Deviations from the spec

Record any place where the code intentionally differs from `docs/spec.md` or `docs/proposals.md`, with the reason. Update the spec afterwards if the deviation is permanent.

- (none)
