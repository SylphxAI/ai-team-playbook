# Migration Strategy — From Manual to Autonomous

> Our journey: v1 (naive) → v2 (structured) → v3 (FSM + triage).
> Each version fixed the failures of the previous one.

## The Three Versions

### v1 — Naive (Day 1)

**Architecture:** One coordinator, one builder, one reviewer. No labels. No FSM. The coordinator kept a mental model of what was happening and made decisions based on vibes.

**What worked:**
- PRs got opened and merged
- Basic code was produced

**What broke:**
- Coordinator merged PRs while CI was still running (PR #381 disaster)
- No quality gate → Scout created junk issues, Builder implemented them
- Duplicate PRs for the same issue (no dedup)
- No timeout handling → stuck PRs stayed stuck forever
- Coordinator did too much: coordinated + reviewed + sometimes coded

**Lessons:**
- Never let the coordinator merge directly
- Separate concerns: one agent, one job
- You need a quality gate (triage) before building

### v2 — Structured (Week 1)

**Architecture:** Coordinator + Builder + Reviewer + Fixer + Rebaser + Auditor (our original Scout). Added basic labels (`in-progress`, `ready`). Coordinator still ran on vibes but with better prompts.

**What worked:**
- Role separation reduced conflicts
- Basic label tracking improved visibility
- Review cycle caught more bugs
- Fixer addressed review feedback without re-creating PRs

**What broke:**
- `strict: true` with auto-merge → cascading conflicts, PRs stuck for hours
- Vercel pending statuses blocked auto-merge indefinitely
- No triage → 30-50% of work was wasted on low-value issues
- GitHub `reviewDecision` stayed `CHANGES_REQUESTED` even after new approvals
- Coordinator overlapping runs caused duplicate agent spawns
- No learning loop → same mistakes repeated

**Lessons:**
- `strict: true` kills throughput at scale
- Triage is the #1 missing piece
- Auto-merge doesn't work with `strict: true` (cascading conflicts)
- Need explicit FSM states, not ad-hoc labels
- GitHub API has quirks you learn the hard way

### v3 — FSM + Triage (Current)

**Architecture:** 6 specialized roles (Scout, Triage, Builder, Reviewer, Merger, Sentinel). GitHub labels as FSM state. Reconciliation loop (Kubernetes controller pattern). No locks. `strict: false`.

**What works:**
- 30 PRs merged on launch day
- 14 PRs batch-merged in 2 minutes
- Triage prevents 30-50% of wasted work
- Sentinel catches post-merge issues
- Learning loop improves agents over time
- Idempotent operations → no duplicate work even with overlapping runs

**Configuration:**
- WIP limit: 10
- Backlog threshold: 15
- Max fix retries: 2
- Cron interval: 5 min
- Branch protection: `strict: false`, `build` check required, 1 review

## Label Migration

When transitioning from v2 to v3, existing issues need pipeline labels.

### Step 1: Audit Existing Issues

```bash
# Find issues with old labels
gh issue list --state open --json number,title,labels,assignees

# Categorize each:
# - Has assignee + open PR → pipeline/pr-open
# - Has assignee + no PR → pipeline/in-progress
# - No assignee + "ready" label → pipeline/approved
# - No labels → pipeline/discovered (needs triage)
```

### Step 2: Create Pipeline Labels

```bash
# Pipeline state labels
gh label create "pipeline/discovered" --color "FBCA04"
gh label create "pipeline/approved" --color "0E8A16"
gh label create "pipeline/rejected" --color "B60205"
gh label create "pipeline/in-progress" --color "1D76DB"
gh label create "pipeline/pr-open" --color "5319E7"
gh label create "pipeline/review-passed" --color "0E8A16"
gh label create "pipeline/review-failed" --color "E99695"
gh label create "pipeline/merged" --color "006B75"
gh label create "pipeline/abandoned" --color "D93F0B"

# Priority labels
gh label create "priority/P0" --color "B60205"
gh label create "priority/P1" --color "D93F0B"
gh label create "priority/P2" --color "FBCA04"

# Type labels
gh label create "type/bug" --color "D73A4A"
gh label create "type/enhancement" --color "A2EEEF"
gh label create "type/chore" --color "D4C5F9"
gh label create "type/security" --color "B60205"
gh label create "type/performance" --color "F9D0C4"

# Size labels
gh label create "size/XS" --color "009800"
gh label create "size/S" --color "77BB00"
gh label create "size/M" --color "FBCA04"
gh label create "size/L" --color "D93F0B"

# Source labels
gh label create "source/scout" --color "C5DEF5"
gh label create "source/human" --color "BFD4F2"
gh label create "source/sentinel" --color "D4C5F9"

# Meta labels
gh label create "agent-authored" --color "EDEDED"
```

### Step 3: Apply Labels to Existing Issues

```bash
# For each existing issue, apply the appropriate pipeline label
gh issue edit 42 --add-label "pipeline/approved,priority/P1"
gh issue edit 43 --add-label "pipeline/in-progress"
# etc.
```

### Step 4: Create Infrastructure Issues

```bash
# Metrics dashboard
gh issue create --title "📊 Pipeline Metrics" --label "pipeline/coordinator-lock" --pin

# Start the coordinator cron
# (platform-specific — see your orchestration tool's docs)
```

## The `strict: true` → `strict: false` Transition

This is the scariest change. Here's how to do it safely:

### 1. Verify Sentinel Is Working

Before changing branch protection:
- Confirm Sentinel health checks are running
- Confirm the production URL responds to health checks
- Confirm P0 issue creation works

### 2. Change Branch Protection

```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":false,"contexts":["build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

### 3. Test with a Small Batch

Merge 2-3 PRs quickly (within the same minute). Check:
- Does production stay healthy?
- Does Sentinel detect the merges and verify?
- Does CI still run correctly on subsequent PRs?

### 4. Scale Up

Once confident, increase WIP limit and let the pipeline ramp up.

## Database Migration Considerations

If your project uses sequential migration files (e.g., Drizzle), parallel agents will create conflicting migrations. Solutions:

1. **Timestamp-based naming**: `20260212_143022_add_feature.sql` instead of `0004_add_feature.sql`
2. **Declarative migrations** (Atlas): Agents modify schema declarations, tool generates migrations
3. **Push-based development** (`drizzle-kit push`): Skip migration files in dev, generate in CI

See the original v0.1 playbook docs for detailed migration strategies.
