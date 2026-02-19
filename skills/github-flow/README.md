# GitHub Flow — PR-First Development

> Open source style: no shared DEV environment. Every PR gets an ephemeral preview URL. Merge to `main` deploys to staging. Manual promote to production.

## Mental Model

This workflow is designed for teams where contributors (human or AI) have no shared local environment — exactly like a major open source project. GitHub, Vercel, and Linear all work this way.

```
feature branch
     │
     ▼
PR opened ──────► Ephemeral Preview
                  • Unique URL per PR (e.g. feat-login-abc.vercel.app)
                  • Own isolated DB (created on PR open, deleted on close)
                  • For: developer review, Gatekeeper testing
     │
     ▼ approved + CI pass → squash merge
     │
   main ──────────► Staging (auto-deploy)
                    • Permanent URL (staging.domain.com)
                    • Persistent staging DB
                    • The "nightly build" — accumulated latest work
                    • For: client preview, integration testing, release decisions
     │
     ▼ "ready to release" → one-click promote
     │
Production ─────────► domain.com
                       • Production DB (Neon main branch)
                       • Stable. Only changes on explicit release.
```

## Three Environments Explained

| | Ephemeral Preview | Staging | Production |
|--|---|---|---|
| **Trigger** | PR opened | Merge to `main` | Manual promote |
| **URL** | Temporary, unique per PR | `staging.domain.com` | `domain.com` |
| **Lifespan** | Lives as long as PR is open | Always running | Always running |
| **Database** | Isolated per-PR Neon branch | Neon `staging` branch | Neon `main` branch |
| **Purpose** | Test this PR in isolation | Integration, client review | Live users |

**Note: DEV has no URL and does not exist as an infrastructure environment.** In open source projects, DEV is only ever local on a contributor's machine. Since AI agents have no local machines, DEV simply doesn't exist.

## Database Strategy

### Ephemeral Preview: Per-PR Neon Branch

Each PR gets its own isolated Neon database branch. Schema is copied from `main` (no production data). Destroyed when PR closes.

```yaml
# .github/workflows/preview-db.yml
on:
  pull_request:
    types: [opened, reopened, synchronize, closed]

jobs:
  create-preview-db:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: neondatabase/create-branch-action@v5
        id: neon
        with:
          project_id: ${{ secrets.NEON_PROJECT_ID }}
          branch_name: preview/pr-${{ github.event.number }}
          parent: main
          api_key: ${{ secrets.NEON_API_KEY }}
      - name: Set DATABASE_URL for this PR's Vercel preview
        run: |
          vercel env add DATABASE_URL preview \
            --value "${{ steps.neon.outputs.db_url_with_pooler }}" \
            --git-branch "${{ github.head_ref }}" \
            --token ${{ secrets.VERCEL_TOKEN }}

  teardown-preview-db:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: neondatabase/delete-branch-action@v3
        with:
          project_id: ${{ secrets.NEON_PROJECT_ID }}
          branch: preview/pr-${{ github.event.number }}
          api_key: ${{ secrets.NEON_API_KEY }}
```

### Staging: Persistent Neon Branch

A dedicated `staging` Neon branch. Persistent. Refresh from production snapshot periodically (e.g. weekly) to prevent drift.

### Production: Neon `main` Branch

Never touched by previews or staging. Only updated via Atlas migrations on production deploys.

## Vercel Setup

Two Vercel projects connected to the same repo:

| Project | Branch | Domain | Auto-deploy? |
|---------|--------|--------|--------------|
| `myapp-staging` | `main` | `staging.domain.com` | ✅ Yes |
| `myapp` (production) | manual promote | `domain.com` | ❌ Manual only |

To promote: in Vercel dashboard, click "Promote to Production" on any staging deployment. Or via API:

```bash
vercel promote <deployment-url> --token $VERCEL_TOKEN
```

## Branch Strategy

Only `main` exists as a permanent branch. No `dev`, no `staging` branch.

```
main          ← permanent, staging deploys from here
feature/...   ← short-lived, one per PR, deleted after merge
```

Branch protection on `main`:
- Require PR (no direct push)
- 1 required approval (Gatekeeper)
- Required CI check: `build`
- Squash merge only
- Auto-delete head branches

## Agent Workflow

Every agent always branches from `main`:

```bash
git checkout main && git pull
git checkout -b feat/issue-123-description
# implement...
gh pr create --base main --title "feat: ..." --body "Closes #123"
```

The Gatekeeper reviews, approves, squash merges. Staging auto-updates. Product owner promotes to production when ready.

## When to Promote to Production

Production releases are intentional decisions, not automatic. Promote when:
- Feature set is complete enough for users
- Client has reviewed and approved on staging (Epiow)
- No open P0/P1 issues
- Staging has been stable for a reasonable period

This gives the product owner full control over what users see.

## Gotchas

**Staging DB can drift from production.** If production data evolves (new rows, new patterns), staging schema stays in sync but data differs. This is expected and fine — staging is for schema + feature testing, not data accuracy.

**Per-PR DB shares the `main` schema.** All PR previews copy from Neon `main`. If production has a migration that staging doesn't, previews may be out of sync. Run migrations on staging first; previews inherit from `main` by default.

**Don't send external clients to PR preview URLs.** Ephemeral URLs are for internal review (Gatekeeper, team). Clients always review on `staging.domain.com` — it's stable, professional, and doesn't expire.

**Atlas migrations run in CI before deploy.** Migrations apply to staging on every merge to `main`. Per-PR previews get a fresh schema copy — no migration history needed.
