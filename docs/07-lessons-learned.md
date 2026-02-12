# Lessons Learned — A Living Document

> Every lesson here cost us time, tokens, or production uptime.
> Read this before you start. Bookmark it. Come back when something breaks.

---

## 1. No CI Gate = Broken Code Merges Freely

**What happened**: We started the project without branch protection. The Coordinator had merge permissions. In the first 24 hours, 3 PRs merged with TypeScript errors. Production went down.

**The error we kept seeing**:
```
Type error: Property 'userId' does not exist on type 'Session | null'.
```

The agent assumed `session` was always defined. TypeScript caught it — but nobody required the build to pass before merging.

**Fix**: Branch protection with required status checks. Day 1. Not day 2. Day 1.

**Rule**: If the CI isn't set up, no agent gets write access. Period.

---

## 2. Coordinator Merging Stale/Duplicate PRs

**What happened**: Two Builders independently fixed the same issue. Both created PRs. The Coordinator saw both were approved and merged them sequentially. The second merge introduced a conflict that somehow passed CI (the files didn't overlap, but the logic conflicted).

**Real scenario**:
```
PR #45: Adds UserProfile component with inline styles
PR #47: Adds UserProfile component with Tailwind classes
Both "fix" issue #40: "Create user profile page"
```

Both passed CI because they were tested independently. When both merged, we had two `UserProfile.tsx` files — actually one file with conflicting implementations smushed together.

**Fix**: 
- Pre-merge duplicate detection (see [Merge Safety](03-merge-safety.md))
- The Coordinator must check if another PR already addresses the same issue
- Close superseded PRs with a comment

---

## 3. Vercel Ignore Build Step Inversion

**What happened**: We configured Vercel's "Ignored Build Step" to skip preview deploys on non-main branches. The exit codes are backwards:

```bash
# What we wrote (WRONG):
if [ "$VERCEL_GIT_COMMIT_REF" != "main" ]; then
  exit 1  # We thought: 1 = "yes, ignore this build"
fi
exit 0    # We thought: 0 = "proceed with build"

# What Vercel actually does:
# exit 0 = SKIP the build
# exit 1 = PROCEED with the build
```

**Result**: Production deploys were skipped. Preview deploys were running. For an entire week.

**The fix**:
```bash
# Correct:
if [ "$VERCEL_GIT_COMMIT_REF" != "main" ]; then
  echo "Skipping build for non-main branch"
  exit 0  # 0 = skip
fi
echo "Building for main"
exit 1  # 1 = proceed
```

**Rule**: Always verify deploy behavior by pushing a trivial change and checking the Vercel dashboard. Don't trust the logic.

---

## 4. Component Outside Provider = Runtime 500

**What happened**: A Builder added `MobileTabBar` to the root layout. The component uses `useAuth()` to show the user's avatar. But it was rendered OUTSIDE the `AuthProvider` boundary.

```tsx
// app/layout.tsx — THE BUG
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <AuthProvider>
          {children}
        </AuthProvider>
        <MobileTabBar />  {/* ← OUTSIDE AuthProvider */}
      </body>
    </html>
  );
}
```

**The insidious part**: This builds fine. TypeScript doesn't complain. ESLint doesn't catch it. The error only appears at runtime:

```
Error: useAuth must be used within an AuthProvider
    at MobileTabBar (MobileTabBar.tsx:12)
    at RootLayout (layout.tsx:8)
```

**Result**: 500 error on every page load on mobile. Desktop was fine (MobileTabBar has a `hidden md:` class, but the hook still runs).

**Fix**: Move the component inside the provider. Add a render test that catches context errors:

```typescript
// __tests__/layout.test.tsx
test('root layout renders without errors', () => {
  expect(() => render(<RootLayout>test</RootLayout>)).not.toThrow();
});
```

---

## 5. Transitive Dependencies in pnpm Strict Mode

**What happened**: A Builder used `@hono/zod-validator` in an API route. It worked locally because `@hono/zod-validator` was a transitive dependency of another package. But pnpm's strict mode doesn't hoist transitive deps.

**CI error**:
```
 ERR_PNPM_PEER_DEP_ISSUES  Unmet peer dependencies

error: Cannot find module '@hono/zod-validator'
  Require stack:
  - /app/src/api/routes/auth.ts
```

**The pattern**: This happens constantly with AI agents. They see an import in another file, assume it's available, and use it. With npm's hoisted `node_modules`, it works. With pnpm, it doesn't.

**Fix**: 
```bash
pnpm add @hono/zod-validator
```

And add to agent prompts: "If you import a package, verify it exists in `package.json` under `dependencies` or `devDependencies`. If not, run `pnpm add <package>` and commit the updated `package.json` and `pnpm-lock.yaml`."

---

## 6. Coordinator Creates Rogue Cron Jobs

**What happened**: The Coordinator prompt said "keep the project healthy." It interpreted this as:
1. Create a cron job to check CI status every 5 minutes
2. Create a cron job to review stale PRs every 10 minutes
3. Create a cron job to audit the codebase every 30 minutes

Each cron job spawned sub-agents. Each sub-agent consumed API tokens. We burned through $50 in API costs in 4 hours before noticing.

**The actual cron jobs it created**:
```
*/5 * * * *  — "Check all PR statuses and comment updates"
*/10 * * * * — "Review any PRs without reviews"
*/30 * * * * — "Full codebase audit for issues"
```

**Fix**: Add to Coordinator prompt, explicitly:
```
## PROHIBITED ACTIONS
- Do NOT create cron jobs
- Do NOT create scheduled tasks
- Do NOT create any persistent background processes
- Do NOT modify system configuration
```

**Rule**: AI agents interpret vague instructions creatively. Be explicit about what they MUST NOT do, not just what they should do.

---

## 7. `cron update enabled:false` Doesn't Stop In-Flight Runs

**What happened**: After discovering the rogue cron jobs (lesson #6), we disabled them:

```bash
openclaw cron update --id abc123 --enabled false
```

But the cron job that was already running continued to execute. Disabling a cron job prevents FUTURE runs — it doesn't stop the current one.

**Fix**: Kill the running process separately:
```bash
# Disable future runs
openclaw cron update --id abc123 --enabled false

# Kill the currently running instance
openclaw cron kill --id abc123
```

**Rule**: "Disabled" ≠ "stopped." Always check for in-flight runs.

---

## 8. Sub-Agents Hit 200k Token Limit on Raw HTML

**What happened**: A sub-agent was tasked with "research the competition." It fetched a competitor's website with `curl`, which returned 180kb of raw HTML. The agent tried to process this, hit the 200k token context limit, and crashed.

**The command that caused it**:
```bash
curl -s https://competitor-website.com | head -c 500000
```

That's 500kb of HTML → roughly 125k tokens. Add the system prompt and conversation history, and you're over the limit.

**Fix**: 
- Use structured search tools (Tavily) instead of raw `curl`
- If you must fetch HTML, extract text first: `curl -s URL | html2text | head -100`
- Set explicit limits in agent prompts: "Never fetch more than 10kb of raw text from any URL"

---

## 9. Single Coordinator Blocks All Channels When Busy

**What happened**: The Coordinator was handling a user's Telegram message (complex question about project status). While it was processing that message (~45 seconds), three other events queued up:
- A PR was ready for review
- A Builder finished and needed the next assignment
- A stale PR needed closing

Everything waited 45 seconds for the Coordinator to finish the Telegram response.

**Fix**: Separate the human-facing agent from the coordination agent. The Coordinator handles infrastructure. A separate agent handles user messages and delegates to the Coordinator.

**Rule**: The human-facing agent should NEVER do work itself. It should always delegate, then respond to the human while work happens asynchronously.

---

## 10. Always Delegate — Human-Facing Agent Must Never Do Work

**What happened**: A user asked "Can you fix the login bug?" The main agent (which was also the Coordinator) started debugging the login bug directly. For 3 minutes, it was deep in code analysis.

During those 3 minutes:
- Two approved PRs waited for auto-merge (fine, that's async)
- Another user sent a message and got no response
- A stale PR cleanup cycle was skipped
- The agent hit its token limit halfway through the fix and had to give up

**Fix**: The main agent's response should be:
```
"On it — I'm assigning this to a Builder agent now."
*spawns Builder sub-agent with the issue*
"Builder is working on it. I'll let you know when the PR is ready."
```

Total time: 5 seconds. Then back to being responsive.

**Rule**: If it takes more than 10 seconds, delegate it.

---

## Meta-Lesson: Write It Down

Every one of these lessons was learned the hard way. We didn't write them down immediately. We hit lesson #1 three times before it stuck.

Now we have this document. When a new failure happens:

1. Write down what happened (specific, not vague)
2. Write down the actual error message or behavior
3. Write down the fix
4. Add it to this document
5. Update agent prompts to prevent recurrence

The playbook is only as good as its maintenance. Keep it alive.
