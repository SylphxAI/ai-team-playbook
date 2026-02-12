# Changelog

## v0.3.0 — 2026-02-12

### V4 roster update: 25 roles, 38 agents

**Roster changes:**
- Merged roles: i18n+SEO+Accessibility → Web Standards Engineer, Dead Code+Deprecation+Error Handler → Code Cleaner, Dep Audit+Update → Dependency Manager, Tech Writer+API Docs → Documentation Engineer, Database Engineer+Infra Builder → Fullstack Builder, Integration Tester → E2E Tester (×2)
- Added roles: Growth Strategist, Content Creator, Growth Engineer, Analytics Engineer, Mobile Optimizer, Fullstack Builder
- Scaled up: UI Builder ×3, API Builder ×3, E2E Tester ×2, Unit Tester ×2, Code Reviewer ×4
- New department: Growth (Strategist, Engineer ×2, Content Creator ×2, Analytics Engineer)

**Documentation philosophy:**
- Agent briefs are now 1-2 sentences — tell WHAT, not HOW
- All V4 docs rewritten to be concise and high-signal
- Removed procedural step-by-step scripts throughout

**Updated files:** agent-roles.md, pipeline-v4-architecture.md, coordinator-guide.md, README.md

## v0.2.0 — 2026-02-12

### Complete rewrite based on v3 pipeline launch

**New Documents:**
- `docs/09-kpi-metrics.md` — Performance tracking framework with launch day results
- `docs/10-open-source-research.md` — How Kubernetes, React, Rust, and GitHub handle scale

**Rewritten Documents:**
- `docs/01-foundation.md` — Kubernetes controller pattern, reconciliation loops, no locks
- `docs/02-agent-roles.md` — 6 roles (Scout → Triage → Builder → Reviewer → Merger → Sentinel)
- `docs/03-merge-safety.md` — `strict: false` everywhere, OSS research, batch merging
- `docs/04-coordinator-design.md` — 11-step reconciliation loop, idempotent design
- `docs/05-ci-pipeline.md` — Minimal required checks, lint non-blocking
- `docs/06-migration-strategy.md` — v1 → v2 → v3 migration journey
- `docs/07-lessons-learned.md` — 16 real failures (up from 10)
- `docs/08-tooling.md` — Updated with label schema, cron config, branch protection API

**Key Findings:**
- No major OSS project uses `strict: true` at scale (Kubernetes, React, Next.js, Rust, GitHub)
- Triage gate prevents 30-50% of wasted work (the #1 missing piece in v1/v2)
- Lock mechanisms are anti-pattern — use idempotent reconciliation
- Post-merge Sentinel monitoring replaces strict mode safety
- 14 PRs batch-merged in 2 minutes (vs 20 min/PR with strict: true)

**Launch Day Results:**
- 30 PRs merged
- 0 production downtime
- 0 reverts needed

## v0.1.0 — 2026-02-12
- Initial release
- Foundation docs from SylphxAI/viral experience
- Covers: CI, agent roles, merge safety, coordinator design, migrations, lessons learned
