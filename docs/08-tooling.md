# Recommended Tooling

> These are the specific tools we use and why. Not an exhaustive comparison — just what works.

## GitHub Apps — Bot Identity

### Why GitHub Apps Instead of Personal Access Tokens

| Feature | PAT | GitHub App |
|---------|-----|-----------|
| Identity | Shows as your account | Shows as `app-name[bot]` |
| Permissions | All or nothing (repo scope) | Granular per-permission |
| Token expiry | Never (or manual) | 1 hour (auto-rotate) |
| Per-repo install | No | Yes |
| Rate limits | 5,000/hr (shared with you) | 5,000/hr per installation |
| Audit trail | "shtse8 merged PR" | "sylphx-builder[bot] opened PR" |

### Our Setup: Two Apps

**sylphx-builder** — the Builder agent's identity
- Permissions: `contents: write`, `pull-requests: write`, `issues: write`
- Used for: creating branches, pushing code, opening PRs, commenting on issues
- Cannot: approve PRs, manage repo settings, delete repos

**sylphx-reviewer** — the Reviewer agent's identity
- Permissions: `pull-requests: write` (for approvals), `contents: read`
- Used for: reviewing PRs, approving or requesting changes
- Cannot: push code, merge PRs, modify repo settings

**Why two apps?** Branch protection requires that the approval comes from a different account than the PR author. If the Builder opens a PR as `sylphx-builder[bot]` and the Reviewer approves as `sylphx-reviewer[bot]`, the branch protection rule is satisfied.

### Token Generation

```javascript
// gh-app-token.js — generates a short-lived installation token
const { createAppAuth } = require('@octokit/auth-app');

const auth = createAppAuth({
  appId: process.env.GITHUB_APP_ID,
  privateKey: process.env.GITHUB_APP_PRIVATE_KEY,
  installationId: process.env.GITHUB_APP_INSTALLATION_ID,
});

const { token } = await auth({ type: 'installation' });
console.log(token); // Valid for 1 hour
```

Usage in scripts:
```bash
export GH_TOKEN=$(node /path/to/gh-app-token.js builder)
gh pr create --title "feat: add login page" --body "Closes #42"
```

### Token Rotation

GitHub App installation tokens expire after 1 hour. For long-running agents:

```typescript
class TokenManager {
  private token: string;
  private expiresAt: Date;

  async getToken(): Promise<string> {
    if (!this.token || this.isExpired()) {
      this.token = await generateToken();
      this.expiresAt = new Date(Date.now() + 55 * 60 * 1000); // 55 min buffer
    }
    return this.token;
  }

  private isExpired(): boolean {
    return new Date() >= this.expiresAt;
  }
}
```

## GitHub Actions — CI

### Why Actions

- Native GitHub integration (no webhook setup, no separate service)
- Free for public repos
- Excellent ecosystem of actions (`actions/checkout`, `actions/setup-node`, etc.)
- Concurrency controls built-in
- Matrix builds for testing across Node versions

### Our CI Config

See [CI/CD Pipeline](05-ci-pipeline.md) for the full workflow. Key actions we use:

```yaml
- actions/checkout@v4          # Checkout code
- actions/setup-node@v4        # Install Node.js with caching
- pnpm/action-setup@v4         # Install pnpm
- oven-sh/setup-bun@v2         # Install Bun
- actions/github-script@v7     # Run JS in CI (for issue creation, comments)
- ariga/setup-atlas@v1         # Install Atlas for migration checks
```

### Cost Control

For private repos, Actions minutes cost money. Tips:
- Use `concurrency` + `cancel-in-progress` to kill stale runs
- Cache `node_modules` aggressively
- Use `timeout-minutes` to prevent runaway builds
- Skip CI on docs-only changes:

```yaml
on:
  pull_request:
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'
```

## Atlas — Database Migrations

### Why Atlas Over Drizzle Migrations

| Feature | Drizzle Migrations | Atlas |
|---------|-------------------|-------|
| Approach | Imperative (write SQL) | Declarative (describe schema) |
| Parallel agents | Broken (sequential numbering) | Works (schema diff) |
| Rollback | Manual | Automatic |
| Lint | None | Built-in (destructive change detection) |
| CI integration | Manual | `atlas migrate lint` |

### Quick Start

```bash
# Install
curl -sSf https://atlasgo.sh | sh

# Initialize from existing database
atlas schema inspect --url "postgresql://..." > schema.hcl

# Generate migration from schema change
atlas migrate diff add_feature --env local

# Apply in production
atlas migrate apply --env production
```

See [Migration Strategy](06-migration-strategy.md) for the full migration path.

## Drizzle ORM — Queries & Type Safety

We still use Drizzle for queries. It's excellent at:

```typescript
// Type-safe queries
const users = await db
  .select()
  .from(usersTable)
  .where(eq(usersTable.email, email))
  .limit(1);

// Type-safe inserts
await db.insert(usersTable).values({
  email: 'user@example.com',
  name: 'Test User',
});

// Relations
const posts = await db.query.posts.findMany({
  with: { author: true, comments: true },
  where: eq(posts.published, true),
});
```

**The separation**: Drizzle handles queries and TypeScript types. Atlas handles migrations. They share the same database, and we keep the Drizzle schema file and Atlas schema file in sync.

## Vercel — Production Deploys

### Why Vercel

- Zero-config Next.js deploys
- Automatic preview deployments on PRs
- Edge functions for API routes
- Built-in analytics and monitoring
- Git-based deploys (push to main = deploy to production)

### Configuration

```json
// vercel.json
{
  "framework": "nextjs",
  "buildCommand": "pnpm build",
  "installCommand": "pnpm install --frozen-lockfile"
}
```

### The Gotchas

1. **Ignore build step** exit codes are backwards (see [Lessons Learned #3](07-lessons-learned.md))
2. **Environment variables** must be set in Vercel dashboard, not just `.env`
3. **Build cache** can cause stale deploys — clear it when things are weird
4. **Preview deploys** use the PR branch, not the merged result — they can pass even if the merge would fail

## Branch Protection + Auto-Merge — The Merge Safety Net

This combination replaces the need for any agent to have merge permissions:

```
Branch Protection:
  ✓ Require CI to pass
  ✓ Require 1 approval
  ✓ Require up-to-date branch

Auto-Merge:
  ✓ Enabled
  ✓ Squash merge only
  ✓ Auto-delete branches
```

**The flow**:
1. Builder opens PR → CI runs
2. Reviewer approves → auto-merge is armed
3. CI passes → PR merges automatically
4. Branch is deleted automatically

No agent touches the merge button. No agent has merge permissions. The infrastructure handles it.

## Tool Stack Summary

```
Code         → AI Agents (Builder, Reviewer, Fixer)
Coordination → Coordinator Agent + GitHub Issues/PRs
CI           → GitHub Actions
Deploy       → Vercel (auto-deploy on main push)
Database     → PostgreSQL
Migrations   → Atlas (declarative)
ORM          → Drizzle (queries + types)
Identity     → GitHub Apps (builder + reviewer)
Merge Safety → Branch Protection + Auto-Merge
Monitoring   → Vercel Analytics + GitHub Actions logs
```

Everything is managed through GitHub. Issues are the task queue. PRs are the review pipeline. Actions are the CI. Branch protection is the safety net. Auto-merge is the closer.
