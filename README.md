# AI Team Playbook

### How to run an autonomous AI-agent development team

> Real patterns from shipping a production app with zero human code.
> 32 agents. 28 specialized roles. No broken deploys.

---

## What This Is

We built [viral](https://github.com/SylphxAI/viral) — a production Next.js app — using fully autonomous AI agents. Not copilot autocomplete. A fleet of specialized agents that discover issues, write code, review each other, merge, and monitor production health.

**This playbook is the operating manual.** Every recommendation comes from a real incident, a real config, or a real production failure. We include actual YAML, actual error messages, and actual war stories.

## Current Architecture: V4 (Perpetual Motion)

V4 replaces the sequential pipeline with a **perpetual motion model** — 28 specialized roles across 10 departments, all working in parallel. No pipeline. No state machine. No handoffs. Just Git.

| Metric | Value |
|--------|-------|
| Specialized roles | **28** |
| Agent instances | **32** |
| Departments | **10** |
| Pipeline stages | **0** (no pipeline) |
| State management | **None** (stateless coordinator) |
| Production downtime | **0** |
| Human code written | **0 lines** |

### V4 Quick Start

1. **[V4 Architecture](docs/pipeline-v4-architecture.md)** — The perpetual motion model. Git-first principles, no-waiting design, convergent quality.
2. **[Agent Roles](docs/agent-roles.md)** — All 28 roles: what they do, what they look for, what they produce.
3. **[Coordinator Guide](docs/coordinator-guide.md)** — The 5-step reconciliation loop. Roster config, scaling, monitoring.
4. **[V3 to V4 Migration](docs/v3-to-v4-migration.md)** — What changed, how to switch, what to watch.

## Key Insights (TL;DR)

- **No pipeline beats any pipeline.** V4 eliminated sequential handoffs entirely. All 32 agents work in parallel from minute one.
- **Specialists beat generalists.** 28 focused roles find deeper issues than 6 generalists ever could.
- **Git is the only coordination layer.** Issues, PRs, and branches. No labels for state, no FSM, no orchestrator.
- **The coordinator is a reconciliation loop, not an orchestrator.** It maintains fleet size and merges approved PRs. That's it.
- **Idle agents exit immediately.** No busywork. If there's nothing to do, the agent is gone in under a minute.
- **The system self-corrects.** Overlapping work gets caught by Reviewers. Inconsistencies get fixed by Refactorers. Type issues get fixed by Type Hardeners. The codebase converges.

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                    COORDINATOR (cron, every 5 min)           │
│  Inventory → Spawn deficits → Merge approved → Health check │
└─────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌───────────────────────────────────────────────────────────────┐
│                     AGENT FLEET (32 instances)                │
│  Product (3) · Engineering (6) · Quality (3) · Security (2)  │
│  Performance (2) · Code Health (4) · Maintenance (2)         │
│  Standards (5) · Infrastructure (1) · Gates (4)              │
└───────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌───────────────────────────────────────────────────────────────┐
│              GITHUB (Single Source of Truth)                   │
│         Issues · Pull Requests · main branch                  │
└───────────────────────────────────────────────────────────────┘
```

All coordinated by a stateless reconciliation loop running every 5 minutes. No locks. No webhooks. No state. Just polling + idempotent operations.

## Table of Contents

### V4 Documentation (Current)

| # | Document | What You'll Learn |
|---|----------|-------------------|
| — | [V4 Architecture](docs/pipeline-v4-architecture.md) | Perpetual motion model, git-first principles, no-waiting design |
| — | [Agent Roles](docs/agent-roles.md) | All 28 roles across 10 departments with full specifications |
| — | [Coordinator Guide](docs/coordinator-guide.md) | Reconciliation algorithm, roster config, scaling, monitoring |
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
