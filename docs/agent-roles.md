# Agent Roles Reference — V4

> Complete reference for all 28 specialized roles across 10 departments.

---

## Role Summary

| # | Role | Department | Mode | Instances |
|---|------|-----------|------|-----------|
| 1 | Product Designer | Product | Thinker | 1 |
| 2 | System Architect | Product | Thinker | 1 |
| 3 | UI/UX Designer | Product | Thinker | 1 |
| 4 | UI Builder | Engineering | Task-driven | 2 |
| 5 | API Builder | Engineering | Task-driven | 2 |
| 6 | Database Engineer | Engineering | Task-driven | 1 |
| 7 | Infra Builder | Engineering | Task-driven | 1 |
| 8 | Unit Tester | Quality | Self-directed | 1 |
| 9 | Integration Tester | Quality | Self-directed | 1 |
| 10 | E2E Tester | Quality | Self-directed | 1 |
| 11 | Security Auditor | Security | Self-directed | 1 |
| 12 | Dependency Auditor | Security | Self-directed | 1 |
| 13 | Frontend Optimizer | Performance | Self-directed | 1 |
| 14 | Backend Optimizer | Performance | Self-directed | 1 |
| 15 | Refactorer | Code Health | Self-directed | 1 |
| 16 | Type Hardener | Code Health | Self-directed | 1 |
| 17 | Dead Code Cleaner | Code Health | Self-directed | 1 |
| 18 | Error Handler | Code Health | Self-directed | 1 |
| 19 | Deprecation Migrator | Maintenance | Self-directed | 1 |
| 20 | Dependency Updater | Maintenance | Self-directed | 1 |
| 21 | Technical Writer | Standards | Self-directed | 1 |
| 22 | API Documenter | Standards | Self-directed | 1 |
| 23 | SEO Engineer | Standards | Self-directed | 1 |
| 24 | Accessibility Engineer | Standards | Self-directed | 1 |
| 25 | i18n Engineer | Standards | Self-directed | 1 |
| 26 | CI/CD Engineer | Infrastructure | Self-directed | 1 |
| 27 | Code Reviewer | Gates | Reviewer | 3 |
| 28 | Sentinel | Gates | Sentinel | 1 |

**28 unique roles → 32 agent instances** (UI Builder ×2, API Builder ×2, Code Reviewer ×3)

---

## Agent Modes

| Mode | How Work Is Found | Creates | Exits When |
|------|-------------------|---------|------------|
| **Thinker** | Analyzes codebase and product holistically | Issues (never code) | Issues created |
| **Task-driven** | Checks Issues first, falls back to self-directed | PRs (code changes) | PR submitted |
| **Self-directed** | Analyzes codebase through specialist lens | PRs (code changes) | PR submitted or nothing to do |
| **Reviewer** | Checks open PR queue | Reviews (approve/request changes) | PR reviewed or no PRs pending |
| **Sentinel** | HTTP health check on production | Revert PRs (if production is down) | Health check complete |

---

## Product Department (3 agents)

The Thinkers. They create the **demand signal** by generating well-scoped Issues. They analyze the product from different angles and never write code.

### Product Designer

| Field | Value |
|-------|-------|
| **Key** | `product_designer` |
| **Mode** | Thinker |
| **Instances** | 1 |
| **Department** | Product |

**Description:** Analyzes the product from a business and user perspective. Thinks about what users need, what features drive engagement, and what's missing from the product experience.

**Lens:** User stories, product strategy, feature gaps, user acquisition/retention/engagement flows.

**Produces:** GitHub Issues with user stories, acceptance criteria, scope definitions, and priority. Labels: `feature`, `enhancement`, `ui`, `api`, `db`.

**Exits when:** 1–3 Issues created, or all valuable features already have Issues.

---

### System Architect

| Field | Value |
|-------|-------|
| **Key** | `system_architect` |
| **Mode** | Thinker |
| **Instances** | 1 |
| **Department** | Product |

**Description:** Analyzes the codebase architecture — data models, API contracts, service boundaries, state management. Identifies structural improvements and documents architectural decisions.

**Lens:** Data model design, API contract consistency, service layer organization, tight coupling, scalability bottlenecks, authentication/authorization patterns.

**Produces:** GitHub Issues for architectural improvements (prefixed `arch:`), or Architecture Decision Records (ADRs) submitted as PRs under `docs/adr/`.

**Exits when:** 1–3 architectural Issues created or an ADR submitted, or architecture is clean.

---

### UI/UX Designer

| Field | Value |
|-------|-------|
| **Key** | `uiux_designer` |
| **Mode** | Thinker |
| **Instances** | 1 |
| **Department** | Product |

**Description:** Analyzes user flows, component hierarchy, and interaction patterns. Identifies UX problems — confusing flows, missing states, poor responsive behavior, inconsistent design.

**Lens:** Component decomposition, user flow completeness, loading/error/empty states, responsive design, form design, interaction feedback, visual consistency.

**Produces:** GitHub Issues describing UI/UX improvements with current behavior, problem statement, proposed solution, and affected components. Labels: `ui`, `ux`, `design`.

**Exits when:** 1–3 UX Issues created, or the UI is polished.

---

## Engineering Department (6 agents)

The Builders. They consume Issues and build features. If no Issues exist, they analyze the codebase and find work independently.

### UI Builder

| Field | Value |
|-------|-------|
| **Key** | `ui_builder` |
| **Mode** | Task-driven |
| **Instances** | 2 |
| **Department** | Engineering |

**Description:** Implements React components, pages, and client-side features. Two instances because frontend has the most surface area.

**Lens:** Open Issues labeled `ui`. If none: missing pages, oversized components, hardcoded strings, missing loading/error states, inconsistent styling.

**Produces:** PRs with React component implementations, page layouts, client-side state management, form handling, styling. Branch: `ui-builder/{description}`.

**Exits when:** PR submitted, or genuinely nothing to build.

---

### API Builder

| Field | Value |
|-------|-------|
| **Key** | `api_builder` |
| **Mode** | Task-driven |
| **Instances** | 2 |
| **Department** | Engineering |

**Description:** Implements API routes, server actions, and backend business logic. Two instances because backend features often parallel frontend work.

**Lens:** Open Issues labeled `api` or `backend`. If none: missing endpoints, duplicated business logic, missing input validation, inconsistent error responses.

**Produces:** PRs with API route handlers, server actions, input validation, business logic, auth guards. Branch: `api-builder/{description}`.

**Exits when:** PR submitted, or genuinely nothing to build.

---

### Database Engineer

| Field | Value |
|-------|-------|
| **Key** | `database_engineer` |
| **Mode** | Task-driven |
| **Instances** | 1 |
| **Department** | Engineering |

**Description:** Handles database schemas, migrations, query optimization, and data modeling.

**Lens:** Open Issues labeled `db` or `database`. If none: missing indexes, N+1 query patterns, schema design issues, Zod schema drift from database models.

**Produces:** PRs with schema changes, migration files, query optimizations, database utility functions, seed scripts. Branch: `database-engineer/{description}`.

**Exits when:** PR submitted, or schema and queries are clean.

---

### Infra Builder

| Field | Value |
|-------|-------|
| **Key** | `infra_builder` |
| **Mode** | Task-driven |
| **Instances** | 1 |
| **Department** | Engineering |

**Description:** Handles Docker configuration, CI/CD workflows, deployment config, and environment setup.

**Lens:** Open Issues labeled `infra` or `devops`. If none: unoptimized Dockerfile, missing CI steps, incomplete `.env.example`, build bottlenecks.

**Produces:** PRs with Dockerfile improvements, GitHub Actions workflows, environment configuration, build optimization, deployment scripts. Branch: `infra-builder/{description}`.

**Exits when:** PR submitted, or infrastructure is solid.

---

## Quality Department (3 agents)

Test coverage and correctness verification across unit, integration, and end-to-end levels.

### Unit Tester

| Field | Value |
|-------|-------|
| **Key** | `unit_tester` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Quality |

**Description:** Finds untested modules and writes focused unit tests. Prioritizes utilities, hooks, and business logic over trivial UI components.

**Lens:** Source files without corresponding test files. Untested edge cases. Missing error path coverage. Critical business logic without assertions.

**Produces:** PRs with unit test files. Branch: `unit-tester/{module}`.

**Exits when:** Tests submitted, or test coverage is comprehensive.

---

### Integration Tester

| Field | Value |
|-------|-------|
| **Key** | `integration_tester` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Quality |

**Description:** Tests how modules work together — API routes with databases, components with hooks, form submission flows.

**Lens:** API contracts (input → output), component + hook interactions, error propagation across module boundaries, auth flow integrity.

**Produces:** PRs with integration test files. Branch: `integration-tester/{integration}`.

**Exits when:** Tests submitted, or all critical integrations are covered.

---

### E2E Tester

| Field | Value |
|-------|-------|
| **Key** | `e2e_tester` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Quality |

**Description:** Writes Playwright end-to-end tests simulating real user behavior — page loads, navigation, form submissions, core feature interactions.

**Lens:** Critical user flows without E2E coverage. Homepage, auth flows, main feature interactions, navigation between pages, form submissions.

**Produces:** PRs with Playwright test files and page object models. Branch: `e2e-tester/{flow}`.

**Exits when:** Tests submitted, or critical flows are covered.

---

## Security Department (2 agents)

Proactive security scanning and dependency safety.

### Security Auditor

| Field | Value |
|-------|-------|
| **Key** | `security_auditor` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Security |

**Description:** Systematically audits the codebase for security vulnerabilities and fixes them. Focuses on one vulnerability category per session.

**Lens:** XSS (`dangerouslySetInnerHTML`, unescaped input), injection (SQL/command), CSRF, auth bypass, data exposure, session management, input validation, missing security headers (CSP, HSTS).

**Produces:** PRs fixing security vulnerabilities with clear explanations of the attack vector. Branch: `security-auditor/{vulnerability-type}`.

**Exits when:** Fix submitted, or no vulnerabilities found.

---

### Dependency Auditor

| Field | Value |
|-------|-------|
| **Key** | `dependency_auditor` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Security |

**Description:** Scans for vulnerable, outdated, or problematic dependencies. CVE scanning, license compliance, unused dependency detection.

**Lens:** Known CVEs (`bun audit`), unused packages (imported nowhere), duplicate packages serving the same purpose, copyleft licenses in commercial projects.

**Produces:** PRs updating vulnerable packages, removing unused dependencies, or flagging license issues. Branch: `dependency-auditor/{package-or-issue}`.

**Exits when:** Fix submitted, or all dependencies are clean.

---

## Performance Department (2 agents)

Frontend and backend performance optimization.

### Frontend Optimizer

| Field | Value |
|-------|-------|
| **Key** | `frontend_optimizer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Performance |

**Description:** Improves frontend performance — bundle size, load times, Core Web Vitals, rendering efficiency.

**Lens:** Heavy library imports, unoptimized images, missing code splitting, client components that could be server components, missing memoization, font loading, deferred third-party scripts.

**Produces:** PRs with performance optimizations and before/after impact notes. Branch: `frontend-optimizer/{optimization}`.

**Exits when:** Optimization submitted, or frontend is already well-optimized.

---

### Backend Optimizer

| Field | Value |
|-------|-------|
| **Key** | `backend_optimizer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Performance |

**Description:** Improves backend performance — query speed, caching, response times, connection management.

**Lens:** N+1 queries, sequential awaits that could be parallel, missing caching, unoptimized WHERE clauses, missing pagination, connection pooling issues.

**Produces:** PRs with backend performance improvements. Branch: `backend-optimizer/{optimization}`.

**Exits when:** Optimization submitted, or backend performance is solid.

---

## Code Health Department (4 agents)

Continuous code quality improvement — structure, types, cleanliness, and error handling.

### Refactorer

| Field | Value |
|-------|-------|
| **Key** | `refactorer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Code Health |

**Description:** Improves code structure — reduces duplication, splits god files, moves misplaced logic, unifies inconsistent patterns. All refactorings are behavior-preserving.

**Lens:** DRY violations, files over 300 lines, business logic in UI components, naming inconsistencies, deep nesting, magic values.

**Produces:** PRs with structural improvements (no behavior changes). Branch: `refactorer/{what-was-refactored}`.

**Exits when:** Refactoring submitted, or code structure is clean.

---

### Type Hardener

| Field | Value |
|-------|-------|
| **Key** | `type_hardener` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Code Health |

**Description:** Eliminates `any` types, tightens type safety, aligns Zod schemas with TypeScript types, removes `@ts-ignore` by fixing the underlying issue.

**Lens:** `any` types, type assertions (`as Type`), `@ts-ignore`/`@ts-expect-error` comments, untyped function parameters, Zod/Prisma schema drift.

**Produces:** PRs with type improvements (no behavior changes). Branch: `type-hardener/{what-was-typed}`.

**Exits when:** Types tightened, or type safety is already strong.

---

### Dead Code Cleaner

| Field | Value |
|-------|-------|
| **Key** | `dead_code_cleaner` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Code Health |

**Description:** Finds and removes unused code — imports, exports, functions, files, commented-out code, empty modules.

**Lens:** Unused imports, unused exports (exported but never imported elsewhere), orphan files, commented-out code blocks, empty files, unused CSS.

**Produces:** PRs removing dead code with safety verification that removed items are truly unused. Branch: `dead-code-cleaner/{what-was-removed}`.

**Exits when:** Dead code removed, or codebase is lean.

---

### Error Handler

| Field | Value |
|-------|-------|
| **Key** | `error_handler` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Code Health |

**Description:** Improves error handling — adds error boundaries, fixes silent failures, standardizes error responses, improves user-facing error messages.

**Lens:** Unhandled promise rejections, empty catch blocks, API routes without error handling, missing React error boundaries, generic "Something went wrong" messages, console.log used for error reporting.

**Produces:** PRs with error handling improvements. Branch: `error-handler/{what-was-fixed}`.

**Exits when:** Error handling improved, or error handling is comprehensive.

---

## Maintenance Department (2 agents)

Keeping the codebase modern and dependencies current.

### Deprecation Migrator

| Field | Value |
|-------|-------|
| **Key** | `deprecation_migrator` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Maintenance |

**Description:** Updates deprecated patterns, APIs, and library usage to their modern replacements. Class components to hooks, pages router to app router, CommonJS to ESM.

**Lens:** Deprecated React patterns (class components, `UNSAFE_` methods), deprecated Next.js patterns (`getServerSideProps` in app router), deprecated Node.js APIs, `var` declarations, `require()` imports.

**Produces:** PRs migrating deprecated patterns (behavior-preserving). Branch: `deprecation-migrator/{what-was-migrated}`.

**Exits when:** Migration submitted, or everything is already modern.

---

### Dependency Updater

| Field | Value |
|-------|-------|
| **Key** | `dependency_updater` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Maintenance |

**Description:** Keeps dependencies up to date. Updates one package (or a small group of related ones) per session, resolving any breaking changes.

**Lens:** Outdated packages (`bun outdated`), security updates (any severity), major framework updates (Next.js, React), major utility updates.

**Produces:** PRs updating dependencies with changelog links and breaking change resolutions. Branch: `dependency-updater/{package}-{version}`.

**Exits when:** Update submitted, or all dependencies are current.

---

## Standards Department (5 agents)

Documentation, SEO, accessibility, and internationalization.

### Technical Writer

| Field | Value |
|-------|-------|
| **Key** | `technical_writer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Standards |

**Description:** Improves documentation — README, architecture docs, setup guides, inline comments, environment variable documentation, contributing guides.

**Lens:** README completeness, missing setup instructions, undocumented architecture decisions, complex logic without comments, incomplete `.env.example`.

**Produces:** PRs with documentation improvements. Branch: `technical-writer/{what-was-documented}`.

**Exits when:** Documentation improved, or docs are comprehensive.

---

### API Documenter

| Field | Value |
|-------|-------|
| **Key** | `api_documenter` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Standards |

**Description:** Documents API routes — request/response shapes, error cases, authentication requirements. Creates or extends OpenAPI specs.

**Lens:** Undocumented API endpoints, missing OpenAPI spec, type exports that aren't documented, missing example requests.

**Produces:** PRs with API documentation, OpenAPI spec additions, endpoint reference docs. Branch: `api-documenter/{endpoints}`.

**Exits when:** API docs updated, or all endpoints are documented.

---

### SEO Engineer

| Field | Value |
|-------|-------|
| **Key** | `seo_engineer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Standards |

**Description:** Improves search engine optimization — meta tags, Open Graph, Twitter Cards, sitemap, robots.txt, structured data (JSON-LD), canonical URLs.

**Lens:** Missing or incomplete meta tags, missing Open Graph/Twitter Card tags, no sitemap, no robots.txt, missing JSON-LD structured data, missing alt text on images.

**Produces:** PRs with SEO improvements. Branch: `seo-engineer/{what-was-improved}`.

**Exits when:** SEO improvement submitted, or SEO is comprehensive.

---

### Accessibility Engineer

| Field | Value |
|-------|-------|
| **Key** | `accessibility_engineer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Standards |

**Description:** Ensures WCAG accessibility compliance — alt text, heading hierarchy, keyboard navigation, ARIA labels, focus management, semantic HTML, color contrast.

**Lens:** Missing alt text, broken heading hierarchy, non-keyboard-accessible elements, missing ARIA labels, missing form labels, div-soup instead of semantic HTML, missing skip navigation, poor color contrast.

**Produces:** PRs with accessibility fixes referencing WCAG guidelines. Branch: `accessibility-engineer/{what-was-fixed}`.

**Exits when:** A11y fix submitted, or the app is accessible.

---

### i18n Engineer

| Field | Value |
|-------|-------|
| **Key** | `i18n_engineer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Standards |

**Description:** Implements internationalization infrastructure — translation key extraction, locale routing, message files, locale-aware formatting.

**Lens:** Hardcoded UI strings, missing i18n setup, translation keys without translations, missing locale routing, hardcoded date/number formatting.

**Produces:** PRs with i18n infrastructure or string extraction. Branch: `i18n-engineer/{what-was-internationalized}`.

**Exits when:** i18n improvement submitted, or i18n is set up and complete.

---

## Infrastructure Department (1 agent)

### CI/CD Engineer

| Field | Value |
|-------|-------|
| **Key** | `cicd_engineer` |
| **Mode** | Self-directed |
| **Instances** | 1 |
| **Department** | Infrastructure |

**Description:** Optimizes CI/CD pipelines — build caching, parallel jobs, conditional runs, build speed, preview deployments, status checks.

**Lens:** Missing dependency/build caching in CI, sequential jobs that could be parallel, unnecessary CI runs for unrelated file changes, slow builds, missing preview deployments.

**Produces:** PRs with CI/CD improvements. Branch: `cicd-engineer/{what-was-optimized}`.

**Exits when:** CI/CD improvement submitted, or pipelines are already optimal.

---

## Gates Department (4 agents)

Quality gates and production safety.

### Code Reviewer

| Field | Value |
|-------|-------|
| **Key** | `code_reviewer` |
| **Mode** | Reviewer |
| **Instances** | 3 |
| **Department** | Gates |

**Description:** Reviews open PRs for code quality, correctness, types, error handling, security, performance, test coverage, scope, and breaking changes. Three instances to keep the review queue moving.

**Lens:** Open PRs needing review (not yet approved, not draft). Reviews the oldest unreviewed PR first.

**Produces:** PR reviews — `APPROVE` for good PRs, `REQUEST_CHANGES` for real issues (bugs, security, broken functionality). Constructive feedback with specific suggestions.

**Review Checklist:**
- Code quality and project conventions
- Correctness (does it do what it claims?)
- TypeScript types (no unnecessary `any`)
- Error handling (graceful, not silent)
- Security (XSS, injection, auth bypass, data exposure)
- Performance (N+1 queries, unnecessary re-renders)
- Scope (focused, not doing too many things)
- Breaking changes

**Exits when:** PR reviewed, or no PRs need review.

---

### Sentinel

| Field | Value |
|-------|-------|
| **Key** | `sentinel` |
| **Mode** | Sentinel |
| **Instances** | 1 |
| **Department** | Gates |

**Description:** Monitors production health. Performs HTTP health checks on the live site. If production is broken and a recent merge is the likely cause, creates an automated revert PR.

**Lens:** Production HTTP status code. Recent merge history (last 2 hours).

**Produces:** Revert PRs when production breaks after a recent merge. Labels: `emergency`.

**Revert criteria:**
- Production returns 5xx or times out
- A PR was merged within the last 2 hours
- Revert is created as a normal PR (never force-push to `main`)

**Exits when:** Health check complete and production is healthy, or revert PR created.

---

*See also: [V4 Architecture](pipeline-v4-architecture.md) · [Coordinator Guide](coordinator-guide.md) · [V3 to V4 Migration](v3-to-v4-migration.md)*
