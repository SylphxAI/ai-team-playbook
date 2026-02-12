# AI Team Playbook

### How to run an autonomous AI-agent development team

> Real patterns from shipping a production app with zero human code.
> 30 PRs merged on launch day. No broken deploys.

---

## What This Is

We built [viral](https://github.com/SylphxAI/viral) — a production Next.js app — using fully autonomous AI agents. Not copilot autocomplete. Six specialized agents that discover issues, triage them, write code, review each other, merge, and monitor production health.

**This playbook is the operating manual.** Every recommendation comes from a real incident, a real config, or a real production failure. We include actual YAML, actual error messages, and actual war stories.

## Results

| Metric | Value |
|--------|-------|
| PRs merged on launch day | **30** |
| Batch merge speed | **14 PRs in 2 minutes** |
| Target throughput | **30 PRs/hour** |
| Production downtime | **0** |
| Human code written | **0 lines** |

## Table of Contents

| # | Document | What You'll Learn |
|---|----------|-------------------|
| 1 | [Foundation](docs/01-foundation.md) | Kubernetes controller pattern, GitHub as source of truth, why no locks |
| 2 | [Agent Roles](docs/02-agent-roles.md) | The 6 agents: Scout, Triage, Builder, Reviewer, Merger, Sentinel |
| 3 | [Merge Safety](docs/03-merge-safety.md) | Why NO major OSS project uses `strict: true` — and what to use instead |
| 4 | [Coordinator Design](docs/04-coordinator-design.md) | The 11-step reconciliation loop, idempotent by design |
| 5 | [CI Pipeline](docs/05-ci-pipeline.md) | Minimal required checks, lint as non-blocking, speed over ceremony |
| 6 | [Migration Strategy](docs/06-migration-strategy.md) | From manual to autonomous: v1 → v2 → v3 |
| 7 | [Lessons Learned](docs/07-lessons-learned.md) | 15+ real failures and how to prevent them |
| 8 | [Tooling](docs/08-tooling.md) | GitHub Apps, cron scheduling, label schema |
| 9 | [KPI & Metrics](docs/09-kpi-metrics.md) | What to measure, target ranges, our launch day numbers |
| 10 | [Open Source Research](docs/10-open-source-research.md) | How Kubernetes, React, Rust, and GitHub handle high-volume merging |

## Quick Start

Setting up a new project for AI-agent development? Read in this order:

1. **[Foundation](docs/01-foundation.md)** — The architecture. Understand the Kubernetes controller pattern before anything else.
2. **[Agent Roles](docs/02-agent-roles.md)** — The six agents and why Triage is the most important one.
3. **[Merge Safety](docs/03-merge-safety.md)** — Branch protection settings that actually work at scale.
4. **[Lessons Learned](docs/07-lessons-learned.md)** — Every mistake we made so you don't have to.
5. **[KPI & Metrics](docs/09-kpi-metrics.md)** — How to know if it's working.

## Key Insights (TL;DR)

- **`strict: true` kills throughput.** No major OSS project uses it at scale. We switched to `strict: false` and went from 20 min/PR to 14 PRs in 2 minutes.
- **Triage is the #1 missing piece.** Without it, 30-50% of agent work is wasted on low-value issues.
- **Locks are an anti-pattern.** Use idempotent reconciliation loops (like Kubernetes controllers). Overlapping runs are harmless.
- **Post-merge monitoring replaces pre-merge serialization.** Sentinel + auto-revert gives you the safety of `strict: true` with 10x the throughput.
- **GitHub is your database.** Labels are FSM state. Issues are the task queue. PRs are the review pipeline. No external state.

## Architecture at a Glance

```
Scout discovers → Issue created (pipeline/discovered)
       ↓
Triage evaluates → Approved or Rejected
       ↓ (approved)
Builder implements → PR opened (pipeline/pr-open)
       ↓
Reviewer reviews → Approved or Changes Requested
       ↓ (approved)
Merger merges → Squash merge to main
       ↓
Sentinel verifies → Production healthy or auto-revert
```

All coordinated by a single reconciliation loop running every 5 minutes. No locks. No webhooks. Just polling + idempotent operations.

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
