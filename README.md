# Claude Code × Codex Skills

Three [Claude Code](https://claude.com/claude-code) skills that hand work to **OpenAI Codex**
— implementing a task in its own worktree (as a PR or as a local commit) and reviewing pull
requests — using the newest Codex model with high reasoning effort.

They build on top of the official Codex plugin for Claude Code:
👉 **[openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)**

> A German version of the skills is available on the
> [`german_language`](https://github.com/SirTobi88/claude-codex-skills/tree/german_language) branch.

## Skills

| Skill | What it does | Example |
|---|---|---|
| [`codex-task`](.claude/skills/codex-task/SKILL.md) | Delegates a coding task to Codex. Picks a sensible start point (default branch, current branch or an open PR), creates a separate git worktree, lets Codex implement it, runs the project's tests and opens a PR. Does **not** review — that's what `pr-review-codex` is for. | `/codex-task Add input validation to the login form` |
| [`codex-local-task`](.claude/skills/codex-local-task/SKILL.md) | Purely local variant: no push, no PR, no access to `origin`. Codex works in a worktree from the current `HEAD` while Claude keeps working on other files; Claude then reads the diff, runs the tests (letting Codex fix it up to twice) and cherry-picks the result as a commit into the current branch. Works on unpushed branches, and Claude may also use it on its own for well-scoped subtasks. | `/codex-local-task Extract the price calculation into the core module` |
| [`pr-review-codex`](.claude/skills/pr-review-codex/SKILL.md) | Lets Codex review an open GitHub PR (default: the oldest one) in a detached worktree, then checks every finding and rates it as *confirmed*, *plausible* or *doubtful*. Changes nothing on the PR. | `/pr-review-codex 25` |

How they work:

- **Your working tree stays untouched** — every run uses its own `git worktree` in the session scratchpad
  (`codex-local-task` only touches it at the end, via `git cherry-pick`).
- **Newest model, resolved at runtime** — the latest `*-sol` model is read from `codex debug models` instead of being hard-coded.
- **High reasoning effort per call** — set via `-c model_reasoning_effort="high"`; your `~/.codex/config.toml` is never modified.
- **Background execution** — Codex runs as a single background command; Claude waits for the notification instead of polling.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code)
- The official Codex plugin — install it in Claude Code:
  ```
  /plugin marketplace add openai/codex-plugin-cc
  /plugin install codex@openai-codex
  /reload-plugins
  /codex:setup
  ```
  See [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) for details
  (requires Node.js 18.18+ and a ChatGPT subscription or OpenAI API key).
- Codex CLI installed and logged in (`/codex:setup` checks this; otherwise `npm install -g @openai/codex` and `codex login`)
- [GitHub CLI](https://cli.github.com/) (`gh`) logged in and a git repository with a GitHub `origin` remote
  (for `codex-task` and `pr-review-codex`; `codex-local-task` only needs a local git repository)

## Installation

**Option A — use this repo as a template**

Click **“Use this template”** on GitHub. The skills live in `.claude/skills/`, so they are
available as project skills in the new repository right away.

**Option B — add to an existing project**

```bash
git clone https://github.com/SirTobi88/claude-codex-skills.git /tmp/claude-codex-skills
mkdir -p <your-project>/.claude/skills
cp -R /tmp/claude-codex-skills/.claude/skills/* <your-project>/.claude/skills/
```

**Option C — install globally (all projects)**

```bash
mkdir -p ~/.claude/skills
cp -R /tmp/claude-codex-skills/.claude/skills/* ~/.claude/skills/
```

Restart Claude Code (or start a new session); `/codex-task`, `/codex-local-task` and `/pr-review-codex` will show up as slash commands.

## Customization

The skills are deliberately generic and defer to your project's `CLAUDE.md` / `AGENTS.md`
for test commands and coding rules. Things you may want to adjust in
[`codex-task/SKILL.md`](.claude/skills/codex-task/SKILL.md):

- the `<rules>` block of the Codex prompt (marked with a `Customize` comment) — add your core project rules,
- the test / check command run before opening the PR,
- PR language and commit message style.

In [`codex-local-task/SKILL.md`](.claude/skills/codex-local-task/SKILL.md), adjust the
`Customize` comments: the dependency-resolve commands (Codex has no network in its sandbox),
project constraints in `<context>` and core rules in `<rules>`, plus which tasks count as
“easy to delegate” for your stack.

## Typical workflow

```
/codex-task Fix the off-by-one error in pagination   → Codex implements, tests, opens PR #42
/pr-review-codex 42                                   → Codex reviews, Claude rates findings

/codex-local-task Add unit tests for the date parser   → Codex implements locally, Claude checks
                                                        and cherry-picks it into the current branch
```

## See also

**[AI_Limitmeter](https://github.com/SirTobi88/AI_Limitmeter)** — an ESP32 desk display
that shows Claude Code's and Codex's usage limits side by side, so you can see what
handing work to Codex costs while you do it.

## Disclaimer

Unofficial community skills — not affiliated with or endorsed by OpenAI or Anthropic.
Codex usage counts toward your Codex usage limits.

## License

[MIT](LICENSE)
