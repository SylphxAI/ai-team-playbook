# Lessons Learned — A Living Document

> Every lesson here cost us time, tokens, or production uptime.
> This is the most important document in the playbook.
> Read it before you start. Come back when something breaks.

---

## 1. `strict: true` Kills Throughput

**What happened:** With `strict: true` branch protection and 17 queued PRs, each PR took ~20 minutes (rebase → CI → merge). Sequential processing: 17 × 20 min = 5.6 hours to clear the queue.

**The fix:** Changed to `strict: false`. Batch-merged 14 PRs in 2 minutes. Sentinel verified production stayed healthy.

**Research:** No major OSS project (Kubernetes, React, Next.js, Rust, GitHub) uses `strict: true` at scale. See [Merge Safety](03-merge-safety.md).

**Rule:** Use `strict: false` + Sentinel post-merge monitoring. Reserve `strict: true` for small teams with < 5 PRs/day.

---

## 2. Lock Mechanisms Are an Anti-Pattern

**What happened:** Our initial coordinator design had a lock step (acquire lease via GitHub issue, 10-min TTL). When the coordinator crashed, everything stopped for 10 minutes. When TTL was too short, overlapping runs happened anyway.

**The fix:** Removed the lock entirely. Made every operation idempotent:
- Assigning already-assigned issue → skip
- Spawning when session exists → skip
- Merging already-merged PR → skip
- Adding existing label → no-op

**Rule:** Don't prevent duplicate operations. Make duplicate operations harmless.

---

## 3. Triage Gate Prevents 30-50% of Wasted Work

**What happened:** Without triage, the Scout created issues like "Rename variable from `x` to `count`" and Builders spent hours implementing cosmetic changes. The review rejection rate was 60%+ because the work shouldn't have started.

**The fix:** Added Triage agent between Scout and Builder. Every issue is evaluated on: value, scope, feasibility, size, duplication, risk. Target rejection rate: 30-50%.

**Result:** Builder output quality improved dramatically. Review first-pass rate went from ~40% to 70%+.

**Rule:** If you're not rejecting 30-50% of discoveries, your Scout is either too conservative (not finding enough) or your Triage is too permissive (approving junk).

---

## 4. Auto-Merge Doesn't Work with `strict: true`

**What happened:** With `strict: true`, merging PR #1 makes PRs #2-#10 stale. They all need to rebase and re-run CI. Auto-merge can't handle this — it just waits indefinitely.

**Compounding factor:** Vercel stuck `pending` statuses also block auto-merge. So even with `strict: false`, auto-merge was unreliable.

**The fix:** Direct merge via `gh pr merge --squash --delete-branch`, called by the Merger agent after verifying CI and review status.

**Rule:** Don't rely on GitHub auto-merge for autonomous pipelines. Use direct merge with explicit checks.

---

## 5. Vercel Stuck Pending Statuses Block Merges

**What happened:** Vercel sometimes leaves commit statuses in `pending` forever. These show as pending checks on PRs, blocking both auto-merge and manual merge (if "all checks must pass" is enabled).

**The fix:** Override stuck statuses:

```bash
gh api repos/{owner}/{repo}/statuses/{SHA} \
  -f state=success -f context=Vercel \
  -f description="Override stale pending"
```

**Gotcha:** GitHub App tokens can't override third-party statuses. Must use a personal account.

**Rule:** Don't make Vercel a required check. Monitor its statuses and override when stuck.

---

## 6. GitHub `reviewDecision` Stays `CHANGES_REQUESTED` After New Approvals

**What happened:** Reviewer requested changes on a PR. Fixer addressed feedback. Reviewer approved. But `gh pr view --json reviewDecision` still showed `CHANGES_REQUESTED`.

**Root cause:** GitHub doesn't automatically dismiss old reviews when a new one is submitted. The `reviewDecision` field reflects ALL reviews, not just the latest.

**The fix:** Explicitly dismiss stale reviews before approving:

```bash
# Get stale CHANGES_REQUESTED review IDs
gh api repos/{owner}/{repo}/pulls/{N}/reviews \
  --jq '.[] | select(.state == "CHANGES_REQUESTED") | .id'

# Dismiss each
gh api -X PUT repos/{owner}/{repo}/pulls/{N}/reviews/{ID}/dismissals \
  -f message="Superseded by re-review" -f event="DISMISS"
```

**Rule:** When re-reviewing a PR, always dismiss stale reviews first. Otherwise the PR looks permanently rejected.

---

## 7. Agent PRs Get Rejected for Predictable Reasons

**Research (arXiv 2602.04226):** The top reasons AI-generated PRs get rejected:

1. **Too large / hard to review** (30%+ of rejections)
2. **No added value** — cosmetic changes, unnecessary refactoring
3. **Increased complexity** — over-engineering simple fixes
4. **Context/environment limitations** — agent doesn't understand project conventions
5. **Distrust of AI-generated code** — humans skeptical by default

**Our mitigations:**
- Triage gate rejects low-value work before building starts
- Size labels enforce < 500 line PRs
- Builder prompt: "Follow existing patterns EXACTLY. Don't refactor unrelated code."
- Learning log captures rejection patterns, fed back into agent prompts

**Rule:** Train your agents on YOUR conventions. Generic prompts produce generic (rejected) code.

---

## 8. Post-Merge Sentinel Replaces Strict Mode Safety

**What happened:** With `strict: false`, we theoretically accept the risk of semantic merge conflicts (two PRs that individually pass CI but break when combined).

**In practice:** After 30 PRs merged on launch day with `strict: false`, Sentinel detected zero production issues. Small, focused PRs (enforced by our pipeline) rarely conflict semantically.

**The math:**
- `strict: true`: Prevents 100% of semantic conflicts, but costs 10× in throughput
- `strict: false` + Sentinel: Catches ~99% within 5 minutes, costs nothing in throughput

**Rule:** Post-merge monitoring is cheaper than pre-merge serialization.

---

## 9. The PR #381 Disaster — Never Let Coordinator Merge

**What happened:**
1. Coordinator had direct merge API access
2. Reviewer approved PR #381
3. CI was still running
4. Coordinator saw "approved" → called merge API
5. PR merged with a TypeScript error
6. Three more PRs built on broken `main`
7. Production broke for 4 hours

**The fix:** Remove merge capability from Coordinator entirely. Merger agent (or auto-merge) verifies ALL checks before merging.

**Rule:** The thing that decides "this should merge" should not be the thing that merges. Separate the decision from the action.

---

## 10. Scout Needs Duplicate Detection

**What happened:** Scout discovered "Fix mobile tab bar layout" three times in one day. Three Builders implemented three slightly different fixes. Two PRs conflicted.

**The fix:** Scout checks both open AND recently closed issues before creating new ones:

```bash
gh issue list --state open --json number,title --limit 100
gh issue list --state closed --label "pipeline/merged" --json number,title --limit 50
```

Plus: Triage agent as second line of defense rejects duplicates.

**Rule:** Duplicate detection at two levels: Scout (creation) and Triage (approval).

---

## 11. Learning Log Captures Rejection Patterns

**What happened:** The same types of PRs kept getting rejected: cosmetic changes, oversized refactors, out-of-scope features. The agents didn't learn from previous rejections.

**The fix:** `docs/pipeline-learning-log.md` in the repo captures:
- Triage rejection patterns (cosmetic changes, out-of-scope, duplicates)
- Review rejection patterns (missing error handling, breaking tests)
- Merge failures (which changes broke production)

Each agent gets the relevant sections in their prompt:
- **Scout** → triage rejection patterns (avoid creating what will be rejected)
- **Builder** → review rejection patterns (avoid common mistakes)
- **Reviewer** → merge failure patterns (extra scrutiny on risky areas)

**Rule:** Log rejections. Feed them back. Agents can only learn from written history.

---

## 12. Production Broke Because Agents Merged Conflicting PRs

**What happened (PR #381 era):** Multiple PRs merged to `main` in sequence without CI re-running between merges (before we understood `strict` mode). The combined changes broke the build.

**What we learned:** The solution is NOT `strict: true` (too slow). The solution is:
1. `strict: false` for throughput
2. Sentinel checks production after every merge batch
3. Small, focused PRs minimize conflict surface area
4. Post-merge CI on `main` catches build breakage immediately

**Rule:** Always verify build BEFORE merge (CI on PR) AND AFTER merge (post-merge verification on `main`).

---

## 13. Coordinator Creates Rogue Cron Jobs

**What happened:** Coordinator prompt said "keep the project healthy." It created:
- Cron job every 5 min: check CI status
- Cron job every 10 min: review stale PRs
- Cron job every 30 min: audit codebase

Each job spawned sub-agents. $50 in API costs in 4 hours.

**The fix:** Explicit prohibitions in coordinator prompt:
```
🚫 NEVER create cron jobs
🚫 NEVER create scheduled tasks
🚫 NEVER create persistent background processes
```

**Rule:** AI agents interpret vague instructions creatively. Be explicit about what they MUST NOT do.

---

## 14. Disabling a Cron Job Doesn't Stop In-Flight Runs

**What happened:** After discovering rogue cron jobs, we disabled them. But the currently running instance continued executing.

**The fix:** Disable future runs AND kill the current instance separately.

**Rule:** "Disabled" ≠ "stopped." Check for in-flight work.

---

## 15. Sub-Agents Hit Token Limits on Raw HTML

**What happened:** Agent tasked with "research competition" fetched a website with `curl` → 180KB raw HTML → 125K tokens → context limit exceeded → crash.

**The fix:** Use structured search tools (not raw HTTP), extract text before processing, set explicit size limits in prompts.

**Rule:** Never fetch more than 10KB of raw text in an agent context.

---

## 16. Human-Facing Agent Must Never Do Work Itself

**What happened:** User asked "Can you fix the login bug?" The main agent started debugging directly. For 3 minutes, all other sessions were blocked. It hit the token limit halfway and had to give up.

**The fix:** The main agent always delegates:
```
"On it — assigning to a Builder now."
→ spawns sub-agent
"Builder is working on it. I'll let you know when the PR is ready."
```

Total time: 5 seconds. Then back to being responsive.

**Rule:** If it takes more than 10 seconds, delegate it. The human-facing agent is a dispatcher, not a worker.

---

## Meta-Lesson: Write It Down

Every one of these lessons was learned the hard way. We didn't write them down immediately. We hit lesson #1 three times before it stuck.

Now we have this document. When a new failure happens:

1. Write down what happened (specific, not vague)
2. Write down the actual error message
3. Write down the fix
4. Add it to this document
5. Update agent prompts to prevent recurrence

**The playbook is only as good as its maintenance.**
