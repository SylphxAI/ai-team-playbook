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

> ⚠️ **MANDATORY for all Turborepo monorepos** — skip this and you will burn build minutes at scale.

**The problem:** A Turborepo monorepo connected to multiple Vercel projects triggers builds on ALL projects for every push, even if only one app changed. With agents merging 20+ PRs/day, this causes hundreds of failed/unnecessary builds — 72+ hours of wasted build minutes per day, ~$30/day in costs.

**The solution:** Set `commandForIgnoringBuildStep: "npx turbo-ignore"` on **every** Vercel project in a Turborepo monorepo. `turbo-ignore` checks whether the app actually changed before allowing the build to proceed.

```bash
# Apply to each Vercel project in a monorepo:
vercel project update <project-name> \
  --command-for-ignoring-build-step "npx turbo-ignore"
```

Or set it via the Vercel dashboard: **Project Settings → Git → Ignored Build Step → Custom** → `npx turbo-ignore`

**This MUST be set whenever connecting a new Turborepo monorepo project to Vercel.** Add it to the new project checklist — do not skip it.

Already applied to: SaaS (platform/puzzled/trivia), Epiow (platform/apps/web), Lens (website).

For monorepos with multiple Vercel projects:
- Each app has its own Vercel project with `rootDirectory` set
- `npx turbo-ignore` handles the rest — no additional flags needed

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
