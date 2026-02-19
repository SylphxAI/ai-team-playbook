# GitHub Flow — PR-First Development

> No integration branch. Every change ships through a PR with its own isolated preview deployment. Merge to `main` = production.

## Problem

Traditional GitFlow uses a `dev` branch as an integration layer:

```
feature → PR → dev → staging → main (production)
```

This made sense when developers needed a shared environment to test combined changes. But for AI agent teams where:
- No one does local development
- Each agent works independently on separate PRs
- Vercel creates a preview deployment for every PR automatically

...the `dev` branch is pure overhead. It adds complexity, requires periodic dev→main promotions, and gives agents an extra thing to reason about (which branch to target).

## Solution

GitHub Flow — the same model used by most major open source projects:

```
feature branch → PR (preview deployment = staging) → main (production)
```

Each PR gets its own Vercel preview URL. That preview IS the staging environment for that change. Test it, approve it, merge it. No shared integration branch needed.

## How It Works

```
┌─────────────────────────────────────────────────────────┐
│  Agent creates feature branch from main                  │
│  Agent opens PR                                          │
│          ↓                                               │
│  Vercel deploys PR preview (isolated staging)            │
│  CI runs (lint, build, tests)                            │
│          ↓                                               │
│  Gatekeeper reviews + approves                           │
│          ↓                                               │
│  Squash merge → main → production deploy                 │
└─────────────────────────────────────────────────────────┘
```

### What Each Layer Does

| Layer | Git | URL | Database |
|-------|-----|-----|----------|
| Production | `main` | your domain | Neon `main` branch |
| Staging (per PR) | feature branch | `*.vercel.app` preview | Neon `dev` branch |

Note: The Neon `dev` database branch is still useful — all PR previews share it. Only the git `dev` branch is removed. Database layer still has dev/prod separation.

## Implementation

### Branch Protection (main only)

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":false,"contexts":["build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null \
  --field required_linear_history=true \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

No branch protection needed on `dev` because `dev` no longer exists as a permanent branch.

### Repo Merge Settings

- ✅ Squash merge only
- ✅ Auto-delete head branches
- ❌ No merge commits
- ❌ No rebase merge

### Agent Workflow

Builders always branch from `main`:

```bash
git checkout main && git pull
git checkout -b feat/my-feature
# ... make changes ...
gh pr create --base main --title "..." --body "..."
```

All PRs target `main`. Never create a `dev` branch.

### Vercel

No changes needed. Vercel already:
- Deploys `main` to production
- Deploys every PR branch to a preview URL
- Uses Preview env vars (pointing to Neon `dev`) for all previews

### Coordinator Update

Update the coordinator prompt:
- PRs target `main` (not `dev`)
- No dev→main promotion step
- Branch protection check is on `main`

## Why This Works for AI Agent Teams

| Factor | Traditional GitFlow | GitHub Flow (AI team) |
|--------|--------------------|-----------------------|
| Local dev needed? | Yes | No — agents don't run local environments |
| Integration testing | Shared `dev` branch | CI + isolated PR previews |
| Merge complexity | Two-step (PR → dev → main) | One-step (PR → main) |
| Agent mental model | Must know to target `dev` | Always target `main` |
| Conflict surface | Agents pile up in `dev` | Each PR is isolated |
| Open source parity | Diverges | Matches — anyone can contribute a PR |

The key insight: AI agents are like open source contributors. No one gives an OSS contributor a shared dev environment — they fork, branch, PR. Same model works perfectly here.

## Lessons & Gotchas

**"What about testing multiple PRs together before production?"**
With isolated PR previews, you test each change independently. This is actually safer — you know exactly what each merge introduces. Combine with solid CI (build + tests) and post-merge monitoring.

**"The Neon `dev` branch is shared across all PR previews"**
All simultaneous PR previews share the same `dev` database. If two PRs run schema migrations, they could conflict. Mitigation: Atlas migrations are declarative (not sequential), reducing conflict risk. Monitor preview DB health.

**"What if main breaks?"**
With direct-to-main merges, main breaking = production breaking. Mitigations:
1. Required CI (`build` check) must pass before merge
2. Gatekeeper must approve — no auto-merge
3. Post-merge monitoring on main CI
4. Fast rollback: revert the squash commit

**Don't confuse git `dev` branch with Neon `dev` branch:**
- Git `dev` branch → removed (no longer needed)
- Neon `dev` database branch → keep (PR previews need a non-production DB)
