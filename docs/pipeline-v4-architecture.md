# V4 Pipeline Architecture

Perpetual-motion system. No pipeline. No stages. 30 autonomous agents, all contributing through Git.

## Principles

1. **Git-first.** PRs and issues only. No custom state.
2. **Nobody waits.** All agents parallel. Issues are guidance, not prerequisites.
3. **AI is fullstack.** Split by cognitive mode, not technical domain.
4. **Split by repetition.** Doing X repeatedly covers Y too? Merge. Otherwise separate.
5. **Tell WHAT, not HOW.** 1-2 sentence briefs.

## Distribution

| Category | Agents | % |
|----------|--------|---|
| Builders | 12 | 40% |
| Specialists | 6 | 20% |
| Quality | 5 | 17% |
| Gates | 5 | 17% |
| Thinkers | 2 | 7% |

Coordinator runs every 5 min: count agents → spawn deficit → merge approved PRs → health check.
