# AI Team Playbook

### How to run an autonomous AI-agent development team

> Real patterns from shipping a production app with zero human code.
> 26 agents. 7 directions. No broken deploys.

---

## What This Is

A playbook for running fully autonomous AI agent teams. Not copilot autocomplete — a fleet of specialized agents that discover issues, write code, review each other, merge, and monitor production health.

Originally developed while building [viral](https://github.com/SylphxAI/viral), but the patterns, role briefs, and architecture are **generic** — applicable to any project.

**This playbook is the operating manual.** Every recommendation comes from a real incident, a real config, or a real production failure.

## Current Architecture: V4 (Perpetual Motion)

V4 replaces the sequential pipeline with a **perpetual motion model** — 7 directions of thought, 26 agents, all working in parallel. No pipeline. No state machine. No handoffs. Just Git.

| Metric | Value |
|--------|-------|
| Directions | **7** |
| Agent instances | **26** |
| Pipeline stages | **0** (no pipeline) |
| State management | **None** (stateless coordinator) |
| Production downtime | **0** |
| Human code written | **0 lines** |

### The 7 Directions

| Direction | Agents | What They Do |
|-----------|--------|--------------|
| **Product** | 1 | Product vision, feature issues, competitive strategy, monetization |
| **Build** | 12 | Fullstack engineering — features, APIs, auth, payments, real-time, infrastructure |
| **Test** | 5 | Unit (2), Integration (1), E2E (2) — catch real bugs, not pad coverage |
| **Improve** | 2 | Code quality, refactoring, types, docs, SEO, accessibility, i18n |
| **Secure** | 1 | Adversarial thinking — XSS, injection, auth bypass, CVEs, secrets |
| **Perf** | 1 | Measurement-based optimization — Core Web Vitals, bundles, queries, mobile |
| **Review** | 4 | Quality gate + merge authority — review, approve, and merge PRs |

### Prompt Philosophy

Agent briefs are **generic keyword lists** — minimum words, maximum coverage. Every agent gets project-specific context prepended separately. This separation means:

- **Role briefs are reusable** across projects — swap the project context, keep the briefs
- **Keywords, not narratives** — AI parses dense keyword clusters faster than paragraphs
- **Comprehensive** — each keyword list covers ALL aspects that direction handles

### V4 Quick Start

1. **[V4 Architecture](docs/pipeline-v4-architecture.md)** — The perpetual motion model. Git-first principles, no-waiting design.
2. **[Agent Roles](docs/agent-roles.md)** — All 7 directions with generic keyword-based role briefs.
3. **[Coordinator Guide](docs/coordinator-guide.md)** — The 4-step reconciliation loop. Roster config, health check, scaling.
4. **[V3 to V4 Migration](docs/v3-to-v4-migration.md)** — What changed, how to switch, what to watch.

## Key Insights (TL;DR)

- **No pipeline beats any pipeline.** V4 eliminated sequential handoffs entirely. All 26 agents work in parallel from minute one.
- **Directions beat departments.** 7 cognitive directions find deeper issues than organizational hierarchies.
- **Git is the only coordination layer.** Issues, PRs, and branches. No labels for state, no FSM, no orchestrator.
- **The coordinator is a reconciler, not an orchestrator.** It maintains fleet size and checks production health. That's it.
- **Reviewers merge.** The agent who reviewed the code is the natural person to merge it. No separate merger.
- **Generic briefs + project context.** Role briefs are keyword lists reusable across any project. Project context is separate.
- **Keywords beat narratives.** Dense keyword clusters give AI better scope understanding than prose paragraphs.
- **Idle agents exit immediately.** No busywork. Nothing to do → gone in under a minute.
- **The system self-corrects.** Overlapping work gets caught by Reviewers. Quality drift gets fixed by Improvers.

## Architecture at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│                 COORDINATOR (cron, every 5 min)               │
│       Inventory → Spawn deficits → Health check → Report     │
└──────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌──────────────────────────────────────────────────────────────┐
│                    AGENT FLEET (26 instances)                 │
│   Product (1) · Build (12) · Test (5) · Improve (2)         │
│   Secure (1) · Perf (1) · Review (4)                        │
└──────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌──────────────────────────────────────────────────────────────┐
│               GITHUB (Single Source of Truth)                 │
│          Issues · Pull Requests · main branch                 │
└──────────────────────────────────────────────────────────────┘
```

## Table of Contents

### V4 Documentation (Current)

| # | Document | What You'll Learn |
|---|----------|-------------------|
| — | [V4 Architecture](docs/pipeline-v4-architecture.md) | Perpetual motion model, git-first principles, no-waiting design |
| — | [Agent Roles](docs/agent-roles.md) | All 7 directions with generic keyword-based role briefs |
| — | [Coordinator Guide](docs/coordinator-guide.md) | 4-step reconciliation algorithm, roster config, health check, scaling |
| — | [V3 → V4 Migration](docs/v3-to-v4-migration.md) | What changed, key differences, migration steps |

### V3 Documentation (Historical)

| # | Document | What You'll Learn |
|---|----------|-------------------|
| 1 | [Foundation](docs/01-foundation.md) | Kubernetes controller pattern, GitHub as source of truth |
| 2 | [Agent Roles (V3)](docs/02-agent-roles.md) | The original 6 agents: Scout, Triage, Builder, Reviewer, Merger, Sentinel |
| 3 | [Merge Safety](docs/03-merge-safety.md) | Why NO major OSS project uses `strict: true` — and what to use instead |
| 4 | [Coordinator Design (V3)](docs/04-coordinator-design.md) | The 11-step reconciliation loop |
| 5 | [CI Pipeline](docs/05-ci-pipeline.md) | Minimal required checks, lint as non-blocking, speed over ceremony |
| 6 | [Migration Strategy](docs/06-migration-strategy.md) | From manual to autonomous: v1 → v2 → v3 |
| 7 | [Lessons Learned](docs/07-lessons-learned.md) | 15+ real failures and how to prevent them |
| 8 | [Tooling](docs/08-tooling.md) | GitHub Apps, cron scheduling, label schema |
| 9 | [KPI & Metrics](docs/09-kpi-metrics.md) | What to measure, target ranges, launch day numbers |
| 10 | [Open Source Research](docs/10-open-source-research.md) | How Kubernetes, React, Rust, and GitHub handle high-volume merging |

## Research Foundation

This playbook is informed by:

- **STRATUS** (NeurIPS 2025) — FSM-based agent coordination with safety specifications
- **MetaGPT** — Structured outputs beat free-form chat
- **"Why Agentic-PRs Get Rejected"** (arXiv 2602.04226) — Top rejection reasons and how to prevent them
- **"Where Do AI Coding Agents Fail"** (arXiv 2601.15195) — CI conventions, review norms, PR sizing
- **Kubernetes Prow/Tide** — How to manage 2,500+ PRs/month
- **GitHub Merge Queue** — Batching 15-30 PRs for testing

See [Open Source Research](docs/10-open-source-research.md) for the full analysis.

## Contributing

This is a living document. If you're running AI agents on your own projects, we want your war stories:

- Different stacks (Python, Rust, Go)
- Multi-repo coordination patterns
- Monitoring and metrics approaches
- Alternative CI/CD configurations
- Failure modes we haven't hit yet

Open a PR. The bar is: "Would this have saved us time if we'd known it on day 1?"

## License

MIT — use this however you want. If it saves you from one broken production deploy, it was worth it.

---

*Built by [SylphxAI](https://github.com/SylphxAI). Powered by AI agents coordinated through [OpenClaw](https://github.com/nicepkg/openclaw).*
