# Agent Roles — The Six Agents

> Every agent has exactly one job. The moment an agent does two things, both break.
> The #1 missing piece in v1/v2 was Triage. Without it, 30-50% of work is wasted.

## Overview

```
Scout → discovers issues
Triage → approves or rejects (THE critical quality gate)
Builder → implements approved issues
Reviewer → reviews PRs
Merger → merges approved PRs
Sentinel → monitors production health post-merge
```

Supporting roles (spawned as needed):
- **Fixer** — addresses review feedback on existing PRs
- **Rebaser** — resolves merge conflicts
- **CI Fixer** — fixes build failures

## 🔍 Scout (Discovery Agent)

**Job**: Find things worth fixing in the codebase.

| Aspect | Detail |
|--------|--------|
| **Trigger** | Coordinator dispatches when backlog < threshold |
| **Input** | Codebase, recent git log, existing issues (for dedup), learning log |
| **Output** | GitHub issues with `pipeline/discovered` label |
| **Constraints** | Max 10 issues per run. MUST check for duplicates. MUST reference specific code. |

**What to look for:** Bugs, security vulnerabilities, performance issues, dead code, missing error handling, accessibility gaps, outdated dependencies.

**What NOT to create issues for:** Cosmetic changes, subjective style preferences, issues already covered by open PRs, features outside the app's scope.

**Why duplicate detection matters:** Without it, the Scout creates the same issue repeatedly. Check against both open AND recently closed issues before creating anything.

### Scout Issue Template

```markdown
## Problem
{What's wrong? Be specific — file paths, line numbers, error messages.}

## Location
- `src/path/to/file.ts:42-67`

## Suggested Approach
{How should this be fixed? Concrete, implementable suggestion.}

## Impact
- **Users affected**: {who/what}
- **Severity**: low/medium/high
- **Frequency**: {how often}
```

---

## ⚖️ Triage (Quality Gate)

**Job**: Decide if discovered issues are worth pursuing. This is THE critical quality gate.

| Aspect | Detail |
|--------|--------|
| **Trigger** | Issues with `pipeline/discovered` label |
| **Input** | Issue content, codebase context, project goals, learning log, current workload |
| **Output** | APPROVE (with priority + size labels) or REJECT (with reasoning) |
| **Constraints** | Every issue MUST get a decision. No "maybe." |

### Why Triage Is the #1 Missing Piece

In v1 and v2 of our pipeline, issues went straight from Scout to Builder. The result:

- Builders spent hours implementing cosmetic changes that added no value
- PRs were opened for features outside the project's scope
- The same bug was "discovered" and built 3 times before anyone noticed
- Review rejection rate was 60%+ because the work shouldn't have been started

**Adding Triage dropped wasted work by 30-50%.** It's the single highest-leverage improvement you can make.

### Triage Criteria

Every issue is evaluated on six dimensions:

1. **Value**: Does this meaningfully improve the product, DX, performance, or security?
   - ❌ Reject: Cosmetic-only, subjective style, trivial naming
   - ✅ Approve: Bug fixes, security patches, meaningful UX improvements

2. **Scope**: Is this within the project's goals?
   - ❌ Reject: Tangential features, yak-shaving, out-of-scope
   - ✅ Approve: Direct product improvements, necessary maintenance

3. **Feasibility**: Can an agent implement this in a single PR?
   - ❌ Reject: Needs external service setup, requires human credentials
   - ✅ Approve: Well-defined code changes, clear acceptance criteria

4. **Size**: Small enough for one PR? (Target: < 500 lines)
   - ❌ Reject (with suggestion to split): Too large
   - ✅ Approve with size label: XS/S/M/L

5. **Duplication**: Already covered by an existing issue or PR?
   - ❌ Reject: Duplicate (reference the existing issue)
   - ✅ Approve: Unique work

6. **Risk**: Could this break things? What's the blast radius?
   - High risk (auth, data, payments) → P0 + extra scrutiny
   - Medium risk (UI, new features) → P1
   - Low risk (docs, minor fixes) → P2

### Learning Loop

Triage rejections are logged in `docs/pipeline-learning-log.md` and fed back into the Scout's prompt. Over time, the Scout stops creating issues that Triage rejects. Target rejection rate: 30-50%.

---

## 🏗️ Builder (Implementation Agent)

**Job**: Take one approved issue and produce one working PR.

| Aspect | Detail |
|--------|--------|
| **Trigger** | Issues with `pipeline/approved` + no assignee |
| **Input** | Issue description, codebase, conventions, learning log |
| **Output** | Pull request linking to the issue |
| **Constraints** | Max 1 issue at a time. PR < 500 lines. MUST pass local build before opening PR. |

### Builder Rules (from rejection research)

Research from arXiv (2602.04226) shows the top reasons agent PRs get rejected:

1. **Too large / hard to review** → Keep PRs small, one concern per PR
2. **No added value** → Don't refactor unrelated code
3. **Increased complexity** → Follow existing patterns exactly
4. **Context limitations** → Read the full codebase context, not just the issue

### Builder Workflow

```
1. Assign self to the issue
2. Create branch: agent/{issue-number}-{short-slug}
3. Read issue + relevant code context + learning log
4. Implement the fix (small, focused changes)
5. Run build + lint locally (MANDATORY — do not skip)
6. Open PR with description: "Closes #{issue_number}"
7. Update issue label: pipeline/in-progress → pipeline/pr-open
8. STOP. Do not review. Do not merge. Do not pick up next issue.
```

### PR Description Template

```markdown
Closes #{issue_number}

## Summary
{What changed and why — not just what, but WHY}

## Changes
- `src/path/file.ts` — {what changed}
- `src/path/other.ts` — {what changed}

## Testing
- [x] TypeScript compiles
- [x] Build passes
- [x] Lint passes
```

---

## 👁️ Reviewer (Review Agent)

**Job**: Review a single PR. Approve or request changes.

| Aspect | Detail |
|--------|--------|
| **Trigger** | PRs with `pipeline/pr-open` + CI green + no review |
| **Input** | PR diff, linked issue, codebase context, CI status |
| **Output** | APPROVE or REQUEST_CHANGES with specific feedback |
| **Constraints** | MUST wait for CI green. Uses separate GitHub App identity from Builder. |

### Review Checklist

- [ ] Does the PR address the linked issue?
- [ ] Are there logic errors or unhandled edge cases?
- [ ] Does it follow project conventions and existing patterns?
- [ ] Is there unnecessary complexity or over-engineering?
- [ ] Could this break existing functionality?
- [ ] Is the change size reasonable (< 500 lines)?
- [ ] Are there security implications?

### Identity Separation

The Reviewer uses a DIFFERENT GitHub App from the Builder:
- `sylphx-builder[bot]` opens the PR
- `sylphx-reviewer[bot]` approves (or requests changes)

Branch protection requires approval from a different account than the PR author. This is enforced automatically.

### GitHub `reviewDecision` Bug

**Critical learning:** GitHub's `reviewDecision` field stays `CHANGES_REQUESTED` even after the reviewer submits a new approval — if the old review isn't explicitly dismissed.

**Fix:** The Reviewer must dismiss stale `CHANGES_REQUESTED` reviews before submitting a new approval:

```bash
# Get stale review IDs
gh api repos/{owner}/{repo}/pulls/{N}/reviews \
  --jq '.[] | select(.state == "CHANGES_REQUESTED") | .id'

# Dismiss each
gh api -X PUT repos/{owner}/{repo}/pulls/{N}/reviews/{REVIEW_ID}/dismissals \
  -f message="Superseded by re-review" -f event="DISMISS"
```

---

## 🚀 Merger (Ship Agent)

**Job**: Merge approved PRs safely.

| Aspect | Detail |
|--------|--------|
| **Trigger** | PRs with `pipeline/review-passed` + CI green + no conflicts |
| **Input** | PR state, CI status, merge conflict status |
| **Output** | Merged PR, closed issue with `pipeline/merged` label |
| **Constraints** | ALL checks must pass. Never force merge. |

### Batch Merging

With `strict: false`, **merge ALL ready PRs in the same cycle**. No need to wait, no need to rebase between merges.

```bash
# Merge all ready PRs in sequence — they don't need to be serialized
gh pr merge 409 --squash --delete-branch
gh pr merge 410 --squash --delete-branch
gh pr merge 411 --squash --delete-branch
# ... all in the same coordinator cycle
```

On launch day, we merged **14 PRs in 2 minutes** using this approach.

### Vercel Stuck Status Workaround

Vercel sometimes leaves commit statuses in `pending` forever. If Vercel is pending for >10 minutes, override:

```bash
gh api repos/{owner}/{repo}/statuses/{SHA} \
  -f state=success \
  -f context=Vercel \
  -f description="Override stale pending"
```

**Note:** GitHub App tokens can't override third-party statuses. Use a personal account for this.

---

## 🛡️ Sentinel (Health Monitor)

**Job**: Post-merge verification and production health monitoring. The safety net that makes `strict: false` safe.

| Aspect | Detail |
|--------|--------|
| **Trigger** | After every merge + every coordinator cycle |
| **Input** | Vercel deployment status, production health endpoint |
| **Output** | Health verification comment on merged PRs, or P0 issue + revert if broken |
| **Constraints** | Must identify which merge caused a failure |

### How Sentinel Replaces `strict: true`

`strict: true` prevents merging stale PRs (PRs that haven't been tested against the latest main). The concern: two PRs that individually pass CI might break when combined.

Sentinel addresses this differently: **let them merge, then check immediately.** If production breaks:

1. Identify the most recent merge as likely culprit
2. Create a P0 issue with full context
3. (Optional) Create a revert PR automatically
4. Alert the coordinator to prioritize the fix

In practice, semantic merge conflicts are rare for small, focused PRs (which our pipeline enforces). Sentinel has never had to revert a merge in our pipeline.

### Health Check

```bash
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://your-app.com)
# 200 = healthy, anything else = alert
```

---

## Supporting Roles

### 🔧 Fixer
Addresses review feedback on an existing PR. Spawned when a Reviewer requests changes. Reads review comments, makes targeted fixes, pushes to the same branch.

### 🔀 Rebaser
Resolves merge conflicts. Fetches latest main, rebases the PR branch, resolves conflicts, force-pushes. Only needed when PRs have been open long enough for conflicts to develop.

### 🔨 CI Fixer
Fixes build failures on PRs. Reads CI logs, diagnoses the root cause, pushes a fix. Common issues: missing imports, type errors, missing dependencies.

---

## Communication Pattern

Agents don't talk to each other. They communicate through GitHub:

```
Coordinator reads issues → spawns Builder
Builder creates PR → Coordinator notices → spawns Reviewer
Reviewer requests changes → Coordinator notices → spawns Fixer
Fixer pushes fix → Coordinator notices → spawns Reviewer again
Reviewer approves → Coordinator notices → Merger merges
Sentinel verifies production health
```

GitHub is the message bus. Issues and PRs are the messages. Everything is logged and auditable.

---

## Scaling

| Project Phase | Builders | Reviewers | WIP Limit | Throughput |
|--------------|----------|-----------|-----------|------------|
| Starting out | 2-3 | 1 | 3 | ~5 PRs/day |
| Active | 5-7 | 1-2 | 5 | ~15 PRs/day |
| Full speed | 10 | 2 | 10 | ~30 PRs/hour |

**Never run more than one Coordinator.** Two coordinators = duplicate agents, conflicting decisions, double API costs.

---

## Identity & Authentication

Each role needs its own GitHub identity:

| Role | Identity | Permissions |
|------|----------|-------------|
| Builder | `sylphx-builder[bot]` (GitHub App) | `contents: write`, `pull-requests: write`, `issues: write` |
| Reviewer | `sylphx-reviewer[bot]` (GitHub App) | `pull-requests: write`, `contents: read` |
| Merger | Uses Builder token | Same as Builder |
| Sentinel | Uses Builder token | Same as Builder |

**Why GitHub Apps?**
- Scoped permissions (builder can't delete repos)
- Separate identities (clear audit trail)
- Auto-expiring tokens (1-hour TTL)
- Per-repo installation (least privilege)
- Separate rate limits (5,000/hr per app)
