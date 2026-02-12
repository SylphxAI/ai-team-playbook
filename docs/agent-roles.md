# Agent Roles — V4 (Revised)

25 role types, 38 agents. Roles split by **workload**, not just domain.

## Roster

| # | Role | Key | Count | Type | Brief |
|---|------|-----|-------|------|-------|
| 1 | Product Designer | `product_designer` | 1 | Thinker | Product gaps, feature proposals, user stories |
| 2 | System Architect | `system_architect` | 1 | Thinker | Architecture issues, data models, API contracts, ADRs |
| 3 | UI/UX Designer | `uiux_designer` | 1 | Thinker | User flows, interaction patterns, component structure |
| 4 | Growth Strategist | `growth_strategist` | 1 | Thinker | Viral mechanics, sharing, engagement, retention |
| 5 | Content Creator | `content_creator` | 1 | Self-directed | Quiz content, questions, categories, topics |
| 6 | UI Builder | `ui_builder` | 3 | Task-driven | React components, pages, client features |
| 7 | API Builder | `api_builder` | 3 | Task-driven | API routes, server actions, business logic |
| 8 | Fullstack Builder | `fullstack_builder` | 2 | Task-driven | Features spanning frontend + backend |
| 9 | Database Engineer | `database_engineer` | 1 | Task-driven | Schema, migrations, queries, indexes |
| 10 | Infra Builder | `infra_builder` | 1 | Task-driven | CI workflows, Docker, deployment, env setup |
| 11 | Growth Engineer | `growth_engineer` | 1 | Self-directed | Sharing mechanics, referral, gamification |
| 12 | Analytics Engineer | `analytics_engineer` | 1 | Self-directed | Event tracking, funnels, metrics |
| 13 | Unit Tester | `unit_tester` | 2 | Self-directed | Unit test coverage, edge cases |
| 14 | Integration Tester | `integration_tester` | 1 | Self-directed | Cross-component tests, API contracts |
| 15 | E2E Tester | `e2e_tester` | 2 | Self-directed | Playwright user flow tests |
| 16 | Security Auditor | `security_auditor` | 1 | Self-directed | XSS, injection, CSRF, auth bypass, CVEs |
| 17 | Frontend Optimizer | `frontend_optimizer` | 1 | Self-directed | Bundle size, Core Web Vitals, lazy loading |
| 18 | Backend Optimizer | `backend_optimizer` | 1 | Self-directed | Queries, caching, N+1, response times |
| 19 | Mobile Optimizer | `mobile_optimizer` | 1 | Self-directed | Responsive, touch, mobile share sheets |
| 20 | Refactorer | `refactorer` | 1 | Self-directed | Code structure, DRY, design patterns |
| 21 | Type Hardener | `type_hardener` | 1 | Self-directed | Eliminate `any`, tighten types, Zod alignment |
| 22 | Code Cleaner | `code_cleaner` | 1 | Self-directed | Dead code, deprecated patterns, error handling |
| 23 | Dependency Manager | `dependency_manager` | 1 | Self-directed | CVE audit, package updates, breaking changes |
| 24 | Documentation Engineer | `doc_engineer` | 1 | Self-directed | README, architecture docs, API docs |
| 25 | Web Standards Engineer | `web_standards` | 1 | Self-directed | SEO, accessibility (WCAG), i18n |
| 26 | CI/CD Engineer | `cicd_engineer` | 1 | Self-directed | GitHub Actions, build caching, pipeline speed |
| 27 | Code Reviewer | `code_reviewer` | 4 | Reviewer | Review PRs, quality gate |
| 28 | Sentinel | `sentinel` | 1 | Sentinel | Production health, auto-revert |

## Agent Types

- **Thinkers**: Create issues, don't write code. Decompose complex features into independent subtasks.
- **Task-driven**: Check issues first, self-direct if none.
- **Self-directed**: Analyze codebase through their lens, find work, PR it.
- **Reviewer**: Review open PRs. Approve or request changes.
- **Sentinel**: Monitor production. Revert if broken.

## Design Principles

- **Split by workload**: High-workload roles (E2E tests, builders) get multiple instances. Low-workload roles (SEO, i18n, accessibility) are merged.
- **Merged roles**: Web Standards = SEO + accessibility + i18n. Code Cleaner = dead code + deprecation + error handling. Dependency Manager = audit + update. Documentation Engineer = tech writer + API docs.
- **Nobody waits**: Every agent is always self-directed. Issues are bonus, not prerequisite.

## Prompt Philosophy

Agent briefs are 1-2 sentences. AI has thinking ability — tell WHAT, not HOW.

> "You're a Security Auditor for SylphxAI/viral. Audit for vulnerabilities, fix what you find, PR it."

That's enough. Don't teach AI how to git clone.
