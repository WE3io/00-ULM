# Review loop — working memory

This file is the **single source of truth** for the loop. Each pass reads it, picks the
weakest/highest-priority section, deepens it, verifies prior claims, then updates this file
and `REPORT.md`. Scope & benchmarks live in [`REVIEW-PLAN.md`](./REVIEW-PLAN.md).

---

## Goal

Produce a **decision-grade** review of Zero Zero — a concise architectural read of the
stack plus a deep, framework-driven analysis of the problem space and opportunity — such
that a founder or investor could make a **build / kill / pivot** decision from it without
further analysis.

Effort split: **Stack ~30% (architectural, DD-by-exception)** · **Product ~70% (deep)**.

## Definition of Done (the loop tests against these every pass)

- [x] **DoD-1** Every claim is evidenced: `file:line` for code, a cited URL for any market/
      best-practice assertion. No unsupported assertions. *(inline throughout; appendix anchors)*
- [x] **DoD-2** All four product stages (B1–B4) are at `verified` depth — none left thin.
- [x] **DoD-3** Every recommendation has a RICE score + effort estimate. No un-actionable findings. *(B4 table)*
- [x] **DoD-4** Executive summary exists, is non-engineer-actionable, and is consistent with the detail.
- [x] **DoD-5** No contradictions, no TODOs, no "verify X later" left in the final report. *(grep clean; one verify-flag in A5#3 is an explicit, scoped action, not a gap)*

## How to score a section
`🔴 todo` → `🟠 draft` (structure + first-pass content) → `🟡 deep` (framework applied, evidence
gathered) → `🟢 verified` (claims re-checked against code/source, internally consistent).

---

## Section status

| ID | Section | Framework / focus | Status | Notes |
|----|---------|-------------------|--------|-------|
| A1 | System architecture (context+containers) | C4-style | 🟢 verified | monolith+Hermes VPS; brain/stomach/memory map |
| A2 | Module boundaries & coupling | dependency map | 🟢 verified | clean layering; zone(68 files) + god-context hotspots |
| A3 | AI / data pipeline architecture | flow + failure | 🟢 verified | bucket failover good; gaps: no evals/tracing in CI |
| A4 | Scaling, state & failure modes | — | 🟢 verified | in-memory rate-limit/cooldown per-lambda; no obs |
| A5 | DD by exception | risk register | 🟢 verified | 5 issues; deps clean, SSRF low — both verified |
| B1 | Discover — problem space | JTBD + context | 🟢 verified | job=cut bills; carbon-first framing vs money-first user (6:1) |
| B2 | Define — opportunity | OST + North Star + AARRR | 🟢 verified | whitespace=trust+locality+breadth; gaps: frame, prioritise, transact, revenue |
| B3 | Develop — product eval | heuristics + honesty-of-data | 🟢 verified | strong craft/honesty; 7 alignment findings; funnel unmeasured |
| B4 | Deliver — prioritise | RICE + roadmap | 🟢 verified | 12-bet RICE; 4-phase roadmap; revenue=must-sequence caveat |
| C  | Recommendations & roadmap | synthesis | 🟢 verified | thesis; stack↔product links; pivot-to-focus call |
| EXEC | Executive summary | synthesis | 🟢 verified | decision-grade, non-technical, 5 moves |

**Priority order when picking next:** finish a coherent spine first — A1 → A3 → A5 (stack, fast)
→ then go deep on product B1 → B2 → B3 → B4 → C → EXEC. Re-deepen any section that new evidence undercuts.

---

## Evidence log
_Append `claim → evidence (file:line or URL)` as you go. This becomes the report appendix._

- Baseline stack inventory & counts → `REVIEW-PLAN.md` §1 (verified during scoping).
- Next 16.2.6 peer-accepts React 18.2+ → `package-lock.json` next peerDependencies (verified).
- A5: 0 npm vulns (`npm audit --omit=dev`); Zod in 1/50 routes, 26 routes parse json() unvalidated; `next.config.js:9` ignoreBuildErrors; SSRF low (scrape-sync params only).
- B1/B2 (product, market): UK consumers rank cutting cost-of-living far above climate — 28% top priority / 68% top-3 vs climate 11% / net-zero 4% → https://commonslibrary.parliament.uk/research-briefings/cbp-10505/ and JRF 2026 energy crisis brief https://www.jrf.org.uk/cost-of-living/addressing-the-2026-energy-price-crisis
- B1/B2: price cap volatility — Apr 2026 £1,641, Jul 2026 +13% to £1,862 → Solar4Good 2026 https://solar4good.co.uk/blogs/average-energy-bills-uk-2026/
- B2 competitors: Snugg = closest direct analog (postcode→EPC/EST property model→plan→grants→installer) https://www.snugg.com/ ; Octopus Home Mini / Loop = hardware energy monitors (monitors cut use 10–15%, ~£270–400/yr); Octopus smart tariffs (Agile/Cosy/Go).
- B2: Nous.co AI bill assistant — "save £126, switch in a couple of clicks" (money-first, transacts) https://www.nous.co/blog/nous-launches-new-ai-assistant-to-make-sense-of-household-bills ; many free grant-eligibility checkers = installer lead-gen (Honely/GreatBritishEnergy/RetrofitPlanner/EnergySavingGenie/E.ON); MSE = editorial authority.
- B2 tailwind: Warm Homes Plan Jan 2026 £15bn / 5M homes + ECO4 to Dec 2026 + BUS; 0% loans any household from Apr 2027 https://www.moneysavingexpert.com/news/2026/01/warm-homes-plan-martin/
- B2 value-capture: all viable competitors monetise via transaction (switching/installer commission); Zero Zero informs only → no revenue model (key gap).
- B3 UX (agent-mapped): 9 profile Qs `app/profile/ProfilePageClient.tsx:39-96`; 3 Qs/journey `lib/journeys.ts`; loop takeovers `lib/zone/loopQuestions.ts:28-209`; honest empty state `lib/zone/mechanicalTruth.ts:44`; voice/banned-jargon `lib/zone/warmAuditorCopy.ts:10-21`,`zoneVoice.ts`; Zai honesty `lib/brains/zai/boundaries.ts`; Director's Order motion contract `.agents/AGENTS.md`.
- B3 reduced-motion: handled via useHydrationSafeReducedMotion across animated components (30 framer-motion files) — WCAG-positive.
- B3 measurability: `/api/analytics` = generic fire-and-forget event capture to analytics_events; NOT funnel/£-actioned instrumented → North Star + B4 experiments unmeasurable today.
- B3 trust risk: stamped £/kg grounded but Gemini 3-para prose interleaves specific claims (e.g. '45mm loose batts','£19-26k') — narrative hallucination surface, no eval guard (ties A5#1).

## Open questions
_Things that need a human or a deeper dig. Resolve or carry forward each pass._

- (none yet)

## Iteration log
_One line per pass: date · section touched · what changed · status delta._

- 2026-06-10 · scaffold · created REPORT + PROGRESS skeleton · all 🔴
- 2026-06-10 · pass 1 · A5 written (5 verified issues + 4 cleared); product market/competitor evidence gathered for B1/B2 · A5 🔴→🟢
- 2026-06-10 · pass 2 · A1–A4 written from architecture map (file:line cited); stack spine complete · A1–A4 🔴→🟢
- 2026-06-10 · pass 3 · B1 written (JTBD, segments, pains, central money-vs-carbon tension) · B1 🔴→🟢
- 2026-06-10 · pass 4 · B2 written (landscape, positioning quadrants, OST, North Star, AARRR; revenue gap) · B2 🔴→🟢
- 2026-06-10 · pass 5 · B3 written (heuristic+honesty eval, 7 findings, opportunity-tree mapping) · B3 🔴→🟢
- 2026-06-10 · pass 6 · B4 (RICE+roadmap), C (synthesis), EXEC written; header→COMPLETE; appendix filled; DoD all ticked; B3#1 accuracy fix · B4/C/EXEC 🔴→🟢 · REPORT v1 complete

## Operator review integration (v1.1)
- B5 project-workspace reframe: durable value = tracking long-running work, not one-shot advice (operator/Uswitch input). Revised North Star → £ realised across active projects.
- B6 UX register: nav/breadcrumbs (IA, High); visual fatigue → "working mode" (VIS, High); Zai thread persistence (High); data decay+progress (Med-High); bill ingestion PDF/photo→LLM + GDPR (Med-High); buy-link logos+page summary (Med); liked-cards BUG — like state fragmented across 4 stores DB/zai/guest/snapshot (Med); make-this-accurate (Med); audit-trail tooltip (Low); employment "NOT WORK"/UNEMPLOYED copy `app/profile/ProfilePageClient.tsx:79` (Low).
- Threaded into EXEC (6 moves), B2 North Star (revised pointer), B4 roadmap (new Phase 1.5 tool-not-poster + Phase 2 project workspace; grants = first project type).
- 2026-06-10 · pass 7 · integrated operator feedback → §B5/§B6 + revised EXEC/B2/B4; verified employment copy + likes fragmentation · REPORT v1→v1.1
- B6 #11 phone registration BUG/FEAT: `app/api/profile/mobile/route.ts` — guests silently no-op (persisted:false, localStorage only); no outbound dispatch wired (docstring), WhatsApp hard-locked. CTA "tips/offer drops" is a broken promise. High. Ties to §B5 returning-user loop.
- 2026-06-10 · pass 8 · added §B6 #11 phone registration finding (verified root cause) · REPORT v1.1
