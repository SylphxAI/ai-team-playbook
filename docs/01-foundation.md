# Foundation — Philosophy & Architecture

> The single most important decision: treat your AI pipeline like a Kubernetes controller.
> Reconciliation loops. Idempotent operations. No locks. GitHub as the single source of truth.

## The Kubernetes Controller Pattern

A Kubernetes controller works like this:

```
loop:
  1. Read desired state (what should exist)
  2. Read actual state (what does exist)
  3. Take action to reconcile the difference
  4. Repeat
```

Every iteration is **idempotent** — running it twice produces the same result. There are no locks, no mutexes, no "am I the only one running?" checks. If two controllers overlap, they both read the same state and take the same (or compatible) actions.

**Our AI pipeline works exactly the same way:**

```
loop (every 5 minutes):
  1. Read GitHub state (issues, PRs, labels, CI status)
  2. Determine what needs to happen (triage, build, review, merge)
  3. Spawn agents for outstanding work
  4. Repeat
```

If two coordinator cycles overlap, they both check conditions and only act if needed. Duplicate work is prevented by GitHub's atomic operations (issue assignment, merge) and session label checks.

### Why This Matters

Most people's first instinct is to build a lock-based system:

```
❌ WRONG: Lock-based coordination
1. Acquire lock
2. Read state
3. Make decisions
4. Execute actions
5. Release lock
```

This creates a single point of failure. If the lock holder crashes, everything stops. If the lock TTL is too short, you get overlapping runs anyway. If it's too long, you waste time waiting.

**The right approach: make every operation idempotent.**

- Assigning an issue? If it's already assigned, skip it.
- Spawning a builder? If a builder session already exists, skip it.
- Merging a PR? If it's already merged, skip it.
- Adding a label? If it's already there, GitHub is a no-op.

No locks needed. Overlapping runs are harmless.

## GitHub as Single Source of Truth

**There are no local state files.** No databases. No Redis. No message queues. GitHub IS the state store:

| Concept | GitHub Implementation |
|---------|---------------------|
| Task queue | Open issues with `pipeline/approved` label |
| FSM state | Labels (`pipeline/discovered`, `pipeline/in-progress`, etc.) |
| Work assignment | Issue assignees |
| Build status | GitHub Actions check runs |
| Review status | PR review decisions |
| Merge queue | PRs with `pipeline/review-passed` label |
| Metrics dashboard | Pinned issue updated each cycle |
| Coordinator lock | Pinned issue with lease timestamp |

### Why GitHub?

1. **No sync issues** — there's only one copy of the truth
2. **Built-in concurrency** — issue assignment is atomic, PR merge is atomic
3. **Full audit trail** — every label change, comment, and status update is logged
4. **Humans can participate** — anyone can add labels, comment, or override decisions
5. **API-friendly** — everything is queryable via `gh` CLI or REST API
6. **Free** — for public repos, unlimited API calls (5,000/hour rate limit)

### Labels as Finite State Machine

Each issue moves through states via label transitions:

```
pipeline/discovered → pipeline/approved   (Triage approves)
pipeline/discovered → pipeline/rejected   (Triage rejects)
pipeline/approved   → pipeline/in-progress (Builder starts)
pipeline/in-progress → pipeline/pr-open   (PR created)
pipeline/pr-open    → pipeline/review-passed (Reviewer approves)
pipeline/pr-open    → pipeline/review-failed (Reviewer rejects)
pipeline/review-passed → pipeline/merged  (Merger merges)
```

Labels are mutually exclusive per pipeline state. An issue should never have both `pipeline/approved` and `pipeline/in-progress` — the transition replaces one with the other.

## Poll-Driven Coordination + Event-Driven CI

The system uses two complementary patterns:

**Poll-driven (coordinator):** Every 5 minutes, read all GitHub state and reconcile. Simple, no webhook infrastructure needed, naturally rate-limited, coordinator can reason about global state.

**Event-driven (CI only):** GitHub Actions triggers on PR push/open for build/test. This is what Actions is designed for — reacting to code events.

### Why Not Webhooks?

Webhooks add complexity:
- Webhook infrastructure to maintain
- Retry logic for failed deliveries
- Authentication and verification
- Can be overwhelmed by event storms (e.g., bulk issue creation)

Polling every 5 minutes has a 0-5 minute delay, which is acceptable for autonomous development. The CI system (GitHub Actions) handles the latency-sensitive part — running builds immediately when code is pushed.

**Future optimization:** Add webhook → coordinator endpoint for P0 events (production health alerts) where 5-minute latency is too much.

## The Non-Negotiable Checklist

Before any agent writes code, you need:

- [x] **Branch protection on `main`**: CI pass + 1 approval required
- [x] **`strict: false`**: Branches do NOT need to be up-to-date (see [Merge Safety](03-merge-safety.md))
- [x] **Squash merge only**: Clean, linear history
- [x] **Auto-delete head branches**: Prevent branch accumulation
- [x] **Required check: `build`**: Single required CI check
- [x] **Lint as non-blocking**: `continue-on-error: true` (see [CI Pipeline](05-ci-pipeline.md))
- [x] **Pipeline labels created**: All 26 labels for FSM state tracking
- [x] **Metrics issue pinned**: Dashboard updated each coordinator cycle
- [x] **GitHub Apps configured**: Separate builder and reviewer identities

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub (Source of Truth)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │  Issues   │  │  Labels  │  │   PRs    │  │ Actions (CI)  │   │
│  │ (tasks)   │  │ (state)  │  │ (code)   │  │ (validation)  │   │
│  └─────┬─────┘  └─────┬────┘  └─────┬────┘  └───────┬───────┘   │
└────────┼───────────────┼─────────────┼───────────────┼──────────┘
         │               │             │               │
         ▼               ▼             ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│              Coordinator (Reconciliation Loop — 5 min)           │
│                                                                  │
│  1. Inventory     → Read all GitHub state                        │
│  2. Timeouts      → Heal stuck work                              │
│  3. Sentinel      → Check production health                      │
│  4. Merge         → Merge approved PRs (batch)                   │
│  5. Review        → Spawn reviewers for open PRs                 │
│  6. Fix           → Spawn fixers for rejected reviews            │
│  7. Rebase        → Handle merge conflicts                       │
│  8. Build         → Spawn builders for approved issues           │
│  9. Triage        → Spawn triage for discovered issues           │
│ 10. Scout         → Discover new work (if backlog low)           │
│ 11. Status        → Update metrics dashboard                     │
│                                                                  │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────┐     │
│  │ Scout  │ │Triage  │ │Builder │ │ Reviewer │ │ Merger │     │
│  └────────┘ └────────┘ └────────┘ └──────────┘ └────────┘     │
│  ┌──────────┐                                                   │
│  │ Sentinel │                                                   │
│  └──────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **GitHub as sole state store** | Single source of truth; no sync issues; leverages built-in concurrency |
| **Labels over project boards** | Labels are API-friendly, queryable, and machine-readable |
| **External orchestration (cron) over GitHub Actions** | Coordinator needs LLM reasoning for triage decisions — can't run in Actions |
| **Optimistic concurrency** | Like open source: assign, branch, PR. Conflicts resolved naturally |
| **FSM-based lifecycle** | Explicit states with transition guards prevent invalid operations |
| **Poll-driven coordination** | Simpler than webhooks, naturally rate-limited, good enough latency |
| **`strict: false`** | 10x throughput improvement; Sentinel replaces strict mode safety |
| **Single coordinator** | Two coordinators = duplicate spawns, conflicting decisions |

## Common Mistakes

### "We'll add triage later"
No. Triage is the #1 missing piece. Without it, your Scout creates issues that waste 30-50% of builder time. Add it from day 1.

### "We need locks to prevent duplicate work"
No. Make every operation idempotent. Check-then-act with GitHub's atomic operations. Overlapping coordinator runs should produce the same result.

### "Let's use `strict: true` for safety"
No. It serializes merging to ~20 min/PR. Use `strict: false` + Sentinel post-merge monitoring. See [Merge Safety](03-merge-safety.md) for the full argument with OSS research.

### "The coordinator should be smart"
No. The coordinator should be a dumb dispatcher. It reads state, checks conditions, spawns agents. The moment it starts "reasoning" about code or making architectural decisions, it breaks.
