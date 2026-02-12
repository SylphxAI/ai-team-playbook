# Merge Safety & Build Protection

> The single most important lesson from running AI agents: **never let anything merge without passing CI.**
> Every major incident we had traces back to a merge that skipped the build check.

## The PR #381 Disaster

Here's what happened:

1. The Coordinator had direct merge API access
2. A Reviewer approved PR #381
3. CI was still running (lint was passing, build hadn't started)
4. The Coordinator saw "approved" and called `gh pr merge 381`
5. The PR merged with a TypeScript error that would have failed the build
6. Three more PRs were based on this broken `main`
7. All three also had build failures, but the CI was reporting against the wrong base
8. We shipped a broken build to production

**Total damage**: 4 hours of debugging, 6 broken PRs that needed reverting, a production outage.

**The fix**: Remove merge permissions from the Coordinator entirely. Use GitHub auto-merge. Auto-merge literally cannot merge until every required status check passes.

## Rule 1: CI Gate Is Non-Negotiable

```yaml
# Branch protection — MUST have:
required_status_checks:
  strict: true          # Must be up-to-date with base branch
  contexts:
    - build             # Your CI job name
```

"Strict" mode is essential. Without it:

```
main: A → B → C
PR #1 branches from B, passes CI
PR #2 branches from B, passes CI
PR #1 merges → main is now A → B → C → D
PR #2's CI result is stale — it tested against B, not D
PR #2 merges → 💥 possible build break
```

With strict mode, PR #2 must rebase on `main` (now including D) and re-run CI before merging.

## Rule 2: Pre-Merge File Overlap Detection

When you have 3-5 agents working in parallel, they WILL touch the same files. Before the Coordinator assigns work:

```typescript
// Check if any open PR touches the same files as the new issue
async function checkFileOverlap(issueFiles: string[], openPRs: PR[]): Promise<PR[]> {
  const conflicts: PR[] = [];
  
  for (const pr of openPRs) {
    const prFiles = await gh.pulls.listFiles(pr.number);
    const overlap = prFiles.filter(f => issueFiles.includes(f.filename));
    
    if (overlap.length > 0) {
      conflicts.push(pr);
    }
  }
  
  return conflicts;
}
```

**If there's overlap:**
1. Don't spawn a new Builder — wait for the existing PR to merge
2. Or, assign the work with explicit instructions about which files NOT to touch
3. Never assume two agents can independently modify the same file

## Rule 3: Post-Merge Build Verification

Even with CI gates, things can slip through. Run a build check after every merge to `main`:

```yaml
# .github/workflows/post-merge.yml
name: Post-Merge Verification

on:
  push:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      
      - name: Alert on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: '🚨 Post-merge build failure on main',
              body: `The build broke after merging ${context.sha}.\n\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['bug', 'priority:critical']
            });
```

This is your safety net. If a broken build somehow reaches `main`, you'll know within minutes.

## Rule 4: Duplicate Fix Detection

AI agents don't coordinate with each other. Two Builders might independently fix the same bug:

```
Issue #42: "Fix login button color"
  → PR #43 by Builder A: changes button color in LoginPage.tsx
  → PR #44 by Builder B: changes button color in LoginPage.tsx (assigned before #43 existed)
```

The Coordinator must check for this:

```typescript
async function detectDuplicatePRs(issueNumber: number): Promise<PR[]> {
  const prs = await gh.pulls.list({ state: 'open' });
  return prs.filter(pr => {
    const body = pr.body || '';
    return body.includes(`#${issueNumber}`) || 
           body.includes(`Closes #${issueNumber}`) ||
           body.includes(`Fixes #${issueNumber}`);
  });
}
```

**When duplicates are found:**
1. Keep the PR that's further along (has reviews, CI passing)
2. Close the other with a comment: "Superseded by #43"
3. Don't merge both — they'll conflict

## Rule 5: Stale PR Handling

PRs go stale when:
- The Builder agent hit an error and stopped mid-implementation
- The Fixer never addressed review feedback
- The PR has been open for 24+ hours with no activity

**Stale PR policy:**

```
Age > 24h with no activity → Comment asking for update
Age > 48h with no activity → Close with "stale" label
Age > 72h with no activity → Delete the branch
```

The Coordinator should run a stale PR check on every audit cycle:

```bash
# Find PRs with no activity in 24 hours
gh pr list --state open --json number,updatedAt --jq '.[] | select(.updatedAt < (now - 86400 | todate))'
```

## Rule 6: Why Sequential Migration Numbering Fails

This is a specific but critical issue. If you use Drizzle ORM with sequential migration files:

```
migrations/
  0001_initial.sql
  0002_add_users.sql
  0003_add_posts.sql
```

And two agents create migrations in parallel:

```
Agent A creates: 0004_add_comments.sql
Agent B creates: 0004_add_likes.sql    ← CONFLICT
```

Both pass CI independently (they each test against a database with migrations 0001-0003). When merged, you have two `0004` files and the migration runner breaks.

**Solutions:**
1. Use timestamp-based naming: `20260212_143022_add_comments.sql`
2. Use a declarative migration tool like Atlas that generates migrations from schema diff
3. Don't let agents create migration files — use `drizzle-kit push` in dev and generate migrations in CI

See [Migration Strategy](06-migration-strategy.md) for the full approach.

## The Merge Flow

With all these rules in place:

```
Builder opens PR
    ↓
CI runs (build + lint + test)
    ↓
Coordinator checks for duplicates and file overlap
    ↓
Reviewer reviews → approves or requests changes
    ↓
If approved + CI green → auto-merge (squash)
    ↓
Post-merge verification runs
    ↓
If post-merge fails → auto-create critical issue
```

No agent has merge permissions. No agent makes merge decisions. The infrastructure handles it.
