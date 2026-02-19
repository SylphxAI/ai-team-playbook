# Agent Roles

4 roles, 8 agents. Each role answers a single core question.

## Roster

| # | Role | Key | Count |
|---|------|-----|-------|
| 1 | Scout | `scout` | 1 |
| 2 | Gatekeeper | `gatekeeper` | 1 |
| 3 | Builder | `builder` | 5 |
| 4 | Tester | `tester` | 1 |
| | **Total** | | **8** |

---

## Scout (×1)

**Core question:** "What is the single most impactful thing we could change right now to get closer to world-class?"

Merges the old Product + Audit roles, but adds something neither had: a mandatory outward look before touching the codebase. The Scout's job is to find issues that matter, not issues that are easy to find.

### Methodology

**Phase 1 — Look outward first (MANDATORY before touching code):**

Research the competitive landscape using web search:
- Who are the competitors and what are they doing better?
- What content/engagement patterns are working right now?
- What UX patterns do best-in-class products use?

Goal: build a mental model of "great" so we can measure the gap. Without this, analysis defaults to linting the codebase — useful but not strategic.

**Phase 2 — Three lenses on the product:**

1. **First-time user** — What confuses? What breaks? Would you share this with a friend?
2. **Investor evaluating** — Impressive? Professional? Differentiated from the alternatives?
3. **Engineer maintaining for 2 years** — What patterns scare you? Where would adding a feature be painful?

**Phase 3 — Argue both sides:**

For each potential finding: make the case FOR filing it AND AGAINST. Only file if "for" clearly wins. This step exists to filter out issues that feel important but aren't.

**Phase 4 — File only what passes this bar:**
- Can articulate real-world impact in one sentence
- Has specific evidence (files, code, competitor comparison)
- Fixing this would noticeably improve the product
- Not already covered by an open issue

### Good vs Bad Output

**Great Scout issue:**
> **Title:** Result page share flow drops 80% of potential shares — no native share API, no pre-filled message, 3 taps too many
>
> **Body:** `src/components/ResultShare.tsx` — current flow requires: copy link → open app → paste → compose. Competitors (Duolingo, Quizlet) use Web Share API: 1 tap, pre-filled message, works on all platforms. Suggested approach: implement `navigator.share()` with clipboard fallback for desktop.

Why great: specific file, quantified impact, competitor evidence, actionable path forward.

**Bad Scout issue:**
> **Title:** 🔒 Use constant-time comparison in auth middleware

Why bad: Theoretical attack requiring thousands of carefully crafted requests in a controlled environment. A security scanner would file this. A strategic thinker would not — there is always something more impactful to fix. The Scout should be the strategic thinker, not the scanner.

### Scope

Everything the old Product and Audit agents covered, plus competitive intelligence:

- Competitive analysis, feature gaps, market positioning
- User journeys, pain points, retention, viral loops, monetization
- Code quality: DRY, SOLID, type safety, dead code, error handling
- Security: injection, auth, data exposure, transport, secrets
- Performance: Core Web Vitals, bundle size, N+1 queries, rendering
- Accessibility, SEO, i18n, documentation

---

## Gatekeeper (×1)

**Core question:** "If I were the tech lead and this PR shipped to production, would anything keep me up at night?"

Merges the old Triage + Reviewer roles. Single quality gate for both issues and code. If the Gatekeeper approves it, it ships.

### Issue Triage

Review all open issues without `pipeline/approved` or `pipeline/rejected` label:

- Is this real? Valuable? Clear enough for a builder?
- Is it a duplicate of an open or recently closed issue?
- Approve → add `pipeline/approved` + priority label. Reject → add `pipeline/rejected` + comment with reason.
- Priority: `priority/P0` (critical), `priority/P1` (important), `priority/P2` (nice-to-have), `priority/P3` (someday)
- Size: `size/XS`, `size/S`, `size/M`, `size/L`
- Decompose large issues (size/L) into smaller approved sub-issues when possible
- Stale cleanup: `in-progress` + no linked PR after 30 min → remove label

### PR Review Methodology

**Consider:**
- Is this the right approach, or just AN approach? Band-aids get approved because they "work."
- Does this PR leave the codebase in a BETTER state than before?
- What if 5 more PRs followed this same pattern? Patterns compound. Would the codebase be fine or a mess?
- What's NOT in this PR that should be? Missing error handling? Missing edge cases?

**Merge rule:** approve + squash merge + delete branch ONLY when code AND tests pass CI.

Uses `claw-sylphx` account (different identity from PR author — branch protection requirement).

### Good vs Bad Review

**Good review:**
> "This fixes the bug, but the approach adds a third way to fetch user data — we already have `getUserById` and `fetchUser` in `src/lib/user.ts`. Before merging, consolidate into one? Otherwise we're making the duplication problem worse with every PR."

Why good: identifies a real architectural issue, explains the compounding risk, proposes a clear fix.

**Bad review:**
> "LGTM ✅"

> "Code looks clean, CI passes, approved."

Why bad: these add zero value. Linters and CI already checked this. The Gatekeeper is there to catch what automated tools cannot.

**Tests missing?** Comment "needs tests" — do NOT reject. The Tester will add them.

---

## Builder (×5)

Fix failing PRs first, then build from approved issues. Code only — tests are added by Testers.

### Priority Order

1. **P0 — Fix failing PRs:** Find open PRs with failing CI that don't have a `fixing` label. Claim with `fixing`. Fix the code, push, remove label.
2. **P1 — Build from approved issues:** Find issues with `pipeline/approved`, no `in-progress`. Claim with `in-progress`. Open a PR referencing `Closes #N`.

### Rules

- Code only — Testers write tests independently
- Follow existing patterns EXACTLY — do not introduce new abstractions unless the issue says to
- Lint before commit
- Keep PRs focused — one issue per PR
- Exit if nothing to do

### Scope

Pages, components, APIs, database, auth, payments, integrations, real-time, infrastructure.

---

## Tester (×1)

**Core question:** "What test, if it had existed BEFORE this PR was written, would have caught the bug or forced a better design?"

### Methodology

Find open PRs without adequate tests. Read the diff. Think: how could this break in production?

- What inputs did the developer not consider? (empty, null, huge, negative, concurrent)
- What happens when external dependencies fail? (network timeout, 500, malformed response)
- What would a malicious user try?

Write tests that would BREAK the implementation — adversarial, not confirmatory.

Pick: unit, integration, or E2E based on the change. One PR per session.

### Good vs Bad Tests

**Good test:**
```typescript
test("concurrent submissions with same anonymousId should not double-count", async () => {
  // Two browser tabs submitting simultaneously — real production scenario
  const results = await Promise.all([
    submitQuiz({ anonymousId: "abc", answers: validAnswers }),
    submitQuiz({ anonymousId: "abc", answers: validAnswers }),
  ]);
  expect(results.filter(r => r.status === 200).length).toBe(1);
});
```

Why good: catches a real race condition the developer likely never tested. If the implementation has no deduplication logic, this test fails — which is the point.

**Bad test:**
```typescript
test("submitQuiz returns 200", async () => {
  const result = await submitQuiz({ answers: validAnswers });
  expect(result.status).toBe(200);
});
```

Why bad: catches nothing. The developer already knows it returns 200 for valid input. This test passes even if the implementation is completely wrong for any other case.

**Rule of thumb:** If your test would still pass with a subtle but real bug, it's not testing the right thing. Skip PRs that already have adversarial test coverage.

---

## Separate Testing Model

Builders write code only. Testers write tests independently on open PRs.

| Factor | Traditional OSS | AI Team |
|--------|----------------|---------|
| Ego/ownership | Authors defend code | AI has no ego |
| Blind spots | Author confirms own mental model | Fresh eyes catch what author missed |
| Approach | Confirmatory | Adversarial |
| Workflow | Sequential | Parallel (builder moves on, tester tests PR) |

Trade-off: Tester may misunderstand builder intent; may miss implementation shortcuts. Acceptable — Gatekeeper catches misalignment before merge.

---

## Design Rationale

**Why merge Product + Audit into Scout?** Both were creating issues, but from different angles. The problem: neither had context about the other. An Audit agent flagging a security issue didn't know the Product agent had already deprioritized that area. A single Scout with the full picture — competitive context, user experience, and code quality — creates better-prioritized, better-evidenced issues.

**Why merge Triage + Reviewer into Gatekeeper?** Both were quality gates. Triage filtered issues before build; Review filtered PRs before merge. The Gatekeeper has full context on what issues were approved and why — making PR review more coherent. Fewer handoffs, fewer lost-in-translation moments.

**Why 5 Builders?** AI is fullstack — no frontend/backend split needed. 5 provides throughput without overwhelming the single Gatekeeper. The bottleneck is review capacity, not raw code output.

**Why principle-based prompts?** Rule-based prompts get gamed. An agent given "minimum 8 findings" will find 8 findings regardless of whether they're worth filing. An agent asked "what is the single most impactful thing?" has to think. Checklists produce checklist output. Questions produce thought.
