# Merge Safety — Branch Protection & Merge Strategy

> **Key finding: NO major open source project uses `strict: true` at scale.**
> We researched Kubernetes, React, Next.js, Rust, and GitHub's own monorepo.
> They all use `strict: false` (or merge queues) with post-merge monitoring.

## The `strict: true` Problem

GitHub's "Require branches to be up to date before merging" (`strict: true`) means every PR must be rebased on the latest `main` and have CI re-run before merging.

**The theory:** This prevents "semantic merge conflicts" — two PRs that individually pass CI but break when combined.

**The reality at scale:**

```
With strict: true and 10 queued PRs:

PR #1 merges (20 min: rebase + CI + merge)
  → PRs #2-#10 are now stale, must rebase and re-run CI

PR #2 merges (20 min)
  → PRs #3-#10 must rebase again

PR #3 merges (20 min)
  → ... you get the idea

Total: 10 PRs × 20 min = 3.3 hours (SERIAL)
```

```
With strict: false and 10 queued PRs:

All 10 merge in parallel: ~2 minutes total
Sentinel verifies production: ~3 minutes
Total: ~5 minutes
```

**We switched from `strict: true` to `strict: false` and merged 14 PRs in 2 minutes.** The same queue would have taken 4+ hours with `strict: true`.

## What Major OSS Projects Do

We researched how high-throughput open source projects handle this:

| Project | `strict: true`? | PR Volume | Strategy |
|---------|----------------|-----------|----------|
| **Kubernetes** | No | 2,500+/month | Prow/Tide batch merge |
| **React** | No | High | CI + human review |
| **Next.js** | No | High | CI + Vercel's own tooling |
| **Rust** | No | High | Homu/Bors → GitHub merge queue |
| **GitHub (monorepo)** | No | 2,500+/month | Merge queue batching 15-30 PRs |

**The consensus:** `strict: true` is designed for small teams who rarely merge. Any project merging more than a few PRs/day uses an alternative.

### How Kubernetes Does It (Prow/Tide)

Kubernetes processes 2,500+ PRs/month using Prow (their bot framework) and Tide (their merge controller):

- Tide batches multiple approved PRs together
- Tests them as a group against `main`
- If the batch passes, all PRs merge
- If it fails, Tide bisects to find the culprit
- No individual PR needs to be "up to date" — the batch is tested as a whole

### How GitHub Does It (Merge Queue)

GitHub's own monorepo uses their merge queue feature:
- PRs enter a queue when approved
- The queue batches 15-30 PRs into groups
- Each group is tested against `main` + all previous groups
- Passed groups merge automatically
- This replaces `strict: true` entirely

### Why Not Use Merge Queue?

Merge queue is the ideal solution for large organizations. For our autonomous pipeline:
- It adds configuration complexity
- `strict: false` + Sentinel achieves the same safety with simpler infrastructure
- Merge queue is designed for human teams; our pipeline has its own coordination

## Our Branch Protection Settings

```yaml
# What we use (recommended)
required_status_checks:
  strict: false          # ← Branches do NOT need to be up-to-date
  contexts:
    - build              # Single required check

required_pull_request_reviews:
  required_approving_review_count: 1
  dismiss_stale_reviews: true

enforce_admins: false    # Emergency escape hatch for humans
allow_force_pushes: false
allow_deletions: false
required_linear_history: true
```

### Repo-Level Merge Settings

```
✅ Allow squash merge (ONLY strategy enabled)
❌ Allow merge commits (disabled)
❌ Allow rebase merge (disabled)
✅ Auto-delete head branches after merge
✅ Allow auto-merge

Squash commit title: PR title
Squash commit message: PR body
```

**Why squash only?** AI agents make messy commits: "fix lint", "try again", "actually fix it." Squash collapses everything into one clean commit per PR. Your `main` history reads like a changelog.

## The Safety Net: Sentinel Post-Merge Monitoring

With `strict: false`, we accept that a rare semantic merge conflict might slip through. Our mitigation: **detect and fix immediately instead of preventing at the gate.**

```
Layer 1: CI on every PR (catches individual build failures)
Layer 2: Review on every PR (catches logic errors)
Layer 3: Sentinel after every merge batch (catches integration failures)
Layer 4: Auto-revert if production breaks (limits blast radius)
```

### Sentinel Check Flow

```
PRs merged in this cycle
        ↓
Wait for Vercel deployment (~2-3 min)
        ↓
Health check: curl -s https://your-app.com
        ↓
HTTP 200? → ✅ Comment "post-merge verified" on each PR
HTTP != 200? → 🚨 Create P0 issue, identify culprit, consider auto-revert
```

### The PR #381 Disaster (Why Auto-Merge Alone Isn't Enough)

Early in our project, the Coordinator had direct merge API access:

1. Reviewer approved PR #381
2. CI was still running (lint passing, build hadn't started)
3. Coordinator saw "approved" and called `gh pr merge 381`
4. PR merged with a TypeScript error
5. Three more PRs built on this broken `main`
6. Production broke

**Total damage:** 4 hours of debugging, 6 broken PRs, a production outage.

**The fix:** Remove merge permissions from the Coordinator. Use auto-merge (merges only when ALL checks pass) or have the Merger agent verify CI status before merging.

## Batch Merging

With `strict: false`, merge ALL ready PRs in the same coordinator cycle:

```bash
# All of these can run in the same cycle — no waiting between them
gh pr merge 409 --squash --delete-branch
gh pr merge 410 --squash --delete-branch
gh pr merge 411 --squash --delete-branch
gh pr merge 412 --squash --delete-branch
# ...
```

On launch day, we merged 14 PRs in 2 minutes. With `strict: true`, this would have taken 4+ hours.

## Auto-Merge vs Direct Merge

**Auto-merge** (GitHub feature): PR merges automatically when all checks pass. You enable it on a PR and forget about it.

**Direct merge** (via API): `gh pr merge --squash --delete-branch`. Immediate, but you must verify checks yourself.

**Our experience with auto-merge:**
- It doesn't work well with `strict: true` — cascading conflicts block everything
- Vercel stuck `pending` statuses block auto-merge indefinitely
- With `strict: false`, auto-merge works but adds latency (waits for next CI cycle)

**Our approach:** Direct merge via the Merger agent, which verifies CI and review status before merging. Faster and more predictable.

## Duplicate PR Detection

AI agents don't coordinate with each other. Two Builders might fix the same issue:

```
Issue #42: "Fix login button color"
  → PR #43 by Builder A: changes button color
  → PR #44 by Builder B: changes button color (different approach)
```

**Prevention:** The Coordinator checks for existing PRs before spawning a Builder.

**When duplicates appear:**
1. Keep the PR that's further along (has reviews, CI passing)
2. Close the other with: "Superseded by #43"
3. Never merge both

## Stale PR Handling

PRs go stale when:
- Builder hit an error and stopped mid-implementation
- Fixer never addressed review feedback
- PR has been open >24 hours with no activity

**Policy:**

| Age | Action |
|-----|--------|
| > 4 hours with no review | Flag as urgent, priority-assign reviewer |
| > 24 hours with no activity | Comment asking for update |
| > 48 hours with no activity | Close PR, re-queue issue as `pipeline/approved` |
