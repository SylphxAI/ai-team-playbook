# Neon Branching

> **Note:** This skill describes Neon-hosted database branching. Our current production infrastructure uses self-hosted Postgres 17 on AX162-R. This skill remains relevant if using Neon as a managed database provider. For our self-hosted setup, see the [GitHub Flow](../github-flow/) skill for database strategy.

> Use Neon's copy-on-write branches to give dev and production separate databases with near-zero storage overhead.

## Problem

Sharing a single database between development and production is dangerous. Schema migrations run in dev can corrupt production data. Debugging with production credentials in preview environments is a security risk.

But maintaining two fully-separate databases is expensive and slow to keep in sync.

## Solution

Neon's branching model: two branches per project, copy-on-write from main. The `dev` branch starts as a snapshot of `main` with near-zero storage cost until written.

**Two branches per project:**
- `main` — production database
- `dev` — development database (copy-on-write from main, near-zero storage cost until written)

## Implementation

### Vercel ↔ Neon Mapping

| Vercel Target | Neon Branch | Env Var |
|---------------|-------------|---------|
| Production | `main` | `DATABASE_URL` (production) |
| Preview | `dev` | `DATABASE_URL` (preview) |

### Setup Steps

1. Create a Neon project — it comes with a `main` branch by default
2. Create a `dev` branch from `main` in the Neon console
3. Copy the connection string for each branch
4. In Vercel project settings, set `DATABASE_URL` **separately** for Production and Preview targets:
   - Production target → Neon `main` connection string
   - Preview target → Neon `dev` connection string

### Setting Environment Variables by Target in Vercel

In Vercel Dashboard → Project → Settings → Environment Variables:
- Add `DATABASE_URL` → select only "Production" → paste Neon `main` URL
- Add `DATABASE_URL` again → select only "Preview" → paste Neon `dev` URL

## Examples

```bash
# Neon dev branch connection string format
postgresql://user:password@ep-xxx.us-east-2.aws.neon.tech/dbname?sslmode=require

# Production (main branch) — different endpoint
postgresql://user:password@ep-yyy.us-east-2.aws.neon.tech/dbname?sslmode=require
```

Each branch has its own compute endpoint (`ep-xxx` vs `ep-yyy`), so there's zero risk of dev queries hitting production.

## Lessons & Gotchas

- **Never share production credentials with preview.** Even "read-only" access is a risk. Preview deployments run in shared environments. Neon branches are free — use them.
- **Dev branch diverges over time.** The `dev` branch only reflects `main` at the moment it was created (or last reset). Schema drift accumulates. Periodically reset dev from main if they get too far apart.
- **Resetting dev is destructive.** Resetting the `dev` branch from `main` drops any data written to dev. Fine for development — just know it's happening.
- **Neon branches are per-project.** If your monorepo has multiple databases (e.g., `web-db`, `api-db`), each Neon project needs its own `main`/`dev` branches set up independently.
- **Copy-on-write means the first write triggers storage.** The dev branch costs nothing until someone writes to it. Once written, it stores only the delta from main.
