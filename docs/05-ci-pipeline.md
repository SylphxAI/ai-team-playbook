# CI Pipeline — Speed Over Ceremony

> Your CI should catch real problems fast. Not gatekeep with slow, flaky checks.
> Required check: `build` only. Lint as non-blocking. Sentinel as safety net.

## Philosophy

With autonomous agents creating 30+ PRs/day, CI speed is directly proportional to throughput. Every minute of CI time is multiplied across every PR.

**Our approach:**
- **One required check:** `build` (includes TypeScript compilation)
- **Lint as non-blocking:** `continue-on-error: true` (agents fix lint issues but CI doesn't block on them)
- **Minimal required checks:** Speed + Sentinel post-merge monitoring as safety net

## CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  build:
    name: Build & Validate
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Install dependencies
        run: bun install --frozen-lockfile

      - name: Lint
        run: bun run lint
        continue-on-error: true  # Non-blocking

      - name: Type check
        run: bun run typecheck

      - name: Build
        run: bun run build

      - name: Test
        run: bun test
        if: hashFiles('**/*.test.*', '**/*.spec.*') != ''
```

## Key Design Decisions

### Why Only `build` Is Required

Adding more required checks (lint, test, type-check as separate jobs) seems safer, but:

1. **More checks = more failure points.** Each check can independently have flaky behavior.
2. **Parallel jobs mean CI takes as long as the slowest.** One slow check blocks everything.
3. **The `build` step already includes type-checking** for Next.js projects.
4. **Sentinel catches anything CI misses** via post-merge monitoring.

### Why Lint Is Non-Blocking

```yaml
- name: Lint
  run: bun run lint
  continue-on-error: true  # ← This is intentional
```

Lint catches style issues, not correctness issues. Blocking a PR because of a missing semicolon while production is waiting for a critical fix is wrong.

Agents fix lint issues naturally (they run lint before pushing). The non-blocking approach means:
- Lint failures are visible in the CI output
- They don't block the merge
- The Reviewer can flag lint issues in review if they're significant

### Why `--frozen-lockfile`

AI agents sometimes run `bun install` and accidentally modify the lockfile. `--frozen-lockfile` fails the build if the lockfile doesn't match `package.json`, forcing agents to commit the correct lockfile.

### Why `cancel-in-progress`

When an agent pushes a fix commit, the old CI run is cancelled immediately. Without this:
- You burn Actions minutes on stale runs
- The PR shows a "pending" check from the old run
- Agents get confused about which status to trust

### Why `timeout-minutes: 10`

AI-generated code sometimes creates:
- Infinite loops in tests
- Build-time fetches to non-existent URLs (30-minute DNS timeout)
- Components that recursively render themselves

Hard cap prevents these from burning unlimited Actions minutes.

## Build Pipeline Order

```
1. Install (bun install --frozen-lockfile)
2. Lint (non-blocking — fast feedback)
3. Type check (catches type errors without full build)
4. Build (full build — catches runtime config issues)
5. Test (only if test files exist)
```

**Why lint before build?** Lint is fast (5-10 seconds). If there's a missing import, fail fast.

**Why type check separately?** `build` runs type checking as part of the build, but errors are mixed with build output. Separate `tsc --noEmit` gives clean, focused type errors.

## Post-Merge Verification

Even with CI gates, verify after every merge to `main`:

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
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run build

      - name: Create issue on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🚨 Build broken on main — ${context.sha.substring(0, 7)}`,
              body: `Post-merge build failure.\n\nCommit: ${context.sha}\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['bug', 'priority/P0', 'pipeline/discovered', 'source/sentinel'],
            });
```

This auto-creates a P0 issue that the Coordinator picks up immediately.

## Common CI Failures from AI Agents

### 1. Missing Dependencies (pnpm/bun strict mode)

```
error: Module not found: @some/package
```

Agents use transitive dependencies (installed by something else) that aren't in `package.json`. Works with npm's hoisting, fails with strict mode.

**Fix:** Agent prompt: "If you import a package, verify it's in `package.json`. If not, add it."

### 2. Import Path Case Sensitivity

```
Module not found: Can't resolve './components/loginPage'
# File is: ./components/LoginPage.tsx
```

macOS is case-insensitive, Linux (CI) is case-sensitive. Agents running on macOS sandboxes don't notice the mismatch.

### 3. Environment Variable Assumptions

```
TypeError: Cannot read properties of undefined (reading 'split')
# process.env.DATABASE_URL.split('/')
```

Agents assume env vars exist. Build step doesn't have them.

**Fix:** Provide dummy env vars in CI, or use `zod` to validate with defaults.

### 4. Build-Only Runtime Errors

```
Error: useAuth must be used within an AuthProvider
```

TypeScript compiles fine. The error only appears at runtime. CI won't catch it without integration tests.

**Fix:** Add basic render tests for components using context providers.

## Cost Control

For private repos, Actions minutes cost money:

```yaml
on:
  pull_request:
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'
```

- Skip CI for docs-only changes
- Use `concurrency` + `cancel-in-progress` to kill stale runs
- Cache `node_modules` aggressively
- Set `timeout-minutes` to prevent runaway builds
