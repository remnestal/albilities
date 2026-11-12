---
name: pr-feedback
description: Work through review comments on your own GitHub PR. For each open thread it shows the comment and any saved note, gives a critical take, agrees a response with you, then clarifies, accepts with a fixup commit, or counters. Replies tersely and never resolves threads. Not for someone else's PR (that is /pr-review).
disable-model-invocation: true
argument-hint: "[PR]"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git show *), Bash(git rev-parse *), Bash(git merge-base *), Bash(git blame *), Bash(git branch --show-current), Bash(gh pr view *), Bash(mktemp -d), Bash(git worktree add *), Bash(git worktree remove *), Bash(gofumpt -d *)
---

# Address feedback on your PR

## Gates

- **Your own PR only.** If the PR author is not the viewer, stop. Reviewing
  someone else's PR is `/pr-review`.
- **Never resolve a thread.** The commenter resolves it.
- **Never rewrite history.** No autosquash, amend, rebase or force-push.
  Fixes land as `fixup!` commits; `/pr-finalize` squashes them.
- **GitHub owns thread state.** Notes add reasoning and fixup SHAs. They
  never decide whether a thread is open.
- **Show before sending.** The user approves every commit, push and reply.
- **Plan mode:** stop before the first write, as gates/plan-mode says.

## 1. Load

`$ARGUMENTS` is the PR number or URL. With none, use the current branch's PR.

Follow plumbing/github to identify the PR, the viewer and `<owner>/<repo>`,
and make `<tmp>`. Then fetch:

- all review threads (plumbing/github);
- review bodies and conversation comments:
  `gh pr view <N> -R <owner>/<repo> --json reviews,comments`. Keep those
  with a non-empty body written by someone other than the viewer. Their `id`
  (`PRR_…` or `IC_…`) is the note key;
- the PR's notes (plumbing/backlog): `backlog note-list <owner>/<repo>#<N>`,
  then `backlog note-get <owner>/<repo>#<N> <id>` for each.

Check the local branch, each command in its own call:

```bash
git branch --show-current
git fetch origin
git merge-base --is-ancestor <headRefOid> HEAD
git merge-base origin/<baseRefName> HEAD
```

- The current branch must be `headRefName`.
- If `--is-ancestor` exits 1, the remote has commits the local branch lacks.
  Stop and report.
- The last command prints the fork point. Write it literally wherever
  `<fork-sha>` appears below.

## 2. Triage

Sort every item by its GitHub state:

- **Done:** the thread is resolved. Delete its note:
  `backlog note-rm <owner>/<repo>#<N> <thread id>`. In plan mode, list the
  notes you would delete and go on with the read-only steps.
- **Waiting:** unresolved, and the last comment is the viewer's. The
  commenter owes a reply. List these and skip them unless the user picks one.
- **Open:** unresolved, and the last comment is someone else's.
- **Answered body:** a review body or conversation comment with a note.
  GitHub can't resolve these, so show the note's conclusion. One is done when
  the user says so (delete its note) or when the PR merges.
- **Skipped:** open, but its note is in state `skipped`. List it, and take
  it up again only if the user picks it.
- A review body or conversation comment without a note is open.

Show the counts and the open list (path:line, commenter, first line of the
comment). Then take the open items one at a time.

## 3. Per item

Finish each item, fixup included, before starting the next. Each fixup then
holds only that item's hunks.

1. **Form a critical take.** Read the code at the path first. Is the
   commenter right? Name the consequence of acting and of not acting. If the
   concern no longer applies, say so. Commit to a recommendation: clarify,
   accept or counter.
2. **Show the item** in the gates/decide format:
   - **Severity:** `nit` or `optional` when the comment carries that prefix,
     otherwise `blocking`.
   - **Where:** path and line (`originalLine` when outdated).
   - **Problem:** the comment, with the rest of the thread below it.
   - **Why it matters:** your take.
   - **Fix:** the recommended response.
   - Then the note, if there is one. A note in state `countered` means the
     commenter has answered a counter-proposal.
3. **Align** with AskUserQuestion: **Clarify**, **Accept**, **Counter** or
   **Skip**, with your recommendation first. Keep asking until you and the
   user agree.
4. **Act** on the outcome:
   - **Clarify:** no code change.
   - **Accept:** make the change the commenter asked for. Show the diff, get
     a yes, then commit it as a fixup (section 4).
   - **Counter:** think through the options first. Present them with
     AskUserQuestion, plus an open slot for the user's own idea. Once one is
     chosen, show the literal change as a diff, get a yes, and commit it as a
     fixup (section 4).
   - **Skip:** no change and no reply. Leave out step 5, and save the note
     in state `skipped` with the reason.
5. **Draft the reply** (section 5) and get the user's yes on its wording.
   Each yes in steps 4 and 5 is asked with the diff or reply as the
   question's preview, as gates/decide says.
6. **Save the note** at once, so the work survives an interruption:
   `backlog note-set <owner>/<repo>#<N> <id> '<body>'`, as plumbing/backlog
   describes. The body holds the state (`clarified`, `accepted`, `countered`
   or `skipped`), the conclusion in one or two sentences, the reasoning, the
   fixup SHA and the approved reply.

A counter the commenter rejects gets a new fixup on top. Never rewrite the
earlier one.

## 4. Fixup commit

1. **Filter format-only hunks** with the classification in
   plumbing/changeset. Show them and ask whether to revert them or leave them
   unstaged. They never go into a fixup.
2. **Find each hunk's target commit.** Blame the lines the hunk replaces,
   using its old-side line numbers, within the PR range:

   ```bash
   git blame -l -L <start>,<end> <fork-sha>..HEAD -- <path>
   ```

   Lines marked `^` predate the PR and have no target in the range. Ask the
   user whether they become a separate commit. A pure insertion takes the
   commit that owns the lines around it; ask when that is ambiguous.
   If blame lands on a `fixup!` commit from an earlier run, target the
   commit it fixes up.
3. **Stage per target.** When every hunk in a file has the same target, use
   `git add <path>`. Otherwise write the chosen hunks to
   `<tmp>/item.patch` and run `git apply --cached <tmp>/item.patch`.
   `git add -p` is interactive and unavailable.
4. **Commit one fixup per target.** Run `git diff --cached`, then
   `git commit --fixup=<target sha>`. Afterwards nothing is staged, and only
   the hunks the user chose to leave remain.

## 5. Replies

Follow the review-comment delta in bars/authoring. One sentence, no thanks,
no recap:

- **Accepted:** name the fixup, for example "Fixed in a1b2c3d."
- **Clarified:** the one fact the commenter lacked.
- **Countered:** what you did instead and why, with the fixup SHA.

## 6. Push and post

After the last item, show `git log --oneline origin/<headRefName>..HEAD` and
the approved replies, and ask once to push and post, with both as the
question's preview. Post only after the
push, so every SHA a reply names exists on GitHub.

```bash
git push
```

A rejected push means the remote moved. Stop and report. Never force.

Write each reply to its own file under `<tmp>` and post it:

- **Thread:** reply to its first comment as plumbing/github describes.
- **Review body or conversation comment:** post a top-level comment that
  quotes or links the original:
  `gh pr comment <N> -R <owner>/<repo> --body-file <tmp>/reply-<n>.md`.

Add each reply's URL to its note.

## 7. Report

Report what was answered and which threads are waiting on commenters. Point
to `/pr-finalize` for squashing the fixups and checking merge readiness.

## Reference

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/github`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/changeset`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`
