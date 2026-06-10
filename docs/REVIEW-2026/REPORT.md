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
  "mechanical truth" honesty rate at 100% (no inflated £).
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
| 1 | **Heavy question load before value.** 9 profile Qs, then up to **3×13 journey Qs** plus Solo-Focus *loop* takeovers ("rail instead of flying?"). | `lib/journeys.ts` (3/journey), `lib/zone/loopQuestions.ts:28-209` | Rivals deliver a number "in seconds" (Nous) / "60-second check" (grant tools). Front-loading profiling is an **activation tax** on a user who wants a fast £ answer. |
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
> _Framework: RICE prioritisation + sequenced roadmap._
> _TODO_

---

# Part C — Recommendations & roadmap

> _Synthesised, sequenced. Strategic narrative tying stack + product together._
> _TODO_

---

## Appendix — evidence log & sources
> _Running list mirrored from PROGRESS.md on finalisation._
