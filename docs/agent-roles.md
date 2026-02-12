# Agent Roles — V4

15 role types, 30 agents. Split by **cognitive mode**, not technical domain.

## Principle

> Will an agent doing this work **repeatedly** also cover the other type naturally?
> **Yes** → merge. **No** → keep separate.

## Roster

| # | Role | Key | Count | Type | Brief |
|---|------|-----|-------|------|-------|
| 1 | Product Designer | `product_designer` | 1 | Thinker | Product features, UX, architecture — holistic product thinking |
| 2 | Growth Strategist | `growth_strategist` | 1 | Thinker | Viral mechanics, sharing, engagement, retention |
| 3 | Builder | `builder` | 10 | Builder | Fullstack. Pick up issues or self-direct. AI has no skill boundaries |
| 4 | Content Creator | `content_creator` | 2 | Self-directed | Quiz content, questions, categories — not code |
| 5 | Unit Tester | `unit_tester` | 2 | Self-directed | Unit tests — repetitive, always more to write |
| 6 | Integration Tester | `integration_tester` | 1 | Self-directed | Cross-component tests — distinct from unit |
| 7 | E2E Tester | `e2e_tester` | 2 | Self-directed | Playwright browser tests — slow, time-consuming |
| 8 | Security Auditor | `security_auditor` | 1 | Self-directed | Adversarial thinking — XSS, injection, auth bypass, CVEs |
| 9 | Performance Optimizer | `perf_optimizer` | 1 | Self-directed | Profiling + optimization — frontend, backend, mobile |
| 10 | Code Improver | `code_improver` | 2 | Self-directed | Refactoring, types, dead code, errors, deprecation, docs |
| 11 | Web Standards | `web_standards` | 1 | Self-directed | SEO + accessibility + i18n |
| 12 | CI/CD Engineer | `cicd_engineer` | 1 | Self-directed | Pipeline optimization, build caching |
| 13 | Analytics Engineer | `analytics_engineer` | 1 | Self-directed | Event tracking, funnels, metrics |
| 14 | Code Reviewer | `code_reviewer` | 4 | Reviewer | Review PRs — different cognitive mode from building |
| 15 | Sentinel | `sentinel` | 1 | Sentinel | Production health, auto-revert |

## Why This Split

- **Builders are fullstack (×10):** AI has no skill boundaries. Don't split frontend/backend.
- **Code Improver, not 4 roles (×2):** One "improve code" lens finds refactoring + types + dead code + errors naturally.
- **Testing stays split (3 types):** Unit tests never produce integration tests. E2E needs a browser. Genuinely distinct.
- **Security stays separate:** Adversarial thinking is a different cognitive mode.
- **1-2 sentence briefs:** AI has thinking ability. Tell WHAT, not HOW.
