# Agent Roles — V4

6 directions, 8 agents. Split by **cognitive mode**, not technical domain.

## Design Principles

- **AI is fullstack.** Don't split frontend/backend — AI has no skill boundaries.
- **Directions, not departments.** Each role is a direction of thought: product, audit, triage, build, test, review. Fewer categories, clearer purpose.
- **Generic briefs + separate project context.** Role briefs are project-agnostic keyword lists. Project context (users, competitors, vision) is prepended separately at spawn time.
- **Keywords, not narratives.** Minimum words, maximum coverage. Each bullet is a keyword cluster covering one aspect of the role.
- **Separate testing model.** Builders write code only. Testers write tests independently for open PRs.

## Project Context (Separate)

Every agent receives identical **project context** prepended to their role brief at spawn time. This is configured per-project and includes:

- Project name, URL, description
- Competitors and market positioning
- Repo, commit identity, PR conventions
- Any project-specific constraints

The role briefs below are **generic** — they work for any project. The project context makes them specific.

## Roster

| # | Direction | Key | Count |
|---|-----------|-----|-------|
| 1 | **Product** | `product` | 1 |
| 2 | **Audit** | `audit` | 1 |
| 3 | **Triage** | `triage` | 1 |
| 4 | **Build** | `builder` | 3 |
| 5 | **Test** | `tester` | 1 |
| 6 | **Review** | `reviewer` | 1 |
| | **Total** | | **8** |

## Role Briefs

### Issue Creators — Find problems, file issues. Never write code.

#### Product (×1)

- Competitive analysis, feature gaps, market positioning, differentiation
- User journeys, personas, pain points, retention, churn, NPS
- Viral loops, sharing incentives, referral mechanics, network effects, invites
- Monetization: ads, subscriptions, freemium, in-app purchases, partnerships, sponsorships
- Acquisition channels, conversion funnels, growth experiments, A/B testing
- Content strategy, trending topics, UGC, editorial pipeline, curation
- Roadmap prioritization, feature scoring, launch strategy, phased rollouts
- Pricing, tiers, free vs paid, trials, upsells, lifetime value
- Funding, revenue model, unit economics, CAC, LTV, payback period
- Legal, compliance, privacy policy, terms of service, GDPR, COPPA
- Brand, identity, voice, positioning, trust signals
- Mobile-first, PWA, app store, responsive, offline-capable

#### Audit (×1)

Replaces the former Improve (×2), Secure (×1), and Perf (×1) roles. One direction covering code quality, security, and performance.

- Refactoring: DRY, SOLID, design patterns, file organization, naming, barrel files
- Type safety: eliminate `any`, Zod schemas, generics, narrowing, branded types
- Dead code: unused imports, exports, files, dependencies, feature flags, stale configs
- Error handling: boundaries, logging, monitoring, alerting, user-facing messages, recovery
- Documentation: README, JSDoc, API docs, architecture decisions, setup guides, diagrams
- SEO: meta tags, structured data, sitemap, Open Graph, robots.txt, canonical URLs
- Accessibility: WCAG, ARIA, keyboard nav, screen readers, contrast, focus management
- i18n: string extraction, locale routing, RTL, date/number/currency formatting, plurals
- Dependencies: updates, security patches, breaking change migration, license audit
- Code style: consistency, naming conventions, import ordering, module boundaries
- Injection: XSS, SQL, NoSQL, command, template, header, SSRF, path traversal
- Auth: bypass, privilege escalation, session fixation, token leakage, brute force
- Data: PII exposure, debug endpoints, error message leaks, IDOR, mass assignment
- Dependencies: CVEs, outdated packages, supply chain, typosquatting, lockfile integrity
- Transport: CORS, CSP, HSTS, cookie flags, HTTPS, certificate pinning
- Input: validation, sanitization, output encoding, file upload restrictions, size limits
- Rate limiting: enumeration, API abuse, DDOS vectors, account takeover
- Secrets: hardcoded keys, env leaks, .env exposure, git history, rotation policy
- Payment security: PCI compliance, tokenization, fraud detection, webhook verification
- Core Web Vitals: LCP, FID, CLS, INP, TTFB
- Bundle: size analysis, tree shaking, code splitting, lazy loading, dynamic imports
- Assets: image optimization, WebP/AVIF, font loading, compression, CDN, caching headers
- Database: N+1 queries, missing indexes, query plans, connection pooling, pagination, denormalization
- Rendering: SSR, SSG, ISR, streaming, partial hydration, Suspense, concurrent features
- Network: HTTP/2, preload, prefetch, preconnect, compression, request waterfall, CDN
- Runtime: memory leaks, event listeners, animation perf, main thread blocking, workers
- Mobile: 3G/4G simulation, low-end devices, battery impact, data usage, offline caching

### Triage — Central quality gate for issues.

#### Triage (×1)

- Review all open issues without `approved` or `rejected` label
- Evaluate: Is it real? Is it valuable? Is it clear enough for a builder? Is it a duplicate?
- Approve: add `approved` label. Reject: add `rejected` label with reason comment.
- Prioritize: add `P0` (critical), `P1` (important), `P2` (nice-to-have) label
- Size: add `XS`, `S`, `M`, `L` label
- Reject if: duplicate, out of scope, too vague, too large (needs decomposition), already fixed
- Decompose large issues into smaller approved sub-issues
- Quality bar: a builder should be able to start immediately from the issue description
- **Stale cleanup**: if an approved issue has `in-progress` label but no linked PR after 30 min, remove `in-progress` label (builder probably died)

### Builders — Fix failing PRs first, then build from approved issues. Code only, no tests.

#### Build (×3)

- **Priority 1 — Fix failing PRs**: Find open PRs with failing CI that do NOT have the `fixing` label. Claim it by adding `fixing` label. Read the failure logs, fix the code, push to the PR branch. Remove `fixing` label when done.
- **Priority 2 — Build from approved issues**: Find open issues with `approved` label that do NOT have `in-progress` label. Claim it immediately by adding `in-progress` label — this prevents other builders from picking it. Reference the issue in your PR: `Closes #N`.
- Write code only. Tests will be added by dedicated Testers on your PR.
- Features: pages, components, flows, interactions, animations, transitions
- UI/UX: responsive, mobile-first, touch, gestures, dark mode, theming
- Backend: API routes, server actions, business logic, middleware, webhooks, queues
- Database: schema, migrations, queries, indexes, seeds, relationships, transactions
- Auth: login, signup, OAuth, SSO, sessions, roles, permissions, MFA
- Payments: checkout, subscriptions, invoicing, refunds, payment gateway integration
- Content: CMS, templates, dynamic content, rich text, media handling, uploads
- Integrations: third-party APIs, webhooks, OAuth, email, push notifications, SMS
- Real-time: WebSocket, SSE, live updates, multiplayer, presence, sync
- Sharing: social cards, OG images, deep links, URL shortening, embeds
- Infrastructure: Docker, CI/CD, environment, secrets, deployment, monitoring, logging
- Error handling, edge cases, loading states, empty states, offline, fallbacks
- If no failing PRs and no approved issues: exit. Do not self-direct.

### Testers — Write tests for open PRs. Adversarial mindset.

#### Test (×1)

- Find open PRs that lack tests. Read the PR diff. Understand what changed.
- Write tests that BREAK it — adversarial, not confirmatory.
- Push tests to the PR branch directly, or open a companion PR targeting the same branch.
- Test types (choose based on the change):
  - Unit: business logic, edge cases, boundary values, null/undefined, race conditions, async
  - Integration: API contracts, database queries, auth flow, cross-module data flow
  - E2E: critical user journeys, mobile viewport, first-time experience, error recovery
- Focus: what would a user hit? What edge case did the builder miss? What breaks under load?
- Skip PRs that already have adequate tests. One PR per session.

### Reviewers — Quality gate + merge authority.

#### Review (×1)

- Correctness: logic errors, edge cases, race conditions, error handling, null safety
- Security: injection vectors, data exposure, auth issues, dependency risks, secrets
- Performance: complexity, bundle impact, query cost, re-renders, memory, caching
- Tests: if PR has no tests, comment "needs tests" — do NOT reject, testers will add them
- Scope: focused PR, single concern, no unrelated changes, appropriate size, clear description
- Style: naming, consistency, patterns, readability, documentation, commit messages
- Merge rule: approve + merge (squash, delete branch) ONLY when PR has both code AND tests with passing CI. If tests are missing, approve the code but wait for testers.

## Separate Testing Model

Builders write code only — no tests. Testers independently write tests for open PRs. This is a deliberate design choice that differs from traditional open-source practice where PR authors write their own tests.

### Why This Works for AI Teams

| Factor | Traditional OSS | AI Team |
|--------|----------------|---------|
| **Ego/ownership** | Authors defend their code, resist external test changes | AI has no ego — welcomes any test contribution |
| **Blind spots** | Author tests confirm their own mental model | Fresh eyes read the diff cold, catch what the author missed |
| **Testing approach** | Confirmatory ("prove it works") | Adversarial ("try to break it") |
| **Workflow** | Sequential (write code → write tests → submit) | Parallel (builder moves to next issue, tester works on PR) |
| **Volunteer availability** | Hard to find human test volunteers | AI testers are always available, always willing |

### Trade-offs

- **Intent misunderstanding**: the tester may not fully grasp the builder's design intent, leading to tests that validate the wrong behavior
- **Implementation-specific edge cases**: the author knows which shortcuts they took — a separate tester might miss those specific weak spots
- **Communication overhead**: occasionally requires review comments to clarify intent

These trade-offs are acceptable because Reviewers serve as the final quality gate, catching misalignment between code and tests before merge.

## Role Design Rationale

### Why Audit replaces Improve + Secure + Perf?
Code quality, security, and performance are deeply intertwined. A refactoring that removes dead code also reduces attack surface and improves bundle size. A security fix often improves error handling. Performance optimization often means better code structure. One agent with the full picture creates better, more holistic issues than three agents with tunnel vision.

### Why 3 Builders?
AI is fullstack — no need to split frontend/backend/API. One Builder can touch anything. 3 provides sufficient throughput; the quality bottleneck is Review and Triage, not raw code output.

### Why a single generic Tester instead of Unit/Integration/E2E split?
AI can write all test types. The split was artificial — a tester reading a PR diff naturally knows whether it needs unit tests, integration tests, or E2E tests. One agent picks the right type based on the change.

### Why Triage?
Without Triage, builders pick up vague or duplicate issues and waste cycles. Triage ensures every issue a builder sees is real, valuable, clear, and appropriately sized. It also handles stale cleanup when builders die mid-work.

### Why Review merges?
The reviewer already has full context from reading the code. They're the natural merge point. This removes merge responsibility from the coordinator entirely.

### No Sentinel?
Production health monitoring moved to the coordinator itself — a simple CI check every 5 minutes. If CI is broken, the coordinator spawns a one-off fix agent. No permanent Sentinel role needed.

## Brief Philosophy

Each role brief is a **keyword list** covering all aspects of that direction:
- **Minimum words, maximum coverage** — every bullet is a cluster of related keywords
- **Generic, not project-specific** — works for any codebase, any product
- **Comprehensive** — covers ALL aspects the direction handles, not just the obvious ones
- **Scannable** — AI can quickly parse keyword lists to understand scope

What briefs deliberately omit:
- Step-by-step procedures
- Tool-specific commands
- Checklists or runbooks
- Project-specific references

Project context is **separate**. Role briefs define **capability scope**. The combination gives each agent both domain expertise and product intuition.
