# imager

A hosted gallery platform for event photographers (weddings, parties) on Cloudflare. Its headline feature is **People search without selfies**: every face in a gallery is detected and grouped into people, *within that gallery only*, so a guest can tap a face and see every photo of that person.

Status: **planning**. No product code yet.

## Read in this order

| Document | What it is |
|---|---|
| [`docs/STATUS.md`](docs/STATUS.md) | **Handoff file.** Where the build is, what is next; each fresh session reads it first |
| [`CLAUDE.md`](CLAUDE.md) | Standing rules for every Claude Code session: non-goals, hard constraints, conventions |
| [`docs/plan.md`](docs/plan.md) | The structured plan: phases, milestones, go/no-go gates, risks |
| [`docs/proposals.md`](docs/proposals.md) | 18 design proposals, each with a recommendation, justification and alternatives |
| [`docs/prior-art.md`](docs/prior-art.md) | Similar products and open-source projects, and what we borrow from each (with licence notes) |
| [`docs/spec.md`](docs/spec.md) | Technical spec: architecture, data model, face pipeline, clustering, privacy rules |
| [`docs/build-guide.md`](docs/build-guide.md) | Step-by-step guide from an empty machine to a private beta |
| [`prompts/`](prompts/) | 15 ready-to-paste Claude Code prompts, one per build step |
| [`docs/decisions.md`](docs/decisions.md) | Open questions the owner must answer, and the record of the answers |
| [`docs/cloudflare-notes.md`](docs/cloudflare-notes.md) | Platform limits checked against Cloudflare's docs, with links |
| [`docs/licences.md`](docs/licences.md) | Licence register for every model and dependency |

## Stack in one line

Cloudflare Workers (Hono, TypeScript) · R2 · D1 · Queues · Durable Objects with SQLite · Containers (Python, OpenCV YuNet + SFace) · React + Vite.
