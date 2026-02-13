# Coordinator Guide

The coordinator is a **reconciler with a CI check**. It maintains fleet size and guards CI health. Nothing else.

## Algorithm

Runs every 5 minutes. Stateless, idempotent.

```
1. INVENTORY     — sessions_list, count active agents by v4-* label prefix
2. SPAWN DEFICIT — for each role, if running < desired, sessions_spawn the difference
3. CI CHECK      — check main branch CI status
                   if not success → spawn one-off CI fix agent (label: v4-cifix)
4. SUMMARY       — print agents running/desired, health status, any spawn failures
```

Four steps. No merging, no code review, no work assignment.

## What the Coordinator Does NOT Do

| Responsibility | Owner | Why not coordinator? |
|---------------|-------|---------------------|
| Merge PRs | **Review** agent | Reviewer has full code context from review — natural merge point |
| Review code | **Review** agent | Code quality judgment doesn't belong in a cron loop |
| Assign work | **Nobody** | Agents self-direct from issues (approved by Triage) |
| Manage state | **Nobody** | Git is the only state. Labels for claiming, not FSM |
| Product decisions | **Product** agent | Strategic thinking doesn't belong in a reconciler |
| Issue quality | **Triage** agent | Approve/reject decisions need judgment, not cron |

## Roster Configuration

| Direction | Key | Desired |
|-----------|-----|---------|
| Product | `v4-product` | 1 |
| Audit | `v4-audit` | 1 |
| Triage | `v4-triage` | 1 |
| Build | `v4-builder` | 3 |
| Test | `v4-tester` | 1 |
| Review | `v4-reviewer` | 1 |
| **Total** | | **8** |

## Spawning

For each role with `running < desired`, call `sessions_spawn` with:
- **Label**: `v4-{key}` (e.g., `v4-builder`)
- **Task**: Project context + role brief

Every agent's task has two parts:
1. **Project context** (configured per-project) — project name, URL, competitors, repo, commit identity, PR conventions
2. **Role brief** (generic keyword list) — the agent's capability scope from the [agent roles](agent-roles.md)

The project context is prepended to every agent's task, giving all agents identical understanding of the product. The role briefs are generic and project-agnostic.

## CI Check & Fix

The coordinator checks main branch CI status every 5 minutes:

- **Success** → healthy, continue
- **Failure** → spawn a one-off CI fix agent with label `v4-cifix`

Main CI broken = P0 — it blocks all development, all merges, all deploys. The fix agent diagnoses the failure, fixes the code, and exits. No permanent monitoring role needed.

## Scaling

Adjust counts based on bottlenecks:

| Symptom | Action |
|---------|--------|
| PR queue growing | Add more Review agents (default: 1) |
| Approved issues piling up | Add more Builders (default: 3) |
| PRs waiting for tests | Add more Testers (default: 1) |
| Code quality drifting | Audit agent creates issues; Triage prioritizes them |
| Issues not getting reviewed | Add more Triage agents (default: 1) |

The coordinator doesn't need to understand scaling strategy — just update the roster numbers.

## Prompt Size

The entire coordinator prompt is ~5KB:
- Project context: ~500 bytes (configured per-project)
- 6 role briefs: ~200-400 bytes each (generic keyword lists)
- Algorithm + roster table: ~1KB

Compact by design. AI reasons better with concise direction than verbose instructions.

## Failure Modes

| Failure | Impact | Recovery |
|---------|--------|----------|
| Coordinator doesn't run | Fleet shrinks as agents finish | Next run respawns — self-healing |
| Spawn fails | Fewer agents than desired | Next run retries — idempotent |
| Too many agents | Resource waste | Agents exit when idle — self-correcting |
| Agent crashes | One less worker | Next reconciliation spawns replacement |
| Main CI breaks | All merges blocked | CI check spawns fix agent |
| Builder dies mid-work | Issue stuck with `in-progress` label | Triage cleans up stale labels after 30 min |

The system tolerates coordinator failures gracefully. Agents are independent — they don't need the coordinator to do their work, only to exist.
