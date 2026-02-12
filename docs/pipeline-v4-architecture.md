# V4 Pipeline Architecture

Perpetual-motion system. No pipeline. No stages. No state machine. 26 autonomous agents across 7 directions, all contributing through Git.

## Principles

1. **Git-first.** PRs and issues are the only coordination layer. No custom state, no labels for workflow.
2. **Nobody waits.** All agents work in parallel from minute one. Issues are guidance, not prerequisites.
3. **AI is fullstack.** Split by cognitive mode, not technical domain.
4. **Directions, not departments.** 7 directions of thought: product, build, test, improve, secure, perf, review.
5. **Generic briefs + project context.** Role briefs are project-agnostic keyword lists. Project context is separate.
6. **Keywords, not narratives.** Minimum words, maximum coverage of all aspects each direction handles.

## Distribution

| Direction | Agents | % | Purpose |
|-----------|--------|---|---------|
| Build | 12 | 46% | Fullstack engineering — features, fixes, content |
| Test | 5 | 19% | Unit (2), Integration (1), E2E (2) |
| Review | 4 | 15% | Quality gate + merge authority |
| Improve | 2 | 8% | Code quality, refactoring, standards |
| Product | 1 | 4% | Product direction, feature issues |
| Secure | 1 | 4% | Adversarial security |
| Perf | 1 | 4% | Measurement-based optimization |
| **Total** | **26** | **100%** | |

## Coordinator

The coordinator is a **reconciler with a health check**. It runs every 5 minutes and does four things:

```
1. INVENTORY     — sessions_list, count active v4-* agents
2. SPAWN DEFICIT — for each role, if running < desired, spawn the difference
3. HEALTH CHECK  — curl production URL, if not 200 → spawn one-off v4-revert agent
4. SUMMARY       — print agents running/desired, health status
```

The coordinator does NOT:
- ❌ Merge PRs (that's **Review**)
- ❌ Assign work to agents
- ❌ Manage labels or workflow state
- ❌ Make product decisions
- ❌ Review code

### Why health check lives in the coordinator

A simple HTTP status check every 5 minutes is lightweight enough for the coordinator's cron loop. If the site is down, it spawns a one-off revert agent — a temporary agent that finds the bad merge, creates a revert PR, and exits. No permanent Sentinel role needed.

## How Agents Coordinate

There is no central coordination. Agents coordinate through Git:

- **Issues** describe work. Product creates them. Builders pick them up.
- **PRs** are the unit of work. One focused PR per agent session.
- **Review** is the quality gate. Reviewers approve and merge.
- **Main branch** is always deployable. Coordinator watches production health.

### Self-Correction

The system converges naturally:
- Overlapping work → Reviewers catch duplicates during review
- Code quality drift → Improvers find and fix it
- Broken deploys → Coordinator detects, spawns revert agent
- Stale issues → agents check for duplicates before starting

## Prompt Architecture

Every agent prompt has two parts:

### 1. Project Context (~500 bytes, configured per-project)

All agents receive identical project context: what the product is, who the users are, who the competitors are, what the market gap is, and where the product is headed. This is configured once per project and prepended to every agent's task.

Example fields:
- Project name, URL, description
- Competitors and market positioning
- Repo, commit identity, PR conventions
- Project-specific constraints

### 2. Role Brief (~200-400 bytes, generic keyword lists)

Each role brief is a set of **keyword clusters** — one bullet per aspect of the role, covering all relevant concerns with minimum words. The briefs are:

- **Generic** — not tied to any specific project
- **Keyword-based** — scannable clusters, not narrative paragraphs
- **Comprehensive** — covers ALL aspects the direction handles
- **Compact** — AI reasons better with concise, dense prompts

See [Agent Roles](agent-roles.md) for the full keyword-based briefs.

The separation means you can swap project context without touching role briefs, or refine a role brief without affecting other projects.

## Key Metrics

| Metric | Value |
|--------|-------|
| Directions | **7** |
| Agent instances | **26** |
| Pipeline stages | **0** (no pipeline) |
| State management | **None** (stateless coordinator) |
| Coordinator steps | **4** (inventory, spawn, health check, report) |
| Prompt size | **~5KB** total |
| Reconciliation interval | **5 minutes** |

## Evolution from V3

| Aspect | V3 | V4 |
|--------|----|----|
| Organization | 6 sequential roles | 7 parallel directions |
| Agents | 6 | 26 |
| Coordination | Pipeline with stages | Git-first, no pipeline |
| Coordinator | Orchestrator (merge + health + assign) | Reconciler + health check |
| Merging | Dedicated Merger agent | Review direction (review + merge) |
| Health checks | Coordinator (complex) | Coordinator (simple HTTP) + one-off revert agent |
| Agent briefs | Procedural checklists | Generic keyword lists |
| Product context | Per-agent, inconsistent | Shared block, identical for all, separate from briefs |
