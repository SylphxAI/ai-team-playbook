# Open Source Research — How the Best Projects Scale

> We researched how the highest-throughput open source projects handle merging,
> CI, and coordination. The findings shaped our entire pipeline design.

## Key Finding: Nobody Uses `strict: true` at Scale

We surveyed the most active open source projects on GitHub. **None of them** use GitHub's "Require branches to be up to date before merging" (`strict: true`) setting.

| Project | Monthly PRs | `strict: true`? | Strategy |
|---------|-------------|-----------------|----------|
| Kubernetes | 2,500+ | No | Prow/Tide batch merge |
| React | High | No | CI + human review |
| Next.js | High | No | CI + Vercel tooling |
| Rust | High | No | Homu/Bors → merge queue |
| GitHub (monorepo) | 2,500+ | No | Merge queue, 15-30 PR batches |

**Why?** `strict: true` serializes merging. Every merge invalidates all other queued PRs. At scale, this creates an hours-long queue for what should be minutes of work.

---

## Kubernetes — Prow/Tide (2,500+ PRs/month)

Kubernetes is the gold standard for high-volume PR management. They built their own tooling:

### Prow
A Kubernetes-native CI/CD system with bot commands:
- `/lgtm` — "looks good to me" (from reviewer)
- `/approve` — (from approver, different from reviewer)
- `/hold` — prevent merge
- `/retest` — re-run failed tests

### Tide
The merge controller that handles batching:
- **Batch testing:** Groups multiple approved PRs and tests them together against `main`
- **If the batch passes:** All PRs merge
- **If the batch fails:** Tide bisects to find the culprit, merges the passing subset
- **No individual PR needs to be "up to date"** — the batch is tested as a whole

### OWNERS Files
Kubernetes uses `OWNERS` files in each directory:
```yaml
# pkg/controller/OWNERS
approvers:
  - senior-maintainer-1
  - senior-maintainer-2
reviewers:
  - contributor-1
  - contributor-2
```

Prow automatically assigns reviewers and requires approval from OWNERS.

### Key Takeaway
Kubernetes never requires individual PRs to be rebased on `main`. Instead, they batch-test groups of PRs. This is fundamentally the same insight behind our `strict: false` + Sentinel approach.

---

## React & Next.js — CI + Human Review

React and Next.js take a simpler approach:

- **Required checks:** CI (flow, jest, lint for React; build, test, lint for Next.js)
- **Required reviews:** 1 from core team
- **No strict mode:** PRs merge when approved + CI green
- **No merge queue:** Direct merge (squash)

### What Makes This Work
- High-quality human reviewers catch integration issues
- Well-factored codebases minimize file overlap between PRs
- Fast CI (< 10 min) means quick feedback loops

### Key Takeaway
For most projects, `CI + review` is sufficient. You don't need `strict: true` OR merge queues if your PRs are small and your CI is reliable.

---

## Rust — Homu/Bors → GitHub Merge Queue

Rust has the longest history of automated merge management:

### Bors (Legacy)
- Reviewers approve with `r+`
- Bors creates a temporary merge branch with the PR + latest `main`
- Runs full test suite on the merged result
- If it passes, fast-forwards `main`
- If it fails, the PR is not merged

### Migration to Merge Queue
Rust is transitioning to GitHub's native merge queue, which provides similar functionality without the custom infrastructure.

### Key Takeaway
Bors was the precursor to GitHub merge queues. The core insight: test the merged result, not the individual PR. Our Sentinel approach is the post-merge version of this.

---

## GitHub's Own Monorepo — Merge Queue

GitHub ships 2,500+ PRs/month to their monorepo. They use their own merge queue feature:

### How Merge Queue Works
1. PR is approved and enters the queue
2. Queue batches 15-30 PRs into groups
3. Each group is rebased on `main` + all previous groups
4. CI runs on the entire group
5. If the group passes, all PRs merge
6. If it fails, the group is bisected

### Parallel Testing
Merge queue can test multiple groups in parallel:
```
Group 1: PRs #1-15 (testing against main)
Group 2: PRs #16-30 (testing against main + Group 1)
Group 3: PRs #31-45 (testing against main + Group 1 + Group 2)
```

If Group 1 fails, Groups 2 and 3 are re-tested. If Group 1 passes, Groups 2 and 3 don't need to re-test.

### Why We Don't Use Merge Queue
- Adds configuration complexity
- Designed for human teams, not autonomous agents
- `strict: false` + Sentinel achieves the same safety with simpler infrastructure
- Our pipeline already ensures PRs are small and focused (the main risk factor for integration failures)

---

## Academic Research

### STRATUS (NeurIPS 2025) — FSM-Based Agent Coordination

**Paper:** "STRATUS: Multi-Agent SRE with State Machine and Safety Specifications"

STRATUS uses a finite state machine to coordinate multiple agents in site reliability engineering:
- Each agent has explicit states with transition guards
- A "Transactional No-Regression" safety specification prevents agents from making things worse
- The state machine prevents invalid operations and makes the system auditable

**How we use this:** Our pipeline labels ARE a finite state machine. Each issue transitions through explicit states (`discovered` → `approved` → `in-progress` → `pr-open` → `review-passed` → `merged`). Transition guards (the coordinator checks) prevent invalid transitions.

### MetaGPT — Structured Outputs

**Paper:** "MetaGPT: Meta Programming for Multi-Agent Collaborative Framework"

MetaGPT's key insight: **structured outputs beat free-form chat.** When agents produce artifacts (code, documents, schemas) instead of messages, hallucination cascading is dramatically reduced.

**How we use this:**
- Scout outputs structured issue templates (not chat messages)
- Triage outputs structured decisions (APPROVE/REJECT with criteria)
- Builder outputs PRs with structured descriptions
- Reviewer outputs structured reviews (APPROVE/REQUEST_CHANGES with checklist)

Every agent produces a concrete artifact, not a conversation.

### "Why Agentic-PRs Get Rejected" (arXiv 2602.04226)

**Key findings — top rejection reasons:**

1. **Too large / hard to review** (30%+ of rejections)
   - Our mitigation: Size labels, < 500 line PRs, one concern per PR

2. **No added value**
   - Our mitigation: Triage gate filters cosmetic/trivial changes

3. **Increased complexity unnecessarily**
   - Our mitigation: Builder prompt: "Follow existing patterns EXACTLY"

4. **Context/environment limitations**
   - Our mitigation: Agents read project conventions, learning log

5. **Distrust of AI-generated code**
   - Our mitigation: Agent review + CI + Sentinel multi-layered verification

### "Where Do AI Coding Agents Fail" (arXiv 2601.15195)

**Key insight:** Agents fail not because they can't write code, but because they don't understand how software teams work — CI conventions, review norms, PR sizing.

**How we address this:**
- Agents are trained on OUR specific conventions (via learning log)
- Builder verifies build locally before opening PR
- PR template enforces structured description
- Review checklist matches our team's standards

### Self-Evolving Agents (Live-SWE-agent, SE-Agent)

The frontier is agents that modify their own toolchains and strategies based on outcomes.

**Our implementation:** The learning loop:
1. Triage rejects an issue → pattern logged
2. Reviewer rejects a PR → pattern logged  
3. Sentinel detects failure → pattern logged
4. All logs fed back into agent prompts next cycle
5. Agents gradually stop making the same mistakes

This isn't full self-evolution (agents don't modify their own code), but it's a practical first step.

---

## Practical Recommendations

Based on all this research:

### For Teams Doing < 5 PRs/Day
- `strict: true` is fine (throughput penalty is small)
- Single CI check + 1 review
- No merge queue needed

### For Teams Doing 5-20 PRs/Day
- Consider `strict: false` with post-merge monitoring
- Or use GitHub merge queue (if available)
- Batch testing saves significant time

### For Autonomous Agent Pipelines (20+ PRs/Day)
- `strict: false` is mandatory (throughput is everything)
- Sentinel post-merge monitoring
- Small, focused PRs (minimize conflict surface)
- Triage gate (prevent wasted work)
- Learning loop (prevent repeated mistakes)
- Single required CI check (speed over ceremony)

### Universal Best Practices
- **Squash merge only** — clean history, easy revert
- **Auto-delete branches** — prevent accumulation
- **1 required review** — sufficient with good CI
- **Single CI check** — avoid check proliferation
- **Labels for state** — machine-readable, queryable
- **Post-merge verification** — catch what pre-merge misses
