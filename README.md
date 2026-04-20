# babysit-pr

A [Claude Code](https://claude.com/claude-code) skill that watches an open pull request until it is merge-ready — resolving bot review comments and fixing failing CI checks in a loop until all checks are green and no unanswered bot comments remain.

It builds on top of [`resolve-agent-reviews`](https://github.com/pbakaus/agent-reviews) (which handles review-bot findings) and adds CI monitoring via the GitHub CLI, so you can hand off a PR and come back to a merge-ready one.

## What it does

On each iteration, the skill alternates between two signals and fixes whatever is broken:

1. **CI checks** — polled via `gh pr checks` / `gh pr checks --watch`. Failing runs have their logs fetched with `gh run view --log-failed`, the code is fixed, and the fix is committed and pushed.
2. **Bot review comments** — watched via `npx agent-reviews --watch --bots-only`. New findings from Cursor Bugbot, Copilot, CodeRabbit, etc. are evaluated, fixed (or dismissed), committed, and replied to.

It terminates only when both signals are clean at the same time. It also has stopping rules (e.g., same check failing 3× in a row) so it asks for help instead of looping forever on a misdiagnosis.

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- `git`
- [`gh`](https://cli.github.com/) (GitHub CLI), authenticated (`gh auth login`)
- Node.js (for `npx agent-reviews`)
- An open PR on the current branch

The `agent-reviews` CLI is fetched on demand via `npx`, so no global install is needed. If the project uses a different package manager, substitute `pnpm dlx`, `yarn dlx`, or `bunx`.

## Installation

Install as a user-level Claude Code skill (available in every project):

```bash
git clone https://github.com/gzaripov/babysit-pr.git ~/.claude/skills/babysit-pr
```

Or as a project-level skill (committed alongside a single repo):

```bash
git clone https://github.com/gzaripov/babysit-pr.git .claude/skills/babysit-pr
```

That's it — Claude Code picks up `SKILL.md` the next time it starts.

### Updating

```bash
cd ~/.claude/skills/babysit-pr && git pull
```

### Uninstalling

```bash
rm -rf ~/.claude/skills/babysit-pr
```

## Usage

From inside a git repository with an open PR on the current branch, ask Claude Code to invoke the skill:

```
/babysit-pr
```

Or phrase it naturally:

> Babysit this PR until it's ready to merge.

Claude will:

1. Sweep any currently unanswered bot comments and resolve them.
2. Check CI status; wait for in-flight checks, fix failing ones.
3. Enter a watch loop — waiting for new bot comments or new CI failures — and handle them as they appear.
4. Stop when all checks are green, no unanswered bot comments remain, and the watcher has gone quiet.

It will ask for input when uncertain (architectural changes, unclear failures, the same check failing repeatedly) rather than guessing.

## What it won't do

- Force-push, reset, or otherwise rewrite history to make a check pass.
- Merge the PR (it stops once the PR is ready; merging is your call).
- Touch branch protection or CI configuration.
- Keep guessing forever — it bails out and asks after repeated failures.

## Related

- [resolve-agent-reviews skill](https://github.com/pbakaus/agent-reviews) — the bot-comment resolver this skill delegates to.
- [Claude Code skills docs](https://docs.claude.com/en/docs/claude-code/skills)

## License

MIT. See [LICENSE](./LICENSE).
