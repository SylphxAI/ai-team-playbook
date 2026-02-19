# GitHub Flow — PR-First Development

> No integration branch. Every change ships through a PR with its own isolated preview. Merge to `main` auto-deploys to staging. Manual promote to production.

## Problem

Traditional GitFlow uses a `dev` branch as a shared integration layer — designed for teams with local development environments. For AI agent teams where:
- No one does local development
- Each agent works independently, submitting PRs like open source contributors
- Vercel creates a preview deployment per PR automatically

...the `dev` branch is pure overhead. But simply merging PRs straight to production is also reckless. The solution is a deployment pipeline that matches how the work actually happens.

## The Flow

```
feature branch
     │
     ▼
  PR open ──────► PR Preview deployment
                   └─ per-PR ephemeral DB (schema only, no prod data)
     │
     ▼ merge to main
     │
  main ──────────► Staging (auto-deploy)
                   └─ staging DB (shared, integration point)
     │
     ▼ manual promote
     │
  Production ─────► Live
                   └─ Neon main branch (prod DB)
```

## Three Environments

| Environment | Trigger | URL | Database |
|-------------|---------|-----|----------|
| PR Preview | PR opened | `*.vercel.app/pr-N` | Ephemeral Neon branch (per PR, schema only) |
| Staging | Push to `main` | `staging.domain.com` | Neon staging branch |
| Production | Manual promote | `domain.com` | Neon `main` branch |

## Database Strategy — Option C: Per-PR Ephemeral DB

**Option A (skip DB in preview):** Simplest, but tests almost nothing — you only verify the app starts.

**Option B (shared staging DB):** All previews share one Postgres. Multiple concurrent PRs can conflict on migrations — one PR's schema change breaks another PR's preview.

**Option C (per-PR ephemeral DB) ← recommended:**
- PR opens → create Neon branch from `main` (schema only, no prod data)
- PR preview uses this isolated branch
- PR closes → delete the branch
- Zero cross-PR contamination
- Neon branches are copy-on-write — near-zero storage cost until written

### Implementation (GitHub Actions)

```yaml
# .github/workflows/preview-db.yml
name: Preview DB

on:
  pull_request:
    types: [opened, reopened, closed]

jobs:
  setup-preview-db:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Create Neon branch
        uses: neondatabase/create-branch-action@v5
        id: neon
        with:
          project_id: ${{ secrets.NEON_PROJECT_ID }}
          branch_name: preview/pr-${{ github.event.number }}
          api_key: ${{ secrets.NEON_API_KEY }}

      - name: Set Preview DB URL on Vercel
        run: |
          vercel env add DATABASE_URL preview \
            --value "${{ steps.neon.outputs.db_url_with_pooler }}" \
            --git-branch "pr-${{ github.event.number }}" \
            --token ${{ secrets.VERCEL_TOKEN }}

  teardown-preview-db:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Delete Neon branch
        uses: neondatabase/delete-branch-action@v3
        with:
          project_id: ${{ secrets.NEON_PROJECT_ID }}
          branch: preview/pr-${{ github.event.number }}
          api_key: ${{ secrets.NEON_API_KEY }}
```

## Staging Environment

Keep a lightweight staging environment — **not a git branch**, a Vercel deployment target.

Why staging (even with PR previews):
- **External client review** (e.g. Epiow): clients review new features before production. You can't send clients to a `*.vercel.app` preview URL — it looks unprofessional and previews may expire.
- **Integration point**: after multiple PRs merge, staging shows the combined state. PR previews only show one PR in isolation.
- **Final sanity check**: catch issues that only appear when features interact.

### Vercel Implementation

Two options:

**Option 1: Separate Vercel project for staging**
- `staging-viral` project auto-deploys from `main`
- `viral` (production) deploys only on manual promote
- Clean separation, separate env vars

**Option 2: Vercel deployment protection + manual promote**
- `main` deploys to production but with manual approval gate
- Use Vercel's "Promote to Production" API after confirming staging looks good

Staging DB: a dedicated Neon branch called `staging` (shared across all staging deploys, persistent).

## Branch Protection

Only `main` needs protection. No `dev` branch:

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":false,"contexts":["build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null \
  --field allow_force_pushes=false \
  --field allow_deletions=false
```

## Agent Workflow

Agents always branch from `main`:

```bash
git checkout main && git pull
git checkout -b feat/issue-123-description
# ... implement ...
gh pr create --base main --title "feat: ..." --body "Closes #123"
```

All PRs target `main`. The coordinator spawns agents with this context. No `dev` branch exists.

## Compatibility with Current Setup

| Current | GitHub Flow |
|---------|-------------|
| All PRs target `dev` | All PRs target `main` |
| Neon `dev` branch (shared, all previews) | Per-PR ephemeral Neon branch |
| `dev` → `main` periodic promotion | Eliminated |
| `main` auto-deploys to production | `main` auto-deploys to staging; manual promote to production |
| No staging env | Lightweight staging env |
| Branch protection on `main` + `dev` | Branch protection on `main` only |

**To migrate:**
1. Update coordinator: PRs target `main`
2. Add GitHub Actions workflow for per-PR Neon branch lifecycle
3. Set up staging Vercel project (or deployment protection)
4. Remove `dev` branch protection / archive `dev` branch

## Lessons & Gotchas

**Per-PR ephemeral DB runs schema migrations on PR open.** The migration must be additive/backward-compatible — production DB (from which the branch schema is derived) must be able to run alongside both old and new code. Breaking schema changes require a multi-step migration strategy.

**Staging DB can drift from production over time.** Refresh staging from production snapshot periodically (weekly or before major deploys).

**Don't send clients to PR preview URLs.** Use the dedicated staging domain for external reviews. PR previews are for developer/agent testing only.

**The `dev` Neon branch becomes obsolete.** With per-PR branches, the shared `dev` Neon branch is no longer needed. Keep it briefly as fallback during migration, then remove.
