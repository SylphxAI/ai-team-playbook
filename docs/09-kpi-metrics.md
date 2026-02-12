# KPI & Metrics — Performance Tracking

> What gets measured gets improved.
> Track throughput, quality, and health. Our launch day: 30 PRs merged.

## Throughput Metrics

| Metric | Definition | Target | Our Launch Day |
|--------|-----------|--------|----------------|
| **PRs merged/day** | Successfully merged PRs | > 20 | **30** |
| **PRs merged/hour** | Sustained merge rate | > 5 | **14 in 2 min** (peak) |
| **Cycle time** | APPROVED → MERGED duration | < 4 hours | ~2 hours (avg) |
| **Lead time** | DISCOVERED → MERGED duration | < 24 hours | ~4 hours |
| **Issues discovered/day** | New issues created by Scout | 5-15 | ~20 |
| **Issues approved/day** | Issues passing triage | 3-10 | ~15 |

### Throughput Formula

```
PRs/hour ≈ WIP_LIMIT × completions_per_builder_per_hour
```

| WIP | Completions/hr | Theoretical Max |
|-----|---------------|-----------------|
| 3 | 3 | ~9 PRs/hr |
| 5 | 3 | ~15 PRs/hr |
| 10 | 3 | ~30 PRs/hr |

The bottleneck shifts at different scales:
- **WIP 1-3:** Builder capacity (not enough parallel work)
- **WIP 5-7:** Review throughput (reviewer can't keep up)
- **WIP 10+:** GitHub API rate limits (5,000 requests/hour)

## Quality Metrics

| Metric | Definition | Target | Alert If |
|--------|-----------|--------|----------|
| **Triage rejection rate** | Rejected / Total discovered | 30-50% | < 20% or > 70% |
| **Review first-pass rate** | Approved on first review / Total reviews | > 70% | < 50% |
| **CI pass rate** | Green CI / Total PR pushes | > 85% | < 70% |
| **Revert rate** | Reverts / Total merges | < 5% | > 10% |
| **Abandon rate** | Abandoned issues / Total in-progress | < 20% | > 30% |

### Interpreting Triage Rejection Rate

- **< 20%:** Scout is too conservative (not finding enough issues) OR Triage is too permissive (approving junk). Result: Builders waste time on low-value work.
- **30-50%:** Sweet spot. Scout is discovering broadly, Triage is filtering effectively.
- **> 70%:** Scout is too noisy (creating junk issues). Fix the Scout prompt.

### Interpreting Review First-Pass Rate

- **< 50%:** Builders are producing low-quality PRs. Check:
  - Are they running build locally before pushing?
  - Is the learning log being fed into Builder prompts?
  - Are issues well-defined enough for agents to implement?
- **> 90%:** Reviewer might be too lenient. Check a sample of approved PRs for quality.

## Health Metrics

| Metric | Definition | Alert Threshold |
|--------|-----------|----------------|
| **Backlog depth** | Approved issues awaiting builders | > 20 |
| **Stuck work** | Issues in same state > timeout | Any |
| **Coordinator skip rate** | Skipped cycles / Total cycles | > 30% |
| **API rate usage** | Requests / 5,000 per hour | > 80% |
| **Production uptime** | Healthy checks / Total checks | < 99% |

### Backlog Management

```
Backlog too deep (> 20 approved issues):
  → Increase WIP limit
  → Or increase Builder count
  → Or slow down Scout

Backlog too shallow (< 3 approved issues):
  → Scout should run more frequently
  → Or lower backlog threshold

Backlog at zero:
  → Pipeline is idle — Scout should discover new work
  → This is fine during low-activity periods
```

## Metrics Dashboard

We track metrics via a pinned GitHub issue, updated each coordinator cycle:

```markdown
# 📊 Pipeline Metrics — Updated 2026-02-12T16:00:00Z

## Pipeline State
| State | Count | Issues |
|-------|-------|--------|
| discovered | 3 | #423, #424, #426 |
| approved | 5 | #412, #413, #414, #415, #416 |
| in-progress | 3 | #417, #418, #419 |
| pr-open | 4 | #420, #421, #422, #423 |
| review-passed | 2 | #424, #425 |
| review-failed | 1 | #426 |

## Active Agents
- viral-build-issue417 (12 min)
- viral-build-issue418 (8 min)
- viral-review-PR420 (3 min)

## Production Health
- Status: HTTP 200 ✅
- Last checked: 2026-02-12T16:00:00Z

## WIP
- In-progress: 3 / 10
- Backlog: 8 / 15 (approved + discovered)

## Today's Stats
- Merged: 30 PRs
- Discovered: 20 issues
- Triage rejection rate: 35%
- Review first-pass rate: 72%
```

## API Rate Budget

GitHub allows 5,000 requests/hour per authenticated user/app.

### Per-Cycle Breakdown

```
Typical cycle (5 min):
  Read state:        ~50 requests
  Spawn agents:      ~10 requests
  Update labels:     ~20 requests
  Merge PRs:         ~10 requests
  Comments:          ~10 requests
  ────────────────────
  Total:             ~100 requests

Busy cycle (all roles active):
  Read state:        ~50 requests
  Scout:             ~30 requests
  Triage:            ~20 per issue
  Builder:           ~100 requests (clone, push, PR)
  Reviewer:          ~50 requests
  Merger:            ~20 requests
  ────────────────────
  Total:             ~270 requests
```

At 12 cycles/hour: ~1,200-3,200 requests/hour. Well within 5,000 limit.

**If approaching limit:** Skip Scout (non-essential), skip metrics update, reduce cycle frequency.

## Launch Day KPI Report

**Date:** February 12, 2026

| KPI | Target | Actual |
|-----|--------|--------|
| PRs merged | 20 | **30** ✅ |
| Production downtime | 0 | **0** ✅ |
| Revert rate | < 5% | **0%** ✅ |
| Triage rejection rate | 30-50% | **~35%** ✅ |
| Batch merge speed | < 5 min | **2 min (14 PRs)** ✅ |

### Timeline

```
12:39 — v3 pipeline design started
14:00 — v3 coordinator launched
14:15 — First 3 PRs merged (#415, #420, #421)
15:00 — 17 PRs queued, bottleneck identified (strict: true)
15:54 — Switched to strict: false
15:56 — Batch merged 14 PRs in 2 minutes
16:00 — OSS best practices research confirmed our approach
17:00 — 30 total PRs merged, pipeline running smoothly
```
