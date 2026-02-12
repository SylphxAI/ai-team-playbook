# V3 to V4 Migration Guide

> How to migrate from the V3 sequential pipeline to the V4 perpetual motion model.

---

## Table of Contents

- [What Changed and Why](#what-changed-and-why)
- [Key Differences](#key-differences)
- [Migration Steps](#migration-steps)
- [What to Monitor After Switching](#what-to-monitor-after-switching)
- [Rollback Plan](#rollback-plan)
- [FAQ](#faq)

---

## What Changed and Why

### The V3 Problem

V3 used a sequential pipeline with 6 generalist agents:

```
Scout → Triage → Builder → Reviewer → Merger → Sentinel
```

This worked well initially, but hit fundamental limits:

1. **Pipeline bottlenecks.** Each stage was a gate. If Triage was slow, Builders waited. If Reviewers were busy, approved PRs queued up.
2. **Generalist agents, specialist problems.** A single "Builder" agent had to handle UI, API, database, infrastructure, testing, and documentation. It could never go deep.
3. **Label-based state machine.** The pipeline state lived in GitHub labels (`pipeline/discovered`, `pipeline/building`, etc.). This added complexity, created race conditions, and required careful state management.
4. **Coordinator as orchestrator.** The V3 coordinator actively managed workflow — assigning Issues to agents, tracking pipeline state, handling retries. It was complex and stateful.
5. **Low parallelism.** Only one Builder could work on a feature at a time (to avoid conflicts). This capped throughput.

### The V4 Solution

V4 replaces the sequential pipeline with a **perpetual motion model**:

- **28 specialist roles** instead of 6 generalists
- **No pipeline** — all agents work in parallel
- **No state machine** — no labels for pipeline state, no FSM transitions
- **Stateless coordinator** — reconciliation loop, not orchestrator
- **Git-native coordination** — Issues and PRs are the only communication channel

The result: higher throughput, deeper specialization, simpler coordination, and automatic self-healing.

---

## Key Differences

| Aspect | V3 (Pipeline) | V4 (Perpetual Motion) |
|--------|---------------|----------------------|
| **Agent count** | 6 generalist roles | 28 specialist roles (32 instances) |
| **Work flow** | Sequential: Scout → Triage → Build → Review → Merge | Parallel: all agents work simultaneously |
| **State management** | Label-based FSM on GitHub Issues | None — stateless |
| **Coordinator role** | Orchestrator (assigns work, manages state) | Reconciliation loop (maintains fleet size) |
| **Work assignment** | Coordinator assigns Issues to agents | Agents find and claim their own work |
| **Blocking** | Yes — each stage waits for the previous | No — agents never wait |
| **Specialization** | Builder does everything | 28 focused specialties |
| **Pipeline labels** | `pipeline/discovered`, `pipeline/triaging`, `pipeline/building`, `pipeline/reviewing`, `pipeline/merging` | None |
| **Triage step** | Dedicated Triage agent evaluates each Issue | Product Thinkers create pre-scoped Issues; no triage needed |
| **Merge strategy** | Merger agent with batch logic | Coordinator auto-merges approved+CI-passing PRs |
| **Scaling model** | Scale entire pipeline | Scale individual roles independently |
| **Recovery** | Manual intervention or retry logic | Automatic respawn next cycle |
| **Communication** | Labels, state transitions, coordinator messages | Git artifacts only (Issues, PRs, branches) |

---

## Migration Steps

### Step 1: Understand the New Model

Read these docs before migrating:
- [V4 Architecture](pipeline-v4-architecture.md) — the full design
- [Agent Roles](agent-roles.md) — all 28 roles
- [Coordinator Guide](coordinator-guide.md) — how the new coordinator works

### Step 2: Clean Up V3 Artifacts

Before switching, clean up V3 pipeline state:

```bash
# Close any Issues stuck in pipeline states
gh issue list --repo SylphxAI/viral --state open --label "pipeline/building" --json number | \
  jq -r '.[].number' | xargs -I{} gh issue edit {} --repo SylphxAI/viral --remove-label "pipeline/building"

gh issue list --repo SylphxAI/viral --state open --label "pipeline/triaging" --json number | \
  jq -r '.[].number' | xargs -I{} gh issue edit {} --repo SylphxAI/viral --remove-label "pipeline/triaging"

# Merge or close any pending V3 PRs
gh pr list --repo SylphxAI/viral --state open --json number,title
# Review each and merge or close as appropriate
```

### Step 3: Stop the V3 Coordinator

Disable the V3 coordinator cron job. Ensure no V3 agents are still running:

```bash
# List any running V3 sessions
sessions_list with label filter "v3-"
# Kill any remaining V3 agents
```

### Step 4: Deploy the V4 Coordinator

Set up the V4 coordinator prompt as a cron job running every 5 minutes. The coordinator prompt includes:
- The ROSTER (all 28 roles with desired counts)
- The reconciliation algorithm (5 steps)
- Agent prompt templates for all roles

See [Coordinator Guide](coordinator-guide.md) for the full details.

### Step 5: Verify First Cycle

After the first coordinator cycle runs:

1. Check the cycle summary output
2. Verify agents are spawning for each role
3. Watch for the first PRs to be created (usually within 5–10 minutes)
4. Verify Code Reviewers are reviewing PRs
5. Verify the auto-merge step is working for approved PRs

### Step 6: Monitor for 24 Hours

See [What to Monitor After Switching](#what-to-monitor-after-switching) below.

---

## What to Monitor After Switching

### First Hour

| Check | What to Look For |
|-------|-----------------|
| Agent spawning | Are agents being created for all 28 roles? |
| PR creation | Are agents creating PRs? (Expect 5–15 PRs in the first hour) |
| PR reviews | Are Code Reviewers reviewing and approving/rejecting PRs? |
| Auto-merge | Are approved PRs being merged by the coordinator? |
| Production health | Is Sentinel reporting healthy? |
| Build stability | Are PRs passing CI? |

### First Day

| Check | What to Look For |
|-------|-----------------|
| PR quality | Are Reviewers catching real issues? Are agents creating meaningful work? |
| Merge conflicts | How many PRs are being skipped due to conflicts? (Some is normal; lots means too much parallelism) |
| Agent exits | Are self-directed agents exiting when nothing to do? (Good — no busywork) |
| Issue creation | Are Thinkers creating well-scoped Issues? |
| Issue consumption | Are Builders picking up and claiming Issues? |
| Coverage | Are all 10 departments producing output? |

### First Week

| Check | What to Look For |
|-------|-----------------|
| Throughput | How many PRs merged per day? (Target: 20–50) |
| Quality trend | Is the codebase improving? Fewer type errors, better test coverage, better docs? |
| Fleet balance | Are some roles always idle? (Scale them down) Are some always behind? (Scale them up) |
| Stale PRs | Are any PRs sitting unreviewed for >12 hours? (Need more Reviewers) |
| Production stability | Any Sentinel reverts? What caused them? |

### Tuning Signals

| Observation | Action |
|-------------|--------|
| Too many merge conflicts | Reduce Builder count; check if agents are making broad changes |
| PR queue growing | Increase Code Reviewer count |
| Agents creating busywork | Tighten agent prompts; agents should exit when nothing meaningful exists |
| Specific role always idle | Reduce its count to 0 (disable) or 1 |
| Feature Issues piling up | Increase relevant Builder count |
| Test coverage not improving | Check if Unit/Integration/E2E Testers are finding gaps |
| Lots of Reviewer rejections | Check agent prompt quality; consider stricter acceptance criteria in prompts |

---

## Rollback Plan

If V4 isn't working and you need to go back to V3:

1. **Stop V4 coordinator** — disable the cron job
2. **Kill running V4 agents** — `sessions_list` with label filter `v4-`, kill all
3. **Clean up V4 branches** — close any open V4 PRs that shouldn't be merged
4. **Re-enable V3 coordinator** — restore the V3 cron job
5. **Verify V3** — check that the V3 pipeline resumes normal operation

V4 agents don't leave any state behind (no labels, no pipeline markers), so cleanup is minimal. Any PRs they created that were already merged are just normal commits on `main`.

---

## FAQ

### Do I need to remove the V3 pipeline labels from GitHub?

You don't *need* to, but it's cleaner. V4 ignores pipeline labels entirely — they won't cause problems if left in place. You can remove them at your convenience:

```bash
# Remove pipeline labels from all open issues
for label in pipeline/discovered pipeline/triaging pipeline/building pipeline/reviewing pipeline/merging; do
  gh issue list --repo SylphxAI/viral --state open --label "$label" --json number | \
    jq -r '.[].number' | xargs -I{} gh issue edit {} --repo SylphxAI/viral --remove-label "$label"
done
```

### Can I run V3 and V4 side by side?

Technically yes — they use different label prefixes and don't interfere with each other. However, this doubles the agent count and may cause duplicate work. It's better to switch cleanly.

### What happens to V3 Issues that are mid-pipeline?

V4 agents will see them as regular open Issues. Task-driven Builders will pick up unassigned ones. The pipeline labels are ignored.

### Will V4 agents conflict with each other more than V3?

V4 has more agents working in parallel, so there's slightly more potential for overlap. However:
- Branch isolation prevents conflicts during development
- Conflicts only surface at merge time
- Small, focused PRs minimize conflict surface area
- The coordinator skips conflicted merges
- Reviewers catch overlapping work

In practice, the conflict rate is low because each agent works in a narrow domain.

### How long until V4 reaches steady state?

Typically 2–3 cycles (10–15 minutes). By the third cycle, most roles will have spawned, produced output, and exited. The coordinator will be maintaining the fleet at steady state.

### What if a role consistently has nothing to do?

That's working as designed. If the Security Auditor finds no vulnerabilities, it exits in under a minute. The coordinator respawns it next cycle, it checks again, and exits again. The resource cost is minimal (1 minute of compute per cycle).

If a role *never* finds work across many cycles, consider reducing its desired count to 0 to disable it.

---

*See also: [V4 Architecture](pipeline-v4-architecture.md) · [Agent Roles](agent-roles.md) · [Coordinator Guide](coordinator-guide.md)*
