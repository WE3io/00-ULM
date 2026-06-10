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
| A1 | System architecture (context+containers) | C4-style | 🔴 todo | |
| A2 | Module boundaries & coupling | dependency map | 🔴 todo | 27 `lib/` sub-domains, 237 modules |
| A3 | AI / data pipeline architecture | flow + failure | 🔴 todo | bucket failover, Firecrawl JIT, Hermes |
| A4 | Scaling, state & failure modes | — | 🔴 todo | serverless pooling, cron, rate-limit durability |
| A5 | DD by exception | risk register | 🔴 todo | tests, ignoreBuildErrors, SSRF, secrets, CVEs |
| B1 | Discover — problem space | JTBD + context | 🔴 todo | UK cost-of-living / energy |
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

## Open questions
_Things that need a human or a deeper dig. Resolve or carry forward each pass._

- (none yet)

## Iteration log
_One line per pass: date · section touched · what changed · status delta._

- 2026-06-10 · scaffold · created REPORT + PROGRESS skeleton · all 🔴
