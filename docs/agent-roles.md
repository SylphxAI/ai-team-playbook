# Agent Roles

6 directions, 8 agents. Split by cognitive mode, not technical domain.

## Roster

| # | Direction | Key | Count |
|---|-----------|-----|-------|
| 1 | Product | `product` | 1 |
| 2 | Audit | `audit` | 1 |
| 3 | Triage | `triage` | 1 |
| 4 | Build | `builder` | 3 |
| 5 | Test | `tester` | 1 |
| 6 | Review | `reviewer` | 1 |
| | **Total** | | **8** |

## Role Briefs

### Issue Creators — find problems, file issues, never write code.

**Product** (×1):
- Competitive analysis, feature gaps, market positioning, differentiation
- User journeys, personas, pain points, retention, churn, NPS
- Viral loops, sharing incentives, referral mechanics, network effects
- Monetization: ads, subscriptions, freemium, in-app purchases, partnerships
- Acquisition channels, conversion funnels, growth experiments, A/B testing
- Content strategy, trending topics, UGC, editorial pipeline, curation
- Roadmap prioritization, feature scoring, launch strategy, phased rollouts
- Pricing, tiers, free vs paid, trials, upsells, LTV
- Legal, compliance, privacy, GDPR, COPPA
- Brand, identity, voice, positioning, trust signals
- Mobile-first, PWA, app store, responsive, offline-capable

**Audit** (×1) — replaces former Improve + Secure + Perf:
- Refactoring: DRY, SOLID, design patterns, naming, barrel files
- Type safety: eliminate `any`, Zod schemas, generics, narrowing
- Dead code: unused imports, exports, files, dependencies, feature flags
- Error handling: boundaries, logging, monitoring, alerting, recovery
- Documentation: README, JSDoc, API docs, architecture decisions
- SEO: meta tags, structured data, sitemap, Open Graph, robots.txt
- Accessibility: WCAG, ARIA, keyboard nav, screen readers, contrast
- i18n: string extraction, locale routing, RTL, date/number formatting
- Dependencies: updates, CVEs, supply chain, lockfile integrity
- Injection: XSS, SQL, NoSQL, command, SSRF, path traversal
- Auth: bypass, privilege escalation, session fixation, token leakage
- Data: PII exposure, debug endpoints, IDOR, mass assignment
- Transport: CORS, CSP, HSTS, cookie flags, certificate pinning
- Input: validation, sanitization, output encoding, file upload limits
- Rate limiting: enumeration, API abuse, DDoS vectors
- Secrets: hardcoded keys, env leaks, .env exposure, git history
- Core Web Vitals: LCP, FID, CLS, INP, TTFB
- Bundle: size analysis, tree shaking, code splitting, lazy loading
- Assets: image optimization, WebP/AVIF, font loading, CDN, caching
- Database: N+1 queries, missing indexes, connection pooling, pagination
- Rendering: SSR, SSG, ISR, streaming, partial hydration, Suspense
- Runtime: memory leaks, event listeners, main thread blocking, workers

### Triage — central quality gate for issues.

**Triage** (×1):
- Review all open issues without `approved` or `rejected` label
- Evaluate: real? valuable? clear enough for a builder? duplicate?
- Approve → add `approved` label. Reject → add `rejected` + reason comment
- Prioritize: `P0` (critical), `P1` (important), `P2` (nice-to-have)
- Size: `XS`, `S`, `M`, `L`
- Reject if: duplicate, out of scope, too vague, too large, already fixed
- Decompose large issues into smaller approved sub-issues
- Stale cleanup: `in-progress` + no linked PR after 30 min → remove label

### Builders — fix failing PRs first, then build from approved issues. Code only, no tests.

**Build** (×3):
- **P1 — Fix failing PRs**: find open PRs with failing CI, no `fixing` label. Claim with `fixing`. Fix code, push, remove label.
- **P2 — Build from approved issues**: find issues with `approved`, no `in-progress`. Claim with `in-progress`. PR refs `Closes #N`.
- Code only — tests added by Testers
- Scope: pages, components, APIs, database, auth, payments, integrations, real-time, infrastructure
- Lint before commit. Exit if nothing to do.

### Testers — write tests for open PRs. Adversarial mindset.

**Test** (×1):
- Find open PRs lacking tests. Read the diff.
- Write tests that BREAK it — adversarial, not confirmatory
- Unit, integration, or E2E — pick based on the change
- Focus: user-facing edge cases, what the builder missed, what breaks under load
- Skip PRs with adequate tests. One PR per session.

### Reviewers — quality gate + merge authority.

**Review** (×1):
- Correctness, security, performance, scope, style
- Tests missing → comment "needs tests", do NOT reject (testers will add)
- Merge rule: approve + squash merge + delete branch ONLY when code AND tests pass CI
- Uses separate identity from builders (branch protection requirement)

## Separate Testing Model

Builders write code only. Testers write tests independently on open PRs.

| Factor | Traditional OSS | AI Team |
|--------|----------------|---------|
| Ego/ownership | Authors defend code | AI has no ego |
| Blind spots | Author confirms own mental model | Fresh eyes catch what author missed |
| Approach | Confirmatory | Adversarial |
| Workflow | Sequential | Parallel (builder moves on, tester tests PR) |
| Volunteer availability | Hard to find | Always available |

**Trade-offs**: tester may misunderstand builder intent; may miss implementation-specific shortcuts. Acceptable — reviewers catch misalignment before merge.

## Design Rationale

**Why Audit replaces Improve + Secure + Perf?** Code quality, security, and performance are intertwined. Dead code removal reduces attack surface AND bundle size. One agent with the full picture creates better issues than three with tunnel vision.

**Why 3 Builders?** AI is fullstack — no frontend/backend split needed. 3 provides throughput; the bottleneck is Review and Triage, not raw code output.

**Why one generic Tester?** AI can write all test types. A tester reading a diff naturally knows whether it needs unit, integration, or E2E tests.

**Why Triage?** Without it, builders pick up vague/duplicate issues and waste cycles. Triage dropped wasted work by 30-50%.

**Why Review merges?** Reviewer already has full context from reading the code. Natural merge point. Removes merge responsibility from coordinator.
