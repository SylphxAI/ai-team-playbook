# Coordinator Guide — V4

> How the V4 coordinator works: reconciliation algorithm, roster configuration, scaling, error handling, and monitoring.

---

## Table of Contents

- [Overview](#overview)
- [Reconciliation Algorithm](#reconciliation-algorithm)
- [Roster Configuration](#roster-configuration)
- [Adding, Removing, and Scaling Roles](#adding-removing-and-scaling-roles)
- [Error Handling](#error-handling)
- [Monitoring and Status Reporting](#monitoring-and-status-reporting)
- [Timing and Performance](#timing-and-performance)
- [Troubleshooting](#troubleshooting)

---

## Overview

The V4 coordinator is a **stateless reconciliation loop** that runs as a cron job every 5 minutes. It is not an orchestrator — it doesn't assign work, track progress, or manage state. It does three things:

1. **Maintains the agent fleet** — ensures the right number of agents are running for each role
2. **Merges approved PRs** — auto-merges PRs that pass CI and review
3. **Reports health** — checks production and logs a summary

### Design Properties

| Property | What It Means |
|----------|---------------|
| **Stateless** | No database, no state file, no memory between runs. Everything is derived from current Git + session state. |
| **Idempotent** | Running the coordinator twice in a row produces the same result. At-capacity roles spawn zero new agents. |
| **Crash-safe** | If the coordinator crashes mid-run, the next 5-minute cron picks up seamlessly. No cleanup needed. |
| **Self-healing** | If an agent crashes, it disappears from the session list. The next cycle detects the deficit and spawns a replacement. |

---

## Reconciliation Algorithm

The coordinator executes 5 steps in order. Each step is independent — if one fails, the coordinator logs the error and continues to the next step.

### Step 1: Inventory Running Agents

List all active sub-agent sessions matching the V4 label pattern:

```
sessions_list with label filter "v4-"
```

Parse the results and count how many agents are running for each role by matching label prefixes. For example, labels starting with `v4-ui_builder` count toward the UI Builder role.

**Output:** A map of role → running count.

**If this step fails:** Log the error, assume 0 running agents, proceed to Step 2 (which will attempt to spawn all roles).

### Step 2: Calculate Deficits and Spawn Agents

For each role in the ROSTER, compare the desired count against the running count:

```
deficit = desired_count - running_count
```

For each deficit, spawn a new agent:

```
sessions_spawn with:
  label: "v4-{role_key}"
  task: {SHARED_PREAMBLE} + {role_specific_prompt}
```

**Rules:**
- Spawn agents one at a time (not in parallel)
- If a spawn fails, log the error and continue to the next role
- Do not retry failed spawns in the same cycle — the next cycle will handle it
- Each spawned agent runs autonomously; the coordinator does not interact with it after spawning

**If this step fails partially:** Log each individual spawn failure. Successfully spawned agents continue running.

### Step 3: Auto-Merge Approved PRs

Query open PRs:

```bash
gh pr list --repo SylphxAI/viral --state open \
  --json number,title,reviewDecision,statusCheckRollup,headRefName \
  --limit 50
```

For each PR that meets ALL of these criteria:
- `reviewDecision` is `APPROVED`
- All status checks in `statusCheckRollup` have `conclusion: SUCCESS`
- The PR is not a draft

Attempt squash merge:

```bash
gh pr merge {number} --repo SylphxAI/viral --squash --delete-branch
```

**If merge fails** (conflict, API error, etc.): Skip that PR and continue to the next. Do not force-merge. The conflicting PR will either be resolved by a future agent or eventually closed as stale.

### Step 4: Health Check

```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://tryit.fun
```

| Result | Action |
|--------|--------|
| **200** | Production healthy. Note in summary. |
| **5xx** | Production may be broken. Check if PRs were merged in the last 30 minutes. Note potential cause in report. Sentinel handles remediation. |
| **Timeout** | Production may be down. Same as 5xx handling. |
| **4xx** | Unexpected for the landing page. Note in report. |

The coordinator does not take remediation action — that's the Sentinel's job.

### Step 5: Status Summary

Output a structured summary:

```
=== V4 Coordinator Cycle Complete ===
Time: 2025-02-12T19:30:00Z
Agents running: 28 / 32 desired
Spawned this cycle: ui_builder, refactorer, code_reviewer
PRs merged this cycle: 3 (#142, #145, #147)
Production health: 200
Errors: Failed to spawn e2e_tester (timeout)
=====================================
```

---

## Roster Configuration

The roster defines the desired fleet composition. Each entry specifies a role key, desired instance count, department, and operating mode.

### Default Roster (32 instances)

| # | Role | Key | Desired | Department | Mode |
|---|------|-----|---------|------------|------|
| 1 | Product Designer | `product_designer` | 1 | Product | Thinker |
| 2 | System Architect | `system_architect` | 1 | Product | Thinker |
| 3 | UI/UX Designer | `uiux_designer` | 1 | Product | Thinker |
| 4 | UI Builder | `ui_builder` | 2 | Engineering | Task-driven |
| 5 | API Builder | `api_builder` | 2 | Engineering | Task-driven |
| 6 | Database Engineer | `database_engineer` | 1 | Engineering | Task-driven |
| 7 | Infra Builder | `infra_builder` | 1 | Engineering | Task-driven |
| 8 | Unit Tester | `unit_tester` | 1 | Quality | Self-directed |
| 9 | Integration Tester | `integration_tester` | 1 | Quality | Self-directed |
| 10 | E2E Tester | `e2e_tester` | 1 | Quality | Self-directed |
| 11 | Security Auditor | `security_auditor` | 1 | Security | Self-directed |
| 12 | Dependency Auditor | `dependency_auditor` | 1 | Security | Self-directed |
| 13 | Frontend Optimizer | `frontend_optimizer` | 1 | Performance | Self-directed |
| 14 | Backend Optimizer | `backend_optimizer` | 1 | Performance | Self-directed |
| 15 | Refactorer | `refactorer` | 1 | Code Health | Self-directed |
| 16 | Type Hardener | `type_hardener` | 1 | Code Health | Self-directed |
| 17 | Dead Code Cleaner | `dead_code_cleaner` | 1 | Code Health | Self-directed |
| 18 | Error Handler | `error_handler` | 1 | Code Health | Self-directed |
| 19 | Deprecation Migrator | `deprecation_migrator` | 1 | Maintenance | Self-directed |
| 20 | Dependency Updater | `dependency_updater` | 1 | Maintenance | Self-directed |
| 21 | Technical Writer | `technical_writer` | 1 | Standards | Self-directed |
| 22 | API Documenter | `api_documenter` | 1 | Standards | Self-directed |
| 23 | SEO Engineer | `seo_engineer` | 1 | Standards | Self-directed |
| 24 | Accessibility Engineer | `accessibility_engineer` | 1 | Standards | Self-directed |
| 25 | i18n Engineer | `i18n_engineer` | 1 | Standards | Self-directed |
| 26 | CI/CD Engineer | `cicd_engineer` | 1 | Infrastructure | Self-directed |
| 27 | Code Reviewer | `code_reviewer` | 3 | Gates | Reviewer |
| 28 | Sentinel | `sentinel` | 1 | Gates | Sentinel |

---

## Adding, Removing, and Scaling Roles

### Adding a New Role

1. **Define the role** in the roster with a unique key, desired count, and department
2. **Write the agent prompt** — a self-contained instruction set that includes:
   - The shared preamble (clone, configure, branch naming, PR standards)
   - Role-specific instructions (what to look for, what to build, quality standards)
3. **Add the entry** to the ROSTER table in the coordinator prompt
4. **Deploy** — the next coordinator cycle will detect a deficit and spawn the new role

**Prompt template for new roles:**

```
{SHARED PREAMBLE}

## Your Role: {Role Name}

You are the {Role Name} for {project}. You {one-sentence description}.

### What You Do
1. Clone the repo and {initial analysis}
2. Find ONE {thing to improve}
3. {Action}
4. Submit a PR

### What to Look For
- {Specific items in their domain}

### Branch Naming
{role-key}/{description}

### Before Submitting
1. bun run build must pass
2. {Role-specific validation}
```

### Removing a Role

1. **Remove the entry** from the ROSTER table
2. **Deploy** — the coordinator will stop spawning that role
3. Any currently running agents for that role will finish their current task and exit naturally
4. No cleanup needed — agents are ephemeral

### Scaling a Role Up

Change the `desired` count in the ROSTER. For example, to add a third UI Builder:

```
| 4 | UI Builder | ui_builder | 3 | Engineering | Task-driven |
```

The next coordinator cycle will detect a deficit of 1 and spawn an additional instance.

### Scaling a Role Down

Reduce the `desired` count. Currently running excess agents will finish their task and exit. The coordinator only spawns new agents — it never kills running ones. The fleet naturally converges to the desired count as agents complete their work.

### When to Scale

| Signal | Action |
|--------|--------|
| PR review queue > 10 PRs | Increase Code Reviewer count |
| Feature Issue backlog growing | Increase Builder counts |
| Lots of merge conflicts | Reduce parallel Builders (agents are stepping on each other) |
| Agents frequently exit with "nothing to do" | Reduce that role's count |
| New domain area added to project | Add specialized role |
| Test coverage is high and stable | Reduce Quality agents |

---

## Error Handling

The coordinator is designed to be resilient. Every step has independent error handling.

### Per-Step Error Handling

| Step | Error | Handling |
|------|-------|----------|
| Inventory | `sessions_list` fails | Assume 0 running agents; spawn all roles |
| Spawn | Individual spawn fails | Log error; continue to next role |
| Merge | `gh pr list` fails | Skip entire merge step; log error |
| Merge | `gh pr merge` fails for one PR | Skip that PR; continue to next |
| Health | `curl` times out | Report as potential outage; continue |
| Health | `curl` returns 5xx | Report as broken; Sentinel handles remediation |

### Common Error Scenarios

**Agent spawn rate-limited:**
If the agent platform has concurrency limits, spawns may fail. The coordinator logs the failure and tries again next cycle. Since agents exit after completing their task (typically 2–15 minutes), slots free up naturally.

**GitHub API rate limit:**
If the GitHub API returns 429 (rate limited), all PR operations will fail for that cycle. The next cycle (5 minutes later) will retry. Agents that can't push will exit with errors and be respawned.

**Stale PRs accumulating:**
If PRs go unmerged for extended periods (e.g., review requested changes but the original agent is gone):
- The next cycle's agent for that role may create a new PR that supersedes it
- Reviewers can close stale PRs
- Consider adding a stale PR cleanup step to the coordinator (close PRs older than 24 hours without activity)

---

## Monitoring and Status Reporting

### Cycle Summary

Every coordinator cycle outputs a structured summary:

```
=== V4 Coordinator Cycle Complete ===
Time: {ISO timestamp}
Agents running: {count} / {desired} desired
Spawned this cycle: {list of role keys, or "none"}
PRs merged this cycle: {count and numbers, or "none"}
Production health: {HTTP status code}
Errors: {list of errors, or "none"}
=====================================
```

### Key Metrics to Track

| Metric | Healthy Range | Concern |
|--------|---------------|---------|
| Agents running vs desired | Within 5 of desired | Large gap persists across cycles |
| PRs merged per cycle | 0–5 | Consistently 0 with open approved PRs |
| PR review queue depth | 0–10 | Growing beyond 15 |
| Production health | 200 | Non-200 persists across cycles |
| Spawn failures per cycle | 0–2 | >5 consistent failures |
| Time to complete cycle | 1–3 minutes | >4 minutes (approaching cron interval) |

### Health Indicators

**Healthy system:**
- Agents near desired count
- PRs merging regularly
- Production returning 200
- No persistent errors

**Degraded system:**
- Multiple spawn failures per cycle
- PR queue growing (not enough Reviewers)
- Agents frequently finding nothing to do (reduce fleet)
- Many merge conflicts (agents stepping on each other)

**Broken system:**
- Production returning 5xx (Sentinel should auto-revert)
- All spawns failing (platform issue)
- GitHub API errors (rate limiting or outage)

---

## Timing and Performance

### Cron Schedule

The coordinator runs every 5 minutes. Each cycle should complete within 2–3 minutes, leaving a 2-minute buffer before the next run.

### Typical Cycle Timing

| Step | Duration |
|------|----------|
| Inventory | 2–5 seconds |
| Spawn (per agent) | 3–10 seconds |
| Spawn (all deficits) | 30–120 seconds |
| PR merge (per PR) | 5–15 seconds |
| Health check | 1–10 seconds |
| Total cycle | 1–3 minutes |

### Overlapping Runs

If a cycle takes longer than 5 minutes and overlaps with the next cron trigger, this is safe because the coordinator is idempotent:
- Inventory counts are point-in-time snapshots
- Spawning at-capacity roles is a no-op
- Merging already-merged PRs fails gracefully
- Health checks are read-only

---

## Troubleshooting

### Agents aren't spawning

1. Check if the session platform is healthy
2. Check for concurrency limits
3. Look at spawn error messages in the cycle log
4. Verify the coordinator prompt has the correct ROSTER

### PRs aren't merging

1. Check if PRs are actually approved (`reviewDecision: APPROVED`)
2. Check if CI is passing (all status checks `SUCCESS`)
3. Check for merge conflicts (coordinator skips conflicted PRs)
4. Verify the coordinator has merge permissions on the repository

### Production is down

1. Check the Sentinel — is it running? Did it create a revert PR?
2. Check recent merges — what was merged in the last 2 hours?
3. Check if it's an infrastructure issue (hosting platform, DNS, etc.)
4. If Sentinel revert PR exists, manually approve it to expedite the merge

### Too many merge conflicts

1. Reduce the number of parallel Builders (they're modifying overlapping files)
2. Check if agents are making overly broad changes (violating the "small PR" principle)
3. Consider domain sharding — assign agents to specific directories

### Agents producing low-quality PRs

1. Check the agent prompts — are quality standards clear?
2. Increase Reviewer count to catch more issues
3. Check if agents are creating busywork (they should exit when nothing meaningful exists)
4. Tighten the Reviewer's criteria for approval

---

*See also: [V4 Architecture](pipeline-v4-architecture.md) · [Agent Roles](agent-roles.md) · [V3 to V4 Migration](v3-to-v4-migration.md)*
