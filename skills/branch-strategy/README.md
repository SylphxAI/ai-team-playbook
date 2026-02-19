# Branch Strategy

> Two-tier branch model with protection rules and squash-only merges for clean, reviewable history in high-velocity agent teams.

## Problem

Without a clear branching model, agents push directly to main, merge unreviewed code, create tangled commit histories, and step on each other's work. High-velocity teams (human or AI) need guardrails that enforce quality without throttling throughput.

## Solution

Three-tier model: `main` is sacred, `dev` is integration, feature branches are ephemeral. Branch protection enforces the rules. Squash merge keeps history clean.

```
main   = production, never touch directly
dev    = development, all PRs target here
PR     = feature branches, always branched from dev
```

## Implementation

### Branch Protection Config

Apply this config to **both** `main` and `dev` branches:

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
      "include": ["refs/heads/main", "refs/heads/dev"],
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
git checkout dev
git pull origin dev
git checkout -b feature/my-change

# Work, commit, push
git commit -m "descriptive message"
git push origin feature/my-change

# Open PR targeting dev (never main)
gh pr create --base dev --title "..." --body "..."

# After approval + CI passes, Gatekeeper squash merges
gh pr merge --squash
```

### Promoting dev → main

Periodically (not on every PR), the Gatekeeper opens a PR from `dev` to `main`:

```bash
gh pr create --base main --head dev --title "chore: promote dev to main" --body "Promoting tested changes to production"
```

This requires the same approval + CI check as any other PR.

## Examples

**Good commit history on `main` after squash merges:**
```
abc1234 feat: add user export to CSV
def5678 fix: handle null profile image in share flow
ghi9012 chore: promote dev to main
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

- **`strict: false` is correct.** With `strict: true` (require up-to-date), each PR must be rebased on latest dev before merging. In a team of 5 Builders, this serializes merges to ~20 min/PR throughput. No major open-source project uses strict mode. Post-merge CI catches real conflicts.

- **Squash merge is non-negotiable for agent teams.** Agents make exploratory commits: "attempt 1", "try different approach", "revert to see if this helps". None of that belongs in `dev` history. Squash gives one clean commit per PR.

- **The CI check name must match exactly.** Branch protection requires a check named `build`. If your GitHub Actions job is named `Build` or `test`, the branch rule will never be satisfied and all PRs will be permanently blocked. Check with `gh pr checks`.

- **Auto-delete head branches prevents sprawl.** With 5 Builders each opening 2-3 PRs/day, you'd accumulate hundreds of stale branches in a week. Auto-delete keeps the repo clean.

- **Never let agents push to `dev` directly.** Even one direct push bypasses review, pollutes history, and sets a precedent. The protection rule must be on — disable only temporarily and with intention.
