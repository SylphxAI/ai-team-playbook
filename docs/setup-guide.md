# Setup Guide — How to Copy This Workflow

Step-by-step guide to set up an autonomous AI agent team for one or more projects.

## Prerequisites

- One or more GitHub repos (public or private)
- [OpenClaw](https://github.com/nicepkg/openclaw) instance running
- Two GitHub user accounts (one for Builders/Scout/Tester, one for Gatekeeper)

## 1. Git Identity Setup (Real User Accounts)

Agents use real GitHub user accounts — not GitHub Apps or bots.

| Role | GitHub Account | Identity |
|------|---------------|----------|
| Scout | shtse8 | Kyle Tse / shtse8@gmail.com |
| Builder | shtse8 | Kyle Tse / shtse8@gmail.com |
| Tester | shtse8 | Kyle Tse / shtse8@gmail.com |
| Gatekeeper | claw-sylphx | Clayton Shaw / clayton@sylphx.com |

**Why two accounts?** Branch protection requires the reviewer to be a different account than the PR author. Scout, Builder, and Tester open PRs as `shtse8`; Gatekeeper reviews and merges as `claw-sylphx`.

**Why real accounts instead of GitHub Apps?**
- Simpler setup — no App ID, no private key, no token generation script
- Commits show on Kyle's GitHub profile (green squares)
- No GitHub App token complexity or 1-hour expiry headaches
- Trade-off: shared identity across roles (acceptable — roles don't conflict)

### Setup

Both accounts must be authenticated in `gh` CLI:

```bash
# Authenticate both accounts (one-time setup)
gh auth login --hostname github.com  # log in as shtse8
gh auth login --hostname github.com  # log in as claw-sylphx

# List authenticated accounts
gh auth status

# Switch between accounts
gh auth switch --user shtse8
gh auth switch --user claw-sylphx
```

At agent spawn time, set the correct identity before any git operations:

**For Scout, Builder, Tester:**
```bash
gh auth switch --user shtse8
git config --global user.name "Kyle Tse"
git config --global user.email "shtse8@gmail.com"
```

**For Gatekeeper:**
```bash
gh auth switch --user claw-sylphx
git config --global user.name "Clayton Shaw"
git config --global user.email "clayton@sylphx.com"
```

## 2. Branch Protection

Configure on your `dev` branch (and optionally `main`):

```bash
gh api repos/{owner}/{repo}/branches/dev/protection \
  --method PUT \
  --field required_status_checks='{"strict":false,"contexts":["build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true}' \
  --field restrictions=null \
  --field required_linear_history=true \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

**Apply this to every repo** in your multi-project setup. All PRs target `dev`, never `main`.

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
    branches: [dev]
  push:
    branches: [dev, main]

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

**Apply this to every repo** — adapt the build steps per project but keep the structure consistent.

Key decisions:
- **`continue-on-error: true` for lint** — catches style issues without blocking merges
- **`--frozen-lockfile`** — prevents agents from accidentally modifying lockfile
- **`cancel-in-progress`** — kills stale CI runs when agents push fixes
- **`timeout-minutes: 10`** — prevents infinite loops and runaway builds

## 4. Repo Context (CLAUDE.md)

Each repo should have a `CLAUDE.md` at its root. This is how agents learn project-specific context:

```markdown
# Project Name

Brief description of what this project does.

## Stack
- Framework, language, database, etc.

## Conventions
- Code style, naming, architecture patterns

## Development
- How to run locally, test, build

## Key Files
- Important paths agents should know about
```

When the coordinator spawns an agent for a specific repo, the agent reads that repo's `CLAUDE.md` first.

## 5. Unified Coordinator Setup

The coordinator runs as **one cron job** managing **all repos** (see [Multi-Project](multi-project.md) for rationale).

### Write the coordinator prompt

Create a prompt file that includes:
1. The list of all repos to manage
2. The 6-step algorithm (inventory → check repos → prioritize → spawn deficit → CI check → summary)
3. Roster table (role, key, desired count)
4. Spawn templates for each role

Key principles:
- Agents are **not assigned to repos** — they pick work by priority across all repos
- Priority: failing CI > broken PRs > PRs needing review > approved issues > untriaged issues
- Coordinator does NOT do work itself — only spawns and monitors

### Register the cron job

```bash
openclaw cron create \
  --name "unified-coordinator" \
  --interval "*/5 * * * *" \
  --model "claude-opus-4-6" \
  --thinking high \
  --task "$(cat unified-coordinator.md)"
```

Or configure via `openclaw.json`:
```json
{
  "cron": [
    {
      "name": "unified-coordinator",
      "schedule": "*/5 * * * *",
      "model": "claude-opus-4-6",
      "thinking": "high",
      "taskFile": "prompts/unified-coordinator.md"
    }
  ]
}
```

## 6. Labels

Apply to every repo:

```bash
for REPO in "org/repo-a" "org/repo-b" "org/repo-c"; do
  # Pipeline
  gh label create "pipeline/approved" --repo $REPO --color "0E8A16" --force
  gh label create "pipeline/rejected" --repo $REPO --color "B60205" --force
  # Priority
  gh label create "priority/P0" --repo $REPO --color "B60205" --force
  gh label create "priority/P1" --repo $REPO --color "D93F0B" --force
  gh label create "priority/P2" --repo $REPO --color "FBCA04" --force
  gh label create "priority/P3" --repo $REPO --color "C5DEF5" --force
  # Source
  gh label create "source/scout" --repo $REPO --color "7057FF" --force
  gh label create "source/gatekeeper" --repo $REPO --color "E4E669" --force
  # Type
  gh label create "type/architecture" --repo $REPO --color "0052CC" --force
  gh label create "type/optimization" --repo $REPO --color "0075CA" --force
  gh label create "type/ux" --repo $REPO --color "CC317C" --force
  gh label create "type/security" --repo $REPO --color "B60205" --force
  gh label create "type/growth" --repo $REPO --color "0E8A16" --force
  gh label create "type/quality" --repo $REPO --color "E4E669" --force
  gh label create "type/infra" --repo $REPO --color "1D76DB" --force
  gh label create "type/bug" --repo $REPO --color "D73A4A" --force
  gh label create "type/enhancement" --repo $REPO --color "A2EEEF" --force
  # Size
  gh label create "size/XS" --repo $REPO --color "F9D0C4" --force
  gh label create "size/S" --repo $REPO --color "F9D0C4" --force
  gh label create "size/M" --repo $REPO --color "F9D0C4" --force
  gh label create "size/L" --repo $REPO --color "F9D0C4" --force
  # Workflow
  gh label create "in-progress" --repo $REPO --color "1D76DB" --force
  gh label create "fixing" --repo $REPO --color "D93F0B" --force
done
```

## 7. Checklist

Before going live:

- [ ] Both GitHub accounts (`shtse8`, `claw-sylphx`) authenticated in `gh` CLI
- [ ] Account switching tested: `gh auth switch --user shtse8` and `gh auth switch --user claw-sylphx`
- [ ] Branch protection configured on `dev` branch (all repos)
- [ ] Squash merge only, auto-delete branches (all repos)
- [ ] CI workflow committed to all repos
- [ ] `CLAUDE.md` written for each repo
- [ ] Labels created on all repos
- [ ] Coordinator prompt written with all repos listed
- [ ] Cron job created in OpenClaw (start disabled, enable after testing)
- [ ] Test: manually trigger coordinator, verify it spawns agents
- [ ] Test: verify Builder can push to all repos as shtse8
- [ ] Test: verify Gatekeeper can approve+merge on all repos as claw-sylphx

## Scaling

| Symptom | Fix |
|---------|-----|
| PR queue growing | Gatekeeper is bottleneck — check for stuck PRs |
| Approved issues piling up | Add Builders |
| PRs waiting for tests | Add Testers |
| Low-quality issues | Gatekeeper is working — check Scout prompt |
| One repo starving | Check priority balance, consider priority boost |

Start with the default roster (8 agents). Adjust after observing bottlenecks.
