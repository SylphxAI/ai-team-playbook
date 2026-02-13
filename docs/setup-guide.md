# Setup Guide — How to Copy This Workflow

Step-by-step guide to set up an autonomous AI agent team on any project.

## Prerequisites

- GitHub repo (public or private)
- [OpenClaw](https://github.com/nicepkg/openclaw) instance running
- GitHub org (for GitHub App — or use fine-grained PAT as alternative)

## 1. GitHub App Setup (Bot Identity)

Agents need a separate identity from your personal account. Two options:

### Option A: GitHub App (recommended)

No seat cost. Commits show as `app-name[bot]`. Scoped permissions. Auto-expiring tokens (1h).

**Create two apps** (builder + reviewer) — branch protection requires approval from a different account than the PR author.

1. Go to **GitHub Org Settings → Developer Settings → GitHub Apps → New GitHub App**
2. App name: `yourproject-builder` (and `yourproject-reviewer`)
3. Homepage URL: your repo URL
4. Uncheck "Webhook → Active" (not needed)
5. Permissions:

| Permission | Builder | Reviewer |
|------------|---------|----------|
| Contents | Write | Read |
| Pull requests | Write | Write |
| Issues | Write | Read |
| Checks | Write | Read |
| Commit statuses | Write | Read |
| Metadata | Read | Read |

6. "Where can this app be installed?" → Only on this account
7. **Generate a private key** → save as `.pem` file
8. Note the **App ID** from the app settings page
9. **Install** the app on your target repos
10. Note the **Installation ID** from the URL after installing (`/installations/XXXXX`)

**Generate tokens:**
```bash
npx github-app-installation-token \
  --appId APP_ID \
  --installationId INSTALL_ID \
  --key /path/to/private-key.pem
```

Or write a script:
```javascript
const { createAppAuth } = require('@octokit/auth-app');
const auth = createAppAuth({
  appId: process.env.GITHUB_APP_ID,
  privateKey: fs.readFileSync('/path/to/key.pem', 'utf8'),
  installationId: process.env.GITHUB_APP_INSTALLATION_ID,
});
const { token } = await auth({ type: 'installation' });
// Token valid for 1 hour
```

Usage in agent prompts:
```bash
export GH_TOKEN=$(node /path/to/gh-app-token.js builder)
gh auth setup-git
```

### Option B: Fine-Grained PAT (simpler, trade-offs)

- Commits show as your personal account
- Costs a seat if you create a separate machine user
- Simpler setup — just generate a token in GitHub settings

Go to **Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens**. Grant repo permissions equivalent to the table above.

### App Limitations

GitHub App tokens **cannot**:
- Read branch protection rules (403)
- Override third-party commit statuses (e.g., Vercel)
- Access repo settings

Use a personal account for these operations when needed.

## 2. Branch Protection

Configure on your `main` branch:

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":false,"contexts":["build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true}' \
  --field restrictions=null \
  --field required_linear_history=true \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

Repo-level merge settings:
- ✅ Allow squash merge (only strategy enabled)
- ❌ Allow merge commits
- ❌ Allow rebase merge
- ✅ Auto-delete head branches
- Squash commit title: PR title
- Squash commit message: PR body

**Why `strict: false`?** `strict: true` serializes merging (~20 min/PR). No major OSS project uses it at scale. Use post-merge CI monitoring instead.

**Why squash only?** AI agents make messy commits ("fix lint", "try again"). Squash collapses into one clean commit per PR.

## 3. CI Setup

Minimal GitHub Actions workflow. Single required check: `build`.

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
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run lint
        continue-on-error: true    # lint = non-blocking
      - run: bun run build         # this is the required check
      - run: bun test
        if: hashFiles('**/*.test.*', '**/*.spec.*') != ''
```

Key decisions:
- **`continue-on-error: true` for lint** — catches style issues without blocking merges
- **`--frozen-lockfile`** — prevents agents from accidentally modifying lockfile
- **`cancel-in-progress`** — kills stale CI runs when agents push fixes
- **`timeout-minutes: 10`** — prevents infinite loops and runaway builds

Optional post-merge verification (auto-creates P0 issue on failure):
```yaml
# .github/workflows/post-merge.yml
name: Post-Merge
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
              body: `Commit: ${context.sha}\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['bug', 'P0'],
            });
```

## 4. OpenClaw Cron Setup

The coordinator runs as a cron job in OpenClaw, every 5 minutes.

### Create the coordinator prompt

Write a coordinator prompt that includes:
1. The 4-step algorithm (inventory → spawn deficit → CI check → summary)
2. Roster table (direction, key, desired count)
3. Project context block (prepended to every agent task)
4. Role briefs for each direction

See the [Agent Roles](agent-roles.md) for brief templates.

### Create the cron job

```bash
openclaw cron create \
  --name "v4-coordinator" \
  --interval "*/5 * * * *" \
  --model "claude-opus-4-6" \
  --thinking high \
  --task "$(cat coordinator-prompt.md)"
```

Or configure via `openclaw.json`:
```json
{
  "cron": [
    {
      "name": "v4-coordinator",
      "schedule": "*/5 * * * *",
      "model": "claude-opus-4-6",
      "thinking": "high",
      "taskFile": "coordinator-prompt.md"
    }
  ]
}
```

### Coordinator prompt structure

```markdown
# V4 Coordinator

Maintain a fleet of 8 agents across 6 directions. Every 5 min. Stateless, idempotent.

## Algorithm
1. INVENTORY — sessions_list, count active v4-* agents
2. SPAWN DEFICIT — if running < desired, spawn the difference
3. CI CHECK — gh run list ... if not success → spawn v4-cifix
4. SUMMARY — print status

## Project Context (prepend to every agent task)
> **Project**: YourOrg/your-repo — description
> **Competitors**: ...
> **Repo**: YourOrg/your-repo. One focused PR per session.
> **Git identity**: export GH_TOKEN=$(node /path/to/gh-app-token.js builder)

## Roster
| Direction | Key | Count |
|-----------|-----|-------|
| Product | v4-product | 1 |
| Audit | v4-audit | 1 |
| Triage | v4-triage | 1 |
| Build | v4-builder | 3 |
| Test | v4-tester | 1 |
| Review | v4-reviewer | 1 |

## Role Briefs
(paste keyword lists from agent-roles.md)
```

## 5. Labels (Optional)

Minimal label set for the workflow:

```bash
REPO="your-org/your-repo"
# Priority
gh label create "P0" --repo $REPO --color "B60205" --force
gh label create "P1" --repo $REPO --color "D93F0B" --force
gh label create "P2" --repo $REPO --color "FBCA04" --force
# Workflow
gh label create "approved" --repo $REPO --color "0E8A16" --force
gh label create "rejected" --repo $REPO --color "B60205" --force
gh label create "in-progress" --repo $REPO --color "1D76DB" --force
gh label create "fixing" --repo $REPO --color "D93F0B" --force
# Size
gh label create "XS" --repo $REPO --color "009800" --force
gh label create "S" --repo $REPO --color "77BB00" --force
gh label create "M" --repo $REPO --color "FBCA04" --force
gh label create "L" --repo $REPO --color "D93F0B" --force
```

## 6. Checklist

Before going live:

- [ ] GitHub App(s) created and installed on repo
- [ ] Token generation script working
- [ ] Branch protection configured (`strict: false`, `build` required, 1 review)
- [ ] Squash merge only, auto-delete branches
- [ ] CI workflow committed (`.github/workflows/ci.yml`)
- [ ] Labels created
- [ ] Coordinator prompt written with project context + role briefs
- [ ] Cron job created in OpenClaw
- [ ] Test: manually trigger coordinator, verify it spawns agents
- [ ] Test: verify builder can push, reviewer can approve+merge

## Scaling

| Symptom | Fix |
|---------|-----|
| PR queue growing | Add Review agents |
| Approved issues piling up | Add Builders |
| PRs waiting for tests | Add Testers |
| Low-quality issues | Triage is working — check Audit/Product prompts |

Start with the default roster (8 agents). Adjust after observing bottlenecks.
