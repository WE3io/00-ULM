# Zero Zero (00-ULM) — Comprehensive Review

> **Status: IN PROGRESS (loop-driven).** This report is built iteratively. Source of
> truth for what's done and what's next is [`PROGRESS.md`](./PROGRESS.md). Do not treat
> any section marked 🟡/🔴 in PROGRESS as final.

**Repo:** WE3io/00-ULM · **Branch:** `claude/forked-project-review-9naqz7` · **Started:** 2026-06-10

---

## Executive summary

_Written last, once Parts A–C are verified. A founder/investor should be able to make a
build / kill / pivot decision from this section alone._

> _TODO — synthesise after Parts A–C reach `verified`._

---

# Part A — Stack & architecture (concise, architectural)

> Focus: the *shape* of the system. DD only where something is clearly problematic.

## A1. System architecture (context + containers)

Zero Zero is a **Next.js 16 App-Router monolith on Vercel** plus one external **Oracle
VPS ("Hermes")** that acts only as a clock. The team's own metaphor is an accurate
container model: **Brain** = Gemini (reasoning/copy), **Stomach** = Firecrawl (scrape UK
gov/energy pages), **Memory** = Neon Postgres, **Nervous system** = Next.js API routes,
**Hermes** = a cron that simply pokes `/api/cron/zone-research`; it runs no AI itself
(`docs/FULL-APP-SPEC.md:15-25`).

**Request/runtime shape:**
- **Client** is a thin React tree under one root layout (`app/layout.tsx:126-129`) that
  mounts a single global state context, `AppProvider`. Nine pages: intro → profile →
  profile/summary → zone (the core "bento wall") → zai/likes/settings/admin
  (per architecture map).
- **State** lives in `AppContext` (`app/context/AppContext.tsx`): profile, likedCards,
  userId, hero £/kg totals, per-journey answers, locality, and Solo-Focus selection —
  hydrated from `localStorage` then reconciled with Neon via `/api/answers` and `/api/likes`.
- **Server** is 50 API routes that follow one pipeline: *scrape (Firecrawl) → structure
  (Gemini) → persist (`research_results`) → JSON to browser*. Reads compose a view model
  (`buildZoneViewModel`); writes flow through `/api/answers`.
- **Data** is Neon (London) with a singleton pool (`lib/db.ts:27-33`).

**Architectural assessment.** For a solo/small-team product this is a *coherent, honestly
layered* design — the brain/stomach/memory split maps cleanly onto agents → intelligence →
db, and the "Hermes only wakes the app" decision keeps AI cost and secrets inside one
runtime. The main structural risks are concentration (one god-context on the client, one
68-file `zone` module on the server) and the absence of a shared observability seam (A4).

## A2. Module boundaries & coupling

`lib/` has 28 sub-domains with mostly clean, non-overlapping responsibilities. The layering
is real: **`agents/`** (orchestration: researchAgent, scraperAgent, sentinel, contentArchitect,
hermes-memory) call into **`intelligence/`** (reusable AI/API: aiGateway, bucketFailover,
geminiModels, llmRateLimit, scrapeBoundaries) and **`brains/`** (pure calculators:
buildUserImpact, liveUnitRates, summaryLogic). Suspected duplicates were checked and
cleared: `lib/brain` (one orchestrator) vs `lib/brains` (calculators) is intentional, and
`agents` vs `intelligence` is a clean caller/library split — **no dead code surfaced**.

**Coupling hotspots (the architectural debt to watch):**
- **`lib/zone/` is a 68-file module** — by far the largest unit, mixing view-model
  construction, grid ordering, session memory, Solo-Focus navigation, injections, copy
  voice, guest sessions, and "Gary mode" demo logic. This is the system's complexity
  centre of gravity and the most likely source of regressions (the recent commit log is
  dominated by `fix(zone): …`). Candidate for decomposition into render / state /
  content sub-modules.
- **`AppContext` is a god-context** (~379 lines) holding profile, likes, answers, totals,
  locality and modal state with cross-effects between them — fine today, but it concentrates
  client-state risk and makes the cascade ordering (the "Director's Order") fragile.

## A3. AI / data pipeline architecture

This is the most sophisticated part of the codebase and is genuinely well-factored for
**cost control**, which is the stated product constraint ("use less, more").

- **Provider failover** (`lib/intelligence/bucketFailover.ts:232-303`): a single
  `generateWithBucketFailover()` tries **Gemini → Groq → Mistral → OpenRouter** in order,
  gated by `MODEL_STRATEGY=bucket_failover`, with **per-provider in-memory 429 cooldown**
  (`llmRateLimit.ts:24-51`). This is the right shape for a zero/low-cost profile.
- **Three-way gateway cascade** (`aiGateway.ts:125-147`): bucket failover → Vercel AI
  Gateway → direct Gemini, with a `getGatewayHealthSnapshot()` for last model/error/tag.
- **Model tiers** (`geminiModels.ts`): `zone` / `article` / `chat`, defaulting to
  `gemini-2.0-flash-lite` (free tier) or `gemini-2.5-flash`, temp 0.2 for precision.
- **Scrape → persist** (`/api/cron/zone-research`): `CRON_SECRET`-bearer cron fetches users
  with a postcode, runs `runZeroResearchWithProfile()` (Firecrawl seed URLs filtered by a
  `topicShield`, then Gemini extraction into `research_results`). Modes: default scrape,
  `?repair=1` (mechanical Ofgem/BUS backfill, no LLM), `?deep=1` (repair + per-row Gemini).
- **"Mechanical truth"** is enforced in code, not just docs: honest empty-state baselines
  (`lib/scraper/uk2026Defaults`) and `COMPUTING` strips when Neon has no row — a real
  integrity property (and exactly the invariant that *should* have an automated eval — A5#1).
- **Cross-journey memory** ("Hermes memory", `lib/agents/hermes-memory.ts:64-100`) derives
  signals (e.g. home-solar → travel/money tips) into `users.user_genome`.

**Architectural gaps vs the 2026 production-LLM benchmark:** (1) **no evals in CI** for the
pipeline or the mechanical-truth invariant; (2) **no request tracing** — observability is
conditional `console.log` plus one health snapshot, so trajectory/cost debugging in prod is
hard; (3) failover state (cooldowns) is in-memory and per-instance, so on Vercel's many
lambdas the "cooldown" is mostly advisory (A4). None are structural flaws — they're the
maturity gap between a working pipeline and a *governable* one.

## A4. Scaling, state & failure modes

The deployment model is serverless (Vercel lambdas) + external cron, which interacts badly
with several **in-memory** state stores:

- **Login/abuse rate limiting is in-memory** (`lib/rateLimit.ts`): 8/IP and 5-failed/email
  per 15 min — but per lambda instance. Under real traffic, attempts spread across instances,
  so the effective brute-force ceiling is far higher than the numbers imply. *Move to Neon or
  a KV/Redis store for a durable limit.* (Severity: medium — it's the one with a security edge.)
- **LLM provider cooldown is in-memory** (`llmRateLimit.ts`) — same caveat; a 429 on one
  instance doesn't inform the others, so the failover may keep hammering a throttled provider.
- **DB access** uses a `globalThis` singleton pool (`lib/db.ts:27-33`) and the Neon serverless
  driver — appropriate for lambdas; connection-storm risk is low, but confirm pool sizing
  under concurrent cron + user load.
- **No centralised error handling / structured logging / alerting.** Errors are logged inline;
  there's no error boundary strategy described and no Sentry-class capture. For a product that
  spends money per request (Gemini/Firecrawl), the lack of cost/error telemetry is the biggest
  operability gap.
- **Failure containment is otherwise good**: the provider cascade degrades gracefully and the
  mechanical-truth path means a total AI outage yields honest `COMPUTING`/empty states rather
  than fabricated numbers — a deliberate and commendable failure mode.

## A5. Due-diligence by exception (clearly problematic only)

Only items that are *clearly* problematic are listed; each is verified, not inferred.

| # | Issue | Severity | Evidence | 2026 benchmark missed |
|---|-------|----------|----------|------------------------|
| 1 | **Almost no automated tests** — 1 Playwright e2e spec + 1 fixture for ~55k LOC; e2e not wired into CI (CI runs typecheck/lint/verify only). | **High** | `e2e/zone-funky-stress.spec.ts`, `__tests__/fixtures/energyMockData.ts`; `.github/workflows/ci.yml` | Coverage <40% is a DD red flag; the AI "mechanical truth" invariant has no automated guard. |
| 2 | **Input validation is ad-hoc** — only 1 of 50 API routes imports Zod; 26 routes call `request.json()` with hand-rolled `typeof` checks or none. | **Medium** | `grep zod app/api` → 1 hit; 26 routes parse JSON without a schema (e.g. `app/api/answers/route.ts`, `app/api/zai/route.ts`, `app/api/marketing-email/route.ts`). | "Validate every Server Action / route input with Zod." Inconsistent validation is an injection/abuse surface and a maintenance hazard. |
| 3 | **`typescript.ignoreBuildErrors: true`** in the Next config — type errors don't fail `next build` itself. | **Medium** | `next.config.js:9` | A separate script gate (`scripts/vercel-build-gate.mjs`) does run `tsc`, so this is mitigated *if* that gate stays in the deploy path — but the in-config escape hatch is fragile and easy to regress. **Verify the gate is the actual Vercel build command.** |
| 4 | **Bespoke tooling sprawl** — 61 scripts and many `dev`/`build` variants with manual manifest fixups and forced polling (`WATCHPACK_POLLING`, `ensure-dev-manifests.js`, `build-with-manifest-fix.js`). | **Medium (maintainability)** | `package.json` scripts 6–88; `scripts/` (61 files) | Signals fighting the framework. Quantify how much disappears under the Next 16 Turbopack default (see A-perf). High bus-factor / onboarding cost. |
| 5 | **Legacy AI SDK** — `@google/generative-ai@0.21` is the deprecated Google SDK (superseded by `@google/genai`). | **Low** | `package.json:91` | Migrate before it loses fixes; low urgency since a provider-abstraction layer exists (confirm in A3). |

**Verified as NOT problematic (avoids false findings):**
- **Dependencies are clean** — `npm audit --omit=dev` → **0 vulnerabilities**.
- **SSRF surface is low** — `scrape-sync` accepts only `postcode`/`category`/`answer`/`question_id` that parameterise an internal seed-URL registry; no user-controlled `fetch()` target found (`app/api/scrape-sync/route.ts`).
- **Auth basics are sound** — bcrypt compare, generic error messages, IP+email login rate limiting, HMAC-signed HTTP-only cookies (`app/api/auth/login/route.ts`, `lib/rateLimit`, `lib/auth`). *Caveat to confirm in A4:* the rate limiter is in-memory, so it resets on serverless cold start / per-instance.
- **React 18.3 is valid here** — Next 16.2.6 still peer-accepts React 18.2+ (`package-lock.json`); React 19 is an opportunity, not a defect.

---

# Part B — Product analysis (deep, Double Diamond)

## B1. Discover — the problem space
> _Framework: Jobs-To-Be-Done + user/context map (UK cost-of-living / energy). Who is this
> for, what job are they hiring it for, what is the real pain and its triggers?_
> _TODO_

## B2. Define — the opportunity
> _Framework: competitive landscape + Opportunity Solution Tree anchored to a North Star
> metric + AARRR funnel. Where is the value and against what alternatives?_
> _TODO_

## B3. Develop — evaluate the current product
> _Framework: UX heuristic eval + honesty-of-data review + the journey/Solo-Focus loop
> mapped onto the opportunity tree (what's covered, what's missing, what's friction)._
> _TODO_

## B4. Deliver — prioritise & sequence
> _Framework: RICE prioritisation + sequenced roadmap._
> _TODO_

---

# Part C — Recommendations & roadmap

> _Synthesised, sequenced. Strategic narrative tying stack + product together._
> _TODO_

---

## Appendix — evidence log & sources
> _Running list mirrored from PROGRESS.md on finalisation._
