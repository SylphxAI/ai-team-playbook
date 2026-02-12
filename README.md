# AI Team Playbook — How to Run an AI-Agent Development Pipeline

> A living document capturing best practices for using AI agents as a development team.
> Born from real experience running [SylphxAI/viral](https://github.com/SylphxAI/viral) with autonomous AI agents.

## Why This Exists

We built a production application — [viral](https://github.com/SylphxAI/viral) — almost entirely with AI agents. Not copilot-style autocomplete. Fully autonomous agents that read issues, write code, create PRs, review each other's work, and ship to production.

It worked. It also broke in spectacular ways.

This playbook captures everything we learned: the infrastructure that makes it possible, the guardrails that keep it safe, the patterns that scale, and the mistakes that cost us hours of debugging.

**This is not theory.** Every recommendation comes from a real incident, a real config file, or a real production failure. We include actual error messages, actual YAML, and actual war stories.

## Table of Contents

| # | Document | What It Covers |
|---|----------|---------------|
| 1 | [Project Foundation](docs/01-foundation.md) | Day 1 setup — CI, branch protection, auto-merge |
| 2 | [Agent Roles](docs/02-agent-roles.md) | Coordinator, Builder, Reviewer, Fixer, Rebaser, Auditor |
| 3 | [Merge Safety](docs/03-merge-safety.md) | CI gates, file overlap detection, duplicate PR handling |
| 4 | [Coordinator Design](docs/04-coordinator-design.md) | Architecture, slot management, prompt patterns |
| 5 | [CI/CD Pipeline](docs/05-ci-pipeline.md) | GitHub Actions, Vercel, build verification |
| 6 | [Migration Strategy](docs/06-migration-strategy.md) | Why sequential numbering breaks, Atlas vs Drizzle |
| 7 | [Lessons Learned](docs/07-lessons-learned.md) | 10+ real failures and how to prevent them |
| 8 | [Tooling](docs/08-tooling.md) | GitHub Apps, Actions, Atlas, Drizzle, Vercel |

## Quick Start

If you're setting up a new project for AI-agent development:

1. **Read [Foundation](docs/01-foundation.md)** — set up CI, branch protection, and auto-merge before anything else
2. **Read [Agent Roles](docs/02-agent-roles.md)** — understand the separation of concerns
3. **Read [Merge Safety](docs/03-merge-safety.md)** — this will save you from the worst failures
4. **Skim [Lessons Learned](docs/07-lessons-learned.md)** — learn from our mistakes

## Contributing

This is a living document. If you're running AI agents on your own projects and have lessons to share, open a PR. We especially want:

- War stories from different stacks (Python, Rust, Go)
- Patterns for multi-repo coordination
- Monitoring and metrics approaches
- Alternative CI/CD configurations

## License

MIT — use this however you want. If it saves you from one broken production deploy, it was worth it.
