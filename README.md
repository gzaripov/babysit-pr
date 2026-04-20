# babysit-pr

A coding-agent skill that watches an open pull request until it is merge-ready — resolving bot review comments and fixing failing CI checks in a loop until all checks are green and no unanswered bot comments remain.

Supports **Claude Code**, **Cursor**, and **OpenAI Codex CLI**.

It builds on top of [`agent-reviews`](https://github.com/pbakaus/agent-reviews) (which handles review-bot findings) and adds CI monitoring via the GitHub CLI, so you can hand off a PR and come back to a merge-ready one.

## What it does

On each iteration, the skill alternates between two signals and fixes whatever is broken:

1. **CI checks** — polled via `gh pr checks` / `gh pr checks --watch`. Failing runs have their logs fetched with `gh run view --log-failed`, the code is fixed, and the fix is committed and pushed.
2. **Bot review comments** — watched via `npx agent-reviews --watch --bots-only`. New findings from Cursor Bugbot, Copilot, CodeRabbit, etc. are evaluated, fixed (or dismissed), committed, and replied to.

It terminates only when both signals are clean at the same time. It also has stopping rules (e.g., same check failing 3× in a row) so it asks for help instead of looping forever on a misdiagnosis.

## Prerequisites (all tools)

- `git`
- [`gh`](https://cli.github.com/) (GitHub CLI), authenticated (`gh auth login`)
- Node.js (for `npx agent-reviews`)
- An open PR on the current branch

The `agent-reviews` CLI is fetched on demand via `npx`, so no global install is needed. If the project uses a different package manager, substitute `pnpm dlx`, `yarn dlx`, or `bunx`.

## Repository layout

```
.
├── SKILL.md                # Claude Code skill (with frontmatter)
├── cursor/
│   └── babysit-pr.mdc      # Cursor rule (with frontmatter)
└── codex/
    └── babysit-pr.md       # Codex CLI prompt
```

The body of each file is the same workflow; only the frontmatter and install location differ per tool.

## Installation

### Claude Code

User-level (available in every project):

```bash
git clone https://github.com/gzaripov/babysit-pr.git ~/.claude/skills/babysit-pr
```

Project-level (committed alongside one repo):

```bash
git clone https://github.com/gzaripov/babysit-pr.git .claude/skills/babysit-pr
```

Claude Code picks up `SKILL.md` the next time it starts. Invoke with `/babysit-pr` or by asking naturally (“babysit this PR”).

### Cursor

User-level (available in every workspace):

```bash
mkdir -p ~/.cursor/rules
curl -fsSL https://raw.githubusercontent.com/gzaripov/babysit-pr/main/cursor/babysit-pr.mdc \
  -o ~/.cursor/rules/babysit-pr.mdc
```

Project-level (committed in one repo):

```bash
mkdir -p .cursor/rules
curl -fsSL https://raw.githubusercontent.com/gzaripov/babysit-pr/main/cursor/babysit-pr.mdc \
  -o .cursor/rules/babysit-pr.mdc
```

The rule has `alwaysApply: false`, so it only loads when relevant. Ask the Cursor Agent to "babysit this PR" to invoke it; you can also `@`-mention the rule to pull it into context explicitly.

### OpenAI Codex CLI

Install as a custom prompt — it becomes a `/babysit-pr` slash command in the Codex REPL:

```bash
mkdir -p ~/.codex/prompts
curl -fsSL https://raw.githubusercontent.com/gzaripov/babysit-pr/main/codex/babysit-pr.md \
  -o ~/.codex/prompts/babysit-pr.md
```

Then from inside `codex`:

```
/babysit-pr
```

Codex will load the prompt and start the workflow on the current branch's PR.

## Usage

From inside a git repo with an open PR on the current branch:

- **Claude Code:** `/babysit-pr` (or “babysit this PR”)
- **Cursor:** ask the Agent to babysit the PR; the rule will be pulled in
- **Codex CLI:** `/babysit-pr`

The agent will:

1. Sweep any currently unanswered bot comments and resolve them.
2. Check CI status; wait for in-flight checks, fix failing ones.
3. Enter a watch loop — waiting for new bot comments or new CI failures — and handle them as they appear.
4. Stop when all checks are green, no unanswered bot comments remain, and the watcher has gone quiet.

It asks for input when uncertain (architectural changes, unclear failures, the same check failing repeatedly) rather than guessing.

## Updating

```bash
# Claude Code
cd ~/.claude/skills/babysit-pr && git pull

# Cursor
curl -fsSL https://raw.githubusercontent.com/gzaripov/babysit-pr/main/cursor/babysit-pr.mdc \
  -o ~/.cursor/rules/babysit-pr.mdc

# Codex
curl -fsSL https://raw.githubusercontent.com/gzaripov/babysit-pr/main/codex/babysit-pr.md \
  -o ~/.codex/prompts/babysit-pr.md
```

## Uninstalling

```bash
rm -rf ~/.claude/skills/babysit-pr
rm -f  ~/.cursor/rules/babysit-pr.mdc
rm -f  ~/.codex/prompts/babysit-pr.md
```

## What it won't do

- Force-push, reset, or otherwise rewrite history to make a check pass.
- Merge the PR (it stops once the PR is ready; merging is your call).
- Touch branch protection or CI configuration.
- Keep guessing forever — it bails out and asks after repeated failures.

## Related

- [agent-reviews](https://github.com/pbakaus/agent-reviews) — the bot-comment resolver this skill delegates to.
- [Claude Code skills docs](https://docs.claude.com/en/docs/claude-code/skills)
- [Cursor rules docs](https://docs.cursor.com/context/rules)
- [Codex CLI custom prompts](https://github.com/openai/codex)

## License

MIT. See [LICENSE](./LICENSE).
