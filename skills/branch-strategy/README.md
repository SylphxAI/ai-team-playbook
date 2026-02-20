# Branch Strategy

> Three-tier branch model with protection rules and squash-only merges for clean, reviewable history in high-velocity agent teams.

## Problem

Without a clear branching model, agents push directly to main, merge unreviewed code, create tangled commit histories, and step on each other's work. High-velocity teams (human or AI) need guardrails that enforce quality without throttling throughput.

## Solution

Three-tier model: `main` is production (sacred), `staging` is the integration point, feature branches are ephemeral. Branch protection enforces the rules. Squash merge keeps history clean.

```
main     = production, never touch directly
staging  = integration point, all PRs target here
feature/ = ephemeral, one per PR, deleted after merge
```

## Implementation

### Branch Protection Config

Apply this config to **both** `main` and `staging` branches:

| Setting | Value | Why |
|---------|-------|-----|
| Require PR | ✅ | No direct pushes |
| Required approvals | 1 | Every change gets a second pair of eyes |
| Required CI check | `build` | Must pass before merge |
| Require branches to be up-to-date | ❌ `strict: false` | Kills throughput — see below |
| Squash merge only | ✅ | Clean, atomic history |
| Auto-delete head branches | ✅ | No branch graveyard |

### GitHub Ruleset (via API or `merge-queue-ruleset.json`)

```json
{
  "name": "branch-protection",
  "target": "branch",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main", "refs/heads/staging"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "pull_request", "parameters": { "required_approving_review_count": 1 } },
    { "type": "required_status_checks", "parameters": { "required_status_checks": [{ "context": "build" }], "strict_required_status_checks_policy": false } },
    { "type": "squash_merge" }
  ]
}
```

### Workflow

```bash
# Start new work
git checkout staging
git pull origin staging
git checkout -b feature/my-change

# Work, commit, push
git commit -m "descriptive message"
git push origin feature/my-change

# Open PR targeting staging (never main)
gh pr create --base staging --title "..." --body "..."

# After approval + CI passes, Gatekeeper squash merges
gh pr merge --squash
```

### Promoting staging → main

When staging has accumulated enough tested changes, the Gatekeeper opens a release PR from `staging` to `main`:

```bash
gh pr create --base main --head staging --title "release: $(date +%Y-%m-%d)" --body "Promoting staging to production"
```

This requires the same approval + CI check as any other PR.

## Examples

**Good commit history on `main` after squash merges:**
```
abc1234 feat: add user export to CSV
def5678 fix: handle null profile image in share flow
ghi9012 release: 2025-06-15
```

**Bad commit history without squash:**
```
abc1234 fix lint
def5678 try again
ghi9012 actually fix it this time
jkl0123 forgot to run tests
mno4567 feat: add user export to CSV
```

## Lessons & Gotchas

- **`strict: false` is correct.** With `strict: true` (require up-to-date), each PR must be rebased on latest staging before merging. In a team of 5 Builders, this serializes merges to ~20 min/PR throughput. No major open-source project uses strict mode. Post-merge CI catches real conflicts.

- **Squash merge is non-negotiable for agent teams.** Agents make exploratory commits: "attempt 1", "try different approach", "revert to see if this helps". None of that belongs in `staging` history. Squash gives one clean commit per PR.

- **The CI check name must match exactly.** Branch protection requires a check named `build`. If your GitHub Actions job is named `Build` or `test`, the branch rule will never be satisfied and all PRs will be permanently blocked. Check with `gh pr checks`.

- **Auto-delete head branches prevents sprawl.** With 5 Builders each opening 2-3 PRs/day, you'd accumulate hundreds of stale branches in a week. Auto-delete keeps the repo clean.

- **Never let agents push to `staging` directly.** Even one direct push bypasses review, pollutes history, and sets a precedent. The protection rule must be on — disable only temporarily and with intention.
