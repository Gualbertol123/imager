# Step 07: Cloudflare Containers spike (Gate M2)

## Context
Phase A, step 7. This is a **throwaway** experiment on real Cloudflare to answer: can Containers run our face service fast enough, cheaply enough, and in the EU? It is the first step that creates cloud resources, so **every resource-creating command needs my explicit OK in this session**.

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/cloudflare-notes.md`, `docs/proposals.md` P9, P10, `docs/reports/step-06.md`.

## Decisions needed
D4 (Workers Paid plan, OK for throwaway resources). Ask me to confirm before the first `wrangler deploy`.

## Tasks
1. Check the current Cloudflare docs for Containers (Worker/DO binding, `wrangler.jsonc` `containers` config, instance types, `constraints.jurisdiction`, cold starts, request timeouts). Update `docs/cloudflare-notes.md` with the URLs.
2. `spikes/container/`: a minimal Worker plus Container class using `@cloudflare/containers` (or the current official package; check), image from `services/face/Dockerfile`, instance type `standard-4`, `constraints: { jurisdiction: "eu" }`.
   - Expose only `/spike/process`, `/spike/cluster` and `/spike/health`, protected by a random bearer token set with `wrangler secret put`.
3. **Test data:** synthetic images or a public dataset with a commercial-friendly licence only. Ask me which; **never upload guest photos**.
4. `spikes/container/load.ts` (Node script) measures:
   - cold start (first request after ≥ 15 min idle, 3 times)
   - warm latency per photo (p50, p95)
   - throughput at concurrency 1, 2, 4, 8 on `standard-4`, and at 4 on `standard-2`
   - `/cluster` with 10k, 50k and 100k random embeddings
   - how placement shows in the response or logs (EU colo)
5. Cost: after the run, ask me to read the Containers usage numbers from the dashboard. Compute cost per 1,000 photos and per 2,000-photo gallery, using the included allowances and overage prices in `cloudflare-notes.md`.
6. **Teardown:** delete the Worker, container application and images (`wrangler delete`, `wrangler containers delete`). Ask me to confirm in the dashboard that nothing is left.

## Constraints
- Nothing from this spike is reused as product code, except the lessons in the report.
- No D1, R2 or Queues in the spike unless measuring something needs them (ask first).

## Done when
- All the measurements exist, the resources are deleted, and `cloudflare-notes.md` is updated.

## Stop and report
`docs/reports/step-07.md` with a **Gate M2** verdict against the targets in `docs/plan.md` (cold start ≤ 20 s, ≥ 1 photo/s on 4 vCPU, EU confirmed, cost per 1,000 photos). Give your recommendation if any target is missed. Update `docs/STATUS.md`. Commit. Stop.
