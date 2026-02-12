# Coordinator Guide — V4

The coordinator is a stateless reconciliation loop. Runs every 5 minutes. Does three things:

1. **Maintains the fleet** — ensures the right number of agents are running
2. **Merges approved PRs** — auto-merges PRs that pass CI and review
3. **Reports health** — checks production and logs a summary

## Algorithm

```
Every 5 minutes:
  1. sessions_list → count v4-* agents by role
  2. For each role: if running < desired → spawn deficit
  3. gh pr list → find approved PRs with passing CI → gh pr merge --squash
  4. curl https://tryit.fun → health check
  5. Print summary
```

If any step fails, log and continue. Never crash the cycle.

## Properties

- **Stateless**: No memory between runs. Reads all state from sessions + GitHub each cycle.
- **Idempotent**: Running twice has no harmful effect. At-capacity roles spawn nothing.
- **Fast**: ~7KB prompt, executes in <1 minute. 5-minute interval gives 4-minute buffer.

## Prompt Philosophy

The coordinator prompt is 7KB, not 74KB. Each agent brief is 1-2 sentences. AI knows how to clone repos, create branches, and submit PRs. Just tell it WHAT to do.

## Scaling

- Need more throughput? Increase role count in the roster.
- Need a new specialty? Add a role with a brief description.
- Role has no work? Set count to 0 or merge with a related role.
