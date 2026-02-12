# Coordinator Guide

The coordinator is a **reconciler with a health check**. It maintains fleet size and guards production uptime. Nothing else.

## Algorithm

Runs every 5 minutes. Stateless, idempotent.

```
1. INVENTORY     — sessions_list, count active agents by v4-* label prefix
2. SPAWN DEFICIT — for each role, if running < desired, sessions_spawn the difference
3. HEALTH CHECK  — curl -s -o /dev/null -w "%{http_code}" <PRODUCTION_URL>
                   if not 200 → spawn one-off revert agent (label: v4-revert)
4. SUMMARY       — print agents running/desired, health status, any spawn failures
```

Four steps. No merging, no code review, no work assignment.

## What the Coordinator Does NOT Do

| Responsibility | Owner | Why not coordinator? |
|---------------|-------|---------------------|
| Merge PRs | **Review** agents | Reviewers have full code context from review — natural merge point |
| Review code | **Review** agents | Code quality judgment doesn't belong in a cron loop |
| Assign work | **Nobody** | Agents self-direct from issues or their own judgment |
| Manage state | **Nobody** | Git is the only state. No labels, no FSM, no workflow |
| Product decisions | **Product** agent | Strategic thinking doesn't belong in a reconciler |

## Roster Configuration

| Direction | Key | Desired |
|-----------|-----|---------|
| Product | `v4-product` | 1 |
| Build | `v4-builder` | 12 |
| Test — Unit | `v4-tester_unit` | 2 |
| Test — Integration | `v4-tester_integration` | 1 |
| Test — E2E | `v4-tester_e2e` | 2 |
| Improve | `v4-improver` | 2 |
| Secure | `v4-security` | 1 |
| Perf | `v4-perf` | 1 |
| Review | `v4-reviewer` | 4 |
| **Total** | | **26** |

## Spawning

For each role with `running < desired`, call `sessions_spawn` with:
- **Label**: `v4-{key}` (e.g., `v4-builder`)
- **Task**: Project context + role brief

Every agent's task has two parts:
1. **Project context** (configured per-project) — project name, URL, competitors, repo, commit identity, PR conventions
2. **Role brief** (generic keyword list) — the agent's capability scope from the [agent roles](agent-roles.md)

The project context is prepended to every agent's task, giving all agents identical understanding of the product. The role briefs are generic and project-agnostic.

## Health Check & Revert

The coordinator runs a simple HTTP check against the production URL every 5 minutes:

- **200** → healthy, continue
- **Not 200** → spawn a one-off revert agent with label `v4-revert`

The revert agent's task: find the most recent merge, determine if it caused the break, create a revert PR. It's a temporary agent — spawned on-demand, exits when done. No permanent monitoring role needed.

## Scaling

Adjust counts based on bottlenecks:

| Symptom | Action |
|---------|--------|
| PR queue growing | Add more Review agents (default: 4) |
| Issues piling up | Add more Builders (default: 12) |
| Test coverage lagging | Add testers |
| Code quality drifting | Add Improve agents |

The coordinator doesn't need to understand scaling strategy — just update the roster numbers.

## Prompt Size

The entire coordinator prompt is ~5KB:
- Project context: ~500 bytes (configured per-project)
- 7 role briefs: ~200-400 bytes each (generic keyword lists)
- Algorithm + roster table: ~1KB

Compact by design. AI reasons better with concise direction than verbose instructions.

## Failure Modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| Coordinator doesn't run | Fleet shrinks as agents finish | Next run respawns — self-healing |
| Spawn fails | Fewer agents than desired | Next run retries — idempotent |
| Too many agents | Resource waste | Agents exit when idle — self-correcting |
| Agent crashes | One less worker | Next reconciliation spawns replacement |
| Site goes down | Users affected | Health check spawns revert agent |

The system tolerates coordinator failures gracefully. Agents are independent — they don't need the coordinator to do their work, only to exist.
