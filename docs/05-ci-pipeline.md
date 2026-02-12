# CI/CD Pipeline Design

> CI is your only quality gate. Make it fast, make it reliable, make it mandatory.

## Overview

```
PR opened/updated
    ↓
GitHub Actions: build + lint + type-check + test
    ↓
Review (agent or human)
    ↓
Auto-merge (squash) when approved + CI green
    ↓
Push to main
    ↓
Vercel auto-deploy to production
    ↓
Post-merge verification
```

## GitHub Actions: PR Validation

### Full CI Workflow

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

env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

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
        env:
          # Suppress Next.js telemetry
          NEXT_TELEMETRY_DISABLED: 1
          # Provide dummy env vars that the build needs but aren't secret
          DATABASE_URL: "postgresql://dummy:dummy@localhost:5432/dummy"

      - name: Test
        run: pnpm test
        if: hashFiles('**/*.test.*', '**/*.spec.*') != ''
```

### Key Design Decisions

**`concurrency` with `cancel-in-progress`**: When an agent pushes a fix commit, the old CI run is cancelled immediately. Without this:
- You burn Actions minutes on stale runs
- The PR shows a "pending" check from the old run alongside the new run
- Agents get confused about which status to trust

**`timeout-minutes: 10`**: Hard cap. We've seen AI-generated code that:
- Created infinite loops in tests
- Imported a module that triggered a build-time fetch to a non-existent URL (30-minute DNS timeout)
- Generated a component that recursively rendered itself

**Dummy env vars for build**: Next.js needs `DATABASE_URL` at build time for Drizzle schema generation, even though no actual database connection happens. Provide dummy values to unblock the build.

**`--frozen-lockfile`**: Agents sometimes run `pnpm install` and accidentally modify the lockfile. This flag ensures CI fails if the lockfile doesn't match `package.json`, forcing agents to commit the correct lockfile.

## Vercel Deployment

### Auto-Deploy on Push to Main

Vercel deploys automatically when code hits `main`. Configuration:

```json
// vercel.json
{
  "framework": "nextjs",
  "buildCommand": "pnpm build",
  "installCommand": "pnpm install --frozen-lockfile",
  "outputDirectory": ".next"
}
```

### The Vercel Ignore Build Step Bug

Vercel has an "Ignored Build Step" feature that lets you skip builds for certain commits. The configuration uses a shell script:

```bash
# vercel-ignore-build-step.sh — THIS IS WRONG
if [ "$VERCEL_GIT_COMMIT_REF" = "main" ]; then
  echo "Building on main"
  exit 1  # 1 = proceed with build
fi
exit 0  # 0 = skip build
```

**The bug**: Exit code semantics are BACKWARDS from what you'd expect:
- `exit 1` = **proceed** with build (non-zero = "yes, build")
- `exit 0` = **skip** build (zero = "no, don't build")

We had this inverted for a week. All PRs were deploying preview builds. No pushes to `main` were deploying to production. The app was stuck on a week-old version while we kept merging PRs.

**Lesson**: Test your ignore build step by pushing a trivial change and checking the Vercel dashboard. Don't trust the logic — verify the behavior.

### Manual Deploy Trigger

For emergencies, add a manual deploy workflow:

```yaml
# .github/workflows/deploy.yml
name: Manual Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy environment'
        required: true
        default: 'production'
        type: choice
        options:
          - production
          - preview

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Vercel
        run: |
          npx vercel --token=${{ secrets.VERCEL_TOKEN }} \
            --prod=${{ github.event.inputs.environment == 'production' }}
```

## Build Pipeline Order

The order matters:

```
1. Install (pnpm install --frozen-lockfile)
2. Lint (catch style/import issues fast)
3. Type check (catch type errors without full build)
4. Build (full Next.js build — catches runtime config issues)
5. Test (only if test files exist)
```

**Why lint before build?** Lint is fast (5-10 seconds). If there's a missing import or unused variable, fail fast. Don't wait 2 minutes for the build to find it.

**Why type check separately?** `pnpm build` runs type checking as part of the Next.js build, but the error messages are mixed with build output. A separate `tsc --noEmit` gives clean, focused type errors.

## Common CI Failures from AI Agents

### 1. Missing Dependencies (pnpm strict mode)

```
error: Module not found: @hono/zod-validator
```

AI agents often use packages that are transitive dependencies (installed because something else depends on them) but not in `package.json`. This works with npm (hoisted `node_modules`) but fails with pnpm's strict isolation.

**Fix**: Always add direct dependencies to `package.json`.

### 2. Build-Only Errors

```
Error: useAuth must be used within an AuthProvider
```

This error doesn't appear at build time — Next.js happily compiles the component. It only appears at runtime. CI won't catch it unless you have integration tests.

**Fix**: Add a basic render test for components that use context providers.

### 3. Import Path Case Sensitivity

```
Module not found: Can't resolve './components/loginPage'
# File is actually: ./components/LoginPage.tsx
```

macOS is case-insensitive, Linux (CI) is case-sensitive. Agents develop in a case-insensitive sandbox but CI runs on Linux.

**Fix**: Add an ESLint rule or build check for case-sensitive imports.

### 4. Environment Variable Assumptions

```
TypeError: Cannot read properties of undefined (reading 'split')
# process.env.DATABASE_URL.split('/')
```

Agents assume env vars exist. Build step doesn't have them.

**Fix**: Use `env:` in the CI workflow to provide dummy values, or use `zod` to validate env vars with defaults.

## Post-Merge Verification

Even with CI gates, run a verification after every merge:

```yaml
# .github/workflows/post-merge.yml
name: Post-Merge Build Check

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

      - name: Create issue on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const { data: commit } = await github.rest.git.getCommit({
              owner: context.repo.owner,
              repo: context.repo.repo,
              commit_sha: context.sha,
            });
            
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🚨 Build broken on main — ${context.sha.substring(0, 7)}`,
              body: [
                `## Post-merge build failure`,
                ``,
                `**Commit**: ${context.sha}`,
                `**Author**: ${commit.author.name}`,
                `**Message**: ${commit.message.split('\n')[0]}`,
                `**Run**: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
                ``,
                `This commit broke the build on main. Immediate fix required.`,
              ].join('\n'),
              labels: ['bug', 'priority:critical'],
            });
```

This auto-creates a critical issue that the Coordinator will pick up on its next cycle.
