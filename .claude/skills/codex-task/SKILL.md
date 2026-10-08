---
name: codex-task
description: Hands a coding task to Codex (newest Sol model, reasoning effort high), has it implemented in a separate worktree from a suitable start point (default branch, current branch or an open PR), checks the tests and then opens a PR. The user starts the review themselves. Example: /codex-task Add input validation to the login form.
argument-hint: "<task>"
allowed-tools: Bash, Read, Write
---

# Hand a task to Codex

Task: `$ARGUMENTS` – empty → ask the user and stop.

No subagent: Codex runs as a single background command, waiting costs nothing.
Do not implement anything yourself and do not review – the user starts that
(e.g. `/pr-review-codex`). Your role here: write the brief, secure the frame, deliver the result.

`S` = this session's scratchpad directory (not `/tmp`).

## 1. Prepare

- **Choose the start point yourself** – look at the current state first: `git branch --show-current`,
  `git status --short`, `git log --oneline origin/<default>..HEAD`, open PRs
  (`gh pr list --state open`). Then choose based on the task:
  | Task … | Start (`<start>`) | PR base (`<base>`) |
  |---|---|---|
  | independent of the current branch | `origin/<default>` (`gh repo view --json defaultBranchRef`) | default branch |
  | builds on commits of the current branch or an open PR (e.g. fixing findings there) | that branch | that branch (stacked PR) |
  | needs the **uncommitted** changes in the working tree | ask first: the user commits themselves (recommended) or the changes are copied into the worktree and end up in the Codex PR | same as the row above |
  If the chosen branch is ahead of `origin` locally or not pushed at all: start from the local
  state; before the PR, ask whether this branch may be pushed (otherwise no PR, branch only).
  When torn between two rows, ask briefly instead of guessing.
  State the choice and the reason in your start message to the user.
- Copying uncommitted changes (only after consent):
  `git diff HEAD --binary | git -C <worktree> apply` and copy untracked files
  (`git ls-files --others --exclude-standard`) one by one.
- Derive the branch name from the task, prefix by kind (`feat/`, `fix/`, `chore/`), short, kebab-case.
  Check that it exists neither locally nor on `origin`.
- Newest Sol model (do not hard-code):
  `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-sol"' | cut -d'"' -f4 | sort -V | tail -1`
  Empty → abort, point to `/codex:setup`.
- Worktree – the user's working tree stays untouched:
  ```
  git fetch origin --quiet
  git worktree add -b <branch> "$S/codex-<branch-without-prefix>" <start>
  ```

## 2. Write the brief

To `$S/codex-<name>-prompt.md`. Take the user's task verbatim, add only the frame,
no solution of your own. If needed, take a quick look at the repo (e.g. `docs/`) and name the
affected documents or files as pointers. Structure:

```
<task>…verbatim…</task>
<context>Pointers, relation to project documentation, if known</context>
<rules>
- Follow the project rules from AGENTS.md/CLAUDE.md.
  <!-- Customize: add your core rules here, e.g. tests first, comment language,
  architecture constraints from docs/. -->
- Stay small and focused, do not refactor anything unrelated.
- Do not commit, do not push, do not create branches.
- Run the affected tests (command per AGENTS.md/CLAUDE.md) until green.
</rules>
<return>
Briefly: what was changed (files), which tests are new, test result,
open points or assumptions. First line: suggested commit message
in the repo's style (e.g. "fix(scope): …").
</return>
```

## 3. Start Codex

Bash with `run_in_background: true`, then briefly tell the user the branch and model and wait
for the notification – do not poll:
```
codex exec -C "$S/codex-<name>" -s workspace-write \
  -m <sol-model> -c model_reasoning_effort='"high"' \
  -o "$S/codex-<name>-result.md" - < "$S/codex-<name>-prompt.md" \
  > "$S/codex-<name>-log.md" 2>&1; echo "exit $?"
```
`-c`/`-m` apply to this call only; do not modify `~/.codex/config.toml`.
In sandbox mode Codex cannot reach the network and cannot write to the main `.git` – that is why
this skill commits, not Codex.

## 4. Check the result

- Read `$S/codex-<name>-result.md`; check the log header for `model:` and `reasoning effort:`.
- `git -C <worktree> status --short` and `diff --stat`. No diff → report, remove the worktree,
  stop.
- Run the project check in the worktree, if there is one (e.g. a test script
  per CLAUDE.md/AGENTS.md, otherwise the affected tests). Red → **no PR**; show the user the errors
  and Codex's return and offer to let Codex fix it
  (same worktree, new `codex exec` run, prompt = error output + "please fix").
- Take a quick look for outliers (files outside the task, secrets, large binaries) –
  no content review.

## 5. Commit, push, PR

Running the skill is the mandate for this; ask beforehand only if there are outliers.
```
git -C <worktree> add -A
git -C <worktree> commit -m "<Codex suggestion>" -m "Implemented by Codex (<sol-model>, effort high)." -m "Co-Authored-By: …per current attribution…"
git -C <worktree> push -u origin <branch>
gh pr create --head <branch> --base <base> --title "<commit title>" --body-file <file>
```
PR body: task, changes, tests, open points from the Codex return,
note "Implemented by Codex (<model>, effort high)", attribution line. No auto-merge.

## 6. Clean up and report

- `git worktree remove "$S/codex-<name>"` (the branch stays for the PR).
- Reply briefly: PR link, branch, model, changed files, test result,
  open points from Codex. Hint: start the review with `/pr-review-codex <nr>`.
