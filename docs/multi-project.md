# Multi-Project — One Team, Multiple Repos

How to run a single AI agent team across multiple repositories with one coordinator.

## Why Unified > Per-Project

The naive approach is one coordinator per project, each with its own agent fleet. This fails:

| Problem | Per-Project | Unified |
|---------|------------|---------|
| Agent sprawl | 8 agents × N repos = 8N agents | 8 agents total |
| Resource waste | Idle agents on quiet repos | Agents flow to where work is |
| Inconsistency | Each coordinator evolves differently | One config, one algorithm |
| Priority blindness | Can't prioritize across repos | Global priority queue |
| Cost | Linear cost scaling | Constant team size |

**One coordinator, one team, N repos.** Agents are generalists who read each repo's `CLAUDE.md` for project-specific context.

## How It Works

### The Coordinator Cycle (every 5 min)

```
1. INVENTORY    — count running agents by role
2. CHECK REPOS  — scan ALL repos: CI status, open PRs, open issues
3. PRIORITIZE   — build ONE queue across all repos
4. SPAWN        — fill empty agent slots with highest-priority work
5. CI CHECK     — failing CI = emergency, bump to top
6. REPORT       — print status across all repos
```

### How Agents Discover Work

Agents don't "belong to" a repo. The coordinator:

1. Scans every repo for work items (failing CI, broken PRs, approved issues, etc.)
2. Ranks them in a single priority queue
3. Assigns the highest-priority item to the next available agent
4. The agent clones that repo, reads its `CLAUDE.md`, and works

When the agent finishes, it exits. Next cycle, the coordinator may assign it (or its replacement) to a different repo entirely.

### Priority System

Global priority, applied across all repos:

| Priority | Type | Example |
|----------|------|---------|
| 🔴 P0 | Failing CI on dev/main | Any repo's CI red |
| 🟠 P1 | Broken PRs | CI failing on PR, changes requested |
| 🟡 P2 | PRs needing review | CI passing, awaiting review |
| 🟢 P3 | PRs needing tests | Open PR without test coverage |
| 🔵 P4 | Approved issues | Triaged and ready to build |
| ⚪ P5 | Untriaged issues | New issues needing triage |

Within the same priority level, the coordinator can use heuristics:
- Older items first (prevent starvation)
- Smaller items first (keep velocity high)
- Critical repos first (if configured)

### Project Context via CLAUDE.md

Each repo has a `CLAUDE.md` at its root that tells agents everything they need:

```markdown
# Project Name — domain.com

## What It Does
Brief product description.

## Stack
Next.js 15, TypeScript, Drizzle ORM, Neon Postgres...

## Conventions
- File naming, code style, architecture patterns
- How to run tests, build, deploy

## Key Paths
- src/app/ — pages
- src/lib/ — shared utilities
```

This replaces hardcoding project context in the coordinator prompt. When the coordinator spawns an agent, it tells the agent which repo to work on. The agent reads that repo's `CLAUDE.md` and gets full context.

**Benefits:**
- Project context lives with the project (version controlled)
- Updated by anyone who changes the project
- No coordinator prompt changes needed when adding repos

## Adding a New Repo

To add a new repo to the unified coordinator:

1. **Create `CLAUDE.md`** in the new repo with project context
2. **Set up CI** (`.github/workflows/ci.yml`) — same structure, adapted build steps
3. **Configure branch protection** on `dev` branch
4. **Install GitHub Apps** (builder + reviewer) on the new repo
5. **Create labels** (approved, rejected, priority labels, etc.)
6. **Update coordinator prompt** — add the repo to the repos table
7. **Done.** Next coordinator cycle will start scanning the new repo.

See [New Project](new-project.md) for the full checklist.

## Scaling Considerations

### When 8 Agents Aren't Enough

If work piles up across repos:
- **Increase builders first** — they're usually the bottleneck
- **Add a second reviewer** if PR queue grows
- Don't scale product/audit/triage — they generate work, not consume it

### Preventing Repo Starvation

If one repo dominates the priority queue:
- The priority system naturally handles this — urgent work (failing CI) always wins
- For chronic imbalance, add a "fairness" rule: no repo gets more than 50% of builder slots
- Monitor via the coordinator's status report

### When to Split

Split into separate coordinators only when:
- Repos have genuinely different teams/owners
- Security boundaries require isolation
- Agent count exceeds ~15 (coordinator overhead grows)

For most setups (2-5 repos, same team), unified is strictly better.

## Example: 3-Repo Setup

```
Repos:
  - org/frontend    (Next.js app)
  - org/backend     (API service)
  - org/shared-lib  (shared utilities)

Coordinator scans all 3 every 5 min.

Cycle 1: frontend CI failing → builder-1 fixes frontend
         backend has approved issue → builder-2 works on backend
         shared-lib has open PR → reviewer reviews it

Cycle 2: frontend CI fixed → builder-1 now free
         backend PR ready for review → reviewer switches to backend
         new issue on shared-lib → triage evaluates it
```

Agents flow naturally to where work exists. No manual assignment needed.
