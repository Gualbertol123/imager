# Cloudflare notes

These are platform facts we rely on, each with its source. Re-check them before relying on any of them in code, and update the "checked" date.

| Fact | Value | Source | Checked |
|---|---|---|---|
| Containers plan | Workers Paid ($5/month) required; **not available on Free** | [Containers pricing](https://developers.cloudflare.com/containers/pricing/) | 2026-10-14 |
| Containers included usage (Paid) | 25 GiB-hours memory, 375 vCPU-minutes, 200 GB-hours disk per month; billed per 10 ms active | same | 2026-10-14 |
| Container instance types | lite (1/16 vCPU, 256 MiB) … standard-4 (4 vCPU, 12 GiB, 20 GB disk); custom types 1–4 vCPU, ≥ 3 GiB per vCPU | [Containers limits](https://developers.cloudflare.com/containers/platform-details/limits/) | 2026-10-14 |
| Container image storage | 50 GB per account; image size ≤ instance disk | same | 2026-10-14 |
| Container GPU | **Not mentioned in the docs.** Design for CPU only | same | 2026-10-14 |
| Container placement | `constraints.jurisdiction = "eu"` maps to the EEUR and WEUR regions (since 2026-04-05) | [Placement](https://developers.cloudflare.com/containers/platform-details/placement/), [changelog](https://developers.cloudflare.com/changelog/2026-04-05-regional-placement/) | 2026-10-14 |
| Containers GA date | No official GA announcement found; only a secondary source | — | 2026-10-14 |
| D1 max database size | 10 GB (Paid, cannot be raised), 500 MB (Free) | [D1 limits](https://developers.cloudflare.com/d1/platform/limits/) | 2026-10-14 |
| D1 bound parameters | **100 per query** (chunk batch inserts accordingly) | same | 2026-10-14 |
| D1 row / BLOB size | 2 MB max | same | 2026-10-14 |
| D1 query duration | 30 s, also caps a whole `batch()` | same | 2026-10-14 |
| Durable Object storage | 10 GB per SQLite-backed object | [DO limits](https://developers.cloudflare.com/durable-objects/platform/limits), [SQLite GA](https://developers.cloudflare.com/changelog/2025-04-07-sqlite-in-durable-objects-ga) | 2026-10-14 |
| Queue message size | 128 KB including about 100 B metadata | [Queues limits](https://developers.cloudflare.com/queues/platform/limits/) | 2026-10-14 |
| Queue batch | ≤ 100 messages; max wait 60 s; `sendBatch` ≤ 100 messages / 256 KB | same | 2026-10-14 |
| Queue consumer | 15 min wall time; CPU 30 s default, up to 5 min via `limits.cpu_ms` | same | 2026-10-14 |
| Queue retries / delay | up to 100 retries; delay up to 24 h; retention 24 h Free, up to 14 d Paid | same | 2026-10-14 |
| Images transformations | 5,000 unique per month free, then $0.50 per 1,000 unique per month | [Images pricing](https://developers.cloudflare.com/images/pricing) | 2026-10-14 |
| R2 multipart | Parts must be equal size except the last; incomplete uploads abort after 7 days | [R2 Workers API](https://developers.cloudflare.com/r2/api/workers/workers-api-reference/) | 2026-10-14 |

## Still to verify (step 7 / step 15)

- Worker request body size limit on our plan (expected 100 MB on Paid).
- Workers memory (128 MB) and CPU limits for the API Worker.
- R2 `eu` jurisdiction bucket creation, and how it affects the S3 endpoint.
- D1 jurisdiction or location option for new databases.
- Durable Object `jurisdiction("eu")` namespaces and their interaction with Containers.
- Container cold-start time with our image (measure in step 7).
- Workers Rate Limiting binding availability and limits.
- Email sending options on the account (P7).
- Cloudflare DPA and sub-processor list (Track L).
