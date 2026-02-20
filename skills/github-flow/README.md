# GitHub Flow — PR-First Development

> Three environments: ephemeral per-PR previews, persistent staging, and stable production. AI agents work in isolation; only Gatekeeper-approved code reaches staging.

## Mental Model

```
feature branch
     │
     ▼
PR opened ──────► Ephemeral Preview
                  • URL: pr-{N}.preview.{domain}
                  • Isolated Docker container (spun up in CI, torn down on PR close)
                  • Isolated Postgres DB (empty + migrations run fresh)
                  • For: Gatekeeper review, tester verification, CI checks
     │
     ▼ CI pass + Gatekeeper approves → squash merge
     │
  staging ──────► staging.{domain}
                  • Permanent URL (e.g. staging.tryit.fun)
                  • Persistent staging Postgres DB
                  • Auto-migrates on every merge
                  • The "nightly build" — latest integrated work
                  • For: product owner review, release decisions
     │
     ▼ release decision → PR: staging → main
     │
  main ──────────► {domain} (production)
                   • Production Postgres DB
                   • Migrations apply on release
                   • Stable — only changes on intentional release
```

## Three Environments

| | Ephemeral Preview | Staging | Production |
|--|---|---|---|
| **Trigger** | PR opened | Merge to `staging` | PR: staging → main |
| **URL** | `pr-{N}.preview.{domain}` | `staging.{domain}` | `{domain}` |
| **Lifespan** | Lives while PR is open | Always running | Always running |
| **Database** | Isolated per-PR Postgres (empty + migrations) | Persistent staging Postgres | Persistent production Postgres |
| **Purpose** | Test this PR in isolation | Integration, release candidate review | Live users |

**Note: DEV has no URL and does not exist as an infrastructure environment.** AI agents have no local machines — DEV is undefined. Features go straight to ephemeral preview.

## Database Strategy

### Ephemeral Preview: Per-PR Postgres Database

Each PR gets its own isolated database. Schema is built from scratch (empty DB + all migrations). No production data — purely for schema and API verification.

```bash
# On PR open: create DB and run migrations
CREATE DATABASE preview_{repo}_pr_{N};
# ...run Atlas/Drizzle migrations against it

# On PR close: destroy
DROP DATABASE preview_{repo}_pr_{N};
```

The AX162-R server has sufficient resources to run dozens of concurrent preview DBs.

### Staging: Persistent Postgres Database

A dedicated persistent Postgres database per app on AX162-R. Auto-migrated on every merge to `staging`. Periodically refreshed from a production snapshot to prevent excessive drift.

### Production: Production Postgres Database

Never touched by previews or staging automation. Updated only via intentional Atlas migrations on release deploys.

## Infrastructure

**All environments run on AX162-R (162.55.233.221) via Docker Compose.**

Not Vercel. Not Neon. Self-hosted.

| Layer | Tool |
|-------|------|
| Hosting | Docker Compose |
| Reverse proxy | Traefik (Cloudflare DNS-01 ACME) |
| Database | Postgres 17 (self-hosted, per-app) |
| Ephemeral preview orchestration | GitHub Actions → SSH → AX162-R |

## Branch Strategy

Two permanent branches:

```
main      ← production (never touch directly)
staging   ← integration point (all PRs target here)
feature/  ← ephemeral, one per PR, deleted after merge
```

Branch protection on both `main` and `staging`:
- Require PR (no direct push)
- 1 required approval (Gatekeeper)
- Required CI check: `build`
- Squash merge only
- Auto-delete head branches

## Release Process

Production releases are intentional. When staging has accumulated enough tested changes:

```bash
# Gatekeeper opens release PR
gh pr create \
  --base main \
  --head staging \
  --title "release: $(date +%Y-%m-%d)" \
  --body "Promoting staging to production"
```

Same approval + CI requirements as any other PR.

## Agent Workflow

Every agent always branches from `staging`:

```bash
git checkout staging && git pull
git checkout -b feat/issue-123-description
# implement...
gh pr create --base staging --title "feat: ..." --body "Closes #123"
```

The Gatekeeper reviews the ephemeral preview, approves, squash merges. Staging auto-updates.

## Gotchas

**Staging DB can drift from production.** If production data evolves and staging diverges in content (though schema stays aligned via migrations), expect differences. This is fine — staging is for schema + feature testing.

**Per-PR DBs start empty.** No seed data by default. Apps must handle empty-state gracefully, or a minimal seed script should run in CI. Test the real first-time user experience.

**Atlas migrations run in CI before deploy.** Migrations apply to staging on every merge. Per-PR previews run full migration chain from empty.

**Don't send external clients to PR preview URLs.** Ephemeral URLs are for internal review. Clients always review on `staging.{domain}` — it's stable, professional, and doesn't expire.
