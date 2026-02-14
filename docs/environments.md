# Environments

## Overview

| Environment | Branch | URL | Database |
|------------|--------|-----|----------|
| Production | `main` | Custom domain | Neon `main` branch |
| Preview | `dev` / PRs | `*.vercel.app` | Neon `dev` branch |

## Vercel

- **Production**: Auto-deploys on push to `main`
- **Preview**: Auto-deploys on push to `dev` and on PR creation
- Each Vercel project has separate env vars for Production and Preview targets

### Environment Variables

- **Production** env vars → only apply to `main` deployments
- **Preview** env vars → apply to `dev` and PR deployments
- **Never share production database credentials with preview** — use Neon dev branch

### Monorepo (Turborepo)

For monorepos with multiple Vercel projects:
- Each app has its own Vercel project with `rootDirectory` set
- Use `npx turbo-ignore --fallback=HEAD~1` as `ignoreCommand` to skip unchanged apps

## Neon (Database)

Each project has a Neon project with two branches:
- `main` branch — production database
- `dev` branch — development database (copy-on-write from main)

The dev branch is a copy-on-write clone — reads from main until written to, near-zero storage cost.

## Vercel ↔ Neon Connection

| Vercel Target | Neon Branch | DATABASE_URL |
|--------------|-------------|--------------|
| Production | `main` | Production connection string |
| Preview | `dev` | Dev branch connection string |

Set `DATABASE_URL` separately for Production and Preview targets in Vercel project settings.
