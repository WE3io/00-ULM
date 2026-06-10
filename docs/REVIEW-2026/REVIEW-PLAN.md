# Zero Zero (00-ULM) — Comprehensive Review: Scope & 2026 Best-Practice Brief

> **Purpose.** This is the *preparation* artifact for a full review of the forked
> Zero Zero project from both a **stack** (engineering) and a **product** perspective.
> It defines what will be reviewed, captures the current state of the codebase as a
> baseline, and sets the 2026 best-practice benchmarks each finding will be scored
> against. The review itself (findings + prioritised remediation) is a separate pass
> that runs against this plan.

**Prepared:** 2026-06-10 · **Branch:** `claude/forked-project-review-9naqz7`

---

## 1. What this project is

Zero Zero is a **mobile-first, UK-first web app** that helps users understand and
reduce everyday impact across money, energy, carbon, and home life, driven by the
user's **postcode**. It pairs a journey/"zone" UX with an AI + web-scraping pipeline
that produces locally-grounded, "mechanically true" savings data (the 12,000 kWh ≈
1 tonne CO₂e baseline). Core external systems: **Neon Postgres**, **Google Gemini**
(via a multi-provider failover bucket), **Firecrawl** scraping, a **Hermes**
intelligence/cron loop, and **Vercel** hosting.

### Stack inventory (baseline snapshot)

| Dimension | Current state |
|---|---|
| Framework | Next.js **16.2.6** (App Router), **webpack** dev (`next dev --webpack`, not Turbopack) |
| UI runtime | React **18.3.1** / react-dom 18.3.1 (Next 16.2.x still peer-accepts React 18.2+) |
| Language | TypeScript 5.x, `strict: true`, `target ES2017`, `moduleResolution: bundler` |
| Styling/motion | Tailwind 3.4, framer-motion 12, lucide-react |
| Data | Neon serverless + `pg`; **36 SQL migrations** in `db/migrations` |
| AI | `ai` SDK v6, `@google/generative-ai` **0.21** (legacy SDK), multi-provider bucket (Gemini→Groq→Mistral→OpenRouter) |
| Auth | Custom: bcryptjs, HMAC-signed HTTP-only cookies, in-code login rate limiting |
| Surface | **50 API routes**, **9 pages**, **50 app components**, **237 `lib` modules**, **61 scripts** |
| Codebase size | ~403 TS/TSX files, ~**55k LOC** (excl. node_modules) |
| Tests | **1 Playwright e2e spec + 1 fixture** — effectively no automated coverage |
| CI | GitHub Actions: typecheck + lint + `verify` only (no tests); Vercel build gate |
| Build posture | `next.config.js` sets `typescript.ignoreBuildErrors: true` (tsc enforced separately via a script gate) |
| Docs | Extensive: a 4,287-line `HANDBOOK.md` + ~18 satellite docs |

---

## 2. Review scope — the eight tracks

The review is organised into eight tracks. Each gets a current-state read, scored
against the 2026 benchmark in §3, and a prioritised finding list (P0–P3).

1. **Architecture & code health** — module boundaries (`lib/` has 27 sub-domains),
   coupling, the 61-script tooling layer, dead code, the `ignoreBuildErrors` posture,
   complexity hotspots, dependency freshness (legacy Gemini SDK, React 19 path).
2. **AI / data pipeline** — the Gemini→Groq→Mistral→OpenRouter failover bucket,
   Firecrawl JIT vs Hermes scraping, prompt construction, cost controls, evals,
   observability/tracing, "mechanical truth" guarantees, and failure/fallback paths.
3. **Security** — authn/z, the cron/admin/gateway secret model, secret handling,
   input validation (Zod coverage across 50 routes), SSRF surface in the scraper,
   rate-limiting durability, PII handling, dependency CVEs.
4. **Data & persistence** — schema design, the 36-migration history and how it is
   applied, indexing, query safety (parameterisation), connection pooling on serverless.
5. **Performance & Core Web Vitals** — RSC vs client boundary placement, Turbopack
   adoption, bundle/First-Load-JS, LCP/INP/CLS on mobile, image strategy, caching.
6. **Testing & release engineering** — coverage gap, CI gates, the deploy/promote
   flow, rollback, DORA-style delivery metrics, the custom build-gate scripts.
7. **Product & UX** — the journey/zone/Solo-Focus loop, onboarding/activation,
   information honesty ("COMPUTING" vs fake numbers), copy system, retention loops,
   empty/error states, mobile ergonomics.
8. **Accessibility & compliance** — WCAG 2.2 AA, the high-contrast neon palette,
   motion-heavy UX vs `prefers-reduced-motion`, keyboard/SR support, UK data/privacy.

---

## 3. 2026 best-practice benchmarks (what "good" looks like)

### 3.1 Next.js 16 / React on the modern stack
- **Turbopack is the Next 16 default** for dev *and* prod builds (2–5× faster builds);
  persist the `.turbopack` cache in CI. This repo still runs `--webpack` — review whether
  that is deliberate (a known incompatibility) or untaken upside.
- Push the **`'use client'` boundary as deep as possible**; full RSC adoption commonly
  cuts First-Load-JS 50–70% and improves LCP.
- **Cache Components / `'use cache'`**: caching is opt-in in 16; dynamic-by-default.
- **Security**: pin Next ≥ 16.0.11 / React ≥ 19.2.4; two 2025 RSC CVEs exist
  (CVE-2025-55184 DoS, CVE-2025-55183 source exposure) — confirm patched range.
- **Validate every Server Action / route input with Zod**; never embed secrets in
  Server Components. `priority` on `<Image>` is deprecated in favour of `preload`.
- **React 19** unlocks the React Compiler, Activity, and `useEffectEvent` — treat as a
  roadmap opportunity, not a blocker (the current Next version still accepts React 18).

### 3.2 Production LLM / agent architecture
- Treat the agent/pipeline as a **system, not a prompt**: strict tool contracts,
  deterministic state transitions, **trace-level observability**, and **evals shipped in CI**.
- Three eval layers: **end-to-end** (task completed), **trajectory** (planning/tool calls),
  **component** (retriever / extraction quality). The "mechanical truth" guarantee is
  exactly the kind of invariant that should have automated evals.
- **Cost control**: semantic/embedding caching can cut LLM calls ~69% and cost ~70%.
  Benchmark the existing bucket-failover + JIT-scrape budgeting against this.
- For retrieval: hybrid (dense + BM25 + metadata) with rerank is the 2026 default if/where
  RAG applies.

### 3.3 Technical due-diligence / code-review standard
- Run **automated scanners first** (Snyk/`npm audit` for deps, SonarQube-class static
  analysis) — they catch ~70% more vulns than manual review alone — then manual review.
- **Coverage**: 60–80% is healthy; **< 40% is a red flag**. This repo's ~2 test files is
  the single biggest gap to call out.
- **Security is the #1 deal-killer**: TLS everywhere, encrypted PII columns, MFA on admin,
  SOC2 readiness posture.
- **DORA delivery metrics**: deploy frequency (daily+), lead time < 1 day, change-failure < 5%.

### 3.4 Product / UX & Core Web Vitals
- KPIs that matter in 2026: task-completion rate, time-on-task, **funnel conversion by step**,
  **WCAG 2.2 AA** score, Core Web Vitals, **7-day & 30-day retention** curves, product NPS.
- **Core Web Vitals targets (p75, field data): LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1.**
- **Mobile-first** is the right instinct here; verify it holds under real device constraints
  (image/script weight, tactile targets).
- **Accessibility is a business lever**, not just compliance: accessible products see ~23%
  higher long-term retention and up to ~17% higher checkout completion — directly relevant to
  a motion-heavy, high-contrast neon UI.

---

## 4. Early signals already visible (to confirm in the review pass)

These are *candidate* findings spotted during scoping — each must be verified before it
lands in the final review.

- **P0? Test coverage:** ~2 test files for ~55k LOC; e2e isn't in CI. Highest-leverage gap.
- **P1 Build honesty:** `typescript.ignoreBuildErrors: true` in `next.config.js`. A
  separate script gate runs tsc, but the in-config escape hatch is fragile — verify the gate
  actually blocks bad builds on Vercel.
- **P1 Tooling sprawl:** 61 scripts + many bespoke `dev`/`build` variants with polling flags
  and manual manifest fixups — signals fighting the framework; assess maintainability and
  whether Turbopack/native flows remove the need.
- **P2 Dependency freshness:** `@google/generative-ai@0.21` is the legacy Google SDK
  (superseded by `@google/genai`); confirm migration path and that pinned Next/React are on
  the CVE-patched range.
- **P2 Webpack vs Turbopack:** dev runs `--webpack`; quantify the build/DX cost and the
  blocker (if any) to adopting the Next 16 default.
- **Security surface to probe:** Zod coverage across all 50 routes, the multi-secret model
  (`CRON_SECRET`/`GATEWAY_TOKEN`/`SESSION_SECRET`/`ADMIN_PASSWORD`), SSRF in the Firecrawl/
  scrape-sync path (`?postcode`/URL handling), and whether the in-memory login rate limiter
  survives serverless cold starts.
- **Accessibility tension:** the frozen "Director's Order" motion sequence + neon palette vs
  `prefers-reduced-motion` and WCAG 2.2 AA — likely a real conflict to evaluate.

---

## 5. Review method & deliverables

1. **Automated sweep** — `npm audit`, dependency/CVE check, typecheck/lint baseline,
   complexity scan, dead-code/duplication pass.
2. **Manual track-by-track read** (the eight tracks in §2), scored against §3 benchmarks.
3. **Security deep-dive** — authn/z, secret model, input validation, SSRF, PII.
4. **Product/UX walkthrough** — the full journey loop on a mobile viewport; Core Web Vitals
   + WCAG 2.2 AA spot-checks; honesty-of-data review.
5. **Findings register** — every finding tagged P0–P3 with evidence (`file:line`), the 2026
   benchmark it misses, and a remediation estimate.
6. **Executive summary** — top risks, quick wins, and a sequenced remediation roadmap.

---

## Sources

- [Next.js 16 (official blog)](https://nextjs.org/blog/next-16) · [Turbopack API reference](https://nextjs.org/docs/app/api-reference/turbopack)
- [Complete Guide to Next.js 16 + React 19.2 in Production — RSC Security, Turbopack (dev.to)](https://dev.to/x4nent/complete-guide-to-nextjs-16-react-192-in-production-rsc-security-view-transitions-turbopack-5090)
- [Next.js 16 Performance Optimization 2026 Cheat Sheet](https://techbytes.app/posts/nextjs-16-performance-optimization-2026-cheat-sheet/)
- [AI Agent Architecture: Build Systems That Work in 2026 (Redis)](https://redis.io/blog/ai-agent-architecture/)
- [The Complete Guide to LLM & AI Agent Evaluation in 2026 (Adaline)](https://www.adaline.ai/blog/complete-guide-llm-ai-agent-evaluation-2026)
- [LLM Agent Evaluation Metrics in 2026 (Confident AI)](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide)
- [Your Ultimate Technical Due Diligence Checklist 2026 (john-pratt.com)](https://www.john-pratt.com/technical-due-diligence-checklist)
- [Technical Due Diligence Key Elements: Checklist for 2026 (Cleveroad)](https://www.cleveroad.com/blog/technical-due-diligence/)
- [Core Web Vitals and WCAG: One Operating System for UX, SEO, and Risk (Siteimprove)](https://www.siteimprove.com/blog/core-web-vitals-wcag/)
- [UX/UI Design Trends 2026 — Data-Backed Guide](https://www.sanjaydey.com/ux-ui-design-trends-2026-biggest/)
- [Core Web Vitals: The Complete 2026 Guide (LCP, INP & CLS)](https://innovisionbiz.com/core-web-vitals-guide/)
