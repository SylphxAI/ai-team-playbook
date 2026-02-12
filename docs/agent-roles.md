# Agent Roles Reference — V4

> 25 specialized roles, 38 agent instances across 10 departments.
> Agent briefs tell WHAT, not HOW. AI can think — give it a mission, not a script.

---

## Roster

| # | Role | Key | Dept | Mode | × |
|---|------|-----|------|------|---|
| 1 | Product Designer | `product_designer` | Product | Thinker | 1 |
| 2 | System Architect | `system_architect` | Product | Thinker | 1 |
| 3 | UI/UX Designer | `uiux_designer` | Product | Thinker | 1 |
| 4 | UI Builder | `ui_builder` | Engineering | Task-driven | 3 |
| 5 | API Builder | `api_builder` | Engineering | Task-driven | 3 |
| 6 | Fullstack Builder | `fullstack_builder` | Engineering | Task-driven | 3 |
| 7 | Unit Tester | `unit_tester` | Quality | Self-directed | 2 |
| 8 | E2E Tester | `e2e_tester` | Quality | Self-directed | 2 |
| 9 | Security Auditor | `security_auditor` | Security | Self-directed | 1 |
| 10 | Dependency Manager | `dependency_manager` | Security | Self-directed | 1 |
| 11 | Frontend Optimizer | `frontend_optimizer` | Performance | Self-directed | 1 |
| 12 | Backend Optimizer | `backend_optimizer` | Performance | Self-directed | 1 |
| 13 | Mobile Optimizer | `mobile_optimizer` | Performance | Self-directed | 1 |
| 14 | Refactorer | `refactorer` | Code Health | Self-directed | 1 |
| 15 | Type Hardener | `type_hardener` | Code Health | Self-directed | 1 |
| 16 | Code Cleaner | `code_cleaner` | Code Health | Self-directed | 1 |
| 17 | Web Standards Engineer | `web_standards_engineer` | Standards | Self-directed | 1 |
| 18 | Documentation Engineer | `documentation_engineer` | Standards | Self-directed | 1 |
| 19 | CI/CD Engineer | `cicd_engineer` | Infrastructure | Self-directed | 1 |
| 20 | Code Reviewer | `code_reviewer` | Gates | Reviewer | 4 |
| 21 | Sentinel | `sentinel` | Gates | Sentinel | 1 |
| 22 | Growth Strategist | `growth_strategist` | Growth | Thinker | 1 |
| 23 | Growth Engineer | `growth_engineer` | Growth | Task-driven | 2 |
| 24 | Content Creator | `content_creator` | Growth | Self-directed | 2 |
| 25 | Analytics Engineer | `analytics_engineer` | Growth | Self-directed | 1 |

**25 roles → 38 instances** (UI Builder ×3, API Builder ×3, Fullstack Builder ×3, Unit Tester ×2, E2E Tester ×2, Growth Engineer ×2, Content Creator ×2, Code Reviewer ×4)

---

## Agent Modes

| Mode | Discovers work by | Produces | Exits when |
|------|-------------------|----------|------------|
| **Thinker** | Holistic analysis of codebase/product | Issues (never code) | Issues created or nothing needed |
| **Task-driven** | Open Issues first, then self-directed scan | PRs | PR submitted or nothing to build |
| **Self-directed** | Specialist analysis of codebase | PRs | PR submitted or nothing to improve |
| **Reviewer** | Open PR queue | Reviews (approve/request changes) | PR reviewed or queue empty |
| **Sentinel** | HTTP health check | Revert PRs (if production is down) | Health check complete |

---

## Product Department (3 agents)

Thinkers who create the demand signal. They analyze and produce Issues — never code.

**Product Designer** — Identifies feature gaps, user needs, and product opportunities from a business and UX perspective.

**System Architect** — Evaluates codebase structure, data models, API contracts, and service boundaries. Produces architectural Issues or ADRs.

**UI/UX Designer** — Finds UX problems: confusing flows, missing states, poor responsiveness, inconsistent interaction patterns.

---

## Engineering Department (9 agents)

Builders who consume Issues and ship code. Fall back to self-directed work when no Issues exist.

**UI Builder (×3)** — Implements React components, pages, and client-side features. Three instances because frontend has the most surface area.

**API Builder (×3)** — Implements API routes, server actions, and backend business logic. Three instances to parallel frontend velocity.

**Fullstack Builder (×3)** — Handles cross-cutting work: database schemas, migrations, infrastructure config, Docker, deployment, and features that span the full stack. Replaces the old specialized Database Engineer and Infra Builder roles with a more flexible generalist.

---

## Quality Department (4 agents)

Test coverage and correctness across all levels.

**Unit Tester (×2)** — Writes focused unit tests for utilities, hooks, and business logic. Prioritizes critical paths over trivial coverage.

**E2E Tester (×2)** — Writes Playwright tests simulating real user flows. Also covers integration-level testing between modules and API contracts.

---

## Security Department (2 agents)

Proactive vulnerability scanning and dependency safety.

**Security Auditor** — Audits for XSS, injection, CSRF, auth bypass, data exposure, and missing security headers.

**Dependency Manager** — Scans for CVEs, updates vulnerable or outdated packages, removes unused dependencies, and checks license compliance. Consolidates the old Dependency Auditor and Dependency Updater roles.

---

## Performance Department (3 agents)

Frontend, backend, and mobile performance optimization.

**Frontend Optimizer** — Improves bundle size, load times, Core Web Vitals, code splitting, and rendering efficiency.

**Backend Optimizer** — Fixes N+1 queries, adds caching, parallelizes sequential awaits, optimizes database access patterns.

**Mobile Optimizer** — Ensures responsive performance, touch interactions, mobile-specific rendering paths, and efficient resource loading on constrained devices.

---

## Code Health Department (3 agents)

Continuous structural improvement — keep the codebase clean and maintainable.

**Refactorer** — Reduces duplication, splits oversized files, moves misplaced logic, and unifies inconsistent patterns. All changes are behavior-preserving.

**Type Hardener** — Eliminates `any` types, removes `@ts-ignore`, tightens type safety, and aligns Zod schemas with TypeScript types.

**Code Cleaner** — Removes dead code, fixes error handling gaps, and migrates deprecated patterns to modern replacements. Consolidates the old Dead Code Cleaner, Error Handler, and Deprecation Migrator roles.

---

## Standards Department (2 agents)

Documentation and web standards compliance.

**Web Standards Engineer** — Handles SEO (meta tags, Open Graph, structured data), accessibility (WCAG compliance, ARIA, keyboard navigation), and internationalization (i18n infrastructure, string extraction, locale routing). Consolidates the old SEO, Accessibility, and i18n Engineer roles.

**Documentation Engineer** — Improves README, architecture docs, API documentation, setup guides, and inline comments. Creates or extends OpenAPI specs. Consolidates the old Technical Writer and API Documenter roles.

---

## Infrastructure Department (1 agent)

**CI/CD Engineer** — Optimizes build caching, parallelizes CI jobs, configures conditional runs, and maintains preview deployments.

---

## Gates Department (5 agents)

Quality gates and production safety.

**Code Reviewer (×4)** — Reviews open PRs for correctness, types, error handling, security, performance, scope, and breaking changes. Four instances to keep the review queue moving. Approves good PRs, requests changes on real issues.

**Sentinel** — Monitors production health via HTTP checks. If production breaks after a recent merge, creates a revert PR. Never force-pushes to `main`.

---

## Growth Department (6 agents)

User acquisition, engagement, and data-driven product evolution.

**Growth Strategist** — Analyzes the product for growth opportunities: viral loops, onboarding friction, retention hooks, conversion funnels. Produces Issues, never code. (Thinker mode.)

**Growth Engineer (×2)** — Implements growth features: referral systems, onboarding flows, A/B test infrastructure, engagement mechanics. Two instances to maintain velocity on growth initiatives.

**Content Creator (×2)** — Produces and optimizes content: landing pages, copy, social proof, email templates, in-app messaging. Two instances to cover both acquisition and retention content.

**Analytics Engineer** — Instruments event tracking, builds dashboards, implements conversion funnels, and ensures data collection supports growth decisions.

---

## V4 Changes from V3

### Merged Roles (fewer, broader specialists)

| V3 Roles | V4 Role | Rationale |
|----------|---------|-----------|
| i18n Engineer + SEO Engineer + Accessibility Engineer | **Web Standards Engineer** | All web standards concerns under one specialist |
| Dead Code Cleaner + Deprecation Migrator + Error Handler | **Code Cleaner** | All code hygiene under one specialist |
| Dependency Auditor + Dependency Updater | **Dependency Manager** | Audit and fix are one workflow |
| Technical Writer + API Documenter | **Documentation Engineer** | All documentation under one specialist |
| Database Engineer + Infra Builder | **Fullstack Builder** | Cross-cutting work benefits from full-stack context |
| Integration Tester | **E2E Tester (×2)** | Integration testing absorbed into expanded E2E scope |

### Added Roles

| Role | Why |
|------|-----|
| Growth Strategist | Product growth needs dedicated strategic thinking |
| Growth Engineer | Growth features need dedicated build capacity |
| Content Creator | Content is a growth lever that needs continuous attention |
| Analytics Engineer | Data-driven decisions need instrumentation |
| Mobile Optimizer | Mobile performance is distinct from desktop optimization |
| Fullstack Builder | Cross-cutting work (DB, infra, full-stack features) needs flexible generalists |

### Scaled Up

| Role | V3 | V4 | Why |
|------|----|----|-----|
| UI Builder | ×2 | ×3 | Frontend surface area growth |
| API Builder | ×2 | ×3 | Backend feature velocity |
| Unit Tester | ×1 | ×2 | Test coverage throughput |
| E2E Tester | ×1 | ×2 | Expanded scope (includes integration) |
| Code Reviewer | ×3 | ×4 | Keep review queue moving with higher PR volume |

---

*See also: [V4 Architecture](pipeline-v4-architecture.md) · [Coordinator Guide](coordinator-guide.md) · [V3 to V4 Migration](v3-to-v4-migration.md)*
