# Issue format

Rules, before anything else:

- **Search for duplicates before drafting.** Never file over an open match
  without the user's say-so.
- **Show the full draft and ask before creating.** An issue is visible to the
  whole team the moment it exists.
- **Never invent repro steps, versions or error text.** Ask for what is
  missing, or mark the gap in the draft.

An issue is read by someone without your context, often months later. Write
it to the authoring bar, including its issue delta.

## Before drafting

Run each `gh` command as its own plain call. Search terms and titles go in
single quotes, never double quotes. For another repo, pass `-R <owner/name>`
to `gh issue` and `gh label` commands. `gh repo view` takes the repo as a
positional argument instead.

1. Find the repo: `gh repo view --json nameWithOwner -q .nameWithOwner`, or
   `gh repo view <owner/name> --json nameWithOwner` to confirm another one.
2. Search for duplicates, open and closed, with two phrasings: the symptom in
   the user's words, then the symbol or error text.
   `gh issue list --search '<two or three key terms>' --state all --limit 20`
3. Read the body of each close match with `gh issue view <N>`.
   - **Open match:** show it and ask whether to comment on it instead.
   - **Closed match:** link it in the new draft ("Possibly a regression of
     #N").
4. Read the labels with `gh label list`. Use only labels that exist.

## Title

The title names the symptom or the capability, concretely, in under 70
characters. "Export drops rows after 10k", not "Export bug". No type prefix
unless the repo's existing issues use one.

## Bug

```
<One sentence: what breaks, where, and how badly.>

## Steps to reproduce
1. <exact command, input or click>
2. …

## Expected
<what should happen>

## Actual
<what happens, with the exact error or log lines in a code block>

## Environment
<version or commit SHA, OS, runtime version, relevant config>

## Notes
<optional: suspected cause, labelled as a guess; workaround if any>
```

## Feature

```
<One sentence: who can't do what today.>

## Problem
<the situation and its cost; a concrete example>

## Proposal
<the smallest change that solves it>

## Non-goals
<what this deliberately leaves out>

## Done when
<an observable criterion someone can check>
```

Lead with the problem, not the solution. A reader who disagrees with the
proposal can still act on the problem.

## Enhancement

```
<One sentence: what improves and for whom.>

## Current behaviour
<what it does now, with an example>

## Desired behaviour
<what it should do instead>

## Why
<the consequence of leaving it: time lost, errors, confusion>
```

## Creating it

Show the title, labels and full body as they will be posted. Ask with
AskUserQuestion: **Create**, **Edit** or **Cancel**. Put the title, labels and
body in the `preview` of every option, so the draft stays visible while the
user chooses. On Edit, apply the changes and show the draft again.

On Create, write the body to `<tmp>/issue.md`, where `<tmp>` is the workflow's
`mktemp -d` directory (run `mktemp -d` once if there is none). Then:

```
gh issue create --title '<title>' --body-file <tmp>/issue.md [--label '<label>']
```

Write an apostrophe in the title as `'\''`. `gh` prints the URL, which ends in
the issue number.
