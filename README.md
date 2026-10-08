# Claude Code × Codex Skills

Two [Claude Code](https://claude.com/claude-code) skills that hand work to **OpenAI Codex**
— implementing a task in its own worktree and reviewing pull requests — using the newest
Codex model with high reasoning effort.

They build on top of the official Codex plugin for Claude Code:
👉 **[openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)**

> A German version of the skills is available on the
> [`german_language`](https://github.com/SirTobi88/claude-codex-skills/tree/german_language) branch.

## Skills

| Skill | What it does | Example |
|---|---|---|
| [`codex-task`](.claude/skills/codex-task/SKILL.md) | Delegates a coding task to Codex. Picks a sensible start point (default branch, current branch or an open PR), creates a separate git worktree, lets Codex implement it, runs the project's tests and opens a PR. Does **not** review — that's what the second skill is for. | `/codex-task Add input validation to the login form` |
| [`pr-review-codex`](.claude/skills/pr-review-codex/SKILL.md) | Lets Codex review an open GitHub PR (default: the oldest one) in a detached worktree, then checks every finding and rates it as *confirmed*, *plausible* or *doubtful*. Changes nothing on the PR. | `/pr-review-codex 25` |

How they work:

- **Your working tree stays untouched** — every run uses its own `git worktree` in the session scratchpad.
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
- [GitHub CLI](https://cli.github.com/) (`gh`) logged in
- A git repository with a GitHub `origin` remote

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

Restart Claude Code (or start a new session); `/codex-task` and `/pr-review-codex` will show up as slash commands.

## Customization

The skills are deliberately generic and defer to your project's `CLAUDE.md` / `AGENTS.md`
for test commands and coding rules. Things you may want to adjust in
[`codex-task/SKILL.md`](.claude/skills/codex-task/SKILL.md):

- the `<rules>` block of the Codex prompt (marked with a `Customize` comment) — add your core project rules,
- the test / check command run before opening the PR,
- PR language and commit message style.

## Typical workflow

```
/codex-task Fix the off-by-one error in pagination   → Codex implements, tests, opens PR #42
/pr-review-codex 42                                   → Codex reviews, Claude rates findings
```

## Disclaimer

Unofficial community skills — not affiliated with or endorsed by OpenAI or Anthropic.
Codex usage counts toward your Codex usage limits.

## License

[MIT](LICENSE)
