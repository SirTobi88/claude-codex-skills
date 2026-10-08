---
name: pr-review-codex
description: Has Codex review an open GitHub PR (default: the oldest) – newest Sol model, reasoning effort high – and returns the findings with your own assessment. Call with an optional PR number, e.g. /pr-review-codex 25.
argument-hint: "[PR number]"
allowed-tools: Bash, Read
---

# PR review with Codex

Argument: `$ARGUMENTS` – empty = oldest open PR, otherwise this PR number.

No subagent: the review is a single background command, waiting costs nothing,
and the output stays unaltered. Do not change anything on the PR, do not comment or push.

## Procedure

`S` = this session's scratchpad directory (not `/tmp`).

1. **Determine the PR**
   - without argument: `gh pr list --state open --json number,headRefName,baseRefName,createdAt --jq 'sort_by(.createdAt)[0] | "\(.number) \(.headRefName) \(.baseRefName)"'`
   - with argument: `gh pr view <nr> --json number,headRefName,baseRefName --jq '"\(.number) \(.headRefName) \(.baseRefName)"'`
   - No open PR → tell the user and stop.

2. **Newest Sol model**, resolved at runtime (do not hard-code):
   `codex debug models 2>/dev/null | grep -o '"slug":"[^"]*-sol"' | cut -d'"' -f4 | sort -V | tail -1`
   Empty → abort and report (Codex CLI missing or not logged in; `/codex:setup`).

3. **Separate worktree** – the user's working tree stays untouched:
   ```
   git fetch origin --quiet
   git worktree add --detach "$S/pr-<nr>" origin/<headRef>
   git -C "$S/pr-<nr>" diff --shortstat origin/<baseRef>...HEAD
   ```

4. **Start the review in the background** (Bash with `run_in_background: true`), then briefly
   tell the user the PR, model and size and wait for the notification – do not poll:
   ```
   cd "$S/pr-<nr>" && codex review --base origin/<baseRef> \
     -c model='"<sol-model>"' -c model_reasoning_effort='"high"' \
     > "$S/pr-<nr>-review.md" 2>&1; echo "exit $?"
   ```
   The plugin script `codex-companion.mjs review` has no `--effort`, hence the CLI directly.
   `-c` applies to this call only; do not modify `~/.codex/config.toml`.

5. **Read the result**: the header (check the lines with `model:` and `reasoning effort:`) and the
   final block starting at `Full review comments:` – the file is long, do not read all of it.

6. **Verify the findings**: look up each finding at the given location in the worktree and rate it
   as *confirmed*, *plausible* or *doubtful* (short reason).

7. **Clean up**: `git worktree remove --force "$S/pr-<nr>"`. When reproducing issues, Codex
   creates its own files (e.g. under `/tmp` or `/private/tmp`) – list them,
   but delete only on request.

## Reply to the user

Briefly:
- Header line: PR (as link `https://github.com/<owner>/<repo>/pull/<nr>`), model, effort, size.
- Per finding: priority, title, file:line as a Markdown link relative to the repo
  (strip the worktree path), Codex's core statement, your own rating.
- Codex's overall verdict (e.g. test situation) in one sentence.
- Path to the full output, note on Codex temp files.
- Closing: offer to fix the findings on the PR branch – do not start unasked.
