# V3 → V4 Migration

## What Changed

| Aspect | V3 | V4 |
|--------|----|----|
| Model | Assembly line (Scout → Triage → Builder → Reviewer → Merger) | Perpetual motion (all agents parallel) |
| Roles | 6 pipeline roles | 25 specialized roles, 38 agents |
| State | FSM labels on GitHub issues | None — Git is the only state |
| Coordination | Orchestrator assigns work per stage | Coordinator only fills headcount + merges |
| Dependencies | Implicit (design → build → test → review) | None between agent types |
| Waiting | Common (agents idle waiting for upstream) | None — everyone self-directs |
| Prompts | Detailed step-by-step scripts | 1-2 sentence professional briefs |
| Prompt size | 11KB coordinator + per-agent templates | 7KB total |

## Why

V3's assembly line serialized work through stages. Most agents were idle waiting for upstream. V4 removes all dependencies — every agent finds its own work and submits PRs independently. The system self-corrects through continuous refactoring and review.

## Key Insight

Assembly lines are wrong for Git. OSS projects prove that independent specialists + strong review > centralized orchestration.
