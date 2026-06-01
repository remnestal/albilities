---
name: pr
description: Open a GitHub pull request from the commits on the current branch. Use when asked to open, create, raise or push up a PR, to ship the branch for review, or to put the work in front of a colleague. Not for a plain git push, questions about an existing PR, checking CI, or addressing review comments.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git branch --show-current), Bash(gh pr view *), Bash(gh issue view *), Bash(mktemp -d), Bash(git worktree add *), Bash(git worktree remove *), Bash(gofumpt -d *)
---

# Open a pull request

## Gates

- **Create nothing until the commit range passes the atomicity check** or the
  user overrides it.
- **Ship commits, not the working tree.** Uncommitted changes go through the
  `albilities:commit` skill first, or stay out of the PR.
- **Don't add reviewers, labels, assignees or a draft flag** unless asked.
- **On the default branch, stop** and ask for a branch name.
- **Plan mode:** stop before the first write, as gates/plan-mode says.

## 1. Gather state

Each Bash call is a fresh shell. Run one plain command per call, with no
variables, pipes or `&&`, and write resolved values literally.

Find the base as plumbing/changeset describes. `<base>` below is the bare
branch name, without `origin/`. Make a temp directory with `mktemp -d` and
reuse its literal path as `<tmp>`. Then:

```bash
git branch --show-current
git status --porcelain
git log --reverse --format='%h %s' origin/<base>..HEAD
git diff origin/<base>...HEAD
gh pr view --json url,state
```

Read the full diff, not only the stat. `gh pr view` failing with "no pull
requests found" means there is none yet. If the branch already has an open
PR, report its URL and stop.

## 2. Clean tree

If `git status --porcelain` prints anything, ask whether those changes belong
in this PR.

- **Yes:** invoke the `albilities:commit` skill with the Skill tool. When it
  returns, resume at step 3 here. The new commits change the range, so the
  atomicity check runs on the final set.
- **No:** leave them alone. A push sends commits only.

## 3. Atomicity check

The range is atomic when it has one reviewable purpose, could be reverted as a
unit, and every commit serves that purpose. It is not atomic when it holds:

- two unrelated features or fixes;
- a feature plus a refactor it doesn't need;
- changes to components that don't depend on each other;
- work that needs two different reviewers.

If it is not atomic, stop and put the split to the user as one gates/decide
item, with no severity in the heading:

- **Where:** the commit range.
- **Problem:** the purposes it mixes, each with its commits.
- **Why it matters:** what a reviewer or a revert would have to untangle.
- **Fix:** which commits go in which PR, in dependency order, a title for
  each, and which PR must merge first.

The options are **Split as proposed** (Recommended) and **Keep one PR**. Act
after the answer. For a stack, open each PR against its predecessor's branch
and put `Depends on #N` as the first line of the dependent PR's body.

## 4. Title

- **One commit:** use its subject unchanged.
- **Several:** write a conventional summary, `type(scope): subject`, following
  the commit-subject delta in bars/authoring.

A title that needs "and" means step 3 missed something. Go back to it.

## 5. Description

Follow the PR-description delta in bars/authoring:

1. **Hook paragraph.** What the change does and which problem it solves.
2. **One more paragraph, only if earned.** It must clear a gotcha, a
   surprising choice or a constraint that a reviewer would otherwise flag.
3. **`Closes #N`,** one line per issue. Take the issue link (`#N`) from the
   branch scope that `backlog scope` prints (plumbing/backlog). Before
   writing it, run `gh issue view <N>` and check that the issue is the work
   the scope describes. If it isn't, ask. With no link, leave the line out.

Cut how the work came about and file-by-file walkthroughs. Write the body to
`<tmp>/body.md` with the Write tool.

## 6. Push

```bash
git push -u origin HEAD
```

If the push is rejected, report the error and stop. Never force-push here.

## 7. Create

```bash
gh pr create --base <base> --title '<title>' --body-file <tmp>/body.md
```

Keep the title in single quotes. Write a `'` inside it as `'\''`.

If `gh` fails on authentication, point to `gh auth login`. Don't build the PR
another way.

## 8. Record and report

Record the PR on the branch scope with `backlog scope set --pr <N>`
(plumbing/backlog). If `backlog scope` printed nothing, pass the PR title as
the scope text, since `scope set` needs text when no scope exists. Report the
PR URL.

## Reference

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/changeset`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`
