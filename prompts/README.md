# Build prompts

There are 15 prompts, one per step in [`docs/plan.md`](../docs/plan.md). For each one: open a **fresh** Claude Code session in the repo, switch to **plan mode** (`Shift+Tab`), paste the whole prompt file, review the plan, approve, then verify using [`docs/build-guide.md`](../docs/build-guide.md).

| # | Prompt | Phase | Gate after |
|---|---|---|---|
| 01 | [Scaffold](01-scaffold.md) | A | |
| 02 | [Detect, align, embed](02-detect-embed.md) | A | |
| 03 | [Quality and tiers](03-quality.md) | A | |
| 04 | [Clustering](04-clustering.md) | A | |
| 05 | [Labelling and evaluation](05-eval.md) | A | **M1** models |
| 06 | [Face service + Docker](06-face-service.md) | A | |
| 07 | [Containers spike](07-container-spike.md) | A | **M2** platform |
| 08 | [API skeleton + auth](08-api-skeleton.md) | B | |
| 09 | [Galleries, uploads, guest sessions](09-galleries-uploads.md) | B | |
| 10 | [Web app](10-web-app.md) | B | **M3** gallery |
| 11 | [Processing pipeline](11-pipeline.md) | C | |
| 12 | [Clustering orchestration](12-clustering-orchestration.md) | C | |
| 13 | [Photographer People tools](13-photographer-people-tools.md) | C | |
| 14 | [Guest People view + privacy](14-guest-people-privacy.md) | C | **M4** feature + legal |
| 15 | [Staging, production, beta](15-staging-beta.md) | D | **M5** beta |

**Fresh session per prompt.** Each prompt is written to run in a brand-new session with no memory. Continuity comes from the repo: `CLAUDE.md` (loaded automatically) defines a start and end protocol, `docs/STATUS.md` is the handoff file, and `docs/reports/step-NN.md` holds each step's details. If a step is too big for one session, start a new session with: *"Continue step NN. Read docs/STATUS.md and docs/reports/step-NN.md (partial) first."* The unfinished session should write a partial report before you close it, so ask it to: *"Write a partial report and update STATUS.md, then stop."*

Every prompt follows the same shape: **Context → Read first → Decisions needed → Tasks → Constraints → Done when → Stop and report.** Every step ends by writing `docs/reports/step-NN.md` and stopping.
