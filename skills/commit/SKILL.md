---
name: commit
description: Turn uncommitted work into atomic git commits. Use when asked to commit, split, amend or reword, save progress or checkpoint, or before a plan step runs git commit. Not for past-commit questions, "commit to an approach", PR text, or PR-feedback fixups.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git show *), Bash(git ls-files *), Bash(git rev-parse *), Bash(git branch -r --contains *), Bash(git symbolic-ref *), Bash(git merge-base *), Bash(git add *), Bash(git apply *), Bash(git restore *), Bash(git rm *), Bash(git stash push *), Bash(git write-tree), Bash(git commit-tree *), Bash(git worktree add *), Bash(git worktree remove *), Bash(git commit *), Bash(gofumpt -l *), Bash(gofumpt -d *), Bash(go build *), Bash(go vet *), Bash(go test *), Bash(golangci-lint run *), Bash(mktemp -d)
---

# Commit

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

Steps 1 to 4 are the read-only part. Hard rules, before anything else:

- **Never amend, reword or rebase a pushed commit unless the user asks.**
  `git status -sb` shows whether the branch is ahead of its upstream. If
  `git branch -r --contains HEAD` prints anything, HEAD is pushed.
- **Show before you remove.** A file whose every hunk is format-only may be
  restored once the user has seen its file list. Anything else is removed only
  after the user has seen the hunk.
- **Never stage blindly.** No `git add -A` or `git add .` without reading
  everything it would pick up.
- **Check and approve every commit before making it.** Step 6 builds and
  tests the staged tree. Step 7 shows the whole message and waits for a yes.
- **One command per Bash call, plainly.** Pre-approvals don't match variables,
  pipes or `&&`, and each call is a fresh shell. Run `mktemp -d` once and write
  its literal path wherever `<tmp>` appears. Only git and the Go tools write
  there: the Write tool and redirects prompt outside the working directory.
- **Patches and messages go on stdin in a quoted heredoc** (`<<'EOF'`), never
  in double quotes, where backticks run under zsh. It matches pre-approvals.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

## Sidetrack gate

A sidetrack is a hunk in the tree, or a change you would propose, that is
outside the intended change (step 2).

**Absorb silently** a hunk the user directed ("also fix X"), or one that serves
the change's purpose in files it already touches, such as a rename inside the
rewritten function. Reformatting the rest of the file is a sidetrack. **Ask
about every other hunk, including unsure ones.** Drop a proposal of your own
that falls below the nitpick bar (step 8).

Classify each item against the actual diff:

- **A:** Is the current work correct and mergeable without the item?
- **B:** Is the item correct and mergeable without the current work?
- **Mergeable:** it builds, tests pass, and no reviewer asks for the rest.

| A | B | Shape | Recommend |
|---|---|---|---|
| no | no | One change | Part of the change; absorb |
| no | yes | Prerequisite | Separate commit, ordered first |
| yes | no | Builds on the current work | Skip; do it after this change |
| yes | yes | Independent | Separate commit if small and in touched files, else skip |

Put each item through the decision gate above, hunk under **Where**. The
**Problem** line names the shape and its answers: "Independent (A yes, B yes):
`Clamp` now treats `hi` as inclusive." Options, recommendation first:

- **Separate commit** on this branch, ordered by its shape.
- **Leave out,** for untracked files: it stays in the tree, uncommitted.
  Recommend it for untracked non-code files, such as notes.
- **Skip.** A hunk in the tree is reversed; an untracked file is deleted.

**The gate records; later steps act:** removals in step 5. An answer
settles the item for the session. Never ask about it again, even if the diff
grows.

## 1. Read the changeset

Use pending-work mode: everything staged, unstaged and untracked.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/changeset`

Also run `git log --oneline -15`: the repo's subject style beats the house
style below. If the tree is clean, say so and stop.

## 2. Confirm the intended change

Infer it from the first source that says: the conversation, then the branch name
(`git rev-parse --abbrev-ref HEAD`) and the diff. Show one line, such as
"**Intended change:** retry failed fetches with backoff (from the
conversation)". Ask with `AskUserQuestion`: Accept (Recommended) or Correct.
The sieve sorts hunks against it.

## 3. Sieve

Sort every hunk before staging anything:

- **The change.** Hunks that serve the intended change. Removing code the
  change itself made dead (an unused function, import or file) is part of it
  and goes in the same commit.
- **Sidetracks.** Everything else the gate asks about. A cleanup the change
  doesn't cause gets its own commit at most.
- **Scar tissue.** Code the change makes dead or wrong that is still in the
  tree: a helper with no callers left, a comment on the old behaviour, a flag
  nothing reads. Grep for it. Each is a gate item: Remove in this commit
  (Recommended) or Skip (leave it).
- **Format-only hunks,** classified as the changeset section says. They form
  two gate items, each showing its file list: touched files and untouched
  files. The options are a `style:` commit placed first, or Skip, which
  restores them. Recommend the `style:` commit for touched files and Skip for
  untouched ones.

Report debris from the changeset check as gate items. Never commit a stray
print or a skipped test without the user's say-so.

## 4. Plan the commits

One commit is one change that builds and passes tests on its own. Split a fix
from the refactor that enabled it, and a rename from a behaviour change. Code
the change made dead leaves in that commit, never in its own `refactor:`.

Order: the `style:` commit, prerequisites (refactors, renames, gate items of
that shape), the behaviour change, then other separate commits. If a split
would break the build midway, keep one commit and say so in its body.

State the plan in one line per commit. Ask only if the user would plausibly
disagree with the split.

## 5. Stage each commit

**The first time through, carry out the gate's removals:**

- **Skip on a hunk:** reverse a patch of only the unwanted hunks with
  `git apply -R --recount - <<'PATCH'`. If an unwanted line shares a hunk with
  wanted ones, split the hunk (below) or remove the line with Edit. Never
  restore a whole file that holds a wanted hunk.
- **Skip on an untracked file:** `rm -- <path>`, which prompts.
- **Formatting taken out:** `git restore -- <file>` for untouched files;
  reverse the hunks in touched ones.

Then run `go build ./...` or the repo's build. A drop must leave the tree
building, so an import only the dropped code used goes too.

**Stage hunks from a patch on stdin;** `git add -p` needs a TTY.

1. Copy this commit's hunks from `git diff -- <file>`, headers and context
   intact. Git applies all or nothing; `--recount` spares fixing headers:

   ```
   git apply --cached --recount - <<'PATCH'
   <hunks>
   PATCH
   ```
2. Whole and new files: `git add -- <path>`. Deletions: `git rm -- <path>`.
3. Changes within three lines share a hunk. To split them, copy from
   `git diff -U0 -- <file>` and add `--unidiff-zero` to the apply.
4. Unstage what doesn't belong with `git restore --staged -- <path>`, then
   restage the part that does.

Check `git diff --cached --stat` and read `git diff --cached`.

## 6. Check the staged tree

Build and test exactly what will be committed, in a scratch worktree, one
call per line:

1. `git write-tree` prints a tree SHA.
2. `git commit-tree <tree> -p HEAD -m check` prints a commit SHA. HEAD stays.
   Before the repo's first commit, leave out `-p HEAD`.
3. `git worktree add --detach <tmp>/check <sha>`
4. `go build -C <tmp>/check ./...` and `go vet -C <tmp>/check ./...` cover the
   whole module, so a broken caller anywhere fails. Then
   `go test -C <tmp>/check <pkgs>`, where `<pkgs>` are the touched packages,
   such as `./fetch/...`.
5. `git worktree remove --force <tmp>/check`, pass or fail.

Commands the repo's docs name (Makefile, Taskfile, CLAUDE.md) win, pointed at
the worktree (`make -C <tmp>/check test`); they may prompt. Outside Go, use
the repo's equivalents.

`golangci-lint` can't lint outside the current module. On the first pass
only, run `golangci-lint run --new-from-rev HEAD <pkgs>` in the user's tree. A
finding in a planned hunk counts as a failure.

On any failure, stop and show the error. Don't commit or fix without asking.

## 7. Approve and commit

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

Follow the commit delta above. End the message with a blank line and the
harness's attribution line, used exactly. Add none otherwise.

Show the whole message, subject and body, in a fenced block. Ask "Commit 2/3
with this message?" with `AskUserQuestion`: Commit (Recommended) or Edit.
After a yes:

```
git commit -F - <<'MSG'
<message>
MSG
```

Check that `git log -1 --stat` matches the plan. Repeat steps 5 to 7 per
commit. Afterwards, `git status --short` shows only what was left out.

## 8. Report

List the commits (short SHA and subject) and what was left unstaged. Nothing
else.

## Amend and reword

- **Amend** only an unpushed HEAD, or a pushed one on request. Run steps 5
  to 7 with `git commit --amend -F -` and the heredoc, or `--no-edit`.
- **Reword** an older unpushed commit only on request, without an interactive
  rebase: say what you would do and let the user choose.
- After amending a pushed commit, say the next push needs
  `--force-with-lease`. Don't push.
