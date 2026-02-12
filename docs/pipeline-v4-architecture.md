# V4 Pipeline Architecture — Perpetual Motion Model

> From sequential pipeline to autonomous swarm. V4 replaces orchestrated handoffs with 28 self-directed specialists that continuously improve the codebase in parallel.

---

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Git-First Principles](#git-first-principles)
- [Architecture Overview](#architecture-overview)
- [Agent Modes](#agent-modes)
- [The "No Waiting" Principle](#the-no-waiting-principle)
- [Coordinator Reconciliation Loop](#coordinator-reconciliation-loop)
- [Work Claiming Mechanism](#work-claiming-mechanism)
- [Feature Decomposition](#feature-decomposition)
- [Conflict Handling via Git](#conflict-handling-via-git)
- [Failure Modes and Recovery](#failure-modes-and-recovery)
- [System Invariants](#system-invariants)
- [Scaling Guide](#scaling-guide)
- [Comparison with Previous Approaches](#comparison-with-previous-approaches)

---

## Design Philosophy

V4 is a **perpetual motion machine** for software development. It maintains a fleet of 28 specialized AI agents (32 instances, accounting for scaled roles) that continuously improve the codebase. There is no pipeline. No assembly line. No sequential handoffs.

Every agent independently:

1. Finds work in its domain
2. Does the work
3. Submits a PR
4. Exits

The coordinator simply ensures the desired fleet size is maintained. It doesn't assign work, manage state, or orchestrate sequences.

**Core insight:** The cost of occasional duplicate work is far less than the cost of coordination overhead. Let agents work in parallel and let Git handle the rest.

---

## Git-First Principles

V4 treats GitHub as the **single source of truth** and the **only communication channel**.

### No Pipeline, No FSM, No Handoffs

| What V3 Used | What V4 Uses |
|--------------|--------------|
| Label-based FSM (`pipeline/discovered` → `pipeline/building` → ...) | Nothing. No pipeline state. |
| Sequential phase gates (scout → triage → build → review → merge) | All agents work in parallel |
| Coordinator assigns work to specific agents | Agents find their own work |
| Webhook-triggered state transitions | Stateless cron reconciliation |

### Git Artifacts as Communication

Agents **never communicate directly** with each other. All coordination flows through Git:

| Artifact | Purpose | Created By | Consumed By |
|----------|---------|------------|-------------|
| **Issues** | Work to be done | Thinkers (Product, Architecture, UX) | Builders (UI, API, DB, Infra) |
| **Pull Requests** | Proposed changes | All agents | Reviewers, Coordinator |
| **`main` branch** | Canonical project state | Coordinator (via merge) | All agents (clone fresh) |
| **Feature branches** | Isolated workspaces | Each agent | That agent only |

### Why Git-First Works

- **Perfect isolation.** Every agent works on its own branch. Parallel work never conflicts during development.
- **Atomic changes.** Each PR is one logical change. Squash merge means one commit per PR on `main`.
- **Built-in conflict detection.** Git tells you when two changes overlap. No custom coordination needed.
- **Auditability.** Every change is a PR with review history, CI results, and a clear description.
- **Recovery.** Every commit can be reverted. Every branch can be abandoned. Nothing is permanent until merged.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    COORDINATOR (cron, every 5 min)           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Inventory │→ │  Spawn   │→ │  Merge   │→ │  Health  │   │
│  │  agents   │  │ deficits │  │ approved │  │  check   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌───────────────────────────────────────────────────────────────┐
│                     AGENT FLEET (32 instances)                │
│                                                               │
│  Product (3)         Engineering (6)       Quality (3)        │
│  ┌──────────────┐   ┌─────────────────┐   ┌──────────────┐  │
│  │ Prod Designer │   │ UI Builder ×2   │   │ Unit Tester  │  │
│  │ Sys Architect │   │ API Builder ×2  │   │ Integ Tester │  │
│  │ UX Designer   │   │ DB Engineer     │   │ E2E Tester   │  │
│  └──────────────┘   │ Infra Builder   │   └──────────────┘  │
│                      └─────────────────┘                      │
│  Security (2)        Performance (2)       Code Health (4)   │
│  ┌──────────────┐   ┌─────────────────┐   ┌──────────────┐  │
│  │ Sec Auditor   │   │ FE Optimizer    │   │ Refactorer   │  │
│  │ Dep Auditor   │   │ BE Optimizer    │   │ Type Hardener│  │
│  └──────────────┘   └─────────────────┘   │ Dead Code    │  │
│                                            │ Error Handler│  │
│  Maintenance (2)     Standards (5)         └──────────────┘  │
│  ┌──────────────┐   ┌─────────────────┐                      │
│  │ Dep Migrator  │   │ Tech Writer     │   Gates (4)         │
│  │ Dep Updater   │   │ API Documenter  │   ┌──────────────┐  │
│  └──────────────┘   │ SEO Engineer    │   │ Reviewer ×3  │  │
│                      │ A11y Engineer   │   │ Sentinel     │  │
│  Infrastructure (1)  │ i18n Engineer   │   └──────────────┘  │
│  ┌──────────────┐   └─────────────────┘                      │
│  │ CI/CD Eng     │                                            │
│  └──────────────┘                                            │
└───────────────────────────────────────────────────────────────┘
        ↕                ↕                ↕
┌───────────────────────────────────────────────────────────────┐
│                    GITHUB (Single Source of Truth)             │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────┐    │
│  │   Issues    │  │    PRs     │  │  main branch (prod) │    │
│  │ (work items)│  │ (changes)  │  │  (auto-deployed)    │    │
│  └────────────┘  └────────────┘  └─────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

### 10 Departments, 28 Roles, 32 Instances

| Department | Roles | Instances | Mode |
|------------|-------|-----------|------|
| Product | Product Designer, System Architect, UI/UX Designer | 3 | Thinker |
| Engineering | UI Builder, API Builder, Database Engineer, Infra Builder | 6 | Task-driven |
| Quality | Unit Tester, Integration Tester, E2E Tester | 3 | Self-directed |
| Security | Security Auditor, Dependency Auditor | 2 | Self-directed |
| Performance | Frontend Optimizer, Backend Optimizer | 2 | Self-directed |
| Code Health | Refactorer, Type Hardener, Dead Code Cleaner, Error Handler | 4 | Self-directed |
| Maintenance | Deprecation Migrator, Dependency Updater | 2 | Self-directed |
| Standards | Technical Writer, API Documenter, SEO Engineer, A11y Engineer, i18n Engineer | 5 | Self-directed |
| Infrastructure | CI/CD Engineer | 1 | Self-directed |
| Gates | Code Reviewer (×3), Sentinel | 4 | Reviewer / Sentinel |

---

## Agent Modes

Each agent operates in one of four modes that determine how it discovers and executes work.

### 1. Thinker Mode

**Agents:** Product Designer, System Architect, UI/UX Designer

Thinkers analyze the codebase and product from their specialist perspective, then create GitHub Issues. They **never write code**. Their output is the demand signal that feeds Task-driven agents.

**Lifecycle:**
1. Clone repo, explore codebase and live product
2. Check existing open Issues to avoid duplicates
3. Create 1–3 well-scoped, independently implementable Issues
4. Exit

**Key property:** The system works even if Thinkers create zero Issues. Task-driven agents fall back to self-directed mode.

### 2. Task-driven Mode

**Agents:** UI Builder, API Builder, Database Engineer, Infra Builder

Task-driven agents **first** check for open Issues matching their specialty. If they find an unassigned Issue, they claim it and implement it. If no Issues exist, they fall back to self-directed analysis.

**Lifecycle:**
1. Clone repo
2. Query Issues matching their domain labels
3. If unassigned Issue found → self-assign, implement, submit PR
4. If no Issues → analyze codebase, find work, submit PR
5. Exit

**Key property:** Builders always find work, whether Issues exist or not.

### 3. Self-directed Mode

**Agents:** All Quality, Security, Performance, Code Health, Maintenance, Standards, and Infrastructure agents

Self-directed agents analyze the codebase through their specialist lens, find one thing to improve, and fix it. They don't need Issues — the codebase itself is their work queue.

**Lifecycle:**
1. Clone repo
2. Analyze codebase through specialist lens (e.g., security auditor scans for vulnerabilities)
3. Check open PRs to avoid duplicating in-flight work
4. Find ONE thing to improve
5. If nothing to improve → exit immediately (no busywork)
6. Implement fix, submit PR
7. Exit

**Key property:** If there's genuinely nothing to do in their domain, they exit instantly. No manufactured work.

### 4. Reviewer / Sentinel Mode

**Code Reviewer (×3):** Reviews open PRs for quality, correctness, and standards. Approves good PRs, requests changes on problematic ones. If no PRs need review, exits immediately.

**Sentinel (×1):** Monitors production health via HTTP checks. If production is down and a recent merge is the likely cause, creates a revert PR. If production is healthy, exits.

---

## The "No Waiting" Principle

Traditional multi-agent systems suffer from pipeline bottlenecks: Agent A waits for Agent B's output before starting. V4 eliminates this entirely.

### How It Works

1. **All 32 agents start simultaneously.** Every agent is finding work or doing work from minute one.

2. **No agent depends on another agent's output.** Each agent operates on the current state of `main`. Whatever exists right now is the starting point.

3. **Issues are a bonus, not a prerequisite.** Thinkers create Issues, Builders consume them — but if no Issues exist, Builders find work independently.

4. **Design follows implementation.** An architect might create an Issue proposing a data model change. Simultaneously, a builder might implement something that creates a different data model. The refactorer later aligns them. The system self-corrects.

5. **Dependencies are soft.** If a UI Builder needs an API endpoint that doesn't exist:
   - Stub it with mock data
   - Build the UI with the stub
   - Submit the PR
   - Later, an API Builder creates the endpoint, and a future UI Builder connects them

### Why This Works

- **Git provides perfect isolation.** Parallel agents never conflict during development.
- **Small PRs resolve quickly.** Each PR is one logical change. Merge conflicts are rare and small.
- **The system converges.** Continuous refactoring, type-hardening, and review cause the codebase to converge toward consistency — even if individual agents make slightly inconsistent choices.
- **Idle agents exit.** No busywork. No spinning. If nothing needs doing, the agent is gone in under a minute.

---

## Coordinator Reconciliation Loop

The coordinator runs as a **cron job every 5 minutes**. It is stateless — each run is independent. It is a reconciliation loop, not an orchestrator.

### The 5-Step Algorithm

```
Step 1: INVENTORY
  └─ List all running agent sessions (label filter: "v4-")
  └─ Count running instances per role

Step 2: RECONCILE
  └─ For each role in ROSTER:
       deficit = desired_count - running_count
       Spawn {deficit} new agents for that role

Step 3: AUTO-MERGE
  └─ List open PRs (limit 50)
  └─ For each PR that is: approved + CI passing + not draft
       Attempt squash merge
       If conflict → skip, continue

Step 4: HEALTH CHECK
  └─ HTTP GET to production URL
  └─ Log status code
  └─ If unhealthy → note in report (Sentinel handles remediation)

Step 5: STATUS REPORT
  └─ Print summary: agents running, spawned, PRs merged, health status
```

### Key Properties

| Property | What It Means |
|----------|---------------|
| **Idempotent** | Running twice produces the same result. At-capacity roles spawn nothing. |
| **Crash-safe** | If the coordinator crashes, the next 5-minute run picks up seamlessly. |
| **Self-healing** | Crashed agents are detected as missing and respawned next cycle. |
| **Stateless** | No database, no state file, no memory between runs. Everything derived from Git + session state. |

### What the Coordinator Does NOT Do

- Assign work to specific agents
- Track which agent is working on what
- Manage pipeline state or transitions
- Store any data between runs
- Retry failed operations (next cycle handles it)

---

## Work Claiming Mechanism

### Task-Driven Agents (Builders)

1. Query open Issues matching their domain label (`ui`, `api`, `db`, `infra`)
2. Find an **unassigned** Issue
3. Self-assign immediately: `gh issue edit {N} --add-assignee shtse8`
4. Implement the Issue
5. Submit a PR referencing the Issue (`Closes #N`)

**Race safety:** GitHub's Issue assignment is atomic. If two agents try to claim the same Issue, the first one to assign wins. The second agent sees it's assigned and picks a different Issue.

### Self-Directed Agents

No claiming needed. They:
1. Clone the repo fresh
2. Analyze the codebase through their specialist lens
3. Check open PRs to avoid duplicating in-flight work
4. Find ONE thing to improve
5. Implement and submit a PR

**Duplicate safety:** Two agents might independently fix similar things. Git handles this naturally — the second PR to merge will have a conflict, the coordinator skips it, and the next cycle's agent rebases.

---

## Feature Decomposition

Complex features decompose naturally without central coordination.

### Example: "Add User Profile Page"

| Step | Agent | Action | Output |
|------|-------|--------|--------|
| 1 | Product Designer | Creates Issue: "Add user profile with avatar, bio, activity" | Issue #42 |
| 2 | System Architect | Creates Issue: "Add profile fields to User model" | Issue #43 |
| 3 | UI/UX Designer | Creates Issue: "Profile page component structure" | Issue #44 |
| 4 | Database Engineer | Claims Issue #43, adds schema fields | PR #50 |
| 5 | API Builder | Creates `/api/profile/[id]` endpoint | PR #51 |
| 6 | UI Builder | Claims Issue #44, builds component (stubs API if needed) | PR #52 |
| 7 | Code Reviewer | Reviews each PR independently | Approvals |
| 8 | Unit Tester | Writes tests for profile service logic | PR #55 |
| 9 | A11y Engineer | Audits new profile page | PR #56 |
| 10 | SEO Engineer | Adds meta tags for profile pages | PR #57 |

Each step is a separate PR. Each is independently mergeable. The order doesn't matter. If the UI PR merges before the API PR, the profile page ships with stubs — and gets connected in the next cycle.

### Decomposition Rules

- Each Issue should be completable in a single agent session
- Each PR should be independently mergeable (no "Part 1 of 5" chains)
- If an agent encounters a dependency, they build what they can and note the dependency in their PR body
- Multiple agents may independently create overlapping solutions — Reviewers catch this and request consolidation

---

## Conflict Handling via Git

V4 relies entirely on Git's native conflict resolution. No custom coordination needed.

### Branch Isolation

Every agent works on a unique branch: `{role}/{description}`. Branches are short-lived (one PR per branch). This means:

- Agents never interfere with each other during development
- Conflicts only surface at merge time
- Conflicts are between one PR and `main`, never between two PRs

### Merge Strategy

The coordinator uses **squash merging** exclusively:

```bash
gh pr merge {number} --squash --delete-branch
```

| Benefit | Why |
|---------|-----|
| Clean linear history | One commit per PR on `main` |
| Atomic changes | Each merge is a single commit |
| Easy reverts | One commit to revert per change |
| Auto-cleanup | Branch deleted after merge |

### Conflict Resolution Flow

1. PR passes review and CI ✅
2. Coordinator attempts squash merge
3. **Success** → branch deleted, done
4. **Conflict** → coordinator skips it, continues to next PR
5. In the next cycle, that role spawns a new agent
6. New agent builds on current `main` (which now includes whatever caused the conflict)
7. The change eventually gets made on a fresh base

This is **eventual consistency**. The system doesn't need every PR to merge — it needs the codebase to continuously improve.

### Overlapping Work

Two agents might submit PRs that do similar things. This is handled by:

1. **Code Reviewers** catch overlap and request changes on the less complete PR
2. **Dead Code Cleaner** removes any duplicated code that slips through
3. **Refactorer** consolidates similar patterns

---

## Failure Modes and Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Agent crashes mid-work | Not in sessions list next cycle | Coordinator spawns replacement; partial branch is abandoned |
| Bad PR merged, production breaks | Sentinel health check fails | Sentinel creates revert PR; Reviewers fast-track approve |
| GitHub API down | All agents fail to clone/push | Agents exit with errors; next cycle retries |
| All Reviewers crash | PR queue grows | Coordinator spawns 3 new Reviewers next cycle |
| Coordinator crashes | Cron skips one cycle | Next cron run resumes normally (stateless) |
| Two agents create conflicting PRs | Merge conflict on second PR | Coordinator skips conflicted PR; next agent rebases |
| Agent creates a broken PR | CI fails | PR never gets merged (CI pass required) |
| Agent creates busywork PR | Reviewer rejects it | REQUEST_CHANGES blocks merge |

---

## System Invariants

These properties must always hold:

1. **`main` is always deployable.** Only PRs with passing CI and review approval get merged.
2. **No agent modifies `main` directly.** All changes go through feature branches and PRs.
3. **Agents are disposable.** Any agent can be killed at any time without data loss. All work is in Git.
4. **The coordinator is stateless.** It can be restarted at any time without consequence.
5. **No agent blocks another.** If there's nothing to do, exit. Never wait.
6. **One PR per agent session.** Agents do one thing, do it well, submit, and exit.
7. **Small PRs only.** Each PR is one logical, focused change. Large PRs get rejected by Reviewers.
8. **Git is the only communication channel.** No messages, no shared state, no databases. Just Issues, PRs, and branches.

---

## Scaling Guide

### Current Scale: 32 Instances

Designed for an early-to-mid stage project where:
- The codebase has room for improvement in every domain
- PRs are small enough to review quickly
- CI can handle 10–20 concurrent PR checks

### Scaling Up

| Lever | How | When |
|-------|-----|------|
| More Builders | Increase UI Builder / API Builder counts | Feature backlog grows faster than builders can consume |
| More Reviewers | Increase Code Reviewer count | PR review queue exceeds 10 PRs |
| Specialized Builders | Split "UI Builder" into "Form Builder", "Layout Builder" | Component types become highly specialized |
| Domain Sharding | Separate agents for `src/auth/`, `src/feed/`, etc. | Codebase exceeds 50k lines |
| Multi-repo | Run separate V4 fleets per repository | Project splits into microservices |

### Scaling Down

| Lever | How | When |
|-------|-----|------|
| Reduce Quality | Drop to 1 tester instead of 3 | Test coverage is high and stable |
| Reduce Code Health | Remove Dead Code Cleaner, Deprecation Migrator | Codebase is clean |
| Reduce Standards | Remove i18n, SEO if not needed | Single-language internal tool |
| Reduce Reviewers | 1 instead of 3 | PR volume drops below 5/day |

### Resource Efficiency

Agents that find nothing to do **exit immediately**. Typical active concurrency is 5–15 agents, not 32:

| Agent Type | Typical Session Duration |
|------------|------------------------|
| Thinkers | 2–5 minutes |
| Builders | 5–15 minutes |
| Self-directed specialists | 3–10 minutes |
| Reviewers | 2–5 minutes |
| Sentinel | ~1 minute |

### API Rate Limiting

GitHub allows 5,000 API requests/hour for authenticated users. Real-world usage:

- Each agent uses ~5–10 API-heavy operations (list PRs, create PR, etc.)
- Git clone/push uses Git protocol, not REST API
- With staggered spawning and fast exits, actual throughput is ~500–800 API calls per cycle
- Well within the 5,000/hour limit

---

## Comparison with Previous Approaches

| Aspect | V3 (Pipeline) | V4 (Perpetual Motion) |
|--------|---------------|----------------------|
| **Work discovery** | Central backlog, assigned by coordinator | Self-directed, pull-based |
| **Sequencing** | Phase gates: scout → triage → build → review → merge | None — all parallel |
| **Agent count** | 6 roles | 28 roles (32 instances) |
| **Coordination** | Orchestrator manages FSM state via labels | Git-native, stateless reconciliation |
| **Blocking** | Common — "waiting for triage", "waiting for review" | Impossible — build what you can now |
| **Failure recovery** | Manual intervention or complex retry logic | Automatic respawn next cycle |
| **Scaling** | Reconfigure pipeline stages | Change ROSTER counts |
| **Communication** | Labels, state transitions, coordinator messages | Issues and PRs only |
| **State management** | Label-based FSM, coordinator memory | None — derive from Git state |
| **Specialization** | Generalist agents (Builder does everything) | Specialist agents (28 distinct focuses) |

---

*See also: [Agent Roles](agent-roles.md) · [Coordinator Guide](coordinator-guide.md) · [V3 to V4 Migration](v3-to-v4-migration.md)*
