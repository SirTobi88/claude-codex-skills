---
name: codex-local-task
description: Delegates a well-scoped coding task to Codex (newest Sol model, effort high) – purely local, no push and no PR. Codex works in its own worktree from the current local state, Claude keeps working in the meantime, then checks diff and tests and takes over the result as a commit in the current branch via cherry-pick. Claude may also use it on its own when a subtask is easy to delegate (pure logic, rule derivable from docs/, checkable with tests, no UI, no network). Example: /codex-local-task Extract the price calculation into the core module.
argument-hint: "<task>"
allowed-tools: Bash, Read, Write, Edit, Grep
---

# Delegate a task to Codex locally

Task: `$ARGUMENTS` – empty → ask (user) or do not invoke (Claude).

Difference to `/codex-task`: **no push, no PR, no access to `origin`**. The start is the
local state, the goal is a commit in the current branch. That is why this skill also fits
work on unpushed branches and autonomous work without asking back.

Roles: Codex implements. Claude writes the brief, keeps working on other files in the
meantime, checks the result (read the diff + tests) and takes it over. Claude is responsible
for the result as if Claude had written it.

`S` = this session's scratchpad directory (not `/tmp`).

## 0. Does the task fit?

Easy to delegate:
- pure logic in a module/package, moving logic from the app into a module, new adapters
  following an existing pattern;
- the rule is in `docs/` or can be stated unambiguously in the brief → tests can be derived;
- checkable with fast, self-contained tests (unit/package tests).

Do not delegate (do it yourself): platform builds, emulators/simulators, UI and design work,
anything needing the network, tasks whose solution only becomes clear while building, and
anything touching the same files as your own ongoing work.

At most **one** Codex run at a time (builds in the worktree plus your own builds compete for
memory and CPU).

## 1. Prepare

- Look at the state: `git branch --show-current`, `git status --short`.
- **Start = `HEAD` of the current branch.** If the task needs uncommitted changes: commit
  your own changes beforehand as usual (follow the project rules); never take along
  changes from others (from the user or from tools, e.g. generated project files) – then
  ask the user or cut the task so it does not need them.
- Branch `codex/<short-name>` (kebab-case), check that it does not exist locally.
- Newest Sol model (do not hard-code):
  `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-sol"' | cut -d'"' -f4 | sort -V | tail -1`
  Empty → abort, point to `/codex:setup`.
- Worktree (the user's working tree stays untouched):
  `git worktree add -b codex/<name> "$S/codex-<name>" HEAD`
- **Resolve dependencies beforehand** – Codex has no network in its sandbox. For every
  package/module the check builds, fetch its dependencies in the worktree first
  (e.g. `npm ci`, `swift package resolve`, `cargo fetch`, `go mod download`).
  <!-- Customize: list the exact commands for your project here. -->
- Remember `START=$(git rev-parse HEAD)`.
- Small preparations for the brief (e.g. clarifying a comment in a data file) may happen
  directly in the worktree before the start – they end up in the Codex commit. Commit larger
  preparations regularly beforehand.

## 2. Write the brief

To `$S/codex-<name>-prompt.md`. Unlike `/codex-task`, the brief is a complete briefing –
Codex does not know the conversation:

```
<task>What exactly should be built, in 2–6 sentences. When invoked by the user, take
their wording verbatim.</task>
<context>
- Pointers: affected files/types with path, example code ("following the pattern of …").
- Specifications: section in docs/ (path + heading) that contains the rule.
- Interface that must stay (signatures used by callers).
- Project constraints that matter here.
  <!-- Customize: e.g. allowed dependencies per layer, platform limits, localization rules. -->
</context>
<boundaries>
- Only change these files/folders: …
- Do not touch: … (files Claude is working on in parallel, …)
</boundaries>
<rules>
- Follow the project rules from AGENTS.md/CLAUDE.md.
  <!-- Customize: add your core rules here, e.g. tests first, comment language,
  no code against docs/ (better approach → update the document first, with reasoning). -->
- Stay small, do not refactor or reformat anything unrelated.
- Do not commit, no branches, no network.
- Run the affected tests (command per AGENTS.md/CLAUDE.md) until green. When changing a
  shared core module, also test the modules that depend on it.
</rules>
<acceptance>Concrete test cases or behaviour that must hold at the end.</acceptance>
<return>
Briefly: changed files, new tests, test result (count), assumptions and open points.
First line: commit message in the repo's style (`git log --oneline -5`), referencing the
document if there is one.
</return>
```

## 3. Start Codex and keep working

Bash with `run_in_background: true`:
```
codex exec -C "$S/codex-<name>" -s workspace-write \
  -m <sol-model> -c model_reasoning_effort='"high"' \
  -o "$S/codex-<name>-result.md" - < "$S/codex-<name>-prompt.md" \
  > "$S/codex-<name>-log.md" 2>&1; echo "exit $?"
```
Do not modify `~/.codex/config.toml`. Tell the user in one line (task, branch, model).
Then **do not wait and do not poll**: keep working on files outside `<boundaries>`, the
notification arrives on its own. During the run, do not start heavy checks (e.g. simulator
or end-to-end runs) in the worktree and do not change the files from `<boundaries>` in your
own working tree.

## 4. Check the result

- Read `$S/codex-<name>-result.md`; check the log for `model:` and `reasoning effort:`.
- Do not take Codex's own test result unchecked: inside the sandbox some APIs need extra
  permissions, runs can turn out differently than outside.
- `git -C "$S/codex-<name>" status --short` and `diff --stat`. No diff → report, clean up
  (step 6), stop.
- **Read the diff** (`git -C … diff`, new files in full): Did Codex fulfil the task, stay
  within the boundaries, write tests for the rule (not just adjust them)? Project rules kept?
  Outliers: files outside the brief, formatting noise, secrets, binaries.
- **Check in the worktree** by area:
  | changed | check |
  |---|---|
  | only isolated modules/packages | the tests of each affected module and the modules depending on it |
  | also app / integration code | the project's full check (e.g. test script per CLAUDE.md/AGENTS.md) in the worktree – never in parallel to your own run |
- Red or wrong in substance → let Codex fix it, at most twice: new `codex exec` in the same
  worktree, prompt = original brief + error output or concrete complaint + "please fix".
  After that finish it yourself or discard it and report. Small things (typo, a comment)
  fix directly yourself in the worktree.

## 5. Take over (commit + cherry-pick)

```
git -C "$S/codex-<name>" add -A
git -C "$S/codex-<name>" commit -m "<commit title>" -m "Implemented by Codex (<sol-model>, effort high), checked by Claude." -m "Co-Authored-By: …per current attribution…"
git cherry-pick codex/<name>
```
- Before that, check that the working tree has no uncommitted changes to the files of the
  commit (`git diff --name-only` ∩ `git diff-tree --no-commit-id --name-only -r codex/<name>`).
- Conflict → `git cherry-pick --abort`; in the worktree `git rebase <current branch>`,
  resolve conflicts (yourself or Codex with a conflict brief), repeat the check, pick again.
- Afterwards run the project's regular check in your own working tree before the task
  counts as done (several takeovers may share one run).
- No push, no PR, no tag – not even "to finish up".

## 6. Clean up and report

- The content is taken over when `git cherry HEAD codex/<name>` shows **nothing** (commit is
  contained – if `HEAD` has not moved since the start, the cherry-pick just fast-forwards)
  or only `-` lines (same content). Then
  `git worktree remove "$S/codex-<name>"` and `git branch -D codex/<name>`.
  Not taken over (discarded, conflict open) → remove the worktree, keep the branch and
  mention it in the report.
- Report briefly: commit (hash, title), changed files, test result, what Claude fixed,
  open points from Codex.
