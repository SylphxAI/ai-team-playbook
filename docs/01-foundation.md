# Project Foundation — Day 1 Setup

> Everything in this document MUST exist before any agent writes a single line of code.
> Skip this and you'll spend your first week cleaning up broken merges instead of shipping features.

## The Non-Negotiable Checklist

Before you give any AI agent write access to your repository:

- [ ] GitHub Actions CI pipeline (build + lint + test)
- [ ] Branch protection on `main` (CI pass + 1 approval required)
- [ ] Auto-merge enabled (approved + CI green → merge automatically)
- [ ] Squash merge only (clean, linear history)
- [ ] Auto-delete head branches after merge

**Why all five?** Because AI agents are fast, parallel, and have no judgment about merge timing. Without these guardrails, you'll merge broken code within the first hour.

## 1. GitHub Actions CI

Every PR must pass a build check before it can be merged. This is your only safety net when agents are creating 10+ PRs per day.

### Example: Next.js + pnpm + Bun

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm lint

      - name: Type check
        run: pnpm tsc --noEmit

      - name: Build
        run: pnpm build

      - name: Test
        run: pnpm test
        if: hashFiles('**/*.test.*') != ''
```

### Key Details

- **`--frozen-lockfile`**: Agents sometimes run `pnpm install` and commit a changed lockfile. This flag fails the build if the lockfile is out of sync, forcing them to commit the correct one.
- **`concurrency` + `cancel-in-progress`**: When an agent pushes a fix to an existing PR, cancel the old run immediately. Without this, you'll burn through Actions minutes fast.
- **`timeout-minutes: 10`**: Agents sometimes create PRs that cause infinite loops or hanging builds. Always set a timeout.
- **`if: hashFiles('**/*.test.*') != ''`**: Skip test step if no test files exist yet. Agents will add tests incrementally.

## 2. Branch Protection

Go to **Settings → Branches → Branch protection rules → Add rule** for `main`:

```
Branch name pattern: main

☑ Require a pull request before merging
  ☑ Require approvals: 1
  ☐ Dismiss stale pull request approvals when new commits are pushed
  ☑ Require review from Code Owners (optional, useful later)

☑ Require status checks to pass before merging
  ☑ Require branches to be up to date before merging
  Status checks that are required:
    - build (or whatever your job name is)

☑ Require linear history

☐ Require signed commits (not needed for bots)

☐ Include administrators (keep unchecked — you need an escape hatch)
```

### Via GitHub API (for automation)

```bash
gh api repos/SylphxAI/viral/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

### Critical: "Require branches to be up to date"

This setting means a PR can only merge if it's rebased on the latest `main`. This is **essential** with parallel agents because:

1. Agent A creates PR touching `components/Header.tsx`
2. Agent B creates PR touching `lib/api.ts`
3. Both pass CI independently
4. Agent A merges first
5. Agent B's CI is now stale — it never built against A's changes
6. Without this setting, B merges and might break the build

With this setting, B's PR is blocked until it's rebased and CI re-runs.

## 3. Auto-Merge

Enable auto-merge in **Settings → General → Pull Requests**:

```
☑ Allow auto-merge
```

When a reviewer approves a PR, they (or the PR author) can enable auto-merge. The PR will merge automatically once all status checks pass.

**Why this matters**: The coordinator agent should NEVER call the merge API directly. Instead:
1. Builder creates PR
2. Reviewer approves PR
3. Auto-merge kicks in when CI passes

This removes the coordinator from the merge path entirely, which prevents a whole class of bugs (see [Merge Safety](03-merge-safety.md)).

## 4. Squash Merge

In **Settings → General → Pull Requests**:

```
☐ Allow merge commits
☑ Allow squash merging
  Default commit message: Pull request title
☐ Allow rebase merging
```

**Why squash only?**

- AI agents make messy commits: "fix lint", "try again", "actually fix it this time"
- Squash merge collapses all of that into one clean commit per PR
- Your `main` branch history reads like a changelog: one commit = one feature/fix
- Makes `git bisect` actually useful when something breaks

## 5. Auto-Delete Branches

In **Settings → General → Pull Requests**:

```
☑ Automatically delete head branches
```

AI agents create a LOT of branches. Without auto-delete, you'll have hundreds of stale branches within a week. This is pure hygiene.

## The Result

With all five in place, your workflow becomes:

```
Agent creates branch → pushes code → opens PR
                                        ↓
                              CI runs (build + lint + test)
                                        ↓
                              Reviewer agent approves (or requests changes)
                                        ↓
                              Auto-merge (squash) when CI green
                                        ↓
                              Branch auto-deleted
```

No human in the loop. No coordinator making merge decisions. Just infrastructure doing its job.

## Common Mistakes

### "We'll add CI later"

No. Add it on day 1, even if the only check is `echo "hello"`. The branch protection rule needs a status check name to require. If you add CI after agents are already working, you'll have a window where broken code can merge freely.

### "Branch protection is too strict for early development"

The whole point is strictness. Agents don't have judgment. They will merge anything that looks green. Branch protection is what gives "green" meaning.

### "Auto-merge is scary"

Auto-merge WITH branch protection is safer than manual merge. A human might click "merge" when CI is still running. Auto-merge literally cannot merge until every check passes.
