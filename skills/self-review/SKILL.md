---
name: self-review
description: Review your own branch, uncommitted work included, before it goes up. Runs the review engine, asks you to accept & fix, defer or skip each finding, and fixes accepted ones in the working tree. Never posts, stages or commits. Not for someone else's PR (use /pr-review) or for replies on your own PR (use /pr-feedback).
disable-model-invocation: true
argument-hint: "[base branch]"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git show *), Bash(git rev-parse *), Bash(git merge-base *), Bash(git blame *), Bash(git ls-files *), Bash(git symbolic-ref *), Bash(gofumpt -l *), Bash(gh issue view *), Bash(gh repo view *), Bash(mktemp -d), Bash(git worktree add *), Bash(git worktree remove *), Bash(gofumpt -d *)
---

# Self-review

## Gates

- **Nothing is posted.** No PR, no comment, no `gh` write of any kind.
- **The index and history are the user's.** Never stage, commit, stash, check
  out, reset or push.
- **Edit only for an accepted finding, and only in the fix loop.** Subagents
  stay read-only. Deleting debris, scar tissue or an untracked file needs its
  own accepted finding too.
- **Pre-approved commands run plain:** one command per Bash call, no shell
  variables, pipes, `&&` or loops. Each call is a fresh shell.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

In plan mode, decide every finding and sidetrack first. The accepted fixes
and the backlog entries become the plan, applied once plan mode ends.

## 1. Changeset

Follow the changeset fragment's whole-branch diff. `$ARGUMENTS` may name the
base branch.

Leave format-only hunks out of the lenses and say how many there were. Debris
the fragment flags becomes findings under the structure lens.

## 2. Problem statement

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/problem`

Sources: the branch's commit messages, issues they reference
(`gh issue view`), the plan from this session, and the branch's backlog scope
(`${CLAUDE_PLUGIN_ROOT}/bin/backlog scope`).

## 3. Review

Run the engine with the fork SHA as `<base>`: fan-out, consolidation, then the
finding gate.

## 4. Finding gate

The options are **Accept & fix** (recommended), **Defer** and **Skip**. The
proposed action is the fix in one concrete line: "guard `cache` with a
mutex", not "handle concurrency".

Defer records the finding at its severity, text in single quotes with no
apostrophes:
`${CLAUDE_PLUGIN_ROOT}/bin/backlog add --severity <level> '<text>'`.
Skip drops the finding: no fix, no backlog entry.

## 5. Fix

Apply each accepted fix before showing the next finding, so the tree always
matches what the user agreed to.

- Make the smallest change that resolves the finding, in the style of the
  surrounding code. No cleanups, renames or reformatting on the way past.
- If the fix turns out bigger than described, stop and ask again before
  editing. Bigger means several files, a signature other packages call, or a
  behaviour change beyond the finding.
- If the real fix is a redesign, leave it and report it as deliberately left.
  A half-fix that hides the symptom is worse than an open finding.
- Add a test only when the finding is about tests or the user asks. Otherwise
  say the fix is untested.
- Run the formatter only on files you edited.

## 6. Verify

Build, lint and test the way the repo's own docs say (Makefile, Taskfile,
CLAUDE.md). Without them, use the language defaults; for Go that is
`go build ./...`, `go vet ./...` and `go test ./...`. Report every failure,
and say whether it involves code you just changed.

## 7. Sidetracks

Decide each sidetrack alone, as the decision gate describes:
**Follow-up** (recommended) records it with the `backlog add` command from
step 4, and **Skip** drops it. Don't ask about intent-creep hunks: name them
in the report so `/commit` can split them out.

## 8. Report

- Fixed, one line each.
- Deferred, with severity.
- Skipped, one line each.
- Left on purpose, with the reason.
- Clean lenses.
- Sidetracks recorded as follow-ups.
- Intent-creep hunks for `/commit`.

Then suggest `/code-review` for a general correctness pass and the `commit`
skill to turn the tree into commits. Run neither.

# Shared rules

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/review`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment review/engine`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment lenses/tests`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/changeset`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`
