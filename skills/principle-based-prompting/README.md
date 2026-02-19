# Principle-Based Prompting

> Write AI prompts that establish purpose and thinking frameworks rather than checklists, so agents reason genuinely instead of satisfying rules mechanically.

## Problem

Rule-based prompts get gamed. Give AI a checklist and it will satisfy the checklist without genuine thought.

- "File minimum 8 issues" → 8 shallow issues, none worth fixing
- "Approval rate must not be 100%" → artificially reject good PRs to hit the quota
- "Check for security vulnerabilities" → file every theoretical attack from OWASP without thinking about real exploitability

The AI isn't trying to deceive — it's optimizing for what you measured instead of what you wanted.

## Solution

Principle-based prompts that:

1. **Establish the core PURPOSE** — one question that defines what success looks like
2. **Provide a THINKING FRAMEWORK** — how to approach the problem, not what outputs to produce
3. **Use EXAMPLES to calibrate quality** — good vs bad, side by side, with explanations
4. **Enforce STRUCTURAL requirements that can't be faked** — "argue both sides" forces genuine evaluation

## Implementation

### Signs your prompt is rule-based (bad)

- Numbered checklists of things to find
- Minimum/maximum counts ("file at least 8 issues")
- Enumerated anti-patterns ("don't do X, don't do Y, don't do Z")
- Specific syntax or format requirements that miss the intent

### Signs your prompt is principle-based (good)

- Starts with a core question the agent must answer
- Teaches HOW to think, not WHAT to think about
- Includes concrete examples of excellent vs poor output
- Asks the agent to argue both sides before deciding

### Template

```
**Core question:** [The one question that defines success for this role]

**How to think:**
[Framework for reasoning, not checklist of outputs]

**What good looks like:**
[Concrete example of excellent output]

**What bad looks like (and why):**
[Concrete example of poor output with explanation]
```

## Examples

### Applying the Template: Scout Role

**Rule-based (bad) Scout prompt:**
```
You are a Scout agent. Your job is to:
1. Review the codebase for issues
2. Check for security vulnerabilities
3. Look for performance problems
4. File at least 8 GitHub issues
5. Add labels: type/bug, type/security, type/performance as appropriate
6. Don't file duplicate issues
```

**Principle-based (good) Scout prompt:**
```
**Core question:** What is the single most impactful thing we could change right now 
to get closer to world-class?

**How to think:**
Start by looking outward — research competitors and best-in-class products before 
touching the codebase. Build a mental model of "great" so you can measure the gap.

Then examine the product through three lenses:
- First-time user: What confuses? What breaks? Would you share this?
- Investor evaluating: Impressive? Professional? Differentiated?
- Engineer maintaining for 2 years: What patterns scare you?

For each finding, argue both sides: make the case FOR and AGAINST filing. 
Only file if "for" clearly wins.

**What good looks like:**
"Result page share flow drops 80% of potential shares — no native share API, 
no pre-filled message, 3 taps too many"
[specific file paths, competitor comparison, suggested approach]

**What bad looks like (and why):**
"🔒 Use constant-time comparison in auth middleware"
Why bad: Theoretical attack with no evidence of real risk. A scanner files this. 
A strategic thinker doesn't.
```

### Applying the Template: Code Reviewer Role

**Rule-based (bad) reviewer prompt:**
```
Review this PR. Check for:
- Code style
- Security issues  
- Performance issues
- Test coverage
- Documentation
Approve if everything looks good.
```

**Principle-based (good) reviewer prompt:**
```
**Core question:** If I were the tech lead and this PR shipped to production, 
would anything keep me up at night?

**How to think:**
- Is this the RIGHT approach, or just AN approach? Band-aids work but shouldn't be approved.
- Does this leave the codebase in a BETTER state?
- What if 5 more PRs followed this same pattern?
- What's NOT in this PR that should be?

**What good looks like:**
"This fixes the bug, but adds a third way to fetch user data — we already have 
getUserById and fetchUser. Before merging, consolidate? Otherwise we make the 
duplication worse with every PR."

**What bad looks like (and why):**
"LGTM ✅"
Why bad: Adds zero value. Linters and CI already verified syntax and tests pass.
Human/AI review exists to catch things automated tools can't.
```

## Lessons & Gotchas

- **Examples do more work than rules.** A single concrete good/bad pair calibrates the agent's judgment better than a paragraph of rules. Show, don't tell.

- **"Argue both sides" is load-bearing.** It's the single best structural requirement for avoiding shallow output. It can't be faked without actually reasoning. Add it to any prompt where you want genuine evaluation.

- **Quantity requirements are always wrong.** "File at least N issues" creates exactly N issues — no more, no fewer — regardless of whether N is the right number. Remove all quantity floors and ceilings.

- **The core question is a forcing function.** If the agent can't answer the core question in one sentence, the output isn't ready. Make the core question prominent and specific.

- **Anti-pattern lists backfire.** Listing anti-patterns tells the agent what the failure modes are, which makes it easier to avoid triggering them while still producing bad output in novel ways. Better: good examples implicitly define what bad looks like.

- **Principle-based prompts require more upfront investment.** Writing a good core question and quality examples takes longer than writing a checklist. The payoff is output that actually moves the needle instead of output that technically satisfies the prompt.
