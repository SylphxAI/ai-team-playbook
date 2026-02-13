# V4 Pipeline Architecture

Perpetual-motion system. No pipeline. No stages. No state machine. 8 autonomous agents across 6 directions, all contributing through Git.

## Principles

1. **Git-first.** PRs and issues are the only coordination layer. Labels for claiming work (`in-progress`, `fixing`), not for workflow state machines.
2. **Nobody waits.** All agents work in parallel from minute one. Issues are guidance, not prerequisites.
3. **AI is fullstack.** Split by cognitive mode, not technical domain.
4. **Directions, not departments.** 6 directions of thought: product, audit, triage, build, test, review.
5. **Generic briefs + project context.** Role briefs are project-agnostic keyword lists. Project context is separate.
6. **Keywords, not narratives.** Minimum words, maximum coverage of all aspects each direction handles.
7. **Separate testing.** Builders write code only. Testers write tests independently for open PRs.

## Distribution

| Direction | Agents | % | Purpose |
|-----------|--------|---|---------|
| Build | 3 | 37% | Fix failing PRs first, then build from approved issues (code only) |
| Product | 1 | 13% | Product direction, feature issues |
| Audit | 1 | 13% | Code quality, security, and performance issues |
| Triage | 1 | 13% | Central quality gate — approve, reject, prioritize issues |
| Test | 1 | 13% | Write tests for open PRs — adversarial, independent |
| Review | 1 | 13% | Quality gate + merge authority |
| **Total** | **8** | **100%** | |

## Coordinator

The coordinator is a **reconciler with a CI check**. It runs every 5 minutes and does four things:

```
1. INVENTORY     — sessions_list, count active v4-* agents
2. SPAWN DEFICIT — for each role, if running < desired, spawn the difference
3. CI CHECK      — check main branch CI status, if failing → spawn one-off CI fix agent
4. SUMMARY       — print agents running/desired, health status
```

The coordinator does NOT:
- ❌ Merge PRs (that's **Review**)
- ❌ Assign work to agents
- ❌ Manage labels or workflow state
- ❌ Make product decisions
- ❌ Review code

### Why CI check lives in the coordinator

A simple CI status check every 5 minutes is lightweight enough for the coordinator's cron loop. If main branch CI is failing, it spawns a one-off fix agent — a temporary agent that diagnoses and fixes the failure, then exits. Main CI broken = P0 (blocks all development, all merges, all deploys).

## How Agents Coordinate

There is no central coordination. Agents coordinate through Git:

- **Issues** describe work. Product and Audit create them. Triage approves them. Builders pick up approved ones.
- **PRs** are the unit of work. One focused PR per agent session.
- **Labels** claim work: `in-progress` (builder claiming an issue), `fixing` (builder claiming a failing PR).
- **Review** is the quality gate. Reviewers approve and merge when code AND tests pass.
- **Main branch** is always deployable. Coordinator watches CI health.

### Self-Correction

The system converges naturally:
- Overlapping work → Reviewers catch duplicates during review
- Stale claims → Triage removes `in-progress` label after 30 min with no linked PR
- Broken CI → Coordinator detects, spawns fix agent
- Missing tests → Reviewers flag "needs tests", Testers add them
- Low-quality issues → Triage rejects them before builders waste cycles

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
| Directions | **6** |
| Agent instances | **8** |
| Pipeline stages | **0** (no pipeline) |
| State management | **None** (stateless coordinator) |
| Coordinator steps | **4** (inventory, spawn, CI check, report) |
| Prompt size | **~5KB** total |
| Reconciliation interval | **5 minutes** |

## Evolution from V3

| Aspect | V3 | V4 |
|--------|----|----|
| Organization | 6 sequential roles | 6 parallel directions |
| Agents | 6 | 8 |
| Coordination | Pipeline with stages | Git-first, no pipeline |
| Coordinator | Orchestrator (merge + health + assign) | Reconciler + CI check |
| Merging | Dedicated Merger agent | Review direction (review + merge) |
| Health checks | Coordinator (complex) | Coordinator (simple CI check) + one-off fix agent |
| Agent briefs | Procedural checklists | Generic keyword lists |
| Product context | Per-agent, inconsistent | Shared block, identical for all, separate from briefs |
| Quality directions | Improve + Secure + Perf (3 separate) | Audit (1 unified) |
| Testing | Unit/Integration/E2E split (5 agents) | Generic tester (1 agent), separate from builders |
| Issue quality | No gate | Triage agent approves/rejects before builders see them |
| Builder priority | Self-directed | Fix failing PRs first, then approved issues |
