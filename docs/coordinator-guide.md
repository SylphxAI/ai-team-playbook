# Coordinator Guide

Stateless reconciliation loop. Every 5 minutes:

1. Count running `v4-*` agents
2. Spawn deficit to fill roster (15 roles, 30 agents)
3. Merge approved PRs with passing CI
4. Health check tryit.fun
5. Report status

Prompt is ~5KB. Each agent brief is 1-2 sentences.
