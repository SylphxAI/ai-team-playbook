# Vercel Monorepo

> Use `npx turbo-ignore` as Vercel's Ignored Build Step to skip builds intelligently in Turborepo monorepos.

## Problem

A Turborepo monorepo connected to multiple Vercel projects triggers builds on **all** projects for every push, even if only one app changed.

With frequent commits (e.g., from agent teams), this causes hundreds of wasted builds per day.

We burned **72 hours of build time = ~$30/day** before catching this.

## Solution

`npx turbo-ignore` as the Ignored Build Step — Vercel's native solution for Turborepo monorepos. It uses Turborepo's dependency graph to only build projects affected by a given commit.

**How it works:**
- Only builds projects affected by the commit
- Skips unchanged projects (zero build minutes consumed)
- Intelligently handles shared package changes (builds all dependents)

## Implementation

### Via Vercel Dashboard

Vercel Dashboard → Project Settings → Git → "Ignored Build Step" → Custom → `npx turbo-ignore`

### Via Vercel API

```bash
curl -X PATCH "https://api.vercel.com/v9/projects/{projectId}?teamId={teamId}" \
  -H "Authorization: Bearer $VERCEL_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"commandForIgnoringBuildStep": "npx turbo-ignore"}'
```

### Convention

**MUST be set whenever connecting a new Turborepo monorepo project to Vercel. No exceptions.**

Add this as a checklist item in your new project setup process.

## Examples

A monorepo with three apps (`web`, `admin`, `api`):

- Push changes only to `apps/web` → only `web` builds, `admin` and `api` skip
- Push changes to `packages/ui` (shared) → all three build (correct — they all depend on it)
- Push changes to root `package.json` → Turborepo determines affected scope and builds accordingly

## Lessons & Gotchas

- **`exit 0` ≠ `npx turbo-ignore`**: `exit 0` as the ignored build step skips **all** builds always (workaround people use). `npx turbo-ignore` skips **intelligently**. Always use the correct one.
- **Set this on every project.** It's easy to forget when adding a new app to the monorepo. Missing it means that project builds for every commit regardless.
- **Turborepo must be in your devDependencies.** `npx turbo-ignore` needs turbo available. If it's not, the command fails and Vercel falls back to always-build.
- **Check the Vercel build logs.** If you see "Ignoring build" on unrelated projects — it's working. If you see builds triggering for untouched apps — `npx turbo-ignore` is missing.
