# V4 Pipeline Architecture — Perpetual Motion Model

> 25 self-directed specialist roles (38 instances) continuously improving the codebase in parallel. No pipeline. No state machine. No handoffs. Just Git.

---

## Design Philosophy

V4 is a **perpetual motion machine** for software development. A fleet of 38 specialized AI agents work in parallel, each finding and completing work independently. The coordinator maintains fleet size. Git handles coordination. That's it.

**Core insight:** The cost of occasional duplicate work is far less than the cost of coordination overhead.

---

## Git-First Principles

GitHub is the **single source of truth** and the **only communication channel**. Agents never communicate directly.

| Artifact | Purpose | Created By | Consumed By |
|----------|---------|------------|-------------|
| **Issues** | Work to be done | Thinkers | Builders |
| **Pull Requests** | Proposed changes | All agents | Reviewers, Coordinator |
| **`main` branch** | Canonical state | Coordinator (merge) | All agents (clone fresh) |
| **Feature branches** | Isolated work | Each agent | That agent only |

No labels for state. No FSM. No webhooks. No orchestrator messages. Just Git primitives.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    COORDINATOR (cron, every 5 min)           │
│  Inventory → Spawn deficits → Merge approved → Health check │
└─────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌───────────────────────────────────────────────────────────────┐
│                     AGENT FLEET (38 instances)                │
│                                                               │
│  Product (3)         Engineering (9)       Quality (4)        │
│  ┌──────────────┐   ┌──────────────────┐  ┌──────────────┐  │
│  │ Prod Designer │   │ UI Builder ×3    │  │ Unit Tester×2│  │
│  │ Sys Architect │   │ API Builder ×3   │  │ E2E Tester ×2│  │
│  │ UX Designer   │   │ Fullstack Bldr×3 │  └──────────────┘  │
│  └──────────────┘   └──────────────────┘                     │
│                                                               │
│  Security (2)        Performance (3)      Code Health (3)    │
│  ┌──────────────┐   ┌──────────────────┐  ┌──────────────┐  │
│  │ Sec Auditor   │   │ FE Optimizer     │  │ Refactorer   │  │
│  │ Dep Manager   │   │ BE Optimizer     │  │ Type Hardener│  │
│  └──────────────┘   │ Mobile Optimizer │  │ Code Cleaner │  │
│                      └──────────────────┘  └──────────────┘  │
│                                                               │
│  Standards (2)       Growth (6)           Gates (5)          │
│  ┌──────────────┐   ┌──────────────────┐  ┌──────────────┐  │
│  │ Web Standards │   │ Growth Strat     │  │ Reviewer ×4  │  │
│  │ Doc Engineer  │   │ Growth Eng ×2    │  │ Sentinel     │  │
│  └──────────────┘   │ Content Crt ×2   │  └──────────────┘  │
│                      │ Analytics Eng    │                     │
│  Infrastructure (1)  └──────────────────┘                    │
│  ┌──────────────┐                                            │
│  │ CI/CD Eng     │                                           │
│  └──────────────┘                                            │
└───────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌───────────────────────────────────────────────────────────────┐
│              GITHUB (Single Source of Truth)                   │
│         Issues · Pull Requests · main branch                  │
└───────────────────────────────────────────────────────────────┘
```

### 10 Departments, 25 Roles, 38 Instances

| Department | Roles | Instances | Mode |
|------------|-------|-----------|------|
| Product | Product Designer, System Architect, UI/UX Designer | 3 | Thinker |
| Engineering | UI Builder (×3), API Builder (×3), Fullstack Builder (×3) | 9 | Task-driven |
| Quality | Unit Tester (×2), E2E Tester (×2) | 4 | Self-directed |
| Security | Security Auditor, Dependency Manager | 2 | Self-directed |
| Performance | Frontend Optimizer, Backend Optimizer, Mobile Optimizer | 3 | Self-directed |
| Code Health | Refactorer, Type Hardener, Code Cleaner | 3 | Self-directed |
| Standards | Web Standards Engineer, Documentation Engineer | 2 | Self-directed |
| Growth | Growth Strategist, Growth Engineer (×2), Content Creator (×2), Analytics Engineer | 6 | Mixed |
| Infrastructure | CI/CD Engineer | 1 | Self-directed |
| Gates | Code Reviewer (×4), Sentinel | 5 | Reviewer / Sentinel |

---

## Agent Modes

### Thinker
Product Designer, System Architect, UI/UX Designer, Growth Strategist. Analyze the product and codebase → create well-scoped Issues → exit. Never write code.

### Task-driven
UI Builder, API Builder, Fullstack Builder, Growth Engineer. Check for matching Issues first → claim and implement. No Issues? Fall back to self-directed analysis.

### Self-directed
All Quality, Security, Performance, Code Health, Standards, Infrastructure, Content Creator, Analytics Engineer. Analyze through their specialist lens → find one thing to improve → submit PR → exit. If nothing needs doing, exit immediately.

### Reviewer / Sentinel
Code Reviewer: reviews the oldest unreviewed PR. Sentinel: HTTP health check, creates revert PR if production breaks.

---

## The "No Waiting" Principle

All 38 agents work from minute one. No agent depends on another agent's output.

- **Issues are a bonus, not a prerequisite.** Builders find work independently when no Issues exist.
- **Dependencies are soft.** Missing an API endpoint? Stub it, ship the UI, connect later.
- **The system self-corrects.** Overlapping work gets caught by Reviewers. Inconsistencies get fixed by Refactorers. The codebase converges.
- **Idle agents exit.** No busywork. Nothing to do → gone in under a minute.

---

## Coordinator Reconciliation Loop

Runs every 5 minutes as a cron job. Stateless — each run is independent.

**Step 1: Inventory** — Count running agents per role.
**Step 2: Reconcile** — Spawn agents for any deficit (desired − running).
**Step 3: Auto-merge** — Squash-merge PRs that are approved + CI passing + not draft.
**Step 4: Health check** — HTTP GET to production. Sentinel handles remediation.
**Step 5: Report** — Log summary of agents, spawns, merges, health.

The coordinator does NOT assign work, manage state, track progress, or retry operations. It maintains fleet size and merges approved PRs. Everything else is the agents' job.

---

## Work Claiming

**Task-driven agents:** Query Issues by domain label → self-assign an unassigned Issue → implement → PR. GitHub's atomic assignment prevents race conditions.

**Self-directed agents:** Clone fresh, analyze through specialist lens, check open PRs to avoid duplicating in-flight work, find ONE thing, fix it, submit PR.

**Duplicate handling:** Two agents might fix similar things. Reviewers catch overlap. The second PR to merge hits a conflict — coordinator skips it, next agent rebases on fresh `main`.

---

## Conflict Handling

Git handles all conflict resolution natively. No custom coordination.

- Every agent works on a unique branch: `{role}/{description}`
- Conflicts only surface at merge time (never during development)
- Coordinator uses squash merge exclusively — clean linear history, atomic changes, easy reverts
- Conflicted PRs are skipped, not force-merged. The change eventually gets made on a fresh base.

This is **eventual consistency**. The system needs continuous improvement, not every individual PR to merge.

---

## Failure Modes

| Failure | Recovery |
|---------|----------|
| Agent crashes | Coordinator spawns replacement next cycle |
| Bad merge breaks production | Sentinel creates revert PR |
| GitHub API down | Agents exit with errors; next cycle retries |
| Two agents create conflicting PRs | Coordinator skips conflicted PR; next agent rebases |
| Agent creates broken PR | CI fails → never gets merged |
| Agent creates busywork | Reviewer rejects → blocks merge |

---

## System Invariants

1. **`main` is always deployable.** Only approved + CI-passing PRs get merged.
2. **No agent modifies `main` directly.** Everything goes through PRs.
3. **Agents are disposable.** Kill any agent at any time — all work is in Git.
4. **The coordinator is stateless.** Restart at any time, zero consequence.
5. **No agent blocks another.** Nothing to do → exit immediately.
6. **One PR per agent session.** One thing, done well.
7. **Git is the only communication.** No messages, no shared state, no databases.

---

## Scaling

### Current: 38 instances for early-to-mid stage projects

| To Scale | Do This |
|----------|---------|
| More features needed | Increase Builder counts |
| Review queue growing | Increase Reviewer count |
| Codebase > 50k lines | Shard agents by directory/domain |
| Multi-repo | Run separate V4 fleets per repo |
| Domain is clean | Reduce specialist counts |

Agents that find nothing to do exit immediately. Typical active concurrency is 10–20 agents, not 38.

---

## Comparison: V3 → V4

| Aspect | V3 (Pipeline) | V4 (Perpetual Motion) |
|--------|---------------|----------------------|
| Roles | 6 generalists | 25 specialists (38 instances) |
| Work discovery | Central backlog, assigned | Self-directed, pull-based |
| Sequencing | Phase gates (scout → merge) | All parallel, no waiting |
| Coordination | Label-based FSM | Git-native, stateless |
| State | Coordinator memory + labels | None — derived from Git |
| Failure recovery | Manual intervention | Automatic respawn |

---

*See also: [Agent Roles](agent-roles.md) · [Coordinator Guide](coordinator-guide.md) · [V3 to V4 Migration](v3-to-v4-migration.md)*
