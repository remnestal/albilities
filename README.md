# albilities

Claude Code skills for the loop around a change: plan the branch, write the
code and tests, commit, open the PR, review, answer feedback, and get it
ready to merge.

## Usage

Install it from a clone, outside the plugin marketplace:

```sh
git clone https://github.com/remnestal/albilities.git ~/projects/albilities
claude --plugin-dir ~/projects/albilities
```

The flag loads the plugin for that session only, so pass it on every launch.
An alias does that for you:

```sh
alias claude='claude --plugin-dir ~/projects/albilities'
```

It loads straight from the clone, so a `git pull` takes effect in the next
session.

Then, in any repo with work in progress, type `/self-review`.

Needs `git` and a POSIX shell, plus `gh`, logged in, for anything that
touches GitHub. Go repos also need `gofumpt`.

## Skills

Claude reaches for these on its own when they fit:

| Skill | Description |
|-------|-------------|
| [`commit`](./skills/commit) | Splits uncommitted work into atomic commits, and asks about anything unrelated: skip it, commit it separately, or keep it for later |
| [`pr-open`](./skills/pr-open) | Opens a pull request from the branch's commits, after showing you what goes up |
| [`go`](./skills/go) | Go rules for design, doc comments, tests, contexts and closing |
| [`test`](./skills/test) | Plans and writes tests from what the code must do, and cuts tests that catch no bug |

You start these yourself:

| Skill | Description |
|-------|-------------|
| [`/backlog`](./skills/backlog) | `plan` records a branch's scope and milestones; `resume` shows what is left, asks which milestones are done, and drops items whose own issue or PR has closed |
| [`/self-review`](./skills/self-review) | Reviews your own branch, uncommitted work included; for each finding you choose to fix it now, defer it to the backlog, or skip it |
| [`/pr-review`](./skills/pr-review) | Reviews someone else's PR and posts one batched review; `resume` handles the replies |
| [`/pr-feedback`](./skills/pr-feedback) | Works through comments on your PR, one fixup commit per accepted fix |
| [`/pr-finalize`](./skills/pr-finalize) | Squashes the fixups once threads and CI allow, force-pushes after asking, and reports when the PR is ready to merge |
| [`/issue`](./skills/issue) | Files a bug, feature or enhancement after a duplicate search |
| [`/readme`](./skills/readme) | Writes or reviews a README |

## Design

- Nothing is posted, pushed or filed without asking first, and nothing is merged
- GitHub is the source of truth for PR state; local notes only keep the reasoning behind a terse comment
- The backlog lives outside the repo, by default in `~/.local/state/albilities`, shared by every worktree
- Shared rules live once, in `fragments/`, and reach each skill through `bin/include-fragment`

## Contributing

Conventions are in [`CLAUDE.md`](./CLAUDE.md). Run `bin/check-plugin` after any change.

[`rules/sidetrack.md`](./rules/sidetrack.md) is three lines to paste into your
own `CLAUDE.md` to have side tracks caught outside `commit` too.
