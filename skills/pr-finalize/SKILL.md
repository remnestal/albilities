---
name: pr-finalize
description: Make your own GitHub PR ready to merge. Checks unresolved threads, CI and approvals against GitHub, squashes fixup commits with a tree-identical autosquash, force-pushes with a lease after you confirm, and merges only when you confirm. Not for answering review comments (that is /pr-feedback).
disable-model-invocation: true
argument-hint: "[PR]"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git rev-parse *), Bash(git merge-base *), Bash(git branch --show-current), Bash(gh pr view *), Bash(gh pr checks *), Bash(gh repo view *), Bash(mktemp -d)
---

# Finalize your PR

## Gates

- **Your own PR only.** If the PR author is not the viewer, stop.
- **Report every check, then stop if any blocks.** Step 2 runs all checks
  before deciding. Nothing is pushed after a blocking result.
- **Never resolve a thread.** An unresolved thread passes only when the user
  overrides that specific thread.
- **Merge only on an explicit yes** given in this run, after the final
  status. Never pass `--admin`, and pass `--auto` only when asked.
- **Push only a clean, tree-identical autosquash.** The rebase exited 0, no
  rebase is in progress, and `git diff --quiet <pre-sha> HEAD` exits 0.
  Otherwise abort the rebase and push nothing.
- **Plan mode:** stop before the first write, as gates/plan-mode says.

## 1. Load

`$ARGUMENTS` is the PR number or URL. With none, use the current branch's PR.

Follow plumbing/github to identify the PR, the viewer and `<owner>/<repo>`.
Add `mergeStateStatus` to the `--json` fields. Write these down and use them
literally from here on:

- `<pre-sha>`: `headRefOid`;
- `<branch>`: `headRefName`;
- `<base>`: `baseRefName`.

Then, each command in its own call:

```bash
git status --porcelain
git fetch origin
git branch --show-current
git rev-parse HEAD
git merge-base origin/<base> HEAD
```

`git status --porcelain` must print nothing, because the rebase and the
reset in step 3 need a clean tree. Otherwise stop and report it. The current branch must be `<branch>`, and `HEAD` must equal
`<pre-sha>`. If not, there are unpushed commits or the remote moved. Stop and
report which. The last command prints `<base-sha>`.

## 2. Check readiness

Run all four checks against GitHub, then report the results together.

1. **Threads.** A resolved thread is done, whoever resolved it. List every
   unresolved thread: path:line, commenter, who spoke last, and the note's
   conclusion if there is one. Bots never resolve, so their threads always
   show here.
2. **CI.**

   ```bash
   gh pr checks <N> -R <owner>/<repo> --required --json name,bucket,link
   ```

   Judge by `bucket`: `pass`, `fail`, `pending`, `skipping` or `cancel`. Any
   `fail` or `cancel` blocks. Exit code 8 means pending, which may continue;
   step 5 waits. "no checks reported" with exit code 1 means there is
   nothing to wait for.
3. **Approvals.**

   ```bash
   gh api graphql -f owner='<owner>' -f repo='<repo>' -F pr=<N> -f query='
   query($owner: String!, $repo: String!, $pr: Int!) {
     repository(owner: $owner, name: $repo) { pullRequest(number: $pr) {
       reviewDecision
       latestOpinionatedReviews(first: 100) { nodes { author { login } state commit { oid } } }
     } } }'
   ```

   Judge by `reviewDecision`, not by counting reviews. It is `APPROVED`,
   `CHANGES_REQUESTED`, `REVIEW_REQUIRED`, or null when no review is
   required. `CHANGES_REQUESTED` blocks. A missing approval blocks the merge,
   not the squash, so report it and go on. Record who approved, for step 4.
4. **Merge state.** `DIRTY` means conflicts with the base, which blocks.
   `BEHIND` means the base moved. This skill never rebases onto a newer
   base. `DRAFT` needs `gh pr ready <N> -R <owner>/<repo>`, run only on
   request.

Then decide, in this order, each item as gates/decide describes:

1. **Any block above** ends the run after the report. Ask nothing.
2. **Behind.** For `BEHIND`, one item with no severity. The options are
   **Stop** (Recommended) and **Continue**.
3. **Threads.** One item per unresolved thread, with its severity taken from
   the comment's prefix: `nit` or `optional`, otherwise `blocking`. Where is
   path:line, Problem is the comment, Why it matters is who spoke last and
   the note's conclusion, Fix is what would close it. The options are
   **Leave open** (Recommended) and **Override**. Put **Override** first only when the note shows the point
   was answered. Tally the answers and act after the last thread.
4. **Re-request.** If any thread stays open, one item per human commenter
   with an open thread. The options are **Re-request review** (Recommended)
   and **Skip**.
   Act per item:
   `gh pr edit <N> -R <owner>/<repo> --add-reviewer <login>`. Then stop.

## 3. Autosquash

List the range:

```bash
git log --format='%h %s' <base-sha>..HEAD
```

The fixups are the lines whose subject starts with `fixup! `. With none, skip
to step 5. Before rebasing, check each fixup's target: the text after its
last `fixup! `. A commit in the listing that is not a fixup must have a
subject that equals or starts with that text, or a hash the text names. If a
fixup has no target there, stop and report it. Otherwise run each command in
its own call and note its exit code:

```bash
GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash <base-sha>
git rev-parse --git-path rebase-merge
test -e '<path printed by the previous command>'
git diff --quiet <pre-sha> HEAD
```

All must pass: the rebase exits 0, `test -e` exits 1 (no rebase in
progress), and `git diff --quiet` exits 0 (trees identical).

- **Rebase failed or still in progress:** run `git rebase --abort` and stop.
  A fixup that conflicts needs a manual decision.
- **Tree differs:** stop and show `git diff <pre-sha> HEAD`.
- **A `fixup!` commit remains** when you rerun the listing: stop and report
  it.

**On any stop from here until the push lands,** offer
`git reset --keep <pre-sha>` to restore the pushed state. The tree was clean
in step 1, so it loses nothing.

Show the squashed `git log --oneline <base-sha>..HEAD`. Run
`git rev-parse HEAD` and write the result down as `<new-sha>`.

## 4. Push

Put the push to the user as one gates/decide item, with no severity and no
Problem line:

- **Where:** `<branch>`, from `<pre-sha>` to `<new-sha>`.
- **Why it matters:** branch rules may demand a fresh approval from someone
  other than you after a force-push. Whether a tree-identical push keeps
  approvals is undocumented.
- **Fix:** the command below.

The options are **Push** (Recommended) and **Stop**. On yes, push with a
lease on the SHA that GitHub has:

```bash
git push --force-with-lease='<branch>:<pre-sha>' origin 'HEAD:<branch>'
```

A rejected lease means someone pushed in the meantime. Stop and report. Never
retry with a plain `--force`.

After the push:

1. **Wait for the new head's checks.** Until GitHub shows `<new-sha>`,
   `gh pr checks` reports the old head. Foreground `sleep` is blocked, so
   poll with the Monitor tool using an until-loop, with a five-minute
   timeout. The loop is a compound command, so it asks for permission once:

   ```bash
   until gh pr view <N> -R <owner>/<repo> --json headRefOid,statusCheckRollup --jq '.headRefOid == "<new-sha>" and (.statusCheckRollup | length) > 0' | grep -qx true; do sleep 30; done
   ```

   Without earlier checks, drop `and (.statusCheckRollup | length) > 0`. On
   timeout, stop and report.
2. **Approvals.** Rerun the approvals query and compare with step 2. Each
   approval that dropped is one gates/decide item, with the options
   **Re-request review** (Recommended) and **Skip**. Act per item with
   `gh pr edit <N> -R <owner>/<repo> --add-reviewer <login>`.

## 5. Merge

If any required check is pending, wait first:

```bash
gh pr checks <N> -R <owner>/<repo> --required --watch --fail-fast
```

Run it as a background command and continue when it exits. In the
foreground it needs the longest timeout, 600000 ms. A failure stops the run.
This applies whether or not step 4 pushed.

Show the final status: threads, CI, approvals, merge state, and the head
SHA (`<new-sha>` after a push, otherwise `<pre-sha>`). Read the allowed
methods:

```bash
gh repo view <owner>/<repo> --json squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed,viewerDefaultMergeMethod
```

Prefer `viewerDefaultMergeMethod` when it is allowed. Put the merge to the
user as one gates/decide item: merge with that method, merge with another
allowed method, or stop here. On a yes:

```bash
gh pr merge <N> -R <owner>/<repo> --<method> --match-head-commit <head-sha>
```

`--match-head-commit` refuses the merge if the head moved. Delete the branch
only if the user asks (`--delete-branch`). If GitHub refuses the merge,
report its reason and stop. On a no, report that the PR is ready and stop.

## 6. Clean up

Confirm the merge with `gh pr view <N> -R <owner>/<repo> --json state`.
Once the state is `MERGED`, delete every note for the PR:

```bash
backlog note-rm <owner>/<repo>#<N>
```

In plan mode, list the notes you would delete instead.

## Reference

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/github`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`
