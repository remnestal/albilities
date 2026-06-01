## Backlog store

The backlog keeps what must outlive a session. That covers each branch's scope
and milestones, deferred items with their issue and PR links, and the notes
behind PR review threads. It lives outside the repo. Worktrees of one repo
share the branch files. PR notes are keyed by `owner/repo#N` alone, so they
work from any directory.

Use only the backlog script, `${CLAUDE_PLUGIN_ROOT}/bin/backlog`. Where a skill
or fragment writes `backlog <command>`, it means this script. Call it as one
plain command per Bash call, with no variables, pipes, `&&` or loops, so the
pre-approval matches. Pass text in single quotes and reword to avoid an
apostrophe. Text starting with `-` goes after `--`. Never read, edit or create
store files yourself. The commands and line format are in `--help`. Don't show
the user the store path unless asked.

### When to write

- **In plan mode, don't write anything.** Collect follow-ups in the plan's
  "Follow-ups" section, one line each with its severity. Write them with the
  script after the user approves the plan. Reading is fine in plan mode.
- Add an item only when the user chose it: a follow-up for a sidetrack, a
  deferred review finding, or an explicit request. Never add one on your own
  initiative. Run `list` first and skip duplicates.
- Scope, milestones, links and PR notes are written by the workflow that owns
  them, as part of its job.

### Reading `list`

Every stored line starts with a header in brackets. Links exist only inside
the header: `#N` is an issue, `!N` a pull request. A `(#5)` in the text after
the header is plain text, never a link.

- `[scope #12 !34] <text>` says what the branch is for: the issue it closes
  and its PR. `scope` prints this line, or nothing when no scope is set.
- `m1. <text>` is a milestone.
- `1. [high !34] <text>` is an item, numbered for `rm`.

### Scope

- `scope set --pr N` or `--issue N` without text keeps the text and the other
  link. When no scope is set yet, pass text as well.
- `scope set` with text replaces the whole line, links included. Pass again
  every link from `scope` that should stay.
- Act on another branch with `--branch <name>`, for example when the scope is
  set from the default branch before the feature branch exists.

### Milestones

A branch has up to five one-line milestones: outcomes that mark progress
across sessions, not the steps of a plan. They are added when the branch is
planned. One is marked done only while resuming work, after the user confirms
it. Never add or remove milestones as a side effect of other work, such as a
commit.

### Items

- An item is a one-line reminder: what to do and where. Leave out history,
  reasoning and who found it.
- Give exactly one severity, using the levels defined in the review bar
  (bars/review).
- Link the item's own issue or PR, if it has one (`--issue N`, `--pr N`), so
  the item can be pruned once that closes.
- Done items are deleted with `rm <n>`. Numbers come from the latest `list`
  and shift after each delete, so delete the highest number first.

### PR notes

A note keeps the reasoning behind a terse review comment: the full finding,
the conclusion reached, and any fixup SHAs. It is enough to resume a thread
without working the finding out again.

- Key a note by `owner/repo#N` and the thread's node ID (`PRRT_...`). For a
  review-summary comment, which has no thread, use its comment ID.
- GitHub is the source of truth for thread state: resolved, outdated, who
  replied. Read it from GitHub every time and never store it in a note.
- Pass the body as one single-quoted argument, which may span lines:

  ```
  backlog note-set acme/api#12 PRRT_kwDOabc 'accepted: retry only on 503
  The 429 path already backs off in the client. Fixup 1a2b3c4.'
  ```

  `note-get` prints nothing when no note exists.
- Delete a note once GitHub shows its thread is done. After the PR merges or
  closes, `note-rm <owner/repo#N>` drops all its notes.
