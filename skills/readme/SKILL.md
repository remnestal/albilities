---
name: readme
description: Write or review a repository's README.md so a reader can tell what the project is, install it and run one working example, with every command and path checked against the repo. Not for CLAUDE.md, AGENTS.md or other contributor docs.
argument-hint: "[write | review] [path to README, default ./README.md]"
disable-model-invocation: true
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(git ls-files *), Bash(git remote -v), Bash(ls *)
---

# README

**Gates, in order:**

1. **Mode.** `write` drafts a new README or rewrites one. `review` judges an
   existing one and changes nothing until findings are accepted. With no
   argument, review if a README exists, else write. The target is the path
   in the arguments, else `./README.md`.
2. **Problem statement.** Run the problem-statement gate below. Its sources
   are the user's request, the existing README, and the repo's manifest and
   docs. The consumer is the README's reader: settle who they are, because
   the reader decides what goes in.
3. **House style.** Read the repo's other docs, and any READMEs of the user's
   other projects they point you to. House style wins on order, headings and
   tone. Missing install steps are still a finding; a missing example is a
   low one.
4. **Every claim is checked against the repo.** A command, path, flag or
   version goes into the README only after you have seen it exist: on disk
   (`ls`, or `git ls-files --cached --others --exclude-standard`, which also
   lists untracked files), in the manifest, a Makefile or task file, or
   `--help` output. A claim about behaviour (what happens, what is asked,
   what is safe) is checked by reading the skill or script that implements
   it. A claim you can't check is left out or asked about. Run each check
   as one plain command per call, so the pre-approvals match.
5. **Show before writing.** Present the draft or the findings and get the
   user's go-ahead before touching the file. Never commit.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/problem`

## Shape

A README answers the reader's questions in the order they ask them:

1. **What is it?** The first line, one sentence.
2. **Why would I use it?** At most one short paragraph, only if the first line
   doesn't settle it.
3. **How do I install it?** Prerequisites, then steps that work when pasted.
4. **Show me.** One example that works when pasted, with its real output if
   it is short.
5. **What else is there?** Only what this reader needs next: the list of
   commands or skills, configuration, where the deeper docs are. Link to
   docs instead of repeating them.

Leave out:
- history of the project or of the work that produced it;
- badges and a table of contents, unless the user asks. When the repo has a
  LICENCE or CONTRIBUTING file, link to it in one line instead of restating
  it;
- anything a contributor needs but this reader doesn't. That goes in the
  contributor or agent docs, and the README links there.

## Write

1. Gather facts: `git ls-files`, the manifest, the entry points, how it is
   installed and run, and the existing docs to link to.
2. Draft to the shape above and the authoring bar below.
3. Check every command and path in the draft against the repo (gate 4).
4. Show the draft, apply the user's corrections, then write the file.

## Review

Judge the README against the shape, the authoring bar and the repo:

- **Stale or false:** a command, path, flag, version or name that doesn't
  match the repo. These are the findings that matter most.
- **Unreachable:** install steps or the example fail when pasted, or depend on
  something the prerequisites don't name.
- **Misplaced:** content for a different reader, duplication of another doc
  that should be a link, history or provenance.
- **Bar:** the first line doesn't say what the project is, or the text breaks
  a rule of the authoring bar below.

Decide each finding with the user as the decision gate describes. Severity
follows the review bar: stale, false or unreachable is high or medium;
misplaced and bar findings are low. Options: **Accept & fix** (recommended
for high and medium) and **Skip**. Apply each accepted fix before the next
finding, then show the result.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/review`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`
