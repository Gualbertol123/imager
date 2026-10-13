# Licence register

Every model and dependency, with its licence and (for models) training-data provenance. Add a row **before** adding the dependency. "Commercial OK" means the licence permits use in a paid hosted service.

## Models

| Model | File | Licence | Training data | Commercial OK | Source | Added |
|---|---|---|---|---|---|---|
| YuNet | `face_detection_yunet_2023mar.onnx` | MIT | WIDER Face (reported; research dataset) | Yes for weights; data question for the lawyer | https://huggingface.co/opencv/face_detection_yunet | planned |
| SFace | `face_recognition_sface_2021dec.onnx` | Apache 2.0 | **Not stated** | Yes for weights; data provenance unknown | https://huggingface.co/opencv/face_recognition_sface | planned |
| ~~InsightFace buffalo_l / antelopev2~~ | — | Non-commercial | — | **No (forbidden)** | — | — |

## Python dependencies (services/face)

| Package | Licence | Purpose |
|---|---|---|
| opencv-python-headless | Apache 2.0 | YuNet, SFace, image ops |
| numpy | BSD-3 | arrays |
| scipy | BSD-3 | Hungarian matching |
| scikit-learn | BSD-3 | agglomerative, DBSCAN/HDBSCAN, NearestNeighbors |
| networkx | BSD-3 | graph utilities (Chinese Whispers) |
| pillow | MIT-CMU (HPND) | decoding, EXIF orientation, WebP |
| fastapi, uvicorn | MIT, BSD-3 | service (step 6) |
| pytest, ruff | MIT | dev |

## JavaScript dependencies (apps)

| Package | Licence | Purpose |
|---|---|---|
| hono | MIT | API |
| zod | MIT | validation |
| react, react-dom, react-router | MIT | frontend |
| @tanstack/react-query | MIT | data fetching |
| react-photo-album | MIT | grid |
| photoswipe | MIT | lightbox |
| @uppy/core, @uppy/dashboard, @uppy/xhr-upload | MIT | uploads |
| vitest, @cloudflare/vitest-pool-workers, wrangler | MIT / MIT-or-Apache | dev |

Re-check each licence when the package is actually installed (versions change).
