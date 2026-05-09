---
name: backlog
description: Plan a branch's scope or resume work from the persistent backlog. `plan` checks the problem statement, then records scope, milestones, issue and PR links. `resume` prunes items whose issue or PR closed, asks which milestones are done, then shows milestones and items by severity. Not for the step-by-step plan of the current session (that is the Claude plan) or for filing an issue (use /issue).
disable-model-invocation: true
argument-hint: "plan [<what or #issue>] | resume"
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *), Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *), Bash(gh issue view *), Bash(gh pr view *), Bash(gh pr list *), Bash(git branch --list *), Bash(git branch --show-current), Bash(git log *)
---

# Backlog

Hard rules, before anything else:

- **Steps belong in the Claude plan, never in the backlog.**
- **In plan mode, `plan` drafts and stops.** The plan carries the exact
  `scope set` and `milestone add` calls as its first step, so approving the
  plan is what leaves plan mode and runs them.
- **In plan mode, `resume` lists what it would delete and waits.** It runs
  its read-only steps and lists the items it would prune, the finished
  branches and the milestones that look done. It deletes nothing and asks
  nothing until the user leaves plan mode.
- **Resume deletes items whose issue or PR closed without asking,** then
  reports each one. Any other delete needs the user's yes after seeing what
  goes.

The backlog fragment's plain-call rule applies to the gh and git calls too.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/plan-mode`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/decide`

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment plumbing/backlog`

## Choose the mode

The first word of `$ARGUMENTS` picks the mode. Without one, run `backlog list`
and `backlog count --all-branches`. Resume if `list` prints anything or the
count is above 0. Otherwise plan.

## Plan

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment gates/problem`

1. **Problem statement.** Apply the gate above to the user's words, the
   linked issue (`gh issue view <N>`) or the PR (`gh pr view <N>`). Go on only
   once the user confirms.
2. **Branch.** If you are on the default branch and the work gets its own
   branch, ask for its name and pass `--branch <name>` to every call. Scope
   set on the default branch goes silent once the work moves.
3. **Scope.** Write one line of the problem and the done-criterion, under 15
   words. Add `--issue N` for the issue this branch closes and `--pr N` once a
   PR exists: `backlog scope set [--issue N] [--pr N] '<text>'`. An existing
   scope is replaced only after the user confirms. Text replaces the links
   too, so pass again any `#N` or `!N` from the old line that still applies.
4. **Milestones.** Derive 3 to 5 outcomes from the done-criterion, one line
   each, every one checkable from the code or a test. Show them for the
   user's yes. Record each with `backlog milestone add '<text>'`. If the
   branch already has milestones, show them and ask which to keep.
5. **Deferred items.** A non-goal the user wants tracked becomes an item with
   a severity from the review bar
   (`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/review`). Anything else
   stays out.
6. **Write.** Outside plan mode, run the confirmed calls now. In plan mode,
   put them in the plan as described in the hard rules.
7. **Report** the scope, milestones and any items, then stop.

## Resume

Every question below follows the decision gate: one item per question, shown
first as formatted text. Each step collects its answers, then acts after its
last question, highest number first.

1. **Read.** `backlog list --all-branches`. If it prints nothing, say so and
   stop. Get the current branch with `git branch --show-current`.
2. **Closed links.** For each distinct `#N` and `!N` in an item's header,
   run `gh issue view <N> --json state` or `gh pr view <N> --json state`.
   Delete every item whose link is closed or merged, without asking:
   `backlog rm <n> --branch <branch>`, highest number first per branch.
   Report each deleted line.
3. **Finished branches.** Run `backlog list --all-branches` again for fresh
   numbers. For each other listed branch, take the PR from its scope header
   (`!N`), or run
   `gh pr list --head <branch> --state all --json number,state`. Check
   whether `git branch --list <branch>` still prints it. A branch is finished
   when its PR merged or closed, or the branch no longer exists. For each
   finished branch:
   - Ask once about the branch: **Where** is the branch name, **Problem** is
     "PR !N merged" or "branch deleted". Options: **Clear** (Recommended),
     which runs `scope clear` and `milestone done <n>` for each milestone,
     or **Keep**.
   - If the answer is Clear, ask once per item: **Move** to the current
     branch (`add` with its severity and links, then `rm`), or **Drop**
     (`rm`). Recommend Move unless the item no longer applies. A kept
     branch keeps its items.

   Pass `--branch <branch>` to every call on the finished branch.
4. **Milestones.** Run `backlog milestone list`. For each milestone on the
   current branch, check the code, the tests and `git log` for its outcome.
   Ask only about those whose work looks done. **Where** is the commit or
   file that shows it. The question is "m1 '<text>': done?", with **Done**
   (Recommended) and **Not yet**. Run `backlog milestone done <n>` for each
   Done, highest number first.
5. **Show what is left.** Run `backlog list` again. Show the current branch
   first: its scope, its milestones, then its items grouped high, medium, low.
   Other branches follow, one line each with their item count. Name a next
   step only when it is clear: the first open milestone.

To file an item as an issue, point the user to `/issue`.

!`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`
