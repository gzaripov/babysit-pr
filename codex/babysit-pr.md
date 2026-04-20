Babysit the current PR until it is merge-ready. Alternate between two signals and fix whatever is broken after each round:

1. **CI checks** — all required checks must pass (`gh pr checks`).
2. **Bot review comments** — no unanswered bot comments (`npx agent-reviews --watch --bots-only`).

Terminate only when both signals are clean at the same time.

## Prerequisites

The current branch must have an associated PR. The `agent-reviews` CLI authenticates via `gh` or env vars; on cloud environments verify `git config --global --get user.email` is set to a real address so CI can attribute commits.

Substitute the user's package manager runner (`pnpm dlx`, `yarn dlx`, `bunx`) for `npx` where appropriate.

## Phase 1: Initial sweep

### Step 1: Resolve existing bot comments

Fetch unanswered bot comments with `npx agent-reviews --bots-only --unanswered --expanded`. For each comment:

- Evaluate whether it is a true positive, false positive, or uncertain.
- Fix true positives. Skip false positives. Ask the user when uncertain.
- After fixing all, commit and push.
- Reply to every comment with `npx agent-reviews --reply <id> "<message>" --resolve`.

Skip to Step 2 if there are no unanswered comments.

### Step 2: Check CI status

Run `gh pr checks` to read the current status.

- **All passing** → Phase 2.
- **Any pending / in-progress** → Step 3.
- **Any failed** → Step 4.

### Step 3: Wait for in-flight checks

Run `gh pr checks --watch` to block until every check finishes. When it returns, re-run Step 2 to re-evaluate.

### Step 4: Fix failing checks

For each failed check:

1. Identify the failing run: `gh pr checks --json name,link,state,workflow` (or `gh pr view --json statusCheckRollup`).
2. Fetch the failing logs: `gh run view <run-id> --log-failed`. For very long logs, grep for the first error.
3. Analyze the root cause. Read the relevant code — do not guess from the log alone.
4. Fix the code minimally. Do not refactor unrelated code.
5. Reproduce the check locally where feasible (lint, type-check, unit tests).
6. Commit with a message describing the fix and push.
7. Return to Step 2.

If the failure is genuinely unclear, or the fix would require architectural changes, stop and ask the user.

## Phase 2: Watch loop

All checks are green and all known bot comments are resolved. Now wait to see whether anything new appears.

### Step 5: Watch for new bot comments

Run `npx agent-reviews --watch --bots-only` and wait for it to exit (default 10 min; override with `--timeout`).

- **`EXITING WITH NEW COMMENTS`** → handle them as in Step 1, then go back to **Step 2** (new commits may have retriggered CI).
- **`WATCH COMPLETE`** → Step 6.

### Step 6: Final CI sanity check

Run `gh pr checks` once more. Intermediate commits may have kicked off a new run.

- **All green** → Summary report. Done.
- **Anything else** → back to Step 2.

## Summary report

```
## Babysit PR Summary
- Iterations: N
- Bot comments addressed: X (fixed Y, won't-fix Z, skipped W)
- CI failures fixed: M
- Final status: All checks green, no unanswered bot comments
```

## Stopping rules

Bail out and ask the user when:

- The same check fails 3 times in a row after your fixes — your diagnosis is probably wrong.
- The same bot keeps re-raising a comment on the same lines after you replied.
- A failure requires judgement outside the scope of the PR.
- The user has not authorized pushing to this branch, or the branch is protected.

Do not use destructive git operations (force push, reset --hard) to work around a check.
