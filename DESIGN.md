# Design

How the skills are split, how they connect, and why. Each `SKILL.md` holds its
own procedure; this file holds what spans them.

## Principles

1. **A skill has its own trigger or its own side effect.** Everything else is
   a fragment in `fragments/`.
2. **A fragment needs two users.** Content only one skill uses lives in that
   skill.
3. **Fragments arrive through `bin/include-fragment`,** run as a pre-approved
   `!` command, because Claude Code blocks `cat` outside the working
   directory. A missing fragment aborts the skill, so `bin/check-plugin`
   checks every reference. A fragment a skill needs only sometimes is loaded
   on demand with the same script.
4. **Few skills trigger on their own.** Their descriptions share a small
   context budget, so only `commit`, `pr-open`, `go` and `test` are
   model-invoked. The rest are user-only and cost nothing until called.
5. **A skill calls another only at a handoff:** `pr-open` hands a dirty tree to
   `commit`, then resumes. Shared logic is a fragment, never a skill call.
6. **Decisions go to the user one at a time** (`gates/decide`), and nothing is
   written while plan mode is on (`gates/plan-mode`).
7. **Gates sit at the top of a skill,** since only its first 5k tokens survive
   compaction.
8. **Built-ins are suggested, never wrapped.** `/self-review` ends by pointing
   to `/code-review`.

## Skills

| Skill | Does | Invoked by |
|---|---|---|
| `commit` | Turns uncommitted work into atomic commits and asks about anything unrelated | Claude |
| `pr-open` | Opens a pull request from the branch's commits, after showing what goes up | Claude |
| `go` | Loads the Go rules for the code being written | Claude |
| `test` | Plans and writes tests from what the code must do | Claude |
| `backlog` | Records a branch's scope, milestones and deferred items, and resumes from them | you |
| `issue` | Files a bug, feature or enhancement | you |
| `self-review` | Reviews your own branch; you fix, defer or skip each finding | you |
| `pr-review` | Reviews someone else's pull request and posts one batched review | you |
| `pr-feedback` | Answers comments on your pull request, one fixup per accepted fix | you |
| `pr-finalize` | Squashes the fixups and gets your pull request ready to merge | you |
| `readme` | Writes or reviews a README against the repo | you |

## How they connect

```
pr-open ──dirty tree──▶ commit ──▶ back to pr-open
commit, self-review ──follow-ups, deferred findings──▶ bin/backlog
self-review, pr-review ──▶ review/engine ──▶ lenses (+ go/* for Go)
pr-open ──▶ review comments ──▶ pr-feedback ──fixups──▶ pr-finalize
SessionStart hook ──▶ open backlog items + the user-only skills
```

## Fragments

- `gates/`: `decide` (how a decision is put to the user), `plan-mode` (when a
  skill may write), `problem` (the problem statement a skill needs first).
- `bars/`: `authoring` (the writing bar, per kind of text), `review` (the
  nitpick bar and severities).
- `review/engine` and `lenses/tests`: the review passes.
- `go/`: `design`, `doc`, `test`, `ctx-close`.
- `formats/issue`: issue templates.
- `plumbing/`: `changeset` (what is being reviewed or committed), `backlog`
  (the store), `github` (pull requests and review threads).

## Decisions

- **GitHub owns pull request state.** Local notes keep only the reasoning
  behind a terse comment, and a note goes once GitHub shows its thread done.
  On your own pull requests, the commenter resolves threads, never you.
- **`pr-finalize` never merges.** Whether a tree-identical force-push keeps
  approvals is undocumented, so it warns before pushing and rechecks after.
- **The backlog lives outside the repo,** in `~/.local/state/albilities`,
  shared by every worktree. Plan mode never writes to it; a plan keeps its own
  follow-ups.
- **Every commit is checked before it is made:** the whole module is built and
  vetted, and the touched packages are tested, in a scratch worktree.
- **Formatting is detected repo-wide and rewritten narrowly.** `gofumpt -l .`
  finds scout work; `gofumpt -w` runs only on touched files.
- **`self-review` and `pr-review` are separate skills** because one edits and
  the other posts. `allowed-tools` doesn't restrict tools, so the split shapes
  the workflow; it is no guarantee.
- **The review engine sorts sidetracks with a short scope rule,** not commit's
  gate, whose parking steps don't fit a review.
- **Language rules apply to new or changed code only.** Existing violations
  are sidetracks.

## Open

- Real-world trials, starting with `commit` and `/self-review`.
- A regression harness: a seeded diff with known findings, one test-authoring
  case, and trigger phrases for each skill Claude invokes.
- Not installed automatically: the rule in `rules/sidetrack.md`, allow rules
  (including the pull request clones in `~/.cache/albilities/pr/`), and a
  permanent plugin registration.
- Only if needed: a hook that injects the branch scope, and a
  `design-research` skill.
