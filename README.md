# AI Team Playbook

Run an autonomous AI-agent development team. 8 agents, 6 directions, zero human code.

Patterns from shipping a production app ([viral](https://github.com/SylphxAI/viral)) — but the architecture is generic. Swap the project context, keep the playbook.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              COORDINATOR (cron, every 5 min)                  │
│        Inventory → Spawn deficits → CI check → Report        │
└──────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌──────────────────────────────────────────────────────────────┐
│                   AGENT FLEET (8 instances)                   │
│   Product(1) · Audit(1) · Triage(1) · Build(3)              │
│   Test(1) · Review(1)                                        │
└──────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌──────────────────────────────────────────────────────────────┐
│              GITHUB (Single Source of Truth)                  │
│         Issues · Pull Requests · main branch                 │
└──────────────────────────────────────────────────────────────┘
```

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
- **Generic briefs + project context.** Role briefs are reusable keyword lists. Project context is prepended separately.
- **Separate testing.** Builders write code only. Testers write tests independently on open PRs.
- **Builders fix first, build second.** Failing PRs = P0. New features from approved issues = P2.
- **Reviewers merge.** The agent who reviewed the code merges it. No separate merger role.
- **Idle agents exit.** Nothing to do → gone in under a minute.

## Coordinator Algorithm

Runs every 5 minutes. Stateless, idempotent.

```
1. INVENTORY     — count active agents by label prefix
2. SPAWN DEFICIT — if running < desired, spawn the difference
3. CI CHECK      — if main CI failing, spawn one-off fix agent
4. SUMMARY       — print status
```

## Prompt Architecture

Every agent prompt = **project context** (~500 bytes) + **role brief** (~200-400 bytes).

- **Project context**: product name, URL, competitors, repo, commit identity. Identical for all agents. Configured per-project.
- **Role brief**: keyword clusters covering all aspects of the direction. Generic, project-agnostic.

Swap project context → same playbook, different product.

## Docs

| File | Content |
|------|---------|
| [Agent Roles](docs/agent-roles.md) | Role briefs, testing model, design rationale |
| [Setup Guide](docs/setup-guide.md) | How to copy this workflow to any project |
| [Lessons Learned](docs/lessons-learned.md) | Hard-won lessons from V3 and V4 |

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
