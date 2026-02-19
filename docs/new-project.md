# New Project Checklist

## 1. Repository

- [ ] Create repo under SylphxAI org
- [ ] Add `CLAUDE.md` at root with project-specific context
- [ ] Add `biome.json`
- [ ] Add `.github/workflows/ci.yml` (see [CI/CD](ci-cd.md))
- [ ] Use Bun as package manager

## 2. Branches

- [ ] Push initial code to `main`
- [ ] Create `dev` branch from `main`
- [ ] Set branch protection on both `main` and `dev`:
  - Require PR before merging
  - Require 1 approval
  - Require `build` status check
  - Block force push and deletion
- [ ] Consider setting default branch to `dev`

## 3. Vercel

- [ ] Import project to Vercel (Sylphx team)
- [ ] Connect GitHub repo
- [ ] Set Production branch to `main`
- [ ] **If this is a Turborepo monorepo:** set Ignored Build Step to `npx turbo-ignore` on EVERY Vercel project connected to this repo (see [Environments](environments.md)). This is mandatory — skipping it causes hundreds of wasted builds per day.
- [ ] Set environment variables separately:
  - **Production target**: production DATABASE_URL, real API keys
  - **Preview target**: dev DATABASE_URL, dev API keys

## 4. Neon Database

- [ ] Create Neon project (Sylphx org)
- [ ] Get `main` branch connection string → Vercel Production env
- [ ] Create `dev` branch (copy-on-write from main)
- [ ] Get `dev` branch connection string → Vercel Preview env

## 5. Schema & Migrations

- [ ] Define Drizzle schema in `src/db/schema/`
- [ ] Add `atlas.hcl` config
- [ ] Generate initial migration
- [ ] Apply to dev database

## 6. Agent Setup

- [ ] Add project to coordinator prompt (repo path + branch + roster)
- [ ] Verify agents can clone and build the project
- [ ] First coordinator run — should create initial issues
