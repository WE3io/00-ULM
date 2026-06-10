# Zero Zero (00-ULM) — Comprehensive Review

> **Status: COMPLETE (v1) — all sections verified.** Built iteratively (loop-driven); working
> notes and the full evidence/citation log are in [`PROGRESS.md`](./PROGRESS.md); scope and 2026
> benchmarks in [`REVIEW-PLAN.md`](./REVIEW-PLAN.md).

**Repo:** WE3io/00-ULM · **Branch:** `claude/forked-project-review-9naqz7` · **Date:** 2026-06-10

**How to read this:** Executive summary first (decision-grade, non-technical). Part A is the
**stack/architecture** read (~30%, architectural + due-diligence-by-exception). Part B is the
**deep product analysis** (~70%) using a Double Diamond — Discover (JTBD) → Define (opportunity)
→ Develop (evaluation) → Deliver (prioritised roadmap). **Part B′** integrates domain-expert
operator feedback (the project-workspace reframe §B5 + a UX findings register §B6). Part C
synthesises everything into a call.

**v1.1 (2026-06-10):** integrated operator review — added §B5/§B6; revised the North Star, the
executive thesis, and the roadmap (new Phase 1.5 + project-workspace Phase 2).

---

## Executive summary

**Zero Zero is a well-built, unusually honest UK home-savings app that is aimed slightly off
the job its market hires for, and has no way to make money yet. The recommendation is to
pivot-to-focus, not kill.**

**What it is.** A mobile-first, postcode-driven Next.js app that gives UK households localised
advice to cut energy cost and carbon across 13 domains, powered by a cost-controlled AI +
web-scraping pipeline with a deliberate "never show a fake number" honesty rule.

**The stack (architectural read).** Coherent and honestly layered for a small team — a Next.js
monolith plus an external cron ("Hermes"), with a clean agents → intelligence → calculators
split and a genuinely well-designed multi-provider AI failover for low cost. The architectural
debt is concentration (`AppContext` god-context; a 68-file `zone` module) and **operability**:
state like rate-limits and AI cooldowns is in-memory per-lambda, and there is **no tracing,
error capture, or funnel telemetry**. Dependencies are clean (0 vulns) and SSRF risk is low; the
clear DD red flag is **almost no automated tests (~2 files for 55k LOC)** with `ignoreBuildErrors`
on and Zod validation on only 1 of 50 routes.

**The product (deeper read).** UK consumers prioritise **cutting cost over climate by ~6:1**, yet
the product is architected **carbon-first** (a "carbon" journey, an "auditor" persona, £ shown
co-equal with kg). It's also **inform-first** in a market where every viable competitor (Nous,
Snugg, grant checkers, MSE) captures value at the point of **transaction** — so it has **no
revenue model**. Its real, defensible whitespace is **trust + locality + breadth**: nobody else
combines honest, postcode-grounded advice across all the levers, and the macro timing (£15bn Warm
Homes Plan, price-cap volatility) is a strong tailwind.

**The sharper thesis (from operator review, §B5).** Acting on this advice is **weeks-to-months of
real work**, so the durable value is **tracking that long-running work** — Zero Zero should become
a **functional home-efficiency *project workspace***, not a one-shot advice poster. That single
reframe resolves the retention *and* revenue gaps more durably than a one-off referral and turns
returning users into a data moat.

**The moves that matter (in order):**
1. **Instrument the funnel and add an eval that guards the honesty moat** — nothing else can be
   proven or safely changed without this (also fixes durable rate-limiting + observability).
2. **Re-point the frame from carbon to money** — £-first hero/copy; the single highest-confidence
   change.
3. **Show one "biggest win for you," fast** — replace breadth-overload with a prioritised next
   action and a quicker time-to-value.
4. **Make it a tool, not a poster** (§B6) — persistent nav/breadcrumbs and a calmer "working"
   visual mode for surfaces users dwell in; plus quick wins (buy-link logos + page summaries, the
   liked-cards bug, copy nits).
5. **Build the project workspace** (§B5) — projects that persist tasks, advisor threads, and data;
   validity-decay + progress; **bill ingestion (PDF/photo → LLM, GDPR-handled)**; grants
   facilitation as the first project type and the revenue model.
6. **Pay down the test/maintainability debt** continuously.

**Bottom line.** Real asset, real moat, favourable timing — held back by a carbon-first,
inform-only, *poster-shaped* stance on an unmeasured, lightly-tested base. **Pivot to a money-first,
action-first, persistent project workspace — built like a tool, not a presentation — and the
foundation is strong enough to build on.**

*(Detail and evidence: Part A — stack §A1–A5; Part B — product §B1–B4; synthesis §C. Working
notes and citations in [`PROGRESS.md`](./PROGRESS.md).)*

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

### The core job (JTBD)
> *When* my energy bills keep rising and I can't tell which money-saving advice is real,
> *I want to* know the specific actions that will actually cut **my** bills for **my** home and
> area, *so I can* feel in control and act — without wading through tariffs, grants and greenwash.

The functional job is **"cut my cost of living / bills."** The emotional job is **"feel in
control and not ripped off."** The social job ("be a responsible, savvy householder") is real
but secondary. This ordering is the single most important strategic fact about the space.

### The evidence — and the central tension
UK consumers rank **reducing the cost of living as the #1 priority (28% top, 68% top-three)**
and **reducing energy bills in the top three for 43%**, while **"tackling climate change" is
top-three for only 11% and "net zero" for 4%** ([House of Commons Library](https://commonslibrary.parliament.uk/research-briefings/cbp-10505/),
[JRF, 2026](https://www.jrf.org.uk/cost-of-living/addressing-the-2026-energy-price-crisis)).
Money beats carbon by roughly **6:1** as a motivator.

**Zero Zero is architected carbon-first.** Its ground truth is the *12,000 kWh ≈ 1 tonne CO₂e*
baseline, its persona is a lowercase **"auditor,"** and "carbon" is a first-class journey
(`docs/HANDBOOK.md`, `.agents/AGENTS.md:30-34`). The honesty engine ("mechanical truth," no
fake numbers) is genuinely differentiated — but if the *headline payoff* a user feels is
carbon/audit rather than **£ in my pocket**, the product is speaking its second language to a
money-first audience. **This job-framing question is the thread that runs through B2–B4.**

### Triggers (when the job arises)
- **Price-cap shocks** — Apr 2026 cap £1,641, then **+13% to £1,862 in July 2026**; volatility
  is itself the recurring trigger ([Solar4Good, 2026](https://solar4good.co.uk/blogs/average-energy-bills-uk-2026/)).
- **Bill arrival / annual statement**, a cold snap, moving home, or a grant headline (BUS,
  ECO4, Warm Home Discount £150).

### User segments (who has the job, ranked by £-leverage the product can serve)
1. **Cost-pressured homeowners** — the sweet spot. They can act on the highest-£ levers Zero
   Zero covers (insulation, heat pumps + **grants**, solar, tariff). Postcode → council →
   grant eligibility is exactly their pain.
2. **Engaged switchers** (already on Octopus/smart tariffs) — want optimisation and the next
   marginal action; lower upside, higher savviness.
3. **Renters / flat-dwellers** — limited fabric control; only the tariff/behaviour/tech/water
   levers apply. Risk: the 13-journey wall shows them mostly actions they *can't* take.
4. **Fuel-poor households** — highest need, lowest ability to pay/act; a moral target but a
   hard commercial one (they need grants and protection, not a savings dashboard).

### Pains in the space, and how well the concept targets them
| Pain | Zero Zero's answer | Fit |
|------|--------------------|-----|
| Advice is **fragmented** (tariffs *vs* grants *vs* behaviour, across many sites) | One postcode-driven wall spanning 13 domains | **Strong** — aggregation is a real wedge |
| **Distrust** of savings claims / greenwash | "Mechanical truth": honest `COMPUTING`/empty over fake £ | **Strong & rare** — a credible trust moat |
| Advice is **generic**, ignores my home/area | Postcode → council, grid intensity, local grants | **Strong** — the core differentiator |
| I don't know **what to do first** | Journey cards + Solo-Focus recommendation + Rock habits | Medium — depends on prioritisation quality (B3) |
| I want to **actually do it** (switch, apply, install) | CTA links to trusted sources | **Weak/unverified** — informs more than it transacts (see B2/B3) |

### Provisional B1 conclusion
The problem is real, large, recurring, and under-served on **trust + locality** — Zero Zero's
two genuine strengths. The **strategic risk is motivational alignment**: a money-first audience
met with a carbon-auditor frame, and an *inform*-first product in a space where the value often
lands at the point of *action* (switch/apply/install). Both are testable and fixable; they
define the opportunity in B2.

## B2. Define — the opportunity

### Competitive landscape
| Player | What it does | Frame | Value capture |
|--------|--------------|-------|---------------|
| **Nous.co** | AI assistant reads household bills, says "you can save £126", **switches in a couple of clicks** | Money-first | Switching commission |
| **Snugg** | Postcode → EPC/EST property model → retrofit plan → **grants → installer** | Carbon/retrofit | Installer lead-gen |
| **Grant checkers** (Honely, GreatBritishEnergy, RetrofitPlanner, EnergySavingGenie, E.ON…) | Free 60-sec grant eligibility check | Money-first | Installer/ECO lead-gen |
| **MoneySavingExpert** | Editorial authority, grants guides, Martin Lewis trust | Money-first | Affiliate/editorial |
| **Octopus / Loop** | Smart tariffs + hardware energy monitors (monitors cut use 10–15%) | Money + carbon | Supplier / hardware |
| **Generic AI** (ChatGPT/SearchGPT) | "Cut my energy use" plans on demand | Neutral | None (free) |

Sources: [Nous](https://www.nous.co/blog/nous-launches-new-ai-assistant-to-make-sense-of-household-bills) ·
[Snugg](https://www.snugg.com/) · [GreatBritishEnergy checker](https://greatbritishenergy.com/eligibility-checker/) ·
[MSE grants](https://www.moneysavingexpert.com/family/housing-and-energy-grants/).

**Market tailwind:** the **Warm Homes Plan (Jan 2026, £15bn, 5M homes)** plus ECO4 (to Dec 2026)
and BUS — and **0% loans for any household from April 2027** — make grant/scheme navigation a
fast-growing, high-intent need ([MSE, Jan 2026](https://www.moneysavingexpert.com/news/2026/01/warm-homes-plan-martin/)).

### Positioning — where Zero Zero sits
Plot the market on two axes — **Money-first ↔ Carbon-first** and **Inform ↔ Transact**:

- The **money + transact** quadrant is crowded and *funded* (Nous, grant checkers, Snugg).
- Zero Zero sits in **carbon-first + inform** — the quadrant with the **weakest commercial
  pull**, and there it competes with *free generic AI*.

**But its genuine whitespace is orthogonal to both axes: postcode-grounded, trust-first
("mechanical truth"), cross-domain breadth.** No competitor combines honest, locality-grounded
advice across all 13 domains; the checkers are single-purpose lead-gen and MSE is generic
editorial. That breadth + honesty is the defensible wedge — *if* it is re-pointed at the money
job and given a path to action.

### Opportunity Solution Tree
**Desired outcome (North Star candidate): _verified £/year of savings a household actually
actions_** — a value-realised metric, not vanity engagement, consistent with the product's
honesty ethos.

| Unmet need (from B1) | Covered today? | Opportunity |
|----------------------|----------------|-------------|
| "I don't trust savings numbers" | ✅ mechanical truth | **Lead with it** as the brand promise |
| "Advice ignores my home/area" | ✅ postcode grounding | Extend to EPC/Home-Analytics depth (Snugg parity) |
| "Show me money, not carbon" | ⚠️ carbon-first frame | **Re-point hero/copy to £; carbon as supporting** |
| "What do I do *first*?" | ⚠️ partial (13-card wall) | **Single prioritised "biggest win for you" action** |
| "Help me *actually do it*" | ❌ inform-only | **Facilitate the highest-£ action** (grants/switch) → also the revenue path |
| "Why come back?" | ⚠️ habits/Hermes | Trigger-based re-engagement on cap/grant changes |

### North Star + AARRR funnel (current-state read)
- **North Star:** *£ saved actioned per activated household / quarter.* Counter-metric: keep the
  "mechanical truth" honesty rate at 100% (no inflated £). **→ Revised in §B5** following operator
  review to *£ realised across active **projects*** (lifecycle), reflecting that the work is
  long-running, not a single action.
- **Acquisition** — postcode entry is low-friction, but **no obvious top-of-funnel hook**; the
  high-intent SEO surface (grant eligibility) is owned by competitors. *Gap.*
- **Activation** — the "aha" should be a believable, **£-led** first personalised result; today
  the cascade is **carbon/auditor-framed and motion-heavy** (B3), risking a diluted money aha.
- **Retention** — strong *natural* triggers exist (cap volatility, Warm Homes Plan changes) and
  some mechanics (Rock habits 2/day, Hermes cross-journey memory) — but retention needs a
  *reason to return* (new savings surfaced), which depends on fresh, money-relevant content.
- **Referral** — none evident. *Gap.*
- **Revenue** — **none evident.** Every viable competitor captures value at the point of
  transaction; an inform-only product has no model. **This is the defining commercial question.**

### Provisional B2 conclusion
The opportunity is real and timely (Warm Homes Plan tailwind) and Zero Zero owns a rare
**trust + locality + breadth** position. To convert that into a business it must (1) **re-point
the frame from carbon to money**, (2) **collapse the 13-card wall into a single prioritised
"biggest win,"** and (3) **add one path to action** on a high-£ lever (grants facilitation is the
natural first, given the tailwind) — which simultaneously creates the **missing revenue model**.
These become the prioritised bets in B4.

## B3. Develop — evaluate the current product

The build quality is high and the craft is real. The issues below are almost all about
**alignment to the money-first job**, not execution polish.

### What's genuinely strong (keep and lead with)
- **A rare, disciplined honesty system.** Empty states say `COMPUTING — HOME` rather than
  faking a number (`lib/zone/mechanicalTruth.ts:44`); a *category contract* forces wrong-journey
  copy back to "Computing…" (`USER-FLOW…md:80`); banned-jargon/AI-filler filters strip
  greenwash and "as an ai" padding (`lib/zone/warmAuditorCopy.ts:10-21`, `zoneVoice.ts`); Zai
  refuses to invent figures and will say *"i don't have enough information to be confident"*
  (`lib/brains/zai/boundaries.ts`). This is the **trust moat from B1** made real in code.
- **Low-friction, well-toned onboarding** — 9 single-question screens, sensible fields, good
  mobile keyboard handling (`app/profile/ProfilePageClient.tsx:39-96`).
- **Restrained, accessible motion** — linear tweens, and `prefers-reduced-motion` is honoured
  via `useHydrationSafeReducedMotion()` across the animated components (30 files use
  framer-motion; reduced-motion collapses to opacity-only). Good WCAG 2.2 instinct.
- **Postcode grounding and a read-only "prove the maths" auditor (Zai)** — differentiated and
  on-brand.

### Where the product fights the job (heuristic + JTBD eval)
| # | Finding | Evidence | Why it matters (vs the money job) |
|---|---------|----------|-----------------------------------|
| 1 | **Heavy question load before value.** 9 onboarding Qs, then a question surfaced per journey in Solo Focus (1 of a 3-per-journey bank) **plus** *loop*-takeover beats ("rail instead of flying?") — easily 20+ prompts to fully populate the board. | `app/profile/ProfilePageClient.tsx:39-96`, `lib/journeys.ts` (3/journey bank, 1 shown), `lib/zone/loopQuestions.ts:28-209` | Rivals deliver a number "in seconds" (Nous) / "60-second check" (grant tools). Front-loading profiling is an **activation tax** on a user who wants a fast £ answer. |
| 2 | **Carbon is co-equal with money everywhere**, not subordinate. Hero shows £ **and** kg together; "carbon" is its own journey; several loop questions are carbon-led (plant-based meals, offsets). | Zone hero (£+kg stamps), `lib/journeys.ts` carbon journey, `loopQuestions.ts` | Re-states the **B1 6:1 money-vs-carbon mismatch** at the surface the user actually sees. The money "aha" is diluted. |
| 3 | **13-lane wall (≤48 cells) + discovery tips + Rock rail = breadth over focus.** No single "biggest win for you." | `app/zone/page.tsx`, `lib/zone/gridOrder.ts` | Directly answers the wrong question. The user asks *"what do I do first?"*; the wall answers *"here are 13 areas."* For renters most lanes are inert. |
| 4 | **Action is a link, not a transaction.** The Solo-Focus "BUY/CLAIM" opens an offer/source URL. | Solo-Focus action trinity (per UX walkthrough) | Confirms the **B2 transact/revenue gap** — value (and monetisation) leaks to gov.uk / installers at the exact moment of intent. |
| 5 | **Brand motion precedes value on the activation path.** First run plays glitch → word-ticker → grid crystallize → hero ping before the £ lands; the "Director's Order" *freezes* this sequence as a contract. | `.agents/AGENTS.md:5-13`, `lib/motion-family.ts`, `lib/animations.ts` | Adds perceived latency to the one moment that decides activation. Restrained ≠ free; on mobile field conditions it competes with LCP/INP. |
| 6 | **Hallucination surface inside the narrative.** Stamped £/kg are grounded, but the 3-paragraph Gemini prose interleaves *specific* claims ("45mm of loose batts", "£19–26k range") around them. | sample architect prose; `lib/agents/contentArchitect.ts` | The honesty moat protects the *stamps*, not the *story*. One invented detail in the prose erodes the very trust that is the differentiator — and there's **no eval guarding it** (ties to A5#1). |
| 7 | **The funnel isn't measured.** `/api/analytics` is generic fire-and-forget page/event capture into `analytics_events`; there's no instrumentation for activation, £-actioned, or retention cohorts. | `app/api/analytics/route.ts` | The North Star (B2) and every B4 experiment are **currently unmeasurable** — the first thing to fix or no bet can be evaluated. |

### Mapping onto the B2 opportunity tree
- **Covered well:** trust/honesty, locality grounding, "prove the maths."
- **Partial:** prioritisation (breadth instead of a single biggest win), retention (mechanics
  exist but no money-fresh reason to return, and it's unmeasured).
- **Missing:** money-first framing, a path to *action*, referral, revenue, and funnel telemetry.

### Provisional B3 conclusion
This is a **well-built product pointed slightly off-target**. The engineering and copy craft are
assets; the honesty system is a genuine moat. The gap is strategic and consistent across B1–B3:
**re-point from carbon-audit-breadth to money-action-focus, protect the trust moat with an eval,
and instrument the funnel** so the re-pointing can be proven. These convert directly into B4.

## B4. Deliver — prioritise & sequence

RICE = **(Reach × Impact × Confidence) ÷ Effort**. Reach is relative (1–10), Impact {0.25,
0.5, 1, 2, 3}, Confidence {0.5, 0.8, 0.9}, Effort in person-weeks. Scores are indicative, for
*ordering* not budgeting.

| Bet (source) | R | I | C | E (wks) | RICE | 
|--------------|---|---|---|---------|------|
| **Re-point frame carbon → money** (B1/B2/B3-2): hero leads with £, carbon as support; goal/copy money-first | 10 | 2 | 0.9 | 2 | **9.0** |
| **Instrument the funnel** (B3-7): activation, £-actioned, retention cohorts — the meta-enabler | 8 | 2 | 0.9 | 2 | **7.2** |
| **Durable rate limiting** (A4): move in-memory limits → Neon/KV | 7 | 1 | 0.9 | 1 | **6.3** |
| **Value-before-brand on first run** (B3-5): show the £ first, motion as reward | 10 | 1 | 0.6 | 1 | **6.0** |
| **Surface one "biggest win for you"** (B2/B3-3) above the 13-lane wall | 10 | 2 | 0.8 | 3 | **5.3** |
| **Cut time-to-value** (B3-1): postcode + 2–3 Qs → believable £ fast; profile progressively | 10 | 2 | 0.8 | 3 | **5.3** |
| **Trust-moat eval in CI** (A5-1/B3-6): guard mechanical-truth + prose grounding | 8 | 2 | 0.9 | 3 | **4.8** |
| **Observability/error+cost capture** (A4) | 8 | 1 | 0.8 | 2 | **3.2** |
| **Smoke tests on critical paths** (A5-1): auth, /api/answers, zone VM | 8 | 2 | 0.8 | 6 | **2.1** |
| **Path to action + revenue: grants facilitation** (B2/B3-4) — Warm Homes Plan tailwind | 6 | 3 | 0.6 | 8 | **1.35** |
| **Zod validation pass** across routes (A5-2) | 7 | 0.5 | 0.9 | 3 | **1.05** |
| **Decompose `lib/zone` + `AppContext`; drop `ignoreBuildErrors`; trim script sprawl** (A2/A5) | 5 | 1 | 0.7 | 8 | **0.44** |

**Critical caveat — RICE measures efficiency, not necessity.** The **revenue/action bet scores
low** (high effort, lower confidence) yet is **existential**: without a path to transaction there
is no business model (B2). Treat it as a *must-sequence strategic bet*, not a backlog item RICE
can defer away. Likewise the decomposition work is low-RICE but is the tax that keeps every other
bet cheap over time.

### Sequenced roadmap
- **Phase 0 — Make bets provable & safe (weeks 1–3).** Funnel instrumentation; trust-moat eval;
  durable rate limiting; basic error/cost observability. *Nothing below can be evaluated without
  Phase 0.*
- **Phase 1 — Re-point to the job (weeks 2–6, overlapping).** Money-first reframe; value-before-
  brand on first run; surface the single biggest win; cut time-to-value via progressive profiling.
  Measure activation lift against Phase 0 baseline.
- **Phase 1.5 — Make it a tool, not a poster (weeks 4–8, from operator review §B6).** Persistent
  global **navigation + breadcrumbs** (#1); a lower-intensity **"working/dashboard" visual mode**
  for dwell surfaces (#2); and the quick wins — buy-link logos + scraped page summary (#6), fix
  the **liked-cards bug** (#7), audit-trail tooltip (#9), employment copy (#10). Low-cost,
  high-trust, and prerequisites for time-on-task.
- **Phase 2 — Build the project workspace (the core bet, weeks 6–16; §B5).** A `projects` model
  where decisions become tracked, long-running projects; **Zai threads persist** into them (#3);
  **data-validity decay + refresh + progress infographic** (#4); **bill ingestion** via PDF/photo
  → LLM with **GDPR handled properly** (#5). **Grants facilitation becomes the first project
  type** inside the workspace (Warm Homes tailwind), establishing the revenue model. A/B against
  inform-only. *This supersedes the original standalone "grants referral" framing.*
- **Phase 3 — Pay down foundation (continuous).** Smoke/integration test coverage; Zod pass;
  decompose `lib/zone`/`AppContext`; remove `ignoreBuildErrors`; evaluate Turbopack + React 19;
  rationalise the 61-script tooling layer.

---

# Part B′ — Operator review (domain-expert feedback, integrated)

> Source: review feedback from an operator with prior Uswitch/comparison-sector experience.
> This section **supersedes parts of the B1–B4 framing where noted** — the core insight is a
> sharper thesis than the original "inform → transact" gap.

## B5. The reframe: from advice *poster* to project *workspace*

**The insight.** Acting on anything Zero Zero surfaces (insulation, heat pump, solar, switching,
grants) is **weeks-to-months of real-world work**, not a one-click action. Therefore the durable
value is **not the advice, and not even the transaction — it is the persistent tracking of the
long-running work**. The product should behave like a **functional tool/workspace**, not a
"design presentation."

This is a sharper version of B2/B3/B4. The original review identified that value leaks because the
product only *informs*; the operator view goes further: even *transacting* is a single moment,
whereas the household's job spans months. **The defensible product is a home-efficiency project
workspace** where decisions become **projects** that hold their own interactions, data, and state
over time. This also resolves the B2 retention and revenue gaps more durably than a one-shot
grant referral: a workspace people return to is both the retention engine and the data moat.

**Revised North Star (supersedes the B2 candidate):** *£ realised across active household
projects* (lifecycle value), with **active projects progressed per quarter** as the leading
indicator — not one-shot "£ actioned." Honesty rate stays the 100% counter-metric.

### What the reframe requires (new first-class capabilities)
| Capability | Why (operator rationale) | Notes / dependencies |
|------------|--------------------------|----------------------|
| **Project space** — a decision becomes a tracked project holding tasks, interactions, documents, status over time | The work is long-running; today nothing persists past a card close | Needs a `projects` data model + task state; this is the core build, not a feature |
| **Advisor threading into projects** — Zai conversations attach to a project and persist | "The advisor loses conversational thread and nothing gets progressed into a longer-term space" | Zai is currently read-only/stateless per surface (`lib/brains/zai/boundaries.ts`); needs conversation persistence keyed to a project |
| **Data-validity decay + refresh** — numbers age out; returning users are prompted to re-confirm, then shown a **progress infographic** | Long projects mean stale inputs; a 6-month return should refresh and *show progress* | Strong retention trigger; pairs with price-cap volatility re-engagement (B2) |
| **Bill ingestion** — upload a PDF bill or a **photo of tariff/usage**, LLM extracts the figures | Removes the estimate guesswork; the highest-confidence path to a believable £ | **GDPR is now first-class** (lawful basis, explicit consent, retention/erasure, DPIA, PII encryption) — ties to the A-track security gaps; pairs with the "make this accurate" action (B6) |

### Implication for the roadmap
The "path to action + revenue" bet in B4 (Phase 2) should be **reframed as the project
workspace**, with grants facilitation as the *first project type* inside it rather than a
standalone referral. The workspace is the container that makes every transaction, and the
returning-user loop, valuable.

## B6. UX findings register (operator-reported, verified where cheap)

Type: **STR**ategic · **FEAT**ure · **BUG** · **COPY** · **IA**/navigation · **VIS**ual.

| # | Finding (operator) | Type | Evidence / status | Recommendation | Priority |
|---|--------------------|------|-------------------|----------------|----------|
| 1 | **Discoverability is poor** — no nav bar / breadcrumbs; movement relies on back/close buttons. "Feels like a tool, so stability matters." | IA | Validated; navigation is modal/cascade-driven (`SoloFocusOverlay`, Director's Order cascade) | Add persistent global nav + breadcrumbs / a stable shell; treat it as a tool, not a linear promo flow | **High** |
| 2 | **Visual system fatigues over sustained use** — all-caps, font choice, colour intensity, drop shadows. "Great for a poster, not a dashboard." | VIS | Extends B3-5 from *motion* to the whole visual language | Introduce a **lower-intensity "working/dashboard" mode** for surfaces users dwell in (sentence-case body, calmer palette, no drop-shadow); keep the punchy look for marketing/first-run | **High** |
| 3 | **Advisor loses the thread** — Zai chat doesn't progress into a persistent project space | STR | `lib/brains/zai/boundaries.ts` (read-only/stateless per surface) | Persist Zai threads against a project (see B5) | **High** |
| 4 | **No data decay / progress view** — returning users aren't asked to refresh; no progress infographic | FEAT | Not present | Add validity decay + refresh prompt + progress visualisation (see B5) | **Med-High** |
| 5 | **Bill ingestion** — let users upload a PDF/photo of a bill; LLM crunches tariff/usage | FEAT | Not present; pipeline already runs Gemini | Add upload → LLM extract; **handle GDPR properly** (consent, retention, DPIA, encryption) | **Med-High** |
| 6 | **Buy links lack context** — no logos, no visibility of the destination | FEAT | Solo-Focus action opens an offer/source URL with no preview | Add provider **logos** + a **scraped summary of the linked page** (you already scrape — reuse it) | **Med** (good quick win) |
| 7 | **Liked cards didn't work** | BUG | Reproduced concern; likely cause = **like state fragmented across 4 stores** (DB `/api/likes`, zai likes, guest likes, `likeCardSnapshots`) reconstructed via `buildZoneViewModel` on the likes page | Consolidate like state to one source of truth; add a regression test | **Med** (real defect) |
| 8 | **"Make this accurate" on spend estimate** | FEAT | Estimate has no correction affordance | Add a "make this accurate" CTA → bill upload (#5) or manual entry | **Med** |
| 9 | **Audit trail needs a tooltip** | FEAT | No explanatory affordance | Add a tooltip explaining what the audit trail is/shows | **Low** (quick win) |
| 10 | **Employment copy "NOT WORK" is clumsy** | COPY | Confirmed: label `NOT WORK`, value `UNEMPLOYED` (`app/profile/ProfilePageClient.tsx:79`) | Use clearer options — e.g. *Employed / Self-employed / Not working / Retired / Student* + *Prefer not to say* for edge cases | **Low** (quick win) |

**Note on visual design (balanced):** the punchy aesthetic is a genuine strength for acquisition
and brand — the recommendation is **not** to discard it, but to recognise that a tool users spend
real time in needs a calmer *working mode*. This reinforces, with operator evidence, the B3-5
"value-before-brand" finding and the B3-1 time-to-value concern.

# Part C — Recommendations & roadmap

### The one-sentence thesis
Zero Zero is a **well-engineered, unusually honest product that is pointed slightly off the job
its market actually hires for** — it audits *carbon-and-everything* when its users want to *cut
their bills and be told the single next move* — and it has **no way to capture value** at the
moment of action; close those two gaps and the genuine trust+locality moat becomes a business.

### How stack and product connect
The stack review and product review reinforce each other rather than compete:
- The **honesty system** (mechanical truth, banned jargon, Zai's "i don't know") is simultaneously
  the **product's moat** (B1) and the thing the **stack most under-protects** (no eval, no tests —
  A5-1/B3-6). The highest-leverage technical work *is* the highest-leverage product work.
- The **in-memory state + no observability** stack gaps (A4) are also the reason the **product
  funnel is unmeasurable** (B3-7). One fix — instrument and persist — unblocks both.
- The **`lib/zone` complexity hotspot** (A2) is where the **product's over-breadth** (13 lanes,
  B3-3) physically lives; focusing the product also shrinks the riskiest module.

### What to do (in order)
1. **Instrument & protect first (Phase 0).** You cannot prove any change without funnel telemetry,
   and you cannot risk the reframe without an eval guarding the trust moat. Add durable rate
   limiting and basic error/cost capture in the same pass.
2. **Re-point to money, and to a single next action (Phase 1).** £-first hero, value-before-brand
   first run, one "biggest win for you," and a fast time-to-value via progressive profiling.
3. **Earn revenue at the point of action (Phase 2).** Grants facilitation first, riding the
   £15bn Warm Homes Plan tailwind; this creates the missing business model.
4. **Pay down foundation continuously (Phase 3).** Tests, Zod, decomposition, drop
   `ignoreBuildErrors`, Turbopack/React 19, tooling cleanup.

### Build / kill / pivot call
**Pivot-to-focus, don't kill.** The asset (honest, postcode-grounded, broad, well-built) is real
and the macro timing (Warm Homes Plan, price-cap volatility) is favourable. The required change
is a **strategic re-pointing** (money-first, action-first, with a revenue path), not a rebuild.
The main risks are (a) the team's strong "Director's Order" attachment to the carbon-auditor frame
and the frozen motion sequence, which this evidence suggests is the thing to revisit, and (b)
executing a transaction/revenue motion the product has not yet attempted.

---

## Appendix — evidence & sources

**Code evidence (file:line)** is cited inline throughout Parts A–B. Key anchors: architecture
`app/layout.tsx:126`, `app/context/AppContext.tsx`, `lib/db.ts:27-33`,
`lib/intelligence/bucketFailover.ts:232-303`, `lib/intelligence/aiGateway.ts:125-147`,
`app/api/cron/zone-research/route.ts`; product `app/profile/ProfilePageClient.tsx:39-96`,
`lib/journeys.ts`, `lib/zone/loopQuestions.ts:28-209`, `lib/zone/mechanicalTruth.ts:44`,
`lib/zone/warmAuditorCopy.ts:10-21`, `lib/brains/zai/boundaries.ts`, `app/api/analytics/route.ts`;
DD `next.config.js:9`, `package.json:91`, `npm audit --omit=dev` (0 vulns).

**External sources (2026)**
- Market / job: [House of Commons Library — electricity bill make-up](https://commonslibrary.parliament.uk/research-briefings/cbp-10505/) ·
  [JRF — addressing the 2026 energy price crisis](https://www.jrf.org.uk/cost-of-living/addressing-the-2026-energy-price-crisis) ·
  [Solar4Good — average UK energy bills 2026](https://solar4good.co.uk/blogs/average-energy-bills-uk-2026/)
- Competitors / tailwind: [Nous AI bills assistant](https://www.nous.co/blog/nous-launches-new-ai-assistant-to-make-sense-of-household-bills) ·
  [Snugg](https://www.snugg.com/) · [GB Energy grant checker](https://greatbritishenergy.com/eligibility-checker/) ·
  [MSE — grants](https://www.moneysavingexpert.com/family/housing-and-energy-grants/) ·
  [MSE — Warm Homes Plan (Jan 2026)](https://www.moneysavingexpert.com/news/2026/01/warm-homes-plan-martin/)
- Benchmarks (full list in `REVIEW-PLAN.md`): [Next.js 16](https://nextjs.org/blog/next-16) ·
  [LLM/agent evaluation 2026 (Adaline)](https://www.adaline.ai/blog/complete-guide-llm-ai-agent-evaluation-2026) ·
  [Technical due-diligence 2026](https://www.cleveroad.com/blog/technical-due-diligence/) ·
  [Core Web Vitals + WCAG (Siteimprove)](https://www.siteimprove.com/blog/core-web-vitals-wcag/)
