# Coordinator Design — The Reconciliation Loop

> The Coordinator is a Kubernetes controller, not a task scheduler.
> It reads state, reconciles, and spawns agents. It never writes code.
> Every step is idempotent. No locks needed. Overlapping runs are harmless.

## Core Principle: Reconciliation, Not Orchestration

Traditional orchestrators:
```
❌ Step 1: Do X
❌ Step 2: Wait for X to finish
❌ Step 3: Do Y based on X's result
❌ Step 4: ...
```

Reconciliation loop:
```
✅ Read current state
✅ Compare to desired state
✅ Take actions to close the gap
✅ Repeat (every 5 minutes)
```

The Coordinator doesn't remember what it did last cycle. It doesn't track state between runs. Every cycle, it reads ALL state from GitHub and decides what needs to happen RIGHT NOW.

## The 11-Step Algorithm

Every coordinator cycle executes these steps in order:

### Step 1 — Inventory

Gather ALL pipeline state from GitHub:

```bash
# All open issues by pipeline state
gh issue list --label "pipeline/discovered" --json number,title,labels,assignees,createdAt
gh issue list --label "pipeline/approved" --json number,title,labels,assignees,createdAt
gh issue list --label "pipeline/in-progress" --json number,title,labels,assignees,createdAt
gh issue list --label "pipeline/pr-open" --json number,title,labels,assignees,createdAt

# All open PRs with review and CI status
gh pr list --state open --json number,title,labels,reviewDecision,mergeable,statusCheckRollup

# Recently merged PRs (for Sentinel)
gh pr list --state merged --json number,title,mergedAt --limit 10

# Active sub-agent sessions (for duplicate prevention)
# Check running sessions, filter by label prefix
```

Build a mental model of the entire pipeline before proceeding.

### Step 2 — Timeouts

Heal stuck work:

| Condition | Action |
|-----------|--------|
| `pipeline/in-progress` + no PR + no active builder for >2h | Re-queue as `pipeline/approved` |
| `pipeline/pr-open` + no review for >4h | Flag as urgent |
| `pipeline/discovered` + older than 24h | Auto-reject (stale) |
| `pipeline/review-failed` + no fix for >4h | Close PR, re-queue issue |

### Step 3 — Sentinel

Check production health after recent merges:

```bash
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://your-app.com)
```

- **HTTP 200:** Comment "✅ post-merge verified" on recently merged PRs
- **HTTP != 200:** Create P0 issue, identify likely culprit merge

### Step 4 — Merge

Find PRs ready to merge: `pipeline/review-passed` + CI green + no conflicts.

Sort by priority (P0 first). **Merge ALL ready PRs in the same cycle** (batch merge).

Handle Vercel stuck statuses: if pending >10 min, override with success.

### Step 5 — Review

Find PRs needing review: `pipeline/pr-open` + CI green + no review.

For each, check if a reviewer session already exists. If not, spawn one.

### Step 6 — Fix

Find PRs with failed reviews: `pipeline/review-failed` or `reviewDecision == CHANGES_REQUESTED`.

Count previous fix attempts. If >= MAX_FIX_RETRIES (2), abandon the PR and re-queue the issue. Otherwise, spawn a Fixer.

### Step 7 — Rebase

Find PRs that are approved but have merge conflicts (`mergeable == CONFLICTING`).

Spawn a Rebaser for each.

### Step 8 — Build

Find approved issues ready for work: `pipeline/approved` + no assignee.

Check WIP limit. Sort by priority, then age (oldest first). Spawn Builders up to available slots.

### Step 9 — Triage

Find discovered issues: `pipeline/discovered`.

Spawn a Triage agent for each that doesn't already have one running.

### Step 10 — Scout

Only if backlog is low: `count(approved + discovered) < BACKLOG_THRESHOLD`.

Spawn a Scout to discover new issues.

### Step 11 — Status

Update the metrics dashboard (pinned GitHub issue) with current pipeline state. Print a console summary.

If nothing was done this cycle → print status and exit. Don't invent work.

## Why No Locks

The initial design included a lock step (acquire/release via a GitHub issue). We removed it. Here's why:

### Every Step Is Already Idempotent

| Operation | Idempotent? | Why |
|-----------|-------------|-----|
| Assigning an issue | Yes | Already assigned → skip |
| Spawning a builder | Yes | Session already exists → skip |
| Merging a PR | Yes | Already merged → skip |
| Adding a label | Yes | GitHub is a no-op if label exists |
| Closing an issue | Yes | Already closed → skip |
| Creating a P0 issue | Yes | Check for existing P0 first |

### What Happens If Two Cycles Overlap?

```
Cycle A (5:00): Reads state, sees PR #100 ready to merge
Cycle B (5:01): Reads state, sees PR #100 ready to merge

Cycle A: Merges PR #100 ✅
Cycle B: Tries to merge PR #100 → already merged → skips ✅
```

Both cycles produce the same result. No data corruption. No duplicate work.

### The Lock Anti-Pattern

```
❌ Acquire lock → lock holder crashes → 10 min timeout → dead time
❌ Lock TTL too short → overlapping runs anyway
❌ Lock TTL too long → artificial delays
❌ Lock contention → reduced throughput
```

Locks solve a problem that doesn't exist when operations are idempotent.

## Configuration

```
WIP_LIMIT: 10           # Max concurrent in-progress issues
BACKLOG_THRESHOLD: 15    # Scout runs when backlog < this
MAX_FIX_RETRIES: 2       # Max fix attempts before abandoning PR
CRON_INTERVAL: 5 min     # Coordinator cycle frequency
```

### Tuning WIP Limit

| WIP | Builders | Theoretical Max | Notes |
|-----|----------|-----------------|-------|
| 3 | 3 | ~12 PRs/hour | Conservative, good for starting |
| 5 | 5 | ~20 PRs/hour | Moderate |
| 10 | 10 | ~30 PRs/hour | Our current setting |

Formula: `PRs/hour ≈ WIP_LIMIT × completions_per_builder_per_hour`

Most builders complete 2-4 PRs/hour depending on issue complexity.

## Priority Scoring

Work is always ordered by:
1. **Priority**: P0 (1000 pts) > P1 (100 pts) > P2 (10 pts)
2. **Age**: +5 pts per day (prevents starvation)
3. **Size**: -1 pt per size tier (slight preference for smaller work)

```
score = priority_weight + (age_days × 5) - size_penalty
```

## Duplicate Prevention

Before spawning ANY agent, the Coordinator checks:

1. **Is there already an active session for this work?** (e.g., `viral-build-issue42`)
2. **Is there already a PR for this issue?**
3. **Is the issue already assigned?**
4. **Is the issue already closed?**
5. **Do we have available WIP slots?**

If any check fails → skip.

### Stale Session Detection

A session is "stale" if it exists but the work hasn't progressed:
- Builder session exists but no PR created after 30 min
- Fixer session exists but PR still has `CHANGES_REQUESTED`
- Rebaser session exists but PR is still `CONFLICTING`

For stale sessions: ignore them in duplicate checks and spawn fresh agents with incremented suffixes (`fix2-PR42`, `rebase2-PR42`).

## Coordinator Anti-Patterns

### ❌ Coordinator reviews code
It doesn't have the context. It will approve bad PRs. That's the Reviewer's job.

### ❌ Coordinator merges directly
Use the Merger role (or auto-merge). The Coordinator shouldn't call merge APIs.

### ❌ Coordinator creates cron jobs
We had an incident where the Coordinator created 3 cron jobs that burned $50 in API tokens in 4 hours. Your prompt must say: "Do NOT create cron jobs, scheduled tasks, or any persistent processes."

### ❌ Coordinator reads raw HTML
Fetching a web page can consume 200k tokens. Restrict to structured data only.

### ❌ Coordinator does work when busy
If handling a user message while coordinating, both block. Separate human-facing agent from coordination.

## Error Handling

If any step fails:
1. Log the error clearly
2. Continue to the next step (don't abort the whole cycle)
3. Always reach Step 11 to output status

If GitHub API is rate-limited:
1. Skip non-essential steps (Scout, metrics update)
2. Priority order: Merge > Review > Fix > Build > Triage > Scout

## Human Override

Humans can interact with the pipeline at any time:
- Add `pipeline/approved` + priority label → skip triage
- Add `source/human` label → mark as human-created
- Close/reopen issues → override pipeline decisions
- Comment with feedback → agents pick up on next cycle

The coordinator respects human labels and never overrides them.
