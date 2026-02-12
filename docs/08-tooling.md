# Tooling — Tools & Setup

> The specific tools we use and why. Not exhaustive — just what works.

## GitHub Apps — Bot Identity

### Why GitHub Apps Over Personal Access Tokens

| Feature | PAT | GitHub App |
|---------|-----|-----------|
| Identity | Shows as your account | Shows as `app-name[bot]` |
| Permissions | All or nothing (repo scope) | Granular per-permission |
| Token expiry | Never (or manual) | 1 hour (auto-rotate) |
| Per-repo install | No | Yes |
| Rate limits | 5,000/hr (shared with you) | 5,000/hr per installation |
| Audit trail | "you merged PR" | "builder-bot[bot] opened PR" |

### Two-App Setup

**Builder App** — the Builder agent's identity:
- Permissions: `contents: write`, `pull-requests: write`, `issues: write`
- Used for: creating branches, pushing code, opening PRs, commenting
- Cannot: approve PRs, manage settings, delete repos

**Reviewer App** — the Reviewer agent's identity:
- Permissions: `pull-requests: write`, `contents: read`
- Used for: reviewing PRs, approving or requesting changes
- Cannot: push code, merge, modify settings

**Why two apps?** Branch protection requires approval from a different account than the PR author. Builder opens as `builder[bot]`, Reviewer approves as `reviewer[bot]`. Requirement satisfied automatically.

### Token Generation

```javascript
// Generate a short-lived installation token
const { createAppAuth } = require('@octokit/auth-app');

const auth = createAppAuth({
  appId: process.env.GITHUB_APP_ID,
  privateKey: process.env.GITHUB_APP_PRIVATE_KEY,
  installationId: process.env.GITHUB_APP_INSTALLATION_ID,
});

const { token } = await auth({ type: 'installation' });
// Token valid for 1 hour
```

Usage:
```bash
export GH_TOKEN=$(node gh-app-token.js builder)
gh pr create --title "feat: add login page" --body "Closes #42"
```

### Token Limitations

GitHub App tokens can't do everything:
- ❌ Can't read branch protection rules (403)
- ❌ Can't override third-party commit statuses (e.g., Vercel)
- ❌ Can't access repo settings

Use a personal account (`gh` default auth) for these operations.

## Cron Scheduling

The Coordinator runs on a cron schedule (every 5 minutes). We use [OpenClaw](https://github.com/nicepkg/openclaw) for scheduling:

```
Interval: 5 minutes
Model: Claude Opus (strongest reasoning for coordination decisions)
Thinking: High
```

### Cron vs Webhook

| Feature | Cron | Webhook |
|---------|------|---------|
| Latency | 0-5 min | Instant |
| Infrastructure | None (built into orchestrator) | Endpoint + auth + retry logic |
| Reliability | Very high | Depends on delivery |
| Event storms | Naturally rate-limited | Can overwhelm |
| Global state | Yes (reads everything each cycle) | No (reacts to individual events) |

**Our choice:** Cron for coordination, GitHub Actions (event-driven) for CI.

## GitHub Labels Schema

The full label set for the pipeline FSM:

### Pipeline State Labels (mutually exclusive per issue)

| Label | Color | Description |
|-------|-------|-------------|
| `pipeline/discovered` | `#FBCA04` | Newly discovered, awaiting triage |
| `pipeline/approved` | `#0E8A16` | Triaged, ready for builder |
| `pipeline/rejected` | `#B60205` | Triaged, not worth doing |
| `pipeline/in-progress` | `#1D76DB` | Builder actively working |
| `pipeline/pr-open` | `#5319E7` | PR created, awaiting review |
| `pipeline/review-passed` | `#0E8A16` | Review approved, ready to merge |
| `pipeline/review-failed` | `#E99695` | Review requested changes |
| `pipeline/merged` | `#006B75` | Successfully merged |
| `pipeline/abandoned` | `#D93F0B` | Work abandoned |

### Priority Labels

| Label | Color | Description |
|-------|-------|-------------|
| `priority/P0` | `#B60205` | Critical — do immediately |
| `priority/P1` | `#D93F0B` | Important — do soon |
| `priority/P2` | `#FBCA04` | Nice-to-have — when idle |

### Type Labels

| Label | Color | Description |
|-------|-------|-------------|
| `type/bug` | `#D73A4A` | Bug fix |
| `type/enhancement` | `#A2EEEF` | Feature or improvement |
| `type/chore` | `#D4C5F9` | Maintenance, cleanup |
| `type/security` | `#B60205` | Security-related |
| `type/performance` | `#F9D0C4` | Performance improvement |

### Size Labels

| Label | Color | Description |
|-------|-------|-------------|
| `size/XS` | `#009800` | < 20 lines |
| `size/S` | `#77BB00` | 20-100 lines |
| `size/M` | `#FBCA04` | 100-300 lines |
| `size/L` | `#D93F0B` | 300-500 lines |

### Source Labels

| Label | Color | Description |
|-------|-------|-------------|
| `source/scout` | `#C5DEF5` | Created by Scout agent |
| `source/human` | `#BFD4F2` | Created by human |
| `source/sentinel` | `#D4C5F9` | Created by Sentinel |

### Meta Labels

| Label | Color | Description |
|-------|-------|-------------|
| `agent-authored` | `#EDEDED` | PR was written by an agent |

### Creating All Labels (Script)

```bash
#!/bin/bash
REPO="your-org/your-repo"

# Pipeline state
gh label create "pipeline/discovered" --repo $REPO --color "FBCA04" --force
gh label create "pipeline/approved" --repo $REPO --color "0E8A16" --force
gh label create "pipeline/rejected" --repo $REPO --color "B60205" --force
gh label create "pipeline/in-progress" --repo $REPO --color "1D76DB" --force
gh label create "pipeline/pr-open" --repo $REPO --color "5319E7" --force
gh label create "pipeline/review-passed" --repo $REPO --color "0E8A16" --force
gh label create "pipeline/review-failed" --repo $REPO --color "E99695" --force
gh label create "pipeline/merged" --repo $REPO --color "006B75" --force
gh label create "pipeline/abandoned" --repo $REPO --color "D93F0B" --force

# Priority
gh label create "priority/P0" --repo $REPO --color "B60205" --force
gh label create "priority/P1" --repo $REPO --color "D93F0B" --force
gh label create "priority/P2" --repo $REPO --color "FBCA04" --force

# Type
gh label create "type/bug" --repo $REPO --color "D73A4A" --force
gh label create "type/enhancement" --repo $REPO --color "A2EEEF" --force
gh label create "type/chore" --repo $REPO --color "D4C5F9" --force
gh label create "type/security" --repo $REPO --color "B60205" --force
gh label create "type/performance" --repo $REPO --color "F9D0C4" --force

# Size
gh label create "size/XS" --repo $REPO --color "009800" --force
gh label create "size/S" --repo $REPO --color "77BB00" --force
gh label create "size/M" --repo $REPO --color "FBCA04" --force
gh label create "size/L" --repo $REPO --color "D93F0B" --force

# Source
gh label create "source/scout" --repo $REPO --color "C5DEF5" --force
gh label create "source/human" --repo $REPO --color "BFD4F2" --force
gh label create "source/sentinel" --repo $REPO --color "D4C5F9" --force

# Meta
gh label create "agent-authored" --repo $REPO --color "EDEDED" --force

echo "All labels created!"
```

## Branch Protection (API)

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

## Tool Stack Summary

```
Code         → AI Agents (Scout, Triage, Builder, Reviewer, Merger, Sentinel)
Coordination → Reconciliation loop (cron, 5 min)
State        → GitHub (issues + labels + PRs)
CI           → GitHub Actions (single required check: build)
Deploy       → Vercel (auto-deploy on main push)
Identity     → GitHub Apps (builder + reviewer)
Merge        → Squash only, strict: false, batch merge
Monitoring   → Sentinel (post-merge health check)
Learning     → pipeline-learning-log.md (rejection patterns)
Metrics      → Pinned GitHub issue (updated each cycle)
```
