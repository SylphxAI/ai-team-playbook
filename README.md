# AI Team Playbook

Run an autonomous AI-agent development team. 8 agents, 6 directions, zero human code.

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
│   Product(1) · Audit(1) · Triage(1) · Build(3)              │
│   Test(1) · Review(1)                                        │
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

## Roster — 6 Directions, 8 Agents

| Direction | Key | Count | Mode |
|-----------|-----|-------|------|
| Product | `product` | 1 | Create issues — features, strategy, monetization |
| Audit | `audit` | 1 | Create issues — code quality, security, performance |
| Triage | `triage` | 1 | Approve/reject/prioritize issues |
| Build | `builder` | 3 | Fix failing PRs first, then build from approved issues |
| Test | `tester` | 1 | Write tests for open PRs (adversarial) |
| Review | `reviewer` | 1 | Review PRs + merge when code+tests+CI pass |

**Audit** = merged Improve + Secure + Perf. Code quality, security, and performance overlap too much for separate agents.

## Core Design

- **No pipeline.** All 8 agents work in parallel. No stages, no handoffs, no state machine.
- **Git is the only coordination.** Issues, PRs, labels (`in-progress`, `fixing`). Nothing else.
- **Coordinator is a reconciler.** Maintains fleet size + checks CI. Does not merge, review, or assign work.
- **Multi-repo by default.** One coordinator scans all repos, agents pick work by priority.
- **Generic briefs + project context.** Role briefs are reusable keyword lists. Project context comes from each repo's `CLAUDE.md`.
- **Separate testing.** Builders write code only. Testers write tests independently on open PRs.
- **Builders fix first, build second.** Failing PRs = P0. New features from approved issues = P2.
- **Reviewers merge.** The agent who reviewed the code merges it. No separate merger role.
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

Every agent prompt = **repo context** (from `CLAUDE.md`) + **role brief** (~200-400 bytes).

- **Repo context**: read from the target repo's `CLAUDE.md` at spawn time. Each repo defines its own stack, conventions, and goals.
- **Role brief**: keyword clusters covering all aspects of the direction. Generic, project-agnostic.

Agents are not assigned to repos — they pick work from any repo based on priority.

## Docs

| File | Content |
|------|---------|
| [Agent Roles](docs/agent-roles.md) | Role briefs, testing model, design rationale |
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
