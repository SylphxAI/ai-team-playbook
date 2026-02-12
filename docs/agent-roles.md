# Agent Roles — V4

7 directions, 26 agents. Split by **cognitive mode**, not technical domain.

## Design Principles

- **AI is fullstack.** Don't split frontend/backend — AI has no skill boundaries.
- **Directions, not departments.** Each role is a direction of thought: build, test, improve, secure, optimize, review. Fewer categories, clearer purpose.
- **Generic briefs + separate project context.** Role briefs are project-agnostic keyword lists. Project context (users, competitors, vision) is prepended separately at spawn time.
- **Keywords, not narratives.** Minimum words, maximum coverage. Each bullet is a keyword cluster covering one aspect of the role.

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
| 2 | **Build** | `builder` | 12 |
| 3 | **Test — Unit** | `tester_unit` | 2 |
| 4 | **Test — Integration** | `tester_integration` | 1 |
| 5 | **Test — E2E** | `tester_e2e` | 2 |
| 6 | **Improve** | `improver` | 2 |
| 7 | **Secure** | `security` | 1 |
| 8 | **Perf** | `perf` | 1 |
| 9 | **Review** | `reviewer` | 4 |
| | **Total** | | **26** |

## Role Briefs

### Product (×1) — Create issues, not code.

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

### Build (×12) — Check issues first, self-direct if none.

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

### Test — Unit (×2)

- Business logic, state management, data transformations, calculations
- Edge cases, boundary values, null/undefined, NaN, empty, overflow
- Race conditions, async, promises, timeouts, retries
- Auth logic, permission checks, validation rules, sanitization
- Utility functions, formatters, parsers, serializers, converters
- Mocking, isolation, deterministic, fast, no side effects

### Test — Integration (×1)

- API contracts, request/response shapes, status codes, error formats, pagination
- Database queries, transactions, migrations, constraint violations, rollbacks
- Auth flow, token lifecycle, session management, refresh, revocation
- Cross-module data flow, event propagation, state consistency, cache invalidation
- Third-party service mocks, webhook handling, retry logic, circuit breakers
- File uploads, streaming, multipart, presigned URLs

### Test — E2E (×2)

- Critical user journeys, happy paths, error paths, recovery flows
- Mobile viewport, touch, orientation, responsive breakpoints, slow network
- First-time experience, onboarding, empty states, progressive disclosure
- Deep links, social sharing, referral landing, UTM tracking
- Cross-browser, visual regression, accessibility audit, keyboard-only
- Auth flows, payment flows, real-time features, file upload/download

### Improve (×2)

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

### Secure (×1)

- Injection: XSS, SQL, NoSQL, command, template, header, SSRF, path traversal
- Auth: bypass, privilege escalation, session fixation, token leakage, brute force
- Data: PII exposure, debug endpoints, error message leaks, IDOR, mass assignment
- Dependencies: CVEs, outdated packages, supply chain, typosquatting, lockfile integrity
- Transport: CORS, CSP, HSTS, cookie flags, HTTPS, certificate pinning
- Input: validation, sanitization, output encoding, file upload restrictions, size limits
- Rate limiting: enumeration, API abuse, DDOS vectors, account takeover
- Secrets: hardcoded keys, env leaks, .env exposure, git history, rotation policy
- Payment security: PCI compliance, tokenization, fraud detection, webhook verification

### Perf (×1) — Always measure before AND after.

- Core Web Vitals: LCP, FID, CLS, INP, TTFB
- Bundle: size analysis, tree shaking, code splitting, lazy loading, dynamic imports
- Assets: image optimization, WebP/AVIF, font loading, compression, CDN, caching headers
- Database: N+1 queries, missing indexes, query plans, connection pooling, pagination, denormalization
- Rendering: SSR, SSG, ISR, streaming, partial hydration, Suspense, concurrent features
- Network: HTTP/2, preload, prefetch, preconnect, compression, request waterfall, CDN
- Runtime: memory leaks, event listeners, animation perf, main thread blocking, workers
- Mobile: 3G/4G simulation, low-end devices, battery impact, data usage, offline caching

### Review (×4) — Quality gate + merge authority.

- Correctness: logic errors, edge cases, race conditions, error handling, null safety
- Security: injection vectors, data exposure, auth issues, dependency risks, secrets
- Performance: complexity, bundle impact, query cost, re-renders, memory, caching
- Tests: coverage, quality, relevance, flakiness, missing edge cases, mocking strategy
- Scope: focused PR, single concern, no unrelated changes, appropriate size, clear description
- Style: naming, consistency, patterns, readability, documentation, commit messages
- Action: approve and merge (squash, delete branch). Merge any approved PR with passing CI. Request specific changes on bad ones.

## Role Design Rationale

### Why 12 Builders?
AI is fullstack — no need to split frontend/backend/API. One Builder can touch anything. 12 provides throughput; Reviewers handle quality.

### Why "Improve" instead of separate refactoring, types, docs, standards roles?
One "improve" lens naturally covers code quality, types, dead code, error handling, docs, SEO, accessibility, and i18n. These overlap too much to justify separate agents. A senior engineer doing "make it better" covers all of them.

### Why testing stays split (3 sub-types)?
Unit tests never produce integration tests. E2E needs a browser runtime. Integration tests need cross-component knowledge. Genuinely distinct cognitive modes — they don't converge when repeated.

### Why Review merges?
The reviewer already has full context from reading the code. They're the natural merge point. This removes merge responsibility from the coordinator entirely.

### No Sentinel?
Production health monitoring moved to the coordinator itself — a simple HTTP check every 5 minutes. If the site is down, the coordinator spawns a one-off revert agent. No permanent Sentinel role needed.

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
