# Team Agent

> A coordinator + 4-role agent fleet for fully autonomous software development across multiple repos.

## Problem

Autonomous AI development breaks down when agents lack structure. Without defined roles, agents duplicate work, leave PRs unreviewed, miss CI failures, and file shallow issues that don't move the product forward.

## Solution

A stateless coordinator runs every 5 minutes, spawning agents by role into a fixed-size team. Roles are specialized: one Scout finds what matters, one Gatekeeper ensures quality, five Builders ship, one Tester breaks things before production does.

## Implementation

### Roster (4 roles, 8 agents total)

| Role | Key | Count | Job |
|------|-----|-------|-----|
| Scout | `scout` | 1 | Competitive research + multi-dimensional analysis → files prioritized issues |
| Gatekeeper | `gatekeeper` | 1 | Issue triage + deep PR review + squash merge |
| Builder | `builder` | 5 | Fix failing PRs first, then build from approved issues |
| Tester | `tester` | 1 | Adversarial tests for open PRs |

- **Scout** = Product + Audit + competitive research
- **Gatekeeper** = Triage + Reviewer

### Coordinator Algorithm (stateless, runs every 5 min)

1. **INVENTORY** — count active agents by label prefix
2. **CHECK REPOS** — scan all repos for CI status, PRs, issues
3. **PRIORITIZE** — failing CI > broken PRs > PRs needing review > approved issues > untriaged
4. **SPAWN DEFICIT** — fill empty slots with highest-priority work
5. **CI CHECK** — failing CI = top priority, always
6. **SUMMARY** — print status table

### Git Identities

| Role | Account | Identity |
|------|---------|----------|
| Scout, Builder, Tester | shtse8 | Kyle Tse / shtse8@gmail.com |
| Gatekeeper | claw-sylphx | Clayton Shaw / clayton@sylphx.com |

Switch accounts:
```bash
gh auth switch --user <account>
```

**Why two accounts:** Branch protection requires reviewer ≠ PR author. The Gatekeeper (claw-sylphx) can approve PRs opened by Builders (shtse8).

### Label System

```
pipeline/approved       pipeline/rejected
priority/P0             priority/P1          priority/P2     priority/P3
source/scout
type/architecture       type/optimization    type/ux         type/security
type/growth             type/quality         type/infra      type/bug     type/enhancement
size/XS                 size/S               size/M          size/L
```

### Branch Strategy

- All PRs target `staging`, **never** `main`
- `main` ← `staging` promotions happen on intentional release
- Branch protection: require PR + 1 approval + `build` CI check
- Squash merge only

### Project Context via CLAUDE.md

Each repo has a `CLAUDE.md` at root. Agents read it before working. Contains: what the project does, stack, conventions, key paths.

This is how one agent fleet works across multiple repos without hardcoding project context in the coordinator prompt.

### Multi-Repo Setup

One coordinator, one team, N repos. Agents are generalists — they pick work from any repo by priority. No per-repo agent assignments.

## Examples

### Scout Methodology (principle-based, not checklist)

**The Scout answers one question:** "What is the single most impactful thing we could change right now to get closer to world-class?"

**Phase 1 — Look outward first (MANDATORY before touching code)**

Research the competitive landscape using web search. Who are the competitors? What are they doing better? What UX patterns do best-in-class products use? Goal: build a mental model of "great" so we can measure the gap.

**Phase 2 — Three lenses on the product**

1. **First-time user** — What confuses? What breaks? Would you share this?
2. **Investor evaluating** — Impressive? Professional? Differentiated?
3. **Engineer maintaining for 2 years** — What patterns scare you? Where would adding a feature be painful?

**Phase 3 — Argue both sides**

For each finding: make the case FOR and AGAINST. Only file if "for" clearly wins.

**Phase 4 — File only what passes this bar**

- Can articulate real-world impact in one sentence
- Has specific evidence (files, code, competitor comparison)
- Fixing this would noticeably improve the product

**GREAT Scout issue:**
> Title: "Result page share flow drops 80% of potential shares — no native share API, no pre-filled message, 3 taps too many"
>
> Body: [specific file paths], competitors use Web Share API (1 tap vs our 5 steps), suggested approach...

**BAD Scout issue:**
> Title: "🔒 Use constant-time comparison in auth middleware"
>
> Why bad: Theoretical attack. A scanner would file this. A strategic thinker would not.

---

### Gatekeeper Methodology

**Core question:** "If I were the tech lead and this PR shipped to production, would anything keep me up at night?"

Consider:
- Is this the right approach, or just AN approach? Band-aids get approved because they "work."
- Does this PR leave the codebase in a **better** state?
- What if 5 more PRs followed this same pattern?
- What's **not** in this PR that should be?

**Good review:**
> "This fixes the bug, but adds a third way to fetch user data — we already have `getUserById` and `fetchUser`. Before merging, consolidate? Otherwise making duplication worse with every PR."

**Bad review:**
> "LGTM ✅"
>
> Why bad: adds zero value. Linters and CI already checked this.

---

### Tester Methodology

**Core question:** "What test, if it had existed BEFORE this PR was written, would have caught the bug or forced a better design?"

Think: how could this break in production?
- What inputs did the developer not consider?
- What happens when external dependencies fail?
- What would a malicious user try?

**Good test** (catches real race condition):
```typescript
test("concurrent submissions should not double-count", async () => {
  // Two browser tabs submitting simultaneously
  const results = await Promise.all([submit({...}), submit({...})]);
  expect(results.filter(r => r.status === 200).length).toBe(1);
});
```
Why good: developer only tested single request — this catches what they missed.

**Bad test** (tests nothing new):
```typescript
test("submit returns 200", async () => {
  expect((await submit({ answers: validAnswers })).status).toBe(200);
});
```
Why bad: developer already knows this works. Catches nothing.

**Rule:** If your test still passes with a subtle bug introduced, it's testing the wrong thing.

## Lessons & Gotchas

- **Stateless coordinator is key.** Any state stored in memory is lost between runs. Design the coordinator to reconstruct full context from GitHub API on every cycle.
- **Label discipline matters.** If labels aren't applied consistently, the coordinator's inventory is wrong and agents get spawned for the wrong work.
- **CI check name must match exactly.** Branch protection requires a check named `build` — if your CI job is named differently, merges will be permanently blocked.
- **Never assign agents to repos.** Agents should be generalists. Per-repo assignments kill parallelism and create idle capacity.
- **Two accounts are non-negotiable.** Tried single account + self-approval — GitHub rejects it. The Gatekeeper identity must be separate.
- **CLAUDE.md is load-bearing.** Without it, agents have to guess project conventions. Wrong guesses cause PRs that fail CI or violate patterns, wasting Builder slots.
