---
name: issue
description: File a GitHub issue (bug, feature or enhancement) from the conversation or a backlog item, after a duplicate search and the user's approval of the draft.
argument-hint: "[what to file, or a backlog item number]"
disable-model-invocation: true
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(gh repo view *), Bash(gh issue list *), Bash(gh issue view *), Bash(gh label list *), Bash(git rev-parse *), Bash(mktemp -d)
---

# Issue

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment formats/issue`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`

## 1. Work out what to file

Read the arguments and the conversation. If the argument is a backlog item
number, run `backlog list` and read that item: its text, severity and links.
If its header already has `#N`, the item is filed: show
`gh issue view <N>` and stop. Otherwise the item text seeds the draft.

Pin down:

- **Type.** Bug (something breaks), feature (a capability that doesn't exist),
  or enhancement (an existing capability done better). If the type is unclear,
  infer it, state it in one line, and carry on. The user corrects it at the
  draft.
- **Repo.** The current repo unless the user names another, such as an
  upstream project.
- **Facts.** For a bug, gather the repro from what you can observe: run the
  failing command, copy the exact error, read the versions
  (`go version`, `git rev-parse --short HEAD`). Ask for anything you can't
  observe.

## 2. Search, draft and create

Follow the issue format above: duplicate search, the template for the type,
the draft question, then creation. Report the URL.

## 3. Track it

**Filed from a backlog item:** link the item to the new issue without asking.

1. Run `backlog list`. If an item already carries `#<N>`, stop: it is tracked.
2. Add the item again with its old severity, its old `!M` if any, and the
   issue:
   `backlog add --severity <old severity> --issue <N> [--pr <M>] '<old text>'`.
   Put text that starts with `-` after `--`. This replaces the old item, so
   the duplicate check doesn't apply.
3. Run `backlog list`, then `backlog rm <n>` on the old line, the one without
   `#<N>`.

**Filed from the conversation:** if the issue is deferred work for the
current branch, offer to track it there with
`backlog add --severity <high|medium|low> --issue <N> '<text>'`.
