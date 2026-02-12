# Coordinator Architecture

> The Coordinator is the most important agent and the most dangerous one.
> Keep it lightweight. Keep it dumb. The moment it gets clever, things break.

## Core Principle: The Coordinator Is a Dispatcher

The Coordinator should do exactly three things:

1. **Spawn agents** — assign work to Builders, Reviewers, Fixers
2. **Heal stale work** — close abandoned PRs, reassign stuck issues
3. **Find new work** — run audit cycles to discover issues

That's it. Not code. Not merge. Not review. Not deploy. Dispatch.

## Why Single Coordinator

Running two coordinators causes:
- Duplicate agent spawns (two Builders for the same issue)
- Conflicting PR closures (both try to clean up the same stale PR)
- Race conditions in slot management
- Double the API rate limit consumption

One coordinator is a bottleneck by design. If it's too slow, make it faster — don't add another.

## Keeping It Lightweight

The Coordinator's token budget is precious. If it spends 100k tokens reading a huge codebase, it has less capacity for coordination decisions.

**What bloats the Coordinator:**
- Reading full file contents (it doesn't need to — that's the Builder's job)
- Reviewing PR diffs (that's the Reviewer's job)
- Debugging build failures (that's the Fixer's job)
- Parsing raw HTML from web fetches (200k tokens gone in one call)

**What the Coordinator should read:**
- Issue titles and labels (not full bodies unless needed)
- PR status (open/closed/merged, CI status, review status)
- Agent status (running/completed/failed)
- File lists (not file contents)

## Cron-Based vs Event-Driven

### Cron-Based (what we use)

```
Every 15 minutes:
  1. Check for new issues → spawn Builders
  2. Check for PRs needing review → spawn Reviewers
  3. Check for review feedback → spawn Fixers
  4. Check for stale PRs → close or reassign
  5. Check for merge conflicts → spawn Rebasers
```

**Pros:**
- Simple to implement
- Predictable resource usage
- Easy to debug (check the cron logs)

**Cons:**
- 0-15 minute delay between events and responses
- Wasted cycles when nothing changed

### Event-Driven (alternative)

```
GitHub webhook → "PR opened" → spawn Reviewer
GitHub webhook → "review submitted" → spawn Fixer (if changes requested)
GitHub webhook → "issue opened" → spawn Builder
```

**Pros:**
- Instant response to events
- No wasted cycles

**Cons:**
- Requires webhook infrastructure
- Harder to debug
- Can be overwhelmed by event storms (e.g., bulk issue creation)

**Our recommendation:** Start with cron. Move to event-driven when latency matters.

## Slot Management

Agents cost money (API calls, compute). You need limits:

```typescript
interface SlotConfig {
  maxBuilders: number;     // e.g., 3
  maxReviewers: number;    // e.g., 2
  maxFixers: number;       // e.g., 2
  maxRebasers: number;     // e.g., 1
}

async function canSpawnBuilder(): Promise<boolean> {
  const activeBuilders = await getActiveAgents('builder');
  return activeBuilders.length < config.maxBuilders;
}
```

**Why limit slots?**
- 5 Builders running simultaneously = 5 PRs that might conflict
- Each agent consumes API tokens ($$$)
- Too many parallel PRs overwhelms the Reviewer
- CI queues get long with 10+ concurrent PRs

**Recommended starting slots:**
- 2-3 Builders
- 1 Reviewer (it can handle multiple PRs sequentially)
- 1 Fixer
- 1 Rebaser

## Duplicate Prevention

Before spawning a Builder for issue #42, check:

```typescript
async function shouldSpawnBuilder(issueNumber: number): Promise<boolean> {
  // 1. Is there already an open PR for this issue?
  const existingPRs = await findPRsForIssue(issueNumber);
  if (existingPRs.length > 0) return false;

  // 2. Is there already a Builder working on this issue?
  const activeBuilders = await getActiveAgents('builder');
  const alreadyAssigned = activeBuilders.some(a => a.issueNumber === issueNumber);
  if (alreadyAssigned) return false;

  // 3. Is the issue already closed?
  const issue = await gh.issues.get(issueNumber);
  if (issue.state === 'closed') return false;

  // 4. Do we have available slots?
  if (activeBuilders.length >= config.maxBuilders) return false;

  return true;
}
```

Without these checks, the Coordinator will happily spawn 3 Builders for the same issue on consecutive cron cycles.

## File-Based Prompt Pattern

Don't hardcode agent prompts in your Coordinator code. Store them in files:

```
prompts/
  builder.md      — system prompt for Builder agents
  reviewer.md     — system prompt for Reviewer agents
  fixer.md        — system prompt for Fixer agents
  coordinator.md  — system prompt for the Coordinator itself
```

**Why files?**
- Edit prompts without redeploying the Coordinator
- Version control on prompts (git blame shows when/why a prompt changed)
- Easy A/B testing (swap prompt files)
- Agents can read their own prompts at startup

**Prompt structure:**

```markdown
# Builder Agent Prompt

## Role
You are a Builder agent. Your job is to implement a single GitHub issue.

## Rules
1. Create a branch named `feat/issue-{number}-{description}`
2. Implement ONLY what the issue describes
3. Run `pnpm build` before pushing
4. Run `pnpm lint` before pushing
5. Open a PR referencing the issue with `Closes #{number}`
6. Do NOT review your own code
7. Do NOT merge the PR
8. Do NOT pick up additional work

## Context
- Repository: {repo}
- Issue: #{issue_number}
- Issue title: {issue_title}
- Issue body: {issue_body}

## Files you may need to reference
{relevant_file_list}
```

## Coordinator Anti-Patterns

### ❌ Coordinator reviews code
"Let me just quickly check if this PR looks right before spawning a reviewer..."
No. The Coordinator doesn't have the context to review code. It will approve bad PRs.

### ❌ Coordinator merges PRs
"The PR is approved and CI passed, let me merge it..."
No. Auto-merge handles this. The Coordinator merging introduces race conditions.

### ❌ Coordinator creates cron jobs
We had an incident where the Coordinator's prompt said "ensure the project stays healthy" and it interpreted that as creating a cron job that ran every 5 minutes, consuming API tokens. Your prompt must explicitly say: "Do NOT create cron jobs, scheduled tasks, or any persistent processes."

### ❌ Coordinator reads raw HTML
A Coordinator that fetches a web page to "understand the project context" can consume 200k tokens on a single HTML page. Always restrict it to reading structured data (JSON, issue bodies, PR metadata).

### ❌ Coordinator does work when busy
If the Coordinator is handling a user message and simultaneously trying to coordinate agents, it blocks both. The human-facing agent should delegate ALL work — never implement anything itself.

## Example Coordinator Loop

```python
# Simplified coordinator loop
async def coordinator_cycle():
    # Phase 1: Assess current state
    open_issues = await get_open_issues(labels=['ready'])
    open_prs = await get_open_prs()
    active_agents = await get_active_agents()
    
    # Phase 2: Handle PRs needing attention
    for pr in open_prs:
        if pr.needs_review and not has_active_reviewer(pr):
            await spawn_reviewer(pr)
        elif pr.has_changes_requested and not has_active_fixer(pr):
            await spawn_fixer(pr)
        elif pr.has_conflicts and not has_active_rebaser(pr):
            await spawn_rebaser(pr)
        elif pr.is_stale(hours=48):
            await close_stale_pr(pr)
    
    # Phase 3: Assign new work
    for issue in open_issues:
        if can_spawn_builder() and should_spawn_builder(issue):
            await spawn_builder(issue)
    
    # Phase 4: Audit (less frequently)
    if should_run_audit():
        await spawn_auditor()
```

This entire loop should complete in under 30 seconds. If it takes longer, the Coordinator is doing too much.
