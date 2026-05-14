# MeshForge — Project Guide for AI Coding Agents

> **SYNC NOTICE.** This file is mirrored with `AGENTS.md` at the repository root.
> They MUST contain identical content (the only difference is the filename).
> If you edit one, edit the other in the same change. The first thing to do
> after any modification is run `pnpm sync-docs` (or manually copy the body)
> to verify the two files match. Tools that read this file: Claude Code (`CLAUDE.md`),
> Codex / Cursor / generic agents (`AGENTS.md`).

---

## 1. Mission & Hard Constraints

**Product.** MeshForge is a web app where a user types a natural-language prompt
and receives a downloadable 3D model. Under the hood: prompt → LLM (Claude / GPT
via paid API) → validated Blender Python script → headless Blender in a sandbox
→ exported `.glb` / `.fbx` / `.obj` / `.stl` → signed download URL.

**Constraints that drive every decision.**

- Solo developer, college-student budget. Total spend to first paying user must
  stay under **$50**. Monthly run-rate before revenue must stay under **$30**.
- US-first market. No GDPR data flows until we expand.
- Public-internet-facing, monetized → **defense in depth** is mandatory, but a
  $5k pentest is not in budget. Substitute with the security stack in §5.
- AI access via **paid APIs only** (Anthropic, OpenAI). Never via Claude Pro,
  ChatGPT Plus, or other consumer products — that violates ToS and there is no
  programmatic access anyway.
- AI-generated Python is **untrusted code**. It runs only in a sandbox with an
  allow-listed import set, no network, and a hard wall-clock timeout.

If a decision conflicts with these constraints, the constraint wins.

---

## 2. Architecture Overview

```
[Browser]  --HTTPS-->  [Cloudflare WAF + CDN]
                            |
                            v
            +------ Next.js App (Vercel) ------+
            |  UI, marketing, server actions   |
            |  Stripe checkout + webhooks      |
            +-------------+--------------------+
                          |
       +------------------+--------------------+
       v                  v                    v
  [Postgres+Redis]  [Stripe webhooks]   [Job queue (BullMQ)]
  (Neon + Upstash)                            |
                                              v
                              +---------------------------+
                              | Orchestrator              |
                              | - LLM call (Claude)       |
                              | - AST validate script     |
                              | - dispatch job to worker  |
                              +-------------+-------------+
                                            |
                                            v
                            +----------------------------+
                            | Blender worker(s)          |
                            | dev: local subprocess       |
                            | prod: Modal Labs container  |
                            | headless `blender --python` |
                            +-------------+--------------+
                                          |
                                          v
                                [Cloudflare R2 storage]
                                          |
                                          v
                            [Signed download URL to user]
```

For the MVP the orchestrator lives inside the Next.js app as a server action +
BullMQ job. Workers stay in a separate process so the sandbox boundary is real.

---

## 3. Tech Stack (fixed — do not swap without an ADR)

| Layer          | Choice                                            |
| -------------- | ------------------------------------------------- |
| Web framework  | Next.js 15 (App Router) + TypeScript strict       |
| UI             | Tailwind CSS + shadcn/ui + react-three-fiber      |
| Auth           | Clerk (free tier, 10k MAU)                        |
| Database       | Postgres on Neon (free tier, branching enabled)   |
| ORM            | Drizzle ORM                                       |
| Cache / queue  | Upstash Redis + BullMQ                            |
| Object storage | Cloudflare R2 (zero egress fees)                  |
| Payments       | Stripe Checkout + Billing + Customer Portal       |
| LLM            | Anthropic API (`claude-sonnet-4-6` default,       |
|                | `claude-opus-4-7` only for premium calls);        |
|                | OpenAI as fallback (`gpt-5`)                      |
| Worker runtime | Local Python subprocess for dev; Modal Labs prod  |
| Email          | Resend (free 3k/mo)                               |
| Errors         | Sentry (free)                                     |
| Analytics      | PostHog (free 1M events/mo)                       |
| CI             | GitHub Actions                                    |
| Secrets        | Vercel + Modal env vars; never in code            |

Anything not on this list requires an ADR in `docs/decisions/` and user
approval before adding.

---

## 4. Repository Layout

```
meshforge/
  apps/
    web/                Next.js 15 app, auth, billing, UI, API routes
    worker/             Python Blender worker (headless bpy runner)
  packages/
    shared/             Shared TS types + Zod schemas
    prompts/            LLM prompt templates + golden-output tests
  infra/
    docker/             Worker Dockerfile + base blender image
    modal/              Modal Labs deploy entrypoints
  docs/
    decisions/          ADRs (one .md per decision, dated)
    prompts/            Prompt engineering notes
    security/           Threat model, incident playbook
  scripts/              Dev helpers (sync-docs, db reset, etc.)
  .github/workflows/    CI pipelines
  CLAUDE.md
  AGENTS.md
  README.md
```

Monorepo via pnpm workspaces. Use Turborepo only when build times justify it.

---

## 5. Security Model (non-negotiable)

These rules cannot be relaxed without a written exception and user approval.

**Input boundary.**
- Every API route / server action validates input with Zod before any work.
- Prompts are passed through a moderation step (Anthropic content safety or
  OpenAI Moderation) before reaching the code-generation LLM.
- Rate limit at the edge (Cloudflare) AND per-user in app (Upstash Ratelimit).

**LLM-generated Python (the highest-risk surface).**
- Parse with `ast` before execution. Walk the tree and reject any of:
  - imports outside the allow-list `{bpy, bmesh, mathutils, math, random}`
  - calls to `eval`, `exec`, `compile`, `__import__`, `open`, `globals`,
    `locals`, `getattr`, `setattr` on dunders
  - attribute access on `os`, `sys`, `subprocess`, `socket`, `pathlib`
  - any string literal containing `..`, absolute paths, or URL schemes
- Execute only inside a container with: read-only rootfs, no network egress,
  dropped Linux capabilities, RLIMIT_AS memory cap, 90-second wall clock.
- All output goes to `/tmp/out` mounted as a fresh tmpfs per job.
- Container is destroyed after every job. No reuse.

**Secrets.**
- Never committed. Never logged. Never sent inside an LLM prompt.
- Loaded from environment only. `.env*` is in `.gitignore`.
- `gitleaks` runs in CI on every PR.

**Payments.**
- Stripe Checkout hosts the card form. We never see PAN/CVV.
- Stripe webhooks are verified with the signing secret and processed
  **idempotently** (use the event id as the dedup key in Postgres).
- Stripe Radar enabled, 3DS required on first charge.

**Auth.**
- Clerk handles password hashing, sessions, 2FA, social.
- 2FA mandatory on any account that has ever made a purchase.
- Session cookies are `Secure`, `HttpOnly`, `SameSite=Lax`.

**Network.**
- Cloudflare in front of everything. HSTS preload after stable.
- Strict CSP, no `unsafe-inline` outside of nonce-tagged Next.js hydration.
- All inter-service calls authenticated by short-lived signed tokens.

**Spend control (security-relevant).**
- Every LLM call writes a row to `llm_usage` with model, tokens, cost.
- Per-user daily cap; on breach, queue the job for tomorrow.
- Global daily cap (circuit breaker); on breach, return 503 and alert.

**Audit.**
- `audit_log` table records: auth events, payment events, admin actions,
  moderation rejections, sandbox violations. Append-only.

**Substitute for paid pentest (run before public launch).**
- `pnpm dlx semgrep --config p/owasp-top-ten` on every PR.
- `trivy` on every Docker image build.
- `dependabot` + `snyk` (free tier) for dependency CVEs.
- OWASP ZAP baseline scan against staging weekly.
- Manual `/security-review` pass with Claude on any change to auth, billing,
  sandbox, or prompt path.
- HackerOne / Intigriti public bug bounty opened once MRR > $1k.

---

## 6. Coding Conventions

- TypeScript: `"strict": true`. No `any`. No non-null `!` assertions outside
  tests. No `as` casts without a comment justifying the narrowing.
- Validate at the boundary with Zod. Trust internal data after that.
- Server components by default; mark with `"use client"` only when state,
  effects, or browser APIs are required.
- Tailwind for styling. shadcn/ui for primitives. No other UI libraries.
- Naming: `kebab-case` files, `PascalCase` components, `camelCase` functions,
  `SCREAMING_SNAKE` env vars.
- Tests live alongside source: `foo.ts` + `foo.test.ts`.
- Vitest for unit tests; Playwright for one happy-path e2e per critical flow
  (signup, checkout, generate, download).
- Lint/format: ESLint + Prettier. `lint-staged` + `husky` pre-commit hook.
- Commit style: Conventional Commits (`feat:`, `fix:`, `chore:`, `security:`,
  `docs:`). Security-relevant changes use `security:` and require review.

**Anti-patterns to reject in review.**
- Premature abstractions (DRY before 3 concrete callers).
- Wrapper utils around one-liners.
- Comments that describe what the code does instead of why.
- Error handling for impossible states.
- Backwards-compat shims for code that never shipped.
- New deps with < 10k weekly downloads or no release in > 12 months.

---

## 7. Implementation Phases

Work top to bottom. Do not start a phase until the previous one is demoable.

**Phase 0 — Foundations (week 1, ~$12).**
- Buy domain. Set up Cloudflare, GitHub repo, free-tier accounts on Vercel,
  Neon, Upstash, R2, Clerk, Stripe, Sentry, PostHog, Resend, Modal.
- Generate ToS / Privacy / AUP via Termly + Claude review.
- Scaffold the monorepo per §4. CI green. CLAUDE.md + AGENTS.md committed.

**Phase 1 — Vertical slice MVP (weeks 2–5, ~$30 LLM spend).**
- Clerk auth + sign-in / sign-up pages.
- Drizzle schema: `users`, `credits`, `generations`, `llm_usage`, `audit_log`.
- One prompt page with a textarea and a "Generate" button.
- Server action enqueues a BullMQ job.
- Orchestrator: moderation → Claude Sonnet with a tight Blender-Python prompt
  template → AST validate → local Python subprocess runs Blender.
- Export GLB only, upload to R2, return a signed URL.
- r3f viewer to display the result in-browser.
- Test with 20 prompts manually. Track failures in `docs/prompts/failures.md`.

**Phase 2 — Payments & polish (weeks 6–7, ~$0).**
- Stripe Checkout for one starter subscription + one credit pack SKU.
- Stripe Customer Portal for self-serve cancel / update card.
- Credits ledger: every successful generation debits 1 credit.
- All four export formats (GLB / FBX / OBJ / STL).
- Free-tier watermark baked into renders.
- Pricing page, dashboard, history view, email receipts via Resend.

**Phase 3 — Hardening (week 8, ~$0).**
- Rate limiting (edge + per-user).
- Per-user + global LLM spend caps with circuit breaker.
- Run Semgrep, Trivy, ZAP baseline. Fix every finding.
- Manual `/security-review` pass. Fix every finding.
- Threat model written into `docs/security/threat-model.md`.
- Soft launch to 20–50 users from a single niche community.

**Phase 4 — Public launch (week 9+).**
- Marketing site, demo videos, SEO landing pages per use case.
- ProductHunt / HN / niche subreddit launch.
- Affiliate program (Rewardful) when MRR ≥ $500.

**Phase 5 — Scale (when MRR ≥ $1k).**
- Migrate workers from local subprocess to Modal Labs containers.
- Multi-model routing (Sonnet default / Opus premium / GPT-5 fallback).
- Public bug bounty.
- Stripe Atlas LLC formation.
- Lawyer review of ToS + AUP.

**Phase 6 — Enterprise (when MRR ≥ $5k).**
- Paid third-party pentest.
- SOC2 Type I via Vanta/Drata if pursuing enterprise deals.
- API access tier for developers.

---

## 8. Cost Discipline

- Free tiers only until $200/mo MRR.
- Default LLM is `claude-sonnet-4-6`. `claude-opus-4-7` is gated behind a
  premium tier or an explicit "deep render" toggle that costs extra credits.
- Track every API call: `llm_usage(user_id, model, input_tokens, output_tokens,
  cost_usd, created_at)`. Surface the totals in the admin dashboard.
- Dev environment uses the smallest model that proves the path works
  (`claude-haiku-4-5` or `gpt-5-mini`) unless we are explicitly testing output
  quality.
- Daily Sentry alert if total spend exceeds $5/day in dev or breaches the
  configured cap in prod.

---

## 9. Rules of Engagement for AI Agents

These rules apply to any AI agent (Claude Code, Codex, Cursor, etc.)
contributing to this repository. Follow them without being reminded.

**Sync.**
- After any edit to `CLAUDE.md` or `AGENTS.md`, ensure both files are
  byte-identical except for the `# MeshForge — ...` filename in the title.
  Run `pnpm sync-docs` if the script is present.

**Scope.**
- Default to editing existing files. Create new files only when the structure
  in §4 demands it or the user asks.
- No new dependency without checking: weekly downloads ≥ 10k, last publish
  within 12 months, OSI-approved license. Ask the user before adding.
- No new abstraction until at least three concrete call sites exist.
- No backwards-compatibility shims for code that has not shipped.

**Safety.**
- Never run `git push --force`, `git reset --hard`, `rm -rf` on tracked paths,
  or any other destructive command without explicit user confirmation in the
  current turn.
- Never skip pre-commit hooks (`--no-verify`) or signing.
- Never commit `.env*`, `*.pem`, `*.key`, or anything matching `*secret*`.
- Security-critical code (auth, billing, sandbox, AST validator, prompt path)
  requires tests in the same PR. Do not merge without them.

**Communication.**
- Before any non-trivial change, state in one sentence what you are about to
  do. Update the user when direction changes or a blocker appears.
- For UI changes: start the dev server and exercise the feature in a browser
  before claiming the task is done. If you cannot, say so explicitly.
- Cite files using `path:line` so the user can jump straight to them.

**Decisions.**
- Anything that locks in tech-stack, schema, or money-handling behaviour goes
  into `docs/decisions/NNNN-title.md` as an ADR (date, context, decision,
  consequences). Reference the ADR id in the commit message.

**Memory & context.**
- This file plus `docs/decisions/` are the source of truth. If they conflict
  with a memory or a recollection, this file wins. Update memories to match.

---

## 10. Glossary

- **MCP** — Model Context Protocol; Anthropic's standard for tool use. The
  product is *inspired by* blender-mcp but does not ship the local MCP server.
- **AST** — Abstract Syntax Tree; how we statically validate LLM-generated
  Python before execution.
- **ADR** — Architecture Decision Record; one Markdown file per locked
  decision under `docs/decisions/`.
- **R2** — Cloudflare's S3-compatible object store with zero egress fees.
- **MRR** — Monthly Recurring Revenue.
- **AUP** — Acceptable Use Policy.

## 11. External References

- Anthropic API docs: https://docs.anthropic.com/
- OpenAI API docs: https://platform.openai.com/docs
- Blender Python API: https://docs.blender.org/api/current/
- Stripe Billing: https://stripe.com/docs/billing
- Modal Labs: https://modal.com/docs
- Clerk: https://clerk.com/docs
- Drizzle ORM: https://orm.drizzle.team
- Cloudflare R2: https://developers.cloudflare.com/r2/
