# Decisions

The owner answers these. Claude Code must not assume an answer. Each step's prompt lists which decisions it needs.

On 2026-10-14 the owner accepted all recommendations, so every decision below whose recommendation was a single clear choice is recorded as answered. Decisions that need facts only the owner has (photos, accounts, provider, domain, lawyer) stay open.

| # | Question | Needed by | Recommendation | Answer | Date |
|---|---|---|---|---|---|
| D1 | Do you have a folder of photos from one event that you have consent to use? Roughly how many? Any with many people per photo and partly covered faces? | Step 2 | Yes, one real wedding, 500–2,000 photos, including group shots | **Open.** Copy the photos into `data/event1/` (git-ignored) and note the rough count here | |
| D2 | Python 3.12, uv, Node 22, pnpm and Docker Desktop with WSL2 installed? | Step 1 (Python, uv), step 6 (Docker), step 8 (Node, pnpm) | Install all before they're needed | **Partly.** Present: Git 2.56, Python 3.12.7, uv 0.13.1, WSL. Missing: Node 22, pnpm, Docker Desktop, GitHub CLI (winget source blocked). See `docs/STATUS.md` | 2026-10-14 |
| D3 | Accept proposals P1–P18? | Step 1 | Accept all | **Accepted, all P1–P18** | 2026-10-14 |
| D4 | Cloudflare account on the Workers Paid plan ($5/month), and OK to create throwaway resources for the step-7 spike? | Step 7 | Yes | Open | |
| D5 | Email provider for magic links | Step 8 | Postmark or Resend (or Cloudflare email sending if available) | Open: pick one | |
| D6 | CSS approach: Tailwind or CSS modules | Step 10 | Tailwind | **Tailwind** | 2026-10-14 |
| D7 | First customers' countries; exclude US biometric-law states? | Step 13 | EU only at launch | **EU only at launch; Illinois, Texas and Washington excluded** | 2026-10-14 |
| D8 | EU jurisdiction for all storage and compute | Step 15 | Yes | **Yes** | 2026-10-14 |
| D9 | Lawyer engaged; consent model (P11) signed off | Before any real guest uses People | — | Open | |
| D10 | Domain name | Step 15 | — | Open | |
| D11 | If M1 is borderline: try AuraFace, or buy an InsightFace licence? | After step 5 | Try AuraFace first (free); licence if still short | **AuraFace first, then licence** (still ask before downloading) | 2026-10-14 |
| D12 | Selfie search in v1? | After M4 | No, post-beta | **No, post-beta** | 2026-10-14 |
| D13 | Task runner on Windows: `just`, or plain commands? | Step 1 | Root `package.json` scripts (pnpm) for JS, `uv run` for Python, and every command written down in the README. No extra tool | **Plain commands + pnpm scripts, no justfile** | 2026-10-14 |
