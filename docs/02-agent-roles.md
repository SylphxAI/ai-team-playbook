# Agent Roles & Responsibilities

> Every agent has exactly one job. The moment an agent starts doing two things, both things break.

## The Six Roles

### 🎯 Coordinator

**Job**: Orchestrate the workflow. Spawn agents. Track progress. NEVER write code.

The Coordinator is the air traffic controller. It:
- Reads open issues and decides which to work on
- Spawns Builder agents with specific issue assignments
- Spawns Reviewer agents when PRs are ready
- Monitors for stale PRs and stuck agents
- Runs audit cycles to find new work

**What it must NOT do:**
- Write code (not even "just a small fix")
- Merge PRs (that's GitHub's job via auto-merge)
- Review PRs (that's the Reviewer's job)
- Create cron jobs without explicit permission
- Make architectural decisions

**Why the Coordinator must never merge:**
We learned this the hard way. When the Coordinator had merge permissions, it would:
1. See an approved PR
2. Call the merge API immediately
3. Skip waiting for CI to finish
4. Merge code that doesn't build

The fix: remove merge capability entirely. Auto-merge handles it.

### 🔨 Builder

**Job**: Implement a single issue. Create one PR.

The Builder receives a specific issue and:
1. Creates a feature branch (`feat/issue-123-add-login-page`)
2. Reads the issue description and any linked context
3. Implements the solution
4. Runs `pnpm build` and `pnpm lint` locally before pushing
5. Opens a PR with a clear description referencing the issue
6. Stops. Does not review. Does not merge. Does not pick up the next issue.

**Naming convention for branches:**
```
feat/issue-{number}-{short-description}
fix/issue-{number}-{short-description}
chore/issue-{number}-{short-description}
```

**PR description template:**
```markdown
Closes #{issue_number}

## What changed
- Brief description of changes

## How to test
- Steps to verify the change works

## Files changed
- `src/components/LoginPage.tsx` — new component
- `src/lib/auth.ts` — added login function
```

### 👀 Reviewer

**Job**: Review a single PR. Approve or request changes.

The Reviewer:
1. Reads the PR diff
2. Checks for: correctness, style, potential bugs, missing tests
3. Either approves with a comment, or requests changes with specific feedback
4. If requesting changes, clearly describes what needs to change and why

**Reviewer rules:**
- Never approve without reading the full diff
- If the PR is too large to review confidently, request it be split
- Don't nitpick style if there's a linter — let the linter handle it
- Focus on logic errors, missing edge cases, and security issues
- If the PR looks like a duplicate of another PR, flag it

**Using GitHub Apps for review identity:**
We use two GitHub Apps: `sylphx-builder` and `sylphx-reviewer`. This separation means:
- The Builder can't approve its own PRs
- Branch protection requires approval from a different account
- Audit trail clearly shows who built and who reviewed

### 🔧 Fixer

**Job**: Address review feedback on an existing PR.

When a Reviewer requests changes:
1. The Coordinator spawns a Fixer for that specific PR
2. The Fixer reads the review comments
3. Makes the requested changes on the same branch
4. Pushes a new commit
5. Comments on the PR that changes are ready for re-review

**Why not let the original Builder fix it?**
In practice, spawning a new agent is simpler than resuming a previous one's context. The Fixer starts fresh, reads the review comments, and makes targeted changes. This avoids context bloat from the original implementation session.

### 🔀 Rebaser

**Job**: Resolve merge conflicts on a PR that's fallen behind `main`.

When a PR has conflicts:
1. The Coordinator spawns a Rebaser for that specific PR
2. The Rebaser fetches latest `main`
3. Rebases the PR branch, resolving conflicts
4. Force-pushes the rebased branch
5. CI re-runs automatically

**This is a targeted role.** Most PRs won't need it — if you have "Require branches to be up to date" enabled, only one PR can merge at a time, and the rest queue naturally. The Rebaser handles the cases where a PR has been open long enough for conflicts to develop.

### 🔍 Auditor

**Job**: Find new work. Identify bugs, improvements, and feature opportunities.

The Auditor:
1. Reads the current codebase
2. Checks for: TODO comments, missing error handling, accessibility issues, performance problems
3. Creates well-specified GitHub issues for each finding
4. Labels issues appropriately (bug, enhancement, chore)

**Auditor output format:**
```markdown
## Title: [Bug] MobileTabBar renders outside AuthProvider

## Description
The MobileTabBar component uses `useAuth()` but is rendered in the root layout
outside the AuthProvider boundary. This causes a runtime error on mobile devices
that doesn't surface in the build step.

## Steps to Reproduce
1. Open the app on mobile
2. Navigate to any page with the tab bar
3. Observe: "Error: useAuth must be used within an AuthProvider"

## Expected Behavior
MobileTabBar should be inside the AuthProvider, or should handle the case
where auth context is unavailable.

## Files Involved
- `src/app/layout.tsx` (line 42)
- `src/components/MobileTabBar.tsx`
```

## How Agents Communicate

Agents don't talk to each other directly. They communicate through GitHub:

```
Coordinator reads issues → spawns Builder
Builder creates PR → Coordinator notices new PR → spawns Reviewer
Reviewer requests changes → Coordinator notices → spawns Fixer
Fixer pushes commit → Reviewer re-reviews → approves
Auto-merge merges the PR
```

GitHub is the message bus. Issues and PRs are the messages. This is intentional:
- Everything is logged and auditable
- No custom communication protocol to maintain
- Any agent can be replaced without changing the protocol
- Humans can participate at any point by commenting on issues/PRs

## Scaling

Start with one Coordinator. Add Builders as needed. You'll typically want:

| Project Phase | Builders | Reviewers | Coordinators |
|--------------|----------|-----------|-------------|
| Early (0-10 issues) | 1 | 1 | 1 |
| Active (10-50 issues) | 2-3 | 1 | 1 |
| Heavy (50+ issues) | 3-5 | 1-2 | 1 |

**Never run more than one Coordinator.** Two coordinators will spawn duplicate agents and create conflicting PRs. If one coordinator is a bottleneck, make it faster — don't add another.

## Identity & Authentication

Each role should have its own GitHub identity:

```
Builder  → sylphx-builder[bot]    (GitHub App)
Reviewer → sylphx-reviewer[bot]   (GitHub App)
```

The Coordinator doesn't need its own GitHub identity — it operates through the other agents' identities by spawning them.

**Why GitHub Apps instead of PATs?**
- Apps have scoped permissions (builder can't delete repos)
- Apps have separate identities (clear audit trail)
- App tokens expire automatically (1-hour TTL)
- Apps can be installed per-repo (principle of least privilege)
