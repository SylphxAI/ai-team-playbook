# V4 Pipeline Architecture

A perpetual-motion software development system. No pipeline. No stages. No state machine. 38 autonomous agents, each running in isolation, all contributing through Git.

## Core Principles

1. **Git-first.** Agents communicate through PRs and issues. No custom state tracking.
2. **Nobody waits.** All agents run simultaneously. Issues are guidance, not prerequisites.
3. **Design follows implementation.** Build first, refactor later. The system self-corrects.
4. **Idempotent.** Parallel agents never conflict destructively. Git handles isolation.
5. **Workload-based roles.** Split by how much work exists, not by domain boundaries.

## Architecture

```
COORDINATOR (cron, every 5 min)
  1. Count running agents
  2. Spawn missing agents to fill roster
  3. Merge approved PRs with passing CI
  4. Health check production
  5. Report status

38 AGENTS (isolated, autonomous, ephemeral)
  → PRs / Issues → GitHub → auto-deploy → tryit.fun
```

## 10 Departments, 25 Roles, 38 Agents

| Department | Agents | Roles |
|-----------|--------|-------|
| Product | 5 | Product Designer, System Architect, UI/UX Designer, Growth Strategist, Content Creator |
| Engineering | 12 | UI Builder ×3, API Builder ×3, Fullstack Builder ×2, DB Engineer, Infra Builder, Growth Engineer, Analytics Engineer |
| Quality | 5 | Unit Tester ×2, Integration Tester, E2E Tester ×2 |
| Security | 1 | Security Auditor |
| Performance | 3 | Frontend Optimizer, Backend Optimizer, Mobile Optimizer |
| Code Health | 3 | Refactorer, Type Hardener, Code Cleaner |
| Maintenance | 1 | Dependency Manager |
| Standards | 2 | Documentation Engineer, Web Standards Engineer |
| Infrastructure | 1 | CI/CD Engineer |
| Gates | 5 | Code Reviewer ×4, Sentinel |

## How It Works

Every agent is a professional. Spawn it with a 1-2 sentence brief. It clones the repo, finds work through its lens, does one focused thing, submits a PR, exits. The coordinator respawns it next cycle.

No handoffs. No assembly line. No waiting. Just Git.

## Complex Features

Thinkers (Product Designer, System Architect, UI/UX Designer) create well-decomposed issues. Each issue is independently implementable. Builders self-serve — claim unassigned issues, implement, PR. Multiple builders work in parallel on different subtasks.

## Quality

- Code Reviewer ×4 as the primary quality gate
- CI must pass before merge
- Sentinel monitors production, auto-reverts bad deploys
- Small focused PRs limit blast radius

## Evolution

V1: Single generalist agent → V2: Pipeline with roles → V3: FSM-based assembly line → **V4: Git-first perpetual motion**

The key insight: assembly lines are wrong for Git. OSS projects with thousands of contributors prove that independent specialists + strong review > centralized orchestration.
