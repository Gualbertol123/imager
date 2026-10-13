# Step 15: Staging, production and private beta (Gate M5)

## Context
Phase D, step 15. This step deploys to real Cloudflare in the EU, with staging and production environments, monitoring and runbooks, then runs a private beta with 2–3 photographers. **Every resource-creating or deploying command needs my explicit OK in this session.**

## Read first
`docs/STATUS.md`, `CLAUDE.md`, `docs/plan.md` §1 (launch criteria), `docs/cloudflare-notes.md`, `docs/proposals.md` P10, `docs/decisions.md` (D8, D9, D10), `docs/reports/step-07.md`, `docs/reports/step-14.md`.

## Decisions needed
D8 (EU), D9 (legal sign-off; if not done, deploy staging only and keep People disabled in production), D10 (domain).

## Tasks
1. **Re-verify** the open items in `docs/cloudflare-notes.md` ("Still to verify") against the current docs, with URLs.
2. **Environments** in `wrangler.jsonc`: `staging` and `production`, with separate resources named `imager-<env>-<thing>`:
   - D1 (EU jurisdiction or location, per the verified option)
   - R2 bucket created with the `eu` jurisdiction
   - Queue plus DLQ
   - Durable Object namespace using the `eu` jurisdiction
   - Container with `constraints.jurisdiction: "eu"`
   - a cron trigger

   Write the exact creation commands into `docs/runbooks/provision.md` **first**, show them to me, and run them only after my OK.
3. **Secrets:** list every secret (`GUEST_TOKEN_SECRET`, `IMAGE_TOKEN_SECRET`, email API key, …). I set them myself with `wrangler secret put --env …`; you don't see the values.
4. **Domain and routes:** a custom domain for production and `staging.` for staging. HTTPS only, HSTS, security headers check.
5. **Observability:**
   - Workers Logs on, with sampling, and the redaction rules verified.
   - A daily scheduled health report: DLQ depth, error counts, processing backlog, cluster run durations, written to `audit_log` and sent by email to me.
   - Alerts for DLQ > 0 and processing errors > 2% (use Cloudflare notifications where available; check).
6. **Backups:** document D1 Time Travel restore. Note that DO storage and R2 are not backed up by us; this is acceptable because the originals are re-uploadable and expiry is intended. Get my confirmation on that.
7. **Cost:** run one 2,000-photo staging gallery (my consented event). Record Containers, Workers, R2, D1, DO and Queue usage, and cost per gallery, in `docs/reports/cost.md`.
8. **Runbooks** in `docs/runbooks/`:
   - provision
   - deploy and rollback (`wrangler versions` / `rollback`)
   - data deletion request (a guest or photographer asks)
   - incident (data exposure suspected)
   - rotate secrets
   - expire now
9. **CI** (GitHub Actions): lint, typecheck, vitest, pytest and Docker build on PRs. A manual-approval deploy to staging; production by tag. Ask before adding the workflow, and use repository secrets I create.
10. **Beta checklist** `docs/beta-checklist.md`: the legal docs published (privacy notice, retention policy, DPA template), support email, feedback form, per-gallery People default off, launch criteria table to fill.

## Constraints
- No People in production until D9 (legal sign-off) is recorded.
- No real guest data in staging except my own consented event.

## Done when
- Staging and production are deployed in the EU and smoke-tested; monitoring works; the cost report is written; runbooks exist.

## Stop and report
`docs/reports/step-15.md` with a **Gate M5** table against plan §1, and a list of what to watch during the beta. Update `docs/STATUS.md`. Commit. Stop.
