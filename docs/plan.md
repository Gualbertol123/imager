# Plan

This is the structured plan: what we build, in what order, and the gates that decide whether we go on. Design detail is in [`spec.md`](spec.md). The reasons behind each choice are in [`proposals.md`](proposals.md).

## 1. Goal and success criteria

**Product goal.** A photographer uploads a wedding gallery of 500 to 5,000 photos. Within an hour, guests who enter the PIN see a strip of faces. Tapping one shows that person's photos with almost no wrong photos.

**Launch criteria (private beta)**

| Area | Target |
|---|---|
| Wrong photos shown under a person (guest-visible precision) | ≤ 2% of photos, measured on at least 3 real labelled galleries |
| People found (recall of people appearing in ≥ 3 photos) | ≥ 85% have a cluster |
| Processing time, 2,000-photo gallery | ≤ 60 min from upload end to People view ready |
| Compute cost | Measured and written down per 1,000 photos (target well under €1) |
| Privacy | Consent attestation, PIN gating, hide-me, expiry deletion, EU data location all working and tested |
| Legal | DPIA done, DPA template for photographers, guest notice text reviewed by a lawyer |

## 2. Phases and milestones

Each phase ends at a milestone with a **gate**. A gate is a yes/no question answered with evidence. A "no" stops the plan until the owner decides what to do.

```
Phase A  De-risk           Steps 1–7    M1: models good enough?  M2: Containers viable?
Phase B  Gallery product   Steps 8–10   M3: usable gallery with no faces (shippable on its own)
Phase C  People feature    Steps 11–14  M4: People view working end to end, privacy complete
Phase D  Beta              Step 15      M5: staging live, real galleries measured, beta go/no-go
Track L  Legal (owner, in parallel from day 1)   L1: consent model approved before M4
```

### Phase A: De-risk (steps 1–7)

The two biggest technical unknowns are **model accuracy on real event photos** and **whether Cloudflare Containers run the pipeline fast and cheaply enough**. Both are answered before any product code is written.

| Step | Prompt | Output | Where |
|---|---|---|---|
| 1 | [01](../prompts/01-scaffold.md) | Monorepo scaffold, tooling, model download script, licence register | local |
| 2 | [02](../prompts/02-detect-embed.md) | `face_core` detect, align, embed; local SQLite; display crops | local |
| 3 | [03](../prompts/03-quality.md) | Quality scoring and gating | local |
| 4 | [04](../prompts/04-clustering.md) | Clustering (three algorithms behind one interface), locks, contact sheets | local |
| 5 | [05](../prompts/05-eval.md) | Labelling helper, evaluation harness, sweep, **M1 report** | local |
| 6 | [06](../prompts/06-face-service.md) | FastAPI service and Dockerfile; derivatives; local container tests | local Docker |
| 7 | [07](../prompts/07-container-spike.md) | Throwaway Cloudflare Containers spike: cold start, throughput, cost, EU placement. **M2 report** | **Cloudflare (needs go-ahead)** |

**M1 gate: are the models good enough?** On the owner's labelled event photos, the best configuration reaches guest-visible precision ≥ 98% with person recall ≥ 70% (before tuning in the beta).
- Yes: continue.
- Borderline: try in order: higher detection resolution or tiling, stricter quality gate, a different clustering algorithm, then the challenger embedder (AuraFace; see proposal P5).
- No: options are a commercial InsightFace licence or another commercially licensed model. The owner decides, after a licence and cost check.

**M2 gate: are Containers viable?** Cold start ≤ 20 s, steady throughput ≥ 1 photo/s on 4 vCPU, cost per 1,000 photos known, and **EU placement confirmed**. If not, the fallback is a small EU VPS or a managed container service running the same image, called from the Worker. The rest of the architecture stays the same.

### Phase B: Gallery product (steps 8–10)

A complete gallery product **without faces**. It is useful on its own, and it is the base the People feature plugs into.

| Step | Prompt | Output |
|---|---|---|
| 8 | [08](../prompts/08-api-skeleton.md) | Worker + Hono skeleton, D1 migrations, data-access layer, magic-link auth, vitest |
| 9 | [09](../prompts/09-galleries-uploads.md) | Gallery CRUD, uploads to R2, PIN guest sessions, signed image URLs, access tests |
| 10 | [10](../prompts/10-web-app.md) | React app: photographer dashboard with uploader; guest gallery with grid, lightbox, download |

**M3 gate:** a photographer can create a gallery, upload 500 photos, set a PIN and share it. A guest can browse and download. All of it runs under `wrangler dev`, and every cross-gallery access test passes.

### Phase C: People feature (steps 11–14)

| Step | Prompt | Output |
|---|---|---|
| 11 | [11](../prompts/11-pipeline.md) | Queue, consumer, Container binding, derivatives and faces stored (GalleryDO SQLite + R2), processing status |
| 12 | [12](../prompts/12-clustering-orchestration.md) | GalleryDO debounce and alarms, cluster endpoint, stable person IDs, cover faces |
| 13 | [13](../prompts/13-photographer-people-tools.md) | People toggle with consent attestation; merge, split, move, hide, rename, set cover; review queue |
| 14 | [14](../prompts/14-guest-people-privacy.md) | Guest People view, AND filter, hide-me, "not me" suggestions, guest notice, expiry cron, deletion tests |

**M4 gate:** end to end on a real consented gallery. Precision measured on the deployed pipeline matches M1 within 1 point. Every item on the privacy checklist ([spec §8](spec.md#8-privacy-requirements)) has a test or a written procedure. **L1 must be closed.**

### Phase D: Beta (step 15)

| Step | Prompt | Output |
|---|---|---|
| 15 | [15](../prompts/15-staging-beta.md) | Staging and production environments, EU jurisdiction, observability, cost report, runbooks, beta checklist |

**M5 gate:** 2–3 friendly photographers, real galleries, numbers against the launch criteria in §1.

### Track L: Legal (owner, in parallel, starting now)

The People feature processes biometric data. This is the risk most likely to change the product, so it runs in parallel from day 1, not at the end.

1. Decide target countries for the first customers. Recommendation: EU only, and exclude Illinois, Texas and Washington (US states with biometric-specific laws) at launch.
2. Brief a privacy lawyer on [proposal P11](proposals.md#p11-consent-model). Key questions: controller/processor roles, Art. 9 legal basis, what consent from the couple and guests must look like, DPIA content, wording of the guest notice.
3. Write a DPIA, a processor agreement (DPA) template for photographers, a guest-facing notice and a retention policy.
4. Check Cloudflare's DPA and sub-processor list, and the effect of EU jurisdiction settings.

**L1 gate (before M4):** written legal sign-off on the consent model. Without it, the People feature does not go to any real guest.

## 3. After the beta (not scheduled)

- Selfie search for open galleries (guest uploads a selfie, matched only inside that gallery; selfie embedding discarded after the query).
- Zip downloads (streamed from R2 by a Worker or the Container).
- Presigned direct-to-R2 uploads for very large galleries.
- Incremental assignment, only if full reclusters become too slow or too costly (measure first).
- Scale tests with 20,000-photo galleries.
- Watermarking and download restrictions per gallery.

## 4. Risk register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R1 | Consent model judged insufficient under GDPR Art. 9 | Medium | Very high | Track L from day 1; feature off by default; processing only after opt-in; EU-only launch | Owner + lawyer |
| R2 | SFace not accurate enough on real event photos (children, low light, profiles) | Medium | High | M1 gate; three clustering algorithms; challenger embedder; precision-first threshold | Step 5 |
| R3 | Model training-data provenance is unclear (Apache-licensed weights, unknown data) | Medium | High | Record provenance in `licences.md`; give it to the lawyer; keep the embedder swappable (`embedding_model` column) | Step 1, Track L |
| R4 | Containers are too slow or costly, cold starts hurt, or EU placement fails | Low–Med | Medium | M2 spike before building on them; the image runs anywhere | Step 7 |
| R5 | Worker memory (128 MB) exceeded when moving embeddings or images | Medium | Medium | Embeddings stay in GalleryDO; images streamed, never buffered; the cluster call sends compact binary | Steps 11–12 |
| R6 | A guest sees another gallery's data | Low | Very high | One data-access layer; signed tokens bound to gallery ID; cross-gallery tests in CI | Steps 8–9 |
| R7 | Abuse: anyone with the PIN hides people or sabotages clusters | Medium | Low | Guest "not me" is a suggestion; hide-me is allowed (it only reduces exposure) and is logged | Step 14 |
| R8 | Expired galleries not fully deleted (R2 leftovers) | Medium | High | Deletion job with verification; tombstone row until R2 listing is empty; test | Step 14 |
| R9 | Scope creep (selfie search, print store) | High | Medium | Non-goals in CLAUDE.md; post-beta list | Owner |

## 5. Rough effort

Effort is measured in Claude Code sessions, each about 2 to 4 hours with review. These are estimates, not commitments.

| Phase | Sessions | Owner time needed |
|---|---|---|
| A (1–7) | 8–11 | Labelling about 300–500 faces (2–3 h); Cloudflare account and Paid plan for step 7 |
| B (8–10) | 5–7 | Reviewing UI; choosing an email provider |
| C (11–14) | 7–10 | Testing with real galleries; legal track |
| D (15) | 2–3 | Domain, beta photographers |
| **Total** | **22–31** | |
