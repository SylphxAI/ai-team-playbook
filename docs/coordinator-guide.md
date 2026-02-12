# Coordinator Guide — V4

> The coordinator is a stateless reconciliation loop. It maintains fleet size, merges approved PRs, and reports health. It does not orchestrate, assign work, or manage state.

---

## Overview

Runs as a cron job every 5 minutes. Each cycle is independent — no memory between runs. Everything derived from current Git + session state.

| Property | Meaning |
|----------|---------|
| **Stateless** | No database, no state file. Derives everything from Git + sessions. |
| **Idempotent** | Running twice produces the same result. |
| **Crash-safe** | Next cron picks up seamlessly. No cleanup. |
| **Self-healing** | Crashed agents detected as missing → respawned next cycle. |

---

## The 5-Step Reconciliation Loop

Each step is independent. If one fails, the coordinator logs the error and continues.

1. **Inventory** — List running agent sessions, count per role.
2. **Reconcile** — For each role: `deficit = desired - running`. Spawn deficit agents.
3. **Auto-merge** — Squash-merge PRs that are approved + CI passing + not draft. Skip conflicts.
4. **Health check** — HTTP GET to production. Log status. Sentinel handles remediation.
5. **Report** — Summary: agents running, spawned, PRs merged, health status, errors.

### What the Coordinator Does NOT Do

- Assign work to agents
- Track which agent is working on what
- Manage pipeline state or transitions
- Store data between runs
- Retry failed operations (next cycle handles it)
- Review code or make merge decisions beyond CI + approval checks

---

## Roster Configuration (38 instances)

| # | Role | Key | × | Dept | Mode |
|---|------|-----|---|------|------|
| 1 | Product Designer | `product_designer` | 1 | Product | Thinker |
| 2 | System Architect | `system_architect` | 1 | Product | Thinker |
| 3 | UI/UX Designer | `uiux_designer` | 1 | Product | Thinker |
| 4 | UI Builder | `ui_builder` | 3 | Engineering | Task-driven |
| 5 | API Builder | `api_builder` | 3 | Engineering | Task-driven |
| 6 | Fullstack Builder | `fullstack_builder` | 3 | Engineering | Task-driven |
| 7 | Unit Tester | `unit_tester` | 2 | Quality | Self-directed |
| 8 | E2E Tester | `e2e_tester` | 2 | Quality | Self-directed |
| 9 | Security Auditor | `security_auditor` | 1 | Security | Self-directed |
| 10 | Dependency Manager | `dependency_manager` | 1 | Security | Self-directed |
| 11 | Frontend Optimizer | `frontend_optimizer` | 1 | Performance | Self-directed |
| 12 | Backend Optimizer | `backend_optimizer` | 1 | Performance | Self-directed |
| 13 | Mobile Optimizer | `mobile_optimizer` | 1 | Performance | Self-directed |
| 14 | Refactorer | `refactorer` | 1 | Code Health | Self-directed |
| 15 | Type Hardener | `type_hardener` | 1 | Code Health | Self-directed |
| 16 | Code Cleaner | `code_cleaner` | 1 | Code Health | Self-directed |
| 17 | Web Standards Engineer | `web_standards_engineer` | 1 | Standards | Self-directed |
| 18 | Documentation Engineer | `documentation_engineer` | 1 | Standards | Self-directed |
| 19 | CI/CD Engineer | `cicd_engineer` | 1 | Infrastructure | Self-directed |
| 20 | Code Reviewer | `code_reviewer` | 4 | Gates | Reviewer |
| 21 | Sentinel | `sentinel` | 1 | Gates | Sentinel |
| 22 | Growth Strategist | `growth_strategist` | 1 | Growth | Thinker |
| 23 | Growth Engineer | `growth_engineer` | 2 | Growth | Task-driven |
| 24 | Content Creator | `content_creator` | 2 | Growth | Self-directed |
| 25 | Analytics Engineer | `analytics_engineer` | 1 | Growth | Self-directed |

---

## Scaling Roles

**To add a role:** Add to roster → write a 1-2 sentence agent brief → deploy. Next cycle spawns it.

**To remove a role:** Remove from roster → deploy. Running agents finish and exit naturally.

**To scale up/down:** Change the desired count. The coordinator only spawns — it never kills running agents. The fleet converges naturally as agents complete work.

### When to Scale

| Signal | Action |
|--------|--------|
| PR review queue > 10 | Increase Reviewer count |
| Feature backlog growing | Increase Builder counts |
| Frequent merge conflicts | Reduce parallel Builders |
| Agents frequently finding nothing | Reduce that role's count |
| New domain area | Add specialized role |

---

## Error Handling

Every step has independent error handling. Failures in one step don't block subsequent steps.

| Step | On Failure |
|------|------------|
| Inventory fails | Assume 0 running, spawn all |
| Individual spawn fails | Log, continue to next role |
| PR list fails | Skip entire merge step |
| Single PR merge fails | Skip that PR, continue |
| Health check times out | Report as potential outage |

The coordinator is resilient by design. Most failures self-resolve on the next 5-minute cycle.

---

## Monitoring

### Healthy System
- Agents near desired count
- PRs merging regularly
- Production returning 200
- No persistent errors

### Degraded System
- Multiple spawn failures per cycle
- PR queue growing (not enough Reviewers)
- Agents frequently exiting with nothing to do
- Many merge conflicts

### Broken System
- Production 5xx (Sentinel should auto-revert)
- All spawns failing (platform issue)
- GitHub API errors persisting across cycles

### Cycle Timing

| Step | Typical Duration |
|------|-----------------|
| Inventory | 2–5 seconds |
| Spawn (all deficits) | 30–120 seconds |
| PR merge (per PR) | 5–15 seconds |
| Health check | 1–10 seconds |
| **Total cycle** | **1–3 minutes** |

Overlapping runs are safe — every operation is idempotent.

---

## Troubleshooting

**Agents not spawning:** Check platform health, concurrency limits, spawn errors in cycle log.

**PRs not merging:** Verify PRs are approved, CI is passing, no merge conflicts. Check coordinator has merge permissions.

**Production down:** Check Sentinel status. Check recent merges. If revert PR exists, approve it.

**Too many conflicts:** Reduce parallel Builders — they're modifying overlapping files.

**Low-quality PRs:** Tighten agent briefs. Increase Reviewer count. Ensure agents exit when nothing meaningful exists.

---

*See also: [V4 Architecture](pipeline-v4-architecture.md) · [Agent Roles](agent-roles.md) · [V3 to V4 Migration](v3-to-v4-migration.md)*
