---
name: pr-review
description: Review someone else's GitHub PR with the review engine, decide each finding with you, and post one batched review of terse comments pinned to the head commit. `resume <PR>` follows up on replies to your threads. Not for your own branch (use /self-review).
disable-model-invocation: true
argument-hint: "<PR number or URL> | resume <PR>"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(gh pr view *), Bash(gh issue view *), Bash(mktemp -d), Bash(gh pr diff *), Bash(git worktree add *), Bash(git worktree remove *), Bash(gofumpt -d *)
---

# PR review

`$ARGUMENTS` is a PR number or URL for a review, or `resume <PR>` for resume
mode. With no argument, ask which PR. A bare number means a PR of the
current directory's repo.

## Gates

- **Post nothing until the user has decided every finding and approved the
  drafted batch.** A review goes out as one batch. In resume mode each reply
  is approved on its own.
- **The review event is `COMMENT`.** Never approve, request changes, merge,
  close or edit the PR unless the user asks.
- **Resolve only threads the user started.** Never the author's, never
  anyone else's.
- **Leave the PR branch and the user's checkout alone.** No push, no checkout
  of the PR into the user's working tree.
- **If the user wrote the PR, stop.** Point to `/self-review`.
- **Plan mode:** stop before the first write, as gates/plan-mode says. A clone
  and a temp dir are writes, so in plan mode review without them: read the
  diff with `gh pr diff <N> -R <owner>/<repo>` and files at the head with
  `gh api repos/<owner>/<repo>/contents/<path>?ref=<head-sha>`. Decide the
  findings, then stop before drafting files or posting.

## Review mode

### 1. Load the PR

Follow plumbing/github to identify the PR, the viewer and `<owner>/<repo>`,
and make `<tmp>`. Add `title,body,closingIssuesReferences` to the `--json`
fields. Write `headRefOid` down as `<head-sha>`: every anchor and the review
itself pin to it.

### 2. Changeset

Follow the changeset fragment's PR mode. It clones the base repo to its own
directory and checks out `refs/pull/<N>/head` there. Confirm the checkout
matches `<head-sha>`. Review and read code in that clone only.

### 3. Problem statement

Run the problem-statement gate. Its sources are the PR body, the linked
issues (`gh issue view <issue> -R <owner>/<repo>`) and the commit messages.

### 4. Review

Run the engine in the clone: fan-out, consolidation, then the finding gate.
Its `<base>` is `origin/<baseRefName>`, written literally.

### 5. Finding gate

The options are **Comment** and **Skip**. The proposed action is the draft
comment and its anchor, written to the authoring bar's review-comment delta.
High and medium findings carry no prefix. Low ones carry `nit:` or
`optional:`. A question is fine when the answer may be "it's fine, because X".
The note keeps the full reasoning, so the comment stays one sentence.

Scope findings become comments this way:

- **Intent creep:** a low `optional:` comment asking whether to split it into
  its own PR.
- **Scar tissue:** a comment asking whether to remove it here or in a
  follow-up.
- **Sidetracks in code the PR doesn't touch:** report them to the user only.
  They never become comments.

Follow-ups belong to the author, so nothing goes into the backlog.

### 6. Draft the batch

Write the review to `<tmp>/review.json`:

```json
{
  "commit_id": "<head-sha>",
  "event": "COMMENT",
  "body": "<findings with no diff line, or empty>",
  "comments": [
    { "path": "pkg/x.go", "line": 42, "side": "RIGHT", "body": "..." },
    { "path": "pkg/y.go", "start_line": 10, "start_side": "RIGHT",
      "line": 14, "side": "RIGHT", "body": "..." }
  ]
}
```

- **Anchor on the new side:** `side: RIGHT` and the file's line number there,
  plus `start_line` for a range. Use `LEFT` only for a removed line.
- **Every line must sit inside a diff hunk.** One bad line returns 422, and
  nothing posts.
- A finding with no line in the diff goes in the body as `path: sentence`.
  Keep the body to those findings. No summary of the PR. Praise, if any, is
  one separate line.

Show the whole batch and ask: **Post**, **Cancel**, or "Other" to edit.

### 7. Post

Check the head again with `gh pr view <N> -R <owner>/<repo> --json headRefOid`.
If it moved, tell the user, re-anchor against the new head, and ask again.
Then post the batch:

```bash
gh api --method POST repos/<owner>/<repo>/pulls/<N>/reviews --input <tmp>/review.json
```

One call, because comments posted one by one hit secondary rate limits. The
response holds the review's `node_id` (`PRR_…`) and `html_url`.

### 8. Notes

List the PR's review threads (plumbing/github) to get each new comment's
thread ID (`PRRT_…`). Write one note per thread with
`backlog note-set <owner>/<repo>#<N> <thread-id> '<body>'`, as
plumbing/backlog describes. A note holds the severity, the full finding and
its evidence, the reasoning, the fix you would accept, and `<head-sha>`. A
finding in the review body has no thread: key its note by the review's
`node_id`.

### 9. Report

The review's URL, the number of comments posted and skipped, and any
sidetracks. Then remove the clone.

## Resume mode

### 1. Read the source

Follow plumbing/github to identify the PR and make `<tmp>`. Load the review
threads and keep those whose first comment the viewer wrote. If the PR is
merged or closed, delete all its notes with
`backlog note-rm <owner>/<repo>#<N>` and stop. In plan mode, here and in
step 2, list the notes you would delete instead of deleting them.

### 2. Match the notes

Load this PR's notes with `backlog note-list` and `note-get`.

- For a `PRRT_` key, delete the note when GitHub shows its thread resolved
  or gone (`note-rm`). Leave other keys alone: they belong to review bodies,
  which have no thread state.
- A thread without a note: work from its comment alone and say so.
- If the head moved since the note's SHA, offer a review of only the new
  commits: review mode, diffing from that SHA in a fresh PR-mode clone.

### 3. Walk the open threads

Take each open thread with a reply or a change to its lines since the user's
last comment. Show:

- the comment and its note;
- the replies since;
- the code at the current head, with the diff since the note's SHA.

Then give your take: does the reply or change address the finding? If the
author disagrees, is the author right?

Draft a reply to the same review-comment rules. Decide each thread alone, as
the decision gate describes:
**Reply**, **Reply and resolve**, **Resolve**, **Skip**. Offer resolve only
when the finding is fixed or the user accepts the author's reasoning, and
only when the thread's `viewerCanResolve` is true.

### 4. Act

Post each approved reply as plumbing/github describes. Resolve each approved
thread:

```bash
gh api graphql -f threadId='<PRRT_id>' -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) { thread { isResolved } }
}'
```

Record the conclusion in the note, and delete the note once its thread is
resolved.

### 5. Report

Replied, resolved, and still waiting on the author.

# Shared rules

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/review`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/problem`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment review/engine`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment lenses/tests`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/changeset`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/github`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`
