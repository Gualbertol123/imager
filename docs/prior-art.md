# Prior art

These are products and projects that already do something similar, and what we can take from each. Researched in October 2026. Vendor claims are marked as such, and licences must be re-checked before reusing any code.

**Licence rule of thumb.** We are building a commercial hosted service, so:
- MIT, BSD and Apache code can be used, with attribution.
- **AGPL code (Immich, PhotoPrism, Ente) must not be copied.** The AGPL network clause would require us to publish our source. We read their docs and design ideas only and write our own code.
- For models, check the weights licence **and** the training data.

## 1. Commercial products (the market we enter)

| Product | What it does | What we take |
|---|---|---|
| **Pic-Time** "Suggested People" + Selfie Search ([help](https://help.pic-time.com/en/articles/9655382-how-can-gallery-guests-view-or-search-the-gallery-using-face-scan), [feature page](https://it.pic-time.com/features/ai-powered-gallery-search)) | The reference product. Guests pick a person from suggested faces and can add a keyword ("speech", "dancing"). The photographer turns it on per account, per gallery or by template, and can **let the main client decide**. Selfie search appears only in galleries with **≥ 10 unique people**. In-house model; says it stores "mathematical representations", not selfies. Advanced plan only. | Per-gallery toggle plus "let the couple decide" (supports our consent model, P11). A minimum number of people before showing the feature (we use 6; we could test 10). Keyword search is a later idea. |
| **Waldo Photos** ([site](https://waldophotos.com/how-facial-recognition-is-revolutionizing-photo-sharing/)) | Face recognition delivers photos to attendees' phones. Markets itself as "100% opt-in". | Opt-in as a selling point. Our "hide me" and notice support the same message. |
| **Kwikpic** / **Locaflash** ([G2](https://www.g2.com/products/kwikpic/discuss)) | Selfie-based personal galleries for events. Claims 99.9% accuracy (vendor claim); a competitor says it misses candid shots in low or mixed light. | A reminder that accuracy *on candids in poor light* is the real test (our per-tag evaluation). |
| **Samaro** ([blog](https://samaro.ai/blogs/face-identification-app-find-guest-photos)) | Selfie "Find Me"; PIN protection, watermarks, restricted downloads. | PIN plus download controls are standard; watermarking goes on the later list. |
| **ShowMyPhoto** ([SourceForge](https://sourceforge.net/projects/showmyphoto/)) | Event galleries with face or selfie search for photographers. Hosted. | Confirms the product category. |

**Takeaway.** Everyone offers **selfie** search, and fewer offer **selfie-free People suggestions** (Pic-Time). Our selfie-free, per-gallery, privacy-first People view is a real differentiator. Its weak points are accuracy on candids and legal clarity, which is exactly where this plan spends its effort.

## 2. Open-source photo managers with face clustering

### Immich
[Docs: facial recognition](https://docs.immich.app/features/facial-recognition) · licence **AGPL-3.0** (do not copy code)

- **Clustering:** derived from DBSCAN and adapted to run **incrementally**. A face is a *core point* if enough faces lie within the distance threshold ("minimum recognised faces", default 3). New faces join the person of the most similar already-assigned face. A face becomes a new person only if it is a core point; otherwise it is an outlier, such as a stranger in the background. Reclustering everything behaves like classic DBSCAN.
- **What we take:**
  1. Core/border/outlier logic as one of our three candidate algorithms (P4, `knn_dbscan`), reimplemented from this description.
  2. "Outliers are strangers" matches our `MIN_PHOTOS` rule.
  3. Their incremental design is the reference if we ever need incremental assignment (P2).
- **Separate ML service.** Immich runs its ML in its own container, called by the server. This is the same split as our Worker plus Container, so the pattern is proven.

### LibrePhotos
[Repo](https://github.com/LibrePhotos/librephotos) · backend listed as **MIT** (verify the LICENSE file before reusing anything)

- Moved face clustering to **HDBSCAN** (2022), then added **classification** back next to clustering (2024): clusters for unknown people, and a classifier once the user has named some.
- **What we take:** the hybrid idea matches our attach pass. Clusters create people; strict nearest-person matching attaches more faces. HDBSCAN is an option for later experiments (`sklearn.cluster.HDBSCAN`, BSD).

### PhotoPrism
Licence **AGPL-3.0**. It has face clustering with manual merges. Nothing specific to borrow beyond the UX of a "people" review page.

### Ente
Licence **AGPL-3.0**. End-to-end encrypted photos with on-device face clustering. Worth reading for the privacy UX, not for code.

## 3. Models

| Model | Licence | Training data | Use |
|---|---|---|---|
| **YuNet** `face_detection_yunet_2023mar` ([card](https://huggingface.co/opencv/face_detection_yunet)) | MIT | Trained on WIDER Face (research dataset; check with the lawyer) | **Primary detector.** About 10–300 px faces; WIDER Face hard AP ≈ 0.70–0.75, so small faces are its weak point, which is why we tile (P17). Fixed 640×640 export; OpenCV's `FaceDetectorYN` resizes input. |
| **SFace** `face_recognition_sface_2021dec` ([card](https://huggingface.co/opencv/face_recognition_sface)) | Apache 2.0 | **Not stated** on the model card | **Primary embedder**, 128-d MobileFaceNet. Record provenance as unknown. |
| **AuraFace v1** ([card](https://huggingface.co/fal/AuraFace-v1), [blog](https://huggingface.co/blog/isidentical/auraface)) | Apache 2.0 | "Commercially and publicly available data", not named | **Challenger**, only if M1 is borderline. 512-d, ArcFace-style; authors say it is weaker than the original ArcFace. |
| **InsightFace** buffalo_l, antelopev2 ([site](https://www.insightface.ai/)) | Code MIT, **weights non-commercial** | MS1M / Glint etc. | **Forbidden** unless we buy a commercial licence (paid fallback). |
| **dlib ResNet** ([dlib-models](https://github.com/davisking/dlib-models)) | Released into the public domain by the author | Includes datasets with non-commercial terms; one dataset creator asked for no commercial use | **Not recommended.** |
| **eDifFIQA** (face image quality, in OpenCV Zoo) | Check | Check | Optional later quality signal for P18; licence check first. |

Background: a [LocalAI write-up](https://localai.io/docs/features/face-recognition/index.html) also lists YuNet + SFace (about 40 MB) as its recommended commercially usable face stack. It is a useful cross-check, but not a licence opinion.

## 4. Clustering research

- **[learn-to-cluster](https://github.com/yl-1993/learn-to-cluster)** (benchmark repo, ICCV/CVPR papers). On MS-Celeb-1M (millions of faces), Chinese Whispers scores about 73 pairwise F, while a k-NN DBSCAN variant scores about 91. **Lesson:** do not lock in Chinese Whispers without testing (P4). These are very large datasets, though; with a few thousand faces per gallery the gap may be much smaller, and our own evaluation decides.
- **Learned clustering** (GCN-based linkage, pairwise classifiers): state of the art, but it needs training data and is out of scope (non-goal: no training on guest faces).
- **Metrics:** pairwise and BCubed definitions follow those papers; we add guest-visible photo precision (P15).

## 5. Building blocks for the gallery

| Need | Project | Licence | Notes |
|---|---|---|---|
| Upload UI with retries | **Uppy** ([uppy.io](https://uppy.io)) | MIT | XHR uploader to our Worker (P8). Its S3 multipart plugin is the later upgrade path. |
| Resumable uploads on Workers + R2 | [fishcakeday/tus-server](https://github.com/fishcakeday/tus-server) | check | Reference only if we ever need tus; tus chunks do not match R2's equal-part rule, so re-chunking is needed. |
| Justified photo grid | **react-photo-album** | MIT | Lays out rows by aspect ratio. |
| Lightbox | **PhotoSwipe** v5 | MIT | Touch gestures, zoom, works with any grid. |
| API framework | **Hono** | MIT | First-class on Workers. |
| Validation | **zod** | MIT | Shared schemas in `packages/shared`. |
| Image host on Workers + R2 | [qwe7002/sliverimage](https://github.com/qwe7002/sliverimage) | check | Small reference for serving R2 images through a Worker with signed URLs. |
| Photography site on Cloudflare | [Jamie Ede write-up](https://jamieede.com/posts/building-a-photography-portfolio-on-cloudflares-full-stack/) | blog | D1 metadata + Cloudflare Images; an example of the Images alternative to P6. |
| Clustering and linear assignment | **scikit-learn**, **scipy** (`linear_sum_assignment`), **numpy** | BSD | Agglomerative, HDBSCAN, Hungarian matching. |

## 6. Legal background (not legal advice)

- Faces processed to tell people apart are treated as **special category biometric data** under GDPR Art. 9. Practitioner guidance says consent for face recognition must be **explicit and informed**; a general photo notice is not enough ([GDPR Local](https://gdprlocal.com/gdpr-for-images/), [sweap event guide](https://www.sweap.io/en/blog/data-protection-events-guide-photos)).
- A Danish case on fitness-centre face scans required consent *before* scanning, for unique identification ([NYCU Law Review](https://lawreview.law.nycu.edu.tw/en/facial-recognition-at-the-fitness-center-under-the-general-data-protection-regulation-article-91-and-92a-2)).
- An event-photo vendor suggests stating face recognition explicitly in event or registration terms ([SportPXL docs](https://documentation.sportpxl.com/?p=8623)).
- These sources support P11: off by default, host consent, guest notice, opt-out, lawyer review. None of them is regulator guidance for weddings; the lawyer must confirm.
