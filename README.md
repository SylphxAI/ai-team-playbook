# AI Team Playbook

Run an autonomous AI-agent development team. 8 agents, 4 roles, zero human code.

Patterns from shipping production apps across multiple repos — but the architecture is generic. Swap the project context, keep the playbook.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│          UNIFIED COORDINATOR (cron, every 5 min)             │
│     Inventory → Check All Repos → Prioritize →              │
│     Spawn Deficits → CI Check → Report                      │
└──────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌──────────────────────────────────────────────────────────────┐
│                   AGENT FLEET (8 instances)                   │
│   Scout(1) · Gatekeeper(1) · Builder(5) · Tester(1)         │
│         Agents pick work from ANY repo by priority           │
└──────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌──────────────────────────────────────────────────────────────┐
│              GITHUB (Single Source of Truth)                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │  Repo A      │ │  Repo B      │ │  Repo C      │        │
│  │  Issues, PRs │ │  Issues, PRs │ │  Issues, PRs │        │
│  │  dev branch  │ │  dev branch  │ │  dev branch  │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

**One coordinator, one team, multiple repos.** See [Multi-Project](docs/multi-project.md) for why.

## Roster — 4 Roles, 8 Agents

| Role | Key | Count | Summary |
|------|-----|-------|---------|
| Scout | `scout` | 1 | Competitive research + multi-dimensional codebase analysis → files prioritized issues |
| Gatekeeper | `gatekeeper` | 1 | Issue triage + deep PR review + squash merge |
| Builder | `builder` | 5 | Fix failing PRs first, then build from approved issues |
| Tester | `tester` | 1 | Adversarial tests for open PRs |

Scout = Product + Audit merged (but more than both — adds competitive research and a strategic lens).
Gatekeeper = Triage + Reviewer merged (single quality gate for both issues and code).

## Core Design

- **No pipeline.** All 8 agents work in parallel. No stages, no handoffs, no state machine.
- **Git is the only coordination.** Issues, PRs, labels (`in-progress`, `fixing`). Nothing else.
- **Coordinator is a reconciler.** Maintains fleet size + checks CI. Does not merge, review, or assign work.
- **Multi-repo by default.** One coordinator scans all repos, agents pick work by priority.
- **Principle-based prompts, not checklists.** Rule-based prompts get gamed; agents satisfy the letter, not the spirit. Principle-based prompts (core question + lenses + good/bad examples) require genuine thought.
- **Separate testing.** Builders write code only. Testers write tests independently on open PRs.
- **Builders fix first, build second.** Failing PRs = P0. New features from approved issues = P2.
- **Gatekeeper merges.** The agent who reviewed the code merges it.
- **Idle agents exit.** Nothing to do → gone in under a minute.

## Coordinator Algorithm

Runs every 5 minutes. Stateless, idempotent.

```
1. INVENTORY     — count active agents by label prefix
2. CHECK REPOS   — scan ALL repos for work (CI, PRs, issues)
3. PRIORITIZE    — failing CI > broken PRs > approved issues > new issues
4. SPAWN DEFICIT — if running < desired, spawn the difference
5. CI CHECK      — if any repo's dev/main CI failing, prioritize fix
6. SUMMARY       — print status across all repos
```

## Prompt Architecture

Every agent prompt = **repo context** (from `CLAUDE.md`) + **role brief** (principle-based, ~400-600 bytes).

- **Repo context**: read from the target repo's `CLAUDE.md` at spawn time. Each repo defines its own stack, conventions, and goals.
- **Role brief**: core question + analytical lenses + examples of good vs bad output. Generic, project-agnostic.

Agents are not assigned to repos — they pick work from any repo based on priority.

## Git Identities

Two real GitHub user accounts (not GitHub Apps):

| Role | GitHub Account | Identity |
|------|---------------|----------|
| Scout | shtse8 | Kyle Tse / shtse8@gmail.com |
| Builder | shtse8 | Kyle Tse / shtse8@gmail.com |
| Tester | shtse8 | Kyle Tse / shtse8@gmail.com |
| Gatekeeper | claw-sylphx | Clayton Shaw / clayton@sylphx.com |

Switch accounts with: `gh auth switch --user <account>`

Branch protection requires reviewer to be different from PR author — hence two accounts.

## Docs

| File | Content |
|------|---------|
| [Agent Roles](docs/agent-roles.md) | Role briefs, methodology, good/bad examples |
| [Setup Guide](docs/setup-guide.md) | How to set up this workflow (single or multi-project) |
| [Multi-Project](docs/multi-project.md) | How one team works across multiple repos |
| [Lessons Learned](docs/lessons-learned.md) | Hard-won lessons from V3 and V4 |
| [Coding Standards](docs/coding-standards.md) | Stack, conventions, linting, testing |
| [Environments](docs/environments.md) | Production, preview, Vercel, Neon |
| [CI/CD](docs/ci-cd.md) | GitHub Actions, deployment, Atlas migrations |
| [New Project](docs/new-project.md) | Checklist for setting up a new project |

## Branch Protection

```
strict: false              # Do NOT require branches to be up-to-date
required checks: build     # Single required check
reviews: 1                 # From different identity than PR author
squash merge only          # Clean history
auto-delete branches       # Prevent accumulation
```

Why `strict: false`? No major OSS project (Kubernetes, React, Rust, GitHub) uses `strict: true` at scale. It serializes merging to ~20 min/PR. Use post-merge monitoring instead.

## License

MIT
