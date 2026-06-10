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

- [ ] **DoD-1** Every claim is evidenced: `file:line` for code, a cited URL for any market/
      best-practice assertion. No unsupported assertions.
- [ ] **DoD-2** All four product stages (B1–B4) are at `verified` depth — none left thin.
- [ ] **DoD-3** Every recommendation has a RICE score + effort estimate. No un-actionable findings.
- [ ] **DoD-4** Executive summary exists, is non-engineer-actionable, and is consistent with the detail.
- [ ] **DoD-5** No contradictions, no TODOs, no "verify X later" left in the final report.

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
| B2 | Define — opportunity | OST + North Star + AARRR | 🔴 todo | needs competitive scan |
| B3 | Develop — product eval | heuristics + honesty-of-data | 🔴 todo | journey/Solo-Focus loop |
| B4 | Deliver — prioritise | RICE + roadmap | 🔴 todo | |
| C  | Recommendations & roadmap | synthesis | 🔴 todo | |
| EXEC | Executive summary | synthesis | 🔴 todo | write last |

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

## Open questions
_Things that need a human or a deeper dig. Resolve or carry forward each pass._

- (none yet)

## Iteration log
_One line per pass: date · section touched · what changed · status delta._

- 2026-06-10 · scaffold · created REPORT + PROGRESS skeleton · all 🔴
- 2026-06-10 · pass 1 · A5 written (5 verified issues + 4 cleared); product market/competitor evidence gathered for B1/B2 · A5 🔴→🟢
- 2026-06-10 · pass 2 · A1–A4 written from architecture map (file:line cited); stack spine complete · A1–A4 🔴→🟢
- 2026-06-10 · pass 3 · B1 written (JTBD, segments, pains, central money-vs-carbon tension) · B1 🔴→🟢
