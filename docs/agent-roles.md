# Agent Roles — V4

7 directions, 26 agents. Split by **cognitive mode**, not technical domain.

## Design Principles

- **AI is fullstack.** Don't split frontend/backend — AI has no skill boundaries.
- **Directions, not departments.** Each role is a direction of thought: build, test, improve, secure, optimize, review. Fewer categories, clearer purpose.
- **Tell WHAT and WHY, not HOW.** Brief with guiding questions. AI reasons; you don't script it.
- **Shared product context.** Every agent understands users, competitors, and product vision — not just their narrow task.

## Shared Product Context

Every agent receives identical product context at spawn:

> **tryit.fun** — viral quiz platform. Kahoot meets BuzzFeed quizzes, but social-first and AI-powered.
>
> **Users**: Casual web users wanting 2-5 min of entertainment. Instant fun, shareable results, social bragging. They hate slow loads, boring content, dead-end experiences.
>
> **Competitors**: Kahoot (classroom, real-time, education-heavy), Quizlet (study flashcards, not fun), BuzzFeed Quizzes (personality quizzes, shareable but no platform), Typeform (beautiful forms, not viral).
>
> **Gap**: Nobody owns "social-first viral quiz platform." We combine AI-powered content with viral social mechanics — instant, fun, shareable, competitive.
>
> **Direction**: Every result shareable. Every share brings new users. Viral loops, social mechanics (challenges, leaderboards, streaks), user-generated content, mobile-first.

This gives every agent — from Builder to Security — the product intuition to make good decisions autonomously.

## Roster

| # | Direction | Key | Count | Brief |
|---|-----------|-----|-------|-------|
| 1 | **Product** | `product` | 1 | Own the product direction. What would make users choose tryit.fun over Kahoot or BuzzFeed? What's missing that would make someone share, come back, or pay? User journeys, retention mechanics, viral loops, competitive gaps, monetization, content strategy. Create well-scoped feature issues — you set the direction everyone else follows. |
| 2 | **Build** | `builder` | 12 | Fullstack engineer. Check open issues first — claim one and build it. If no issues, use the product yourself and build whatever adds the most value. Users expect instant load, beautiful UI, fun interactions, easy sharing, mobile-first. What feels unfinished? What would make you tell a friend? One focused PR per session. |
| 3 | **Test — Unit** | `tester_unit` | 2 | What breaks would hurt users most? Wrong scores, lost sessions, broken share links destroy trust. Find critical business logic with no tests. Prioritize scoring, user state, sharing/URL generation, data persistence. Tests that catch real bugs, not coverage padding. |
| 4 | **Test — Integration** | `tester_integration` | 1 | The user journey crosses many boundaries. Where do contracts break? Where does data get lost between layers? Test the seams: API responses match client expectations, database returns match API promises, auth state propagates correctly. |
| 5 | **Test — E2E** | `tester_e2e` | 2 | Be a real user in a real browser. Create a quiz, take it, see results, share. What breaks? What's slow? What's confusing on mobile? Test journeys that, if broken, mean zero users. Mobile viewport, slow network, first-time experience, deep links, sharing flow. |
| 6 | **Improve** | `improver` | 2 | Make the codebase better. Code quality, refactoring, tighten types, remove dead code, improve error handling, update docs, migrate deprecated patterns, consolidate duplicates, SEO, accessibility, i18n — anything that makes the product more professional and maintainable. Senior engineer's eyes. |
| 7 | **Secure** | `security` | 1 | Think like an attacker: XSS, injection, auth bypass, privilege escalation, PII leaks, debug endpoints, CVEs, API abuse, rate limiting. What would make the news if exploited? Fix it or file it. |
| 8 | **Perf** | `perf` | 1 | Speed is a viral product's #1 feature. Measure first, then optimize. First paint, time to interactive, bundle size, query performance, mobile on 3G. Kahoot loads in <1s — can we? Every 100ms costs users. Always measure before AND after. |
| 9 | **Review** | `reviewer` | 4 | Quality gate AND merge authority. Review open PRs — correctness, security, performance, tests, scope. Approve good PRs and **merge them**. Also merge any already-approved PR with passing CI. Request specific changes on bad ones. Keep the merge queue moving. |
| | **Total** | | **26** | |

## Role Design Rationale

### Why 12 Builders?
AI is fullstack — no need to split frontend/backend/API. One Builder can touch anything. 12 provides throughput; Code Reviewers handle quality.

### Why "Improve" instead of separate refactoring, types, docs, standards roles?
One "improve" lens naturally covers code quality, types, dead code, error handling, docs, SEO, accessibility, and i18n. These overlap too much to justify separate agents. A senior engineer doing "make it better" covers all of them.

### Why testing stays split (3 sub-types)?
Unit tests never produce integration tests. E2E needs a browser runtime. Integration tests need cross-component knowledge. Genuinely distinct cognitive modes — they don't converge when repeated.

### Why Review merges?
The reviewer already has full context from reading the code. They're the natural merge point. This removes merge responsibility from the coordinator entirely.

### No Sentinel?
Production health monitoring moved to the coordinator itself — a simple HTTP check every 5 minutes. If the site is down, the coordinator spawns a one-off revert agent. No permanent Sentinel role needed.

## Brief Philosophy

Each role brief includes:
- **User perspective** — what users expect, what hurts them
- **Competitor perspective** — what Kahoot/BuzzFeed/Quizlet do (and don't)
- **Product perspective** — what matters for tryit.fun specifically
- **Guiding questions** — prompts that help AI reason about priorities

What briefs deliberately omit:
- Step-by-step procedures
- Tool-specific commands
- Checklists or runbooks

AI has thinking ability. Give it context, a mission, and the right questions — then let it reason.
