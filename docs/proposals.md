# Proposals

Each proposal gives a recommendation, the reasons for it, the alternatives considered, and what would make us change our minds. **All of P1–P18 were accepted by the owner on 2026-10-14** (decision D3). To change one, add a new decision in [`decisions.md`](decisions.md) and update the proposal's status.

Summary:

| # | Topic | Recommendation | Status |
|---|---|---|---|
| P1 | Where face data lives | Per-gallery Durable Object with SQLite storage; D1 only for global tables | Accepted |
| P2 | Clustering mode in v1 | Full recluster per run plus stable ID mapping; no incremental assignment | Accepted |
| P3 | How corrections are stored | Face locks plus person-level cannot-links, not face-pair constraints | Accepted |
| P4 | Clustering algorithm | Evaluate three (agglomerative average-linkage, Chinese Whispers, Immich-style DBSCAN); choose by data | Accepted |
| P5 | Embedding model | SFace primary; AuraFace challenger; InsightFace commercial licence as paid fallback | Accepted |
| P6 | Thumbnails and web sizes | Generate in the Container during processing, store in R2 | Accepted |
| P7 | Photographer auth | Email magic link plus session cookie (sessions in D1) | Accepted |
| P8 | Uploads | Uppy in the browser, one request per photo to the Worker, streamed to R2 | Accepted |
| P9 | Processing pipeline | Queue per photo, batched consumer, image streamed to the Container, JSON out | Accepted |
| P10 | Data location | EU jurisdiction for R2, D1, Durable Objects and Containers | Accepted |
| P11 | Consent model | Opt-in per gallery, host consent, guest notice, PIN gating, guest opt-out; lawyer sign-off | Accepted |
| P12 | Frontend | React + Vite served by Workers static assets; PhotoSwipe, react-photo-album, Uppy, TanStack Query | Accepted |
| P13 | Guest corrections | "Not me" is a suggestion for the photographer, not an automatic edit | Accepted |
| P14 | "Hide me" semantics | Faces excluded, embeddings and crops deleted; only the hidden person's centroid kept until expiry to block re-creation | Accepted |
| P15 | Evaluation metrics | Add guest-visible photo precision and cluster purity next to pairwise and BCubed; report per tag (occlusion, pose, group size) | Accepted |
| P16 | Docs and session structure | Short CLAUDE.md for rules; spec and plan in `docs/`; one prompt per step | Accepted (this repo) |
| P17 | Group photos (many people per photo) | Adaptive tiled detection, same-photo cannot-link, a `photo_persons` index for AND queries, group-aware UI | Accepted |
| P18 | Partially covered faces | Two tiers: "core" faces cluster, "partial" faces only attach to an existing person with a strict threshold and margin; occlusion score from landmarks, detector score and feature norm | Accepted |

---

## P1. Face data lives in the gallery's Durable Object

**Recommendation.** Each gallery has one `GalleryDO` (a Durable Object with SQLite storage). It stores that gallery's `faces`, `persons`, `face_locks`, `person_cannot_links`, `exclusions`, `suggestions` and `cluster_runs`, including embeddings. D1 keeps global tables only: `owners`, `sessions`, `galleries`, `photos`, `login_tokens`, `audit_log`.

**Why**
- **Isolation by construction.** The non-goal "no cross-gallery face queries" is enforced by the platform: no single query can reach two DOs' storage. In D1 it would be a convention that one bad `WHERE` clause breaks.
- **Serialisation for free.** A DO handles one request at a time per object (input gates), so clustering runs, corrections and new faces cannot race. We need this object anyway for debouncing.
- **Deletion is provable.** `ctx.storage.deleteAll()` wipes every embedding of a gallery in one call.
- **Size and memory.** A 20,000-photo gallery has about 100,000 faces, and 128-d float32 embeddings take about 51 MB. That fits easily under the 10 GB per-DO limit. Reading it from D1 into a 128 MB Worker would be tight, and D1 has a 10 GB cap per database that all galleries would share.

**Alternatives**
- *Everything in D1.* Simpler admin queries, but isolation is only a convention, the 10 GB shared cap is a ceiling, and races must be handled separately.
- *One D1 database per gallery.* Isolation is good, but it means dynamic database creation and migrations across thousands of databases. Too heavy.

**Revisit if** we need analytics across galleries. Even then, export aggregate counts only, never embeddings.

## P2. Full recluster in v1, with stable person IDs

**Recommendation.** Each clustering run reclusters all usable faces in the gallery. New clusters are then mapped to existing person IDs by maximum face overlap (Hungarian matching on the overlap matrix), so names, hides and covers survive. Incremental assignment (centroids, margins, unassigned pool) is postponed.

**Why**
- Galleries arrive in one or two bulk uploads, not a constant stream. Incremental assignment solves a problem we may not have.
- Building a k-NN graph over 100,000 128-d vectors is about 10¹⁰ multiply-adds. With chunked numpy that is seconds to a minute or two on 4 vCPU, well within a background job.
- Incremental logic is the most complex and most bug-prone part of the original spec: drift, order dependence and margins. Removing it removes a whole class of bugs.
- A full run with locks (P3) is deterministic: the same inputs and seed always give the same output, which makes evaluation meaningful.

**Revisit if** the M2 spike or the beta shows reclusters over about 5 minutes or noticeable cost per upload batch. Immich's incremental DBSCAN design ([prior-art](prior-art.md#immich)) is the reference.

## P3. Corrections are stored as locks

**Recommendation.** Three structures, all in GalleryDO:
- `face_locks(face_id → person_id)`: set by move, split and merge. A locked face always ends up in that person.
- `person_cannot_links(person_a, person_b)`: "these are different people". Enforced through each person's locked faces.
- `exclusions(face_id)`: hidden faces (hide-me, "not a face", stranger).

Merging A into B relocks A's faces to B and leaves a redirect `A → B` so old URLs still work.

**Why.** Must-link pairs grow as O(n²): merging two 300-face people needs either 90,000 pairs or a careful spanning set. Locks are O(n), easy to explain in the UI, and easy to apply after clustering: assign locked faces, remove them from other clusters, split any cluster that would join two cannot-linked persons.

## P4. Clustering algorithm: evaluate three, pick by data

**Recommendation.** Put three algorithms behind one function signature and choose the default with the evaluation harness:
1. **Agglomerative clustering, average linkage, distance threshold** (scikit-learn). Average linkage is conservative: a cluster grows only if the *average* similarity stays high, which favours precision.
2. **Chinese Whispers on a k-NN graph** (the original spec). Fast and simple, but it tends to chain look-alikes through bridge faces. Published face-clustering benchmarks rank it well below density-based methods at scale.
3. **Immich-style DBSCAN on a k-NN graph** (min samples = 3). Core and border faces, with outliers left unassigned. This is proven at scale in Immich, though their code is AGPL so we reimplement from the published description only.

All three apply the same-photo rule and locks in the same way, in a shared post-processing step.

**Why.** The original plan fixes Chinese Whispers before any data exists, which goes against "measure before tuning". The cost of testing three is small, since they share the graph, post-processing and tests.

## P5. Embedding model

**Recommendation**
- **Primary:** SFace (OpenCV Zoo, Apache 2.0, 128-d) with YuNet (MIT) for detection. Both are small and CPU-friendly, and are called through OpenCV's built-in `cv2.FaceDetectorYN` and `cv2.FaceRecognizerSF` with no extra runtime.
- **Challenger (only if M1 is borderline):** AuraFace v1 (fal, Apache 2.0, ArcFace-style 512-d). The authors say it was trained on commercially licensed data but do not name the dataset, and admit it is weaker than the original ArcFace. Ask the owner before downloading it.
- **Paid fallback:** a commercial InsightFace licence (the strongest open models; pricing on request).
- **Not recommended:** dlib's ResNet model. The author released the weights as public domain, but its training data includes datasets with non-commercial terms, and one dataset creator asked that it not be used commercially.

**Training-data provenance.** The OpenCV Zoo card for SFace does not name the training set. Record "unknown" in `licences.md` and include it in the legal briefing. Store `embedding_model` on every face so a later model swap is a re-embed, not a migration.

## P6. Thumbnails and web sizes are made in the Container

**Recommendation.** The Container already decodes every photo (with EXIF orientation applied) for face detection. In the same pass it writes three derivatives to R2: `thumb` (400 px, WebP), `web` (2048 px, WebP/JPEG) and, for galleries with People enabled, face display crops (256 px, padded).

**Why**
- **Cost.** Cloudflare Images charges $0.50 per 1,000 *unique transformations per month* after 5,000 free. A 5,000-photo gallery in 3 sizes is 15,000 transformations, about $5 a month; a 20,000-photo gallery is about $27 a month. Pre-generated R2 objects cost cents.
- **One decode.** It reuses work already done for detection.
- **Determinism.** Fixed files in R2 are easy to cache, sign and delete.

**Trade-off.** Thumbnails appear only after the photo is processed, typically seconds. Until then the UI shows a placeholder. For galleries *without* People, the Container still makes derivatives but skips face detection.

**Alternative.** Cloudflare Images transformations. Good for prototyping, and worth reconsidering if we need many more sizes.

## P7. Photographer authentication

**Recommendation.** Email magic link: a single-use token with 15-minute expiry, stored hashed in D1. A session cookie (`HttpOnly`, `Secure`, `SameSite=Lax`) maps to a `sessions` row in D1. Rate limiting uses the Workers Rate Limiting binding.

**Why.** No passwords to store or reset. Photographers log in rarely, from a few devices. Cloudflare Access is designed for an organisation's own staff, not public customers.

**Needs.** An email-sending provider. Options: Cloudflare's own email sending if it is available on the account (verify), Postmark or Resend. These only see photographer email addresses, never face data. **The owner picks one.**

**Alternative.** OAuth "Sign in with Google" can be added later alongside magic links.

## P8. Uploads

**Recommendation.** Uppy (MIT) in the browser, with the XHR uploader:
- one request per photo to `POST /api/galleries/:id/photos`
- 4–6 photos in parallel
- automatic retry per file
- client-side check of type and size

The Worker streams each request body straight into `R2.put()` without buffering. It writes the `photos` row, computes nothing heavy, and enqueues a processing message.

**Why.** Event photos are 5–40 MB JPEGs, well under the Worker request-size limit on the Paid plan. Per-file retries give practical resumability without the tus protocol. Proxying through the Worker keeps one auth path and one place to enforce quotas.

**Revisit** with presigned multipart uploads straight to R2 if uploads of thousands of large files are too slow through the Worker.

## P9. Processing pipeline

**Recommendation**
- One queue message per photo: `{gallery_id, photo_id, r2_key, want_faces}` (well under the 128 KB limit).
- Consumer batch size 10, max wait 5 s, and a dead-letter queue after 5 retries.
- For each message, the consumer streams the original from R2 to the Container (`POST /process`, body = image bytes, query = options). It processes a few in parallel inside the batch, up to the Container's capacity.
- The Container returns JSON with photo info, derivatives (base64, a few hundred KB) and faces (bbox, landmarks, quality, embedding as base64 float32, display crop).
- The consumer writes derivatives to R2, updates `photos` in D1, and calls `GalleryDO.addFaces()`. The DO stores faces and schedules a clustering alarm.

**Why.** Streaming bytes in means the Container needs no R2 credentials. JSON out keeps the protocol simple to test. The 15-minute consumer wall time and the 5-minute CPU limit are far above what a batch of 10 needs, because the time is spent waiting on the Container.

**Revisit** if the Worker-to-Container hop is a bottleneck in the M2 spike. The alternative is for the Container to read and write R2 through the S3 API with a bucket-scoped token.

## P10. Data location

**Recommendation.** Set the EU jurisdiction everywhere it exists:
- an R2 bucket created with the `eu` jurisdiction
- D1 created with the EU jurisdiction (verify the current option)
- Durable Objects namespace fetched with `jurisdiction: "eu"`
- Containers with `constraints.jurisdiction = "eu"` (available since April 2026)

**Why.** It reduces GDPR transfer risk for EU customers and is a strong sales point. Note that Cloudflare is a US company. Whether EU residency is enough under the CLOUD Act is a question for the lawyer, not for code.

## P11. Consent model

**Recommendation** (to be confirmed by a lawyer; this is not legal advice):
1. **No processing by default.** Face detection runs only after the photographer turns People on for the gallery.
2. **Photographer attestation.** They confirm they have the host's (couple's) agreement and that guests were informed. Stored with a timestamp, version of the wording and user ID.
3. **Guest notice before first use.** The first time a guest opens the People view, a short notice explains what it does, that data stays in this gallery and is deleted on expiry, and how to opt out. This is recorded per guest session.
4. **Opt-out at any time** with "Hide me" (P14). Opt-out removes the person immediately, without anyone's approval.
5. **PIN gating.** People view is never shown without a gallery session.
6. **Retention.** Embeddings are deleted at gallery expiry, when People is switched off, or on owner request.
7. **Roles.** The photographer is the controller and the platform is the processor, with a DPA.
8. **Markets.** EU at launch; Illinois, Texas and Washington are excluded until a pre-event written-consent flow exists.

**Why.** Biometric data used to tell people apart is special category data under GDPR Art. 9. A guest seeing a face strip has not consented to their face being processed. The strongest practical position is to keep processing scoped, opt-in, transparent and easy to undo, and to get a lawyer's view on the legal basis early. This is the plan's top risk (R1).

## P12. Frontend

**Recommendation.** React 18 + Vite + TypeScript, served by **Workers static assets** from the same Worker as the API (one deploy, same origin, no CORS). Libraries, all MIT:
- **react-photo-album** for justified grids
- **PhotoSwipe** for the lightbox
- **Uppy** for uploads
- **TanStack Query** for data fetching
- **React Router**

Plain CSS modules or Tailwind; the owner chooses.

**Why.** It is boring, well-documented tech. Same origin removes a class of CORS and cookie bugs.

## P13. Guest "not me" is a suggestion

**Recommendation.** "This isn't this person" on a photo creates a `suggestion` row and hides that photo from the guest's own current view. The photographer sees suggestions in a review queue and accepts (which creates an exclusion or a move) or dismisses. Suggestions are rate-limited per guest session.

**Why.** Anyone with the PIN could otherwise rewrite the clusters for everyone. Only the photographer's actions become locks.

## P14. "Hide me" semantics

**Recommendation.** When a guest chooses "Hide me" for a person:
1. That person and all its faces are marked `excluded`.
2. Their embeddings are overwritten with NULL and their crops are deleted from R2, immediately.
3. The person disappears from the strip for all guests.
4. To stop new uploads re-creating the person, the DO keeps the hidden person's **centroid only** for the rest of the gallery's life (deleted at expiry), and auto-excludes new faces that match it at a strict threshold.

**Why.** If the embeddings were simply deleted, the next recluster would rebuild the person from the photos. Keeping one centroid is the smallest amount of data that respects the opt-out. Disclose this in the notice; the lawyer should confirm it.

**Simpler alternative.** Delete everything and block new clustering of that gallery's later uploads near the hidden faces' photos. This is less precise.

**Abuse.** Anyone with the PIN can hide anyone. We accept that, because hiding only reduces exposure. It is logged and the photographer can see it, but cannot undo it without the person's request.

## P15. Evaluation metrics

**Recommendation.** For every setting, report:
- pairwise precision and recall
- BCubed precision, recall and F
- **guest-visible photo precision**: for each predicted person, the share of their photos that do contain the true person, averaged over people weighted by photo count
- **cluster purity**: the share of clusters with zero wrong faces
- **person recall**: the share of true people with ≥ 3 photos who get a cluster

Choose the operating point with guest-visible photo precision ≥ 0.98, then maximise person recall.

**Why.** Pairwise precision weights large clusters quadratically and can hide a small cluster that is entirely wrong. Guests see photos, not face pairs.

Also report **detection recall** on about 20 hand-checked group photos. Clustering metrics cannot see faces that were never detected.

## P16. Docs and session structure (accepted)

CLAUDE.md holds the rules only, because it is loaded every session and long files dilute attention. The spec, plan and proposals live in `docs/`. Each build step has its own prompt in `prompts/` and ends with a short report in `docs/reports/`.

## P17. Group photos: many people in one photo

Weddings are full of group shots: tables, families, the whole party on the stairs. These are often the photos guests want most, and also where faces are smallest. The design must treat "several people per photo" as the normal case.

**Recommendation**

*Detection*
- Detect at 2048 px on the long edge by default.
- **Adaptive tiling.** If the first pass finds ≥ 8 faces, or any face with inter-eye distance < 2 × the usable minimum, run a second pass on overlapping tiles of the **full-resolution** image: 1024 px tiles with 20% overlap. Merge the two passes with non-maximum suppression (IoU 0.4), keeping the higher score.
- Record `detect_pass` (`full` / `tile`) and the face's pixel size in the original image. Tiny faces are kept for counting, and only used when they meet quality rules (P18).

*Clustering*
- **Same-photo cannot-link.** Two faces in the same photo are never the same person. Edges between them are removed before clustering, and any cluster that still holds two faces from one photo is split: the face with the higher mean similarity to the cluster stays, and the other returns to the pool. A group photo with 40 faces also gives 780 free negative pairs, which make the clusters sharper.
- Mirror and reflection duplicates (the same face twice in one photo, for example in a mirror) are rare. The rule above handles them by keeping one.

*Data and queries*
- GalleryDO keeps a derived table `photo_persons(photo_id, person_id, face_id, confidence)`, rebuilt after each clustering run and on every correction.
- A **multi-person AND query** ("photos with Anna *and* Marco") is `GROUP BY photo_id HAVING COUNT(DISTINCT person_id) = n`, which is cheap on the DO's local SQLite.
- Also offer an **"only these people"** option, which excludes photos where other *recognised* people appear. This answers "photos of just the two of us", a very common request.
- `photos.face_count` lets the UI offer "group photos" and "portraits" filters later.

*UI*
- In the lightbox, show the recognised people as small face chips. Tapping one jumps to that person.
- Never draw boxes on other people's faces for guests. It draws attention to strangers and leaks who was recognised. Boxes are for photographers' correction tools only.

**Why.** Group photos contain most of the faces and most of the errors: small faces, motion, partial cover. If the pipeline is tuned only on portraits, the evaluation will be too optimistic.

**Evaluation.** Tag each labelled photo with a group-size bucket (1, 2–4, 5–15, 16+) and report detection recall and precision per bucket.

## P18. Partially covered faces

Faces at events are often partly hidden: hands, glasses, sunglasses, veils, hair, hats, a glass raised for a toast, another person's shoulder, profile views, face paint, masks. Covered faces give weaker, noisier embeddings, and they are the main cause of wrong merges.

**Recommendation: two tiers of faces.**

| Tier | Rule | Used for |
|---|---|---|
| **core** | Passes all quality limits, including occlusion | Building clusters; candidates for the person's cover face and exemplars |
| **partial** | Detected with reasonable confidence, but fails occlusion, pose or size limits within a looser band | **Attach-only.** Joins an existing person only if its best match passes a **strict** threshold *and* beats the second-best person by a **margin**, and never creates or merges people |
| **unusable** | Below the loose band (tiny, very blurred, mostly covered) | Stored for counts; never shown in People |

**Occlusion score.** It combines signals that are cheap with YuNet + SFace, and the evaluation decides the weights:
1. **Detector confidence.** YuNet's score drops for covered faces.
2. **Landmark geometry.** Ratios between eyes, nose and mouth corners compared with a frontal template. Covered regions make landmarks implausible: a mouth too close to the nose, eyes asymmetric beyond what yaw explains.
3. **Embedding feature norm.** The length of the SFace feature *before* L2 normalisation tends to fall for low-quality and covered faces. (This is known behaviour for ArcFace-style models; MagFace and AdaFace use it explicitly.) It is a **hypothesis for SFace**, to be confirmed in step 5.
4. **Optional, later:** a face image quality model (OpenCV Zoo has eDifFIQA). Only if 1–3 are not enough, and **only after a licence check.**

**Also**
- Faces with sunglasses or heavy cover are never chosen as cover faces or exemplars.
- Sunglasses and veils at a wedding usually come and go. The same person appears uncovered in other photos, which is why attach-only gets back most of the recall without risking merges.
- Profiles: yaw from landmarks above about 45° is `partial`; above about 70° is `unusable`. SFace is trained mostly on near-frontal faces.

**Why.** Precision first: covered faces must never be able to *bridge* two different people into one cluster, and attach-only makes that structurally impossible. Recall second: photos where someone is partly covered still show up in their set when the match is strong.

**Evaluation.** Each labelled face gets `occlusion` (none / partial / heavy) and `pose` (frontal / three-quarter / profile) tags. Report precision and recall per tag, and the attach-only gain: recall with and without the partial tier, at equal precision.
