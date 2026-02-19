# Lessons Learned

Hard-won lessons from V3 and V4. Each cost real time, tokens, or production uptime.

## Merge Strategy

- **`strict: true` kills throughput.** 17 queued PRs × 20 min each = 5.6 hours serial. Switched to `strict: false`, batch-merged 14 PRs in 2 minutes. No major OSS project uses `strict: true` at scale.
- **Post-merge monitoring > pre-merge serialization.** Small focused PRs rarely conflict semantically. Check production after merge, not before. Cost: ~0. Catch rate: ~99%.
- **Squash merge only.** AI commits are messy ("fix lint", "try again"). Squash collapses to one clean commit per PR.
- **Vercel stuck `pending` statuses block merges.** Override after 10 min: `gh api repos/{o}/{r}/statuses/{SHA} -f state=success -f context=Vercel`. Note: GitHub App tokens can't override third-party statuses — needs personal account.

## Coordinator

- **Coordinator is a reconciler, not an orchestrator.** It reads state, spawns missing agents, checks CI. It does NOT merge, review, assign work, or make product decisions.
- **Never let the coordinator merge directly.** PR #381: coordinator merged while CI was still running → TypeScript error in production → 4 hours of debugging. Merge belongs to the Review agent.
- **Coordinator creates rogue cron jobs.** Vague prompt ("keep the project healthy") → 3 cron jobs spawning sub-agents → $50 in API costs in 4 hours. Explicitly prohibit: `🚫 NEVER create cron jobs, scheduled tasks, or persistent processes`.
- **Idempotent operations, not locks.** Lock holder crashes → 10 min dead time. Instead: assigning already-assigned issue → skip. Merging already-merged PR → skip. Overlapping runs are harmless.

## Issue Quality

- **Triage prevents 30-50% of wasted work.** Without it, builders implement cosmetic changes, review rejection rate hit 60%+. With triage, first-pass approval rate went from ~40% to 70%+.
- **Duplicate detection at two levels.** Scout checks open AND recently closed issues before creating. Triage catches what Scout misses. Without both, same bug gets built 3 times.
- **Rejection rate sweet spot: 30-50%.** Below 20%: Scout too conservative or Triage too permissive. Above 70%: Scout too noisy.

## GitHub API Quirks

- **`reviewDecision` stays `CHANGES_REQUESTED` after new approvals.** GitHub doesn't auto-dismiss old reviews. Fix: explicitly dismiss stale reviews before approving:
  ```bash
  gh api repos/{o}/{r}/pulls/{N}/reviews \
    --jq '.[] | select(.state == "CHANGES_REQUESTED") | .id'
  gh api -X PUT repos/{o}/{r}/pulls/{N}/reviews/{ID}/dismissals \
    -f message="Superseded" -f event="DISMISS"
  ```
- **GitHub App tokens can't** read branch protection rules (403), override third-party statuses, or access repo settings. Use personal account for these.

## Agent Behavior

- **Agent PRs get rejected for predictable reasons.** Top causes (arXiv 2602.04226): too large (30%+), no added value, increased complexity, context limitations. Mitigations: triage gate, size labels (<500 lines), "follow existing patterns EXACTLY" in builder prompt.
- **Sub-agents hit token limits on raw HTML.** Agent fetched a webpage → 180KB HTML → 125K tokens → crash. Use structured search tools, extract text first, limit to 10KB.
- **Human-facing agent must delegate.** User asks "fix the login bug" → main agent starts debugging → blocks all other sessions for 3 min → hits token limit. Always spawn a sub-agent. Total time: 5 seconds.
- **Idle agents must exit immediately.** No busywork. Nothing to do → exit in under a minute. Explicitly state this in prompts.
- **Disabling a cron job doesn't stop in-flight runs.** Disable future runs AND kill the current instance separately.

## CI

- **Single required check: `build`.** More checks = more failure points. Lint as non-blocking (`continue-on-error: true`). Sentinel/post-merge CI catches the rest.
- **`--frozen-lockfile` in CI.** Agents accidentally modify lockfiles. Fail the build if lockfile doesn't match `package.json`.
- **`timeout-minutes: 10`.** AI-generated code creates infinite loops, build-time fetches to nonexistent URLs (30-min DNS timeout), recursive renders.
- **`cancel-in-progress` on PR CI.** Agent pushes a fix → old CI run is stale. Without cancellation: burned minutes, confusing pending status.

## Database

- **Parallel agents create conflicting migrations.** If using sequential migration files (e.g., Drizzle `0004_*.sql`), multiple builders collide. Fix: timestamp-based naming (`20260212_143022_*.sql`), declarative migrations (Atlas), or push-based development (`drizzle-kit push`).

## Meta

- **Write it down.** "Mental notes" don't survive agent restarts. Log rejections, failures, and patterns. Feed them back into prompts. Agents only learn from written history.
- **Keywords beat narratives in prompts.** Dense keyword clusters give AI better scope understanding than prose paragraphs. Minimum words, maximum coverage.

## Vercel + Turborepo

- **Turborepo monorepos without `turbo-ignore` burn build minutes at scale.** Every push triggers builds on ALL connected Vercel projects, even if only one app changed. With agents merging 20+ PRs/day: 300+ failed builds = 72 hours of wasted build minutes = ~$30/day. Fix: set `commandForIgnoringBuildStep: "npx turbo-ignore"` on every Vercel project. This is now a mandatory convention — see [Environments](environments.md).
- **Set `turbo-ignore` before you accumulate build history.** The cost compounds silently. You won't notice until you check the Vercel usage dashboard and see thousands of wasted build minutes. Do it when connecting a new project, not after.
- **`npx turbo-ignore` (no flags) is correct.** Earlier docs referenced `--fallback=HEAD~1`. That flag is unnecessary and was removed from turbo-ignore's default behavior. Use the plain command.

## Prompt Design

- **Principle-based prompts outperform rule-based prompts.** Rule-based prompts (keyword lists, "MUST do X", numerical quotas like "minimum 8 findings") get gamed: agents produce the required output regardless of quality. Principle-based prompts (a core question, three lenses, good/bad examples, self-argumentation) require genuine thought. The agent has to justify each output, which filters out noise and increases signal.
- **Numerical quotas produce exactly that many outputs — good or bad.** "File at least 8 issues per run" → 8 issues filed, including low-value theoretical findings that waste builder time. Remove quotas. Replace with: "only file what passes this specific bar."
- **Good/bad examples are more effective than anti-pattern lists.** Showing a concrete example of a great Scout issue and a concrete example of a bad one teaches more than listing 10 things to avoid. Examples are memorable; lists are not.
