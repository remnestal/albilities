## Problem statement

Run the problem-statement gate before any lens. If the user neither confirms
nor corrects the inferred parts, review without a statement: skip the
traceability and intent-creep checks and say so in the report.

## Orientation

Read the commit list, the diffstat and the problem statement yourself. When
the review fans out, the passes read the diff, not you. Whoever runs a lens
also reads the code around each hunk.

`<base>` below is the diff base the skill resolved, written literally: the
fork SHA for your own branch, `origin/<baseRefName>` for a PR.

**Dependency lookups go through the `LSP` tool** (load it with `ToolSearch` if
it isn't listed). `hover` gives a symbol's signature and doc, `goToDefinition`
reaches its source, and `findReferences` lists its callers. Read dependency
source from the module cache only when LSP can't answer. Never vendor or
download dependencies into the tree.

**Scope.** Rules apply to new and changed code. A problem in code the diff
doesn't touch is a finding only when the change makes it worse or newly
reachable. Code the change makes unused or wrong counts as made worse.
Anything else is a sidetrack, one line each. The skill decides what happens
to sidetracks.

## Lenses

### Conventions

Does the change look like the code around it? Compare naming, error handling,
package layout, logging and test style with neighbouring files and with
`git log -p` on the touched paths, not with a general ideal.

Namespace and environment consistency: new env vars, config keys, flags,
metric names, log fields, routes and package names follow the existing prefix
and casing. One concept keeps one name across code, config, deploy manifests
and docs. A default set in code matches the one in config and docs.

### Structure

Logic in the wrong layer, a function doing two jobs, or a new helper that
duplicates one the repo already has (search for it).

Scar tissue is what the change leaves behind: code, parameters, flags, config
keys or helpers it made unused (confirm with `findReferences`), shims with no
caller left, workarounds for a cause it fixed, and debris. Propose removal.
The user confirms before anything is deleted.

### Security and runtime

Injection, missing authorization, secrets in code or logs, untrusted input
crossing a trust boundary unvalidated. Nil dereferences, ignored errors,
leaked goroutines, connections or file handles, N+1 queries, missing timeouts,
migrations that lock tables or break the previous release.

Limits and overflow: integer truncation on conversion, arithmetic that can
overflow, reads without a size cap, unbounded growth of maps, queues, retries
or result sets, missing pagination, and off-by-one at a limit.

Leave races and consistency to the next lens.

### Concurrency and consistency

Races and deadlocks: shared state reached from two goroutines or threads
without a guard; a lock held across a blocking call; two locks taken in
different orders; a mutex, or a struct holding one, copied by value; an unlock
deferred inside a loop; a wait group or worker pool that can leak or send on a
closed channel; a cancellation nobody observes. Name the two paths that reach
the state, or drop the finding. Note whether CI runs the touched packages with
a race detector. Don't run it yourself.

Read-after-write: report only patterns the diff shows, and phrase each as a
question. Examples: a write to the primary followed by a read from a replica; a
response built from a re-read instead of what was written; a cache invalidated
before the database write commits; a job or message published before its
transaction commits.

Event hydration is the shape most often missed. A consumer takes an event and
calls another service to fetch the data it refers to, which may not be readable
yet. Ask what happens on "not found": retry with backoff, redeliver, drop, or
write a wrong result. Treating it as final is the finding. An early return that
neither acks nor nacks a broker message leaks prefetch.

### Design

Dependencies injected rather than built inline or reached as globals.
Interfaces declared by the consumer. Configuration that should be a parameter.
Code that can't be tested without its real dependencies.

**Intent creep.** Changes that serve no clause of the problem statement:
features, refactors or renames it doesn't need. List each by name, apart from
the findings. The skill decides how intent creep is handled.

**Traceability ratio.** List every new abstraction, interface, option, config
key, extension point and file. Map each to a clause of the problem or its
constraints. Report the untraced items by name, plus the ratio (for example
"3 of 7 untraced"). Signals: an interface with one implementation and no
substitute; an option no caller or test sets; a `switch` with one case; a
helper extracted from two call sites with flags covering their differences; a
layer that only forwards; generics, reflection, codegen or concurrency the
problem never asks for; names like `Generic`, `Pluggable`, `Registry`, `v2`.
Don't flag what traces to a stated constraint, what adds no complexity, an
option tests exercise, a public API with known external users, or an interface
a test double uses.

**Forking paths.** Read the branch history (`git log -p <base>..HEAD`) and ask
which assumption failed and whether the design was rethought or patched.
Signals: a later commit editing lines an earlier commit on the branch added; a
revert followed by a re-add; a new guard or special case per bug; churn large
relative to the final diff. Ask, don't assert: these signals are often wrong.
Don't flag test-first red/green cycles, review rounds, commits framed as a
spike, or squashed history. Also flag a costly approach where an obviously
simpler one exists and nothing says why it lost.

**Chesterton's fence.** A deleted guard, retry or special case with no stated
reason. Check `git log -S` or blame for why it was added. Flag it unless that
reason clearly no longer holds.

### Prose

A comment earns its place by saying what the code can't: a constraint not
visible locally, a non-obvious invariant, a link to a spec or issue. Flag
comments that narrate the next line, restate the signature, speculate about
the future, carry a TODO with no owner or issue, or drift to neighbouring
topics. Flag a doc comment that no longer matches the code, or that spells out
obvious parameters but omits the error or concurrency contract. A missing
comment is a finding where load-bearing reasoning has no record.

The same bar covers markdown the diff touches. A doc the change makes wrong is
a finding even if the diff doesn't touch it. Group small prose findings into
one finding per file.

### Tests

Apply the tests lens to the changed tests, and its mutant question to the
changed production code.

## Fan-out

Count changed lines with `git diff --numstat <base>` for your own branch, or
`git diff --numstat origin/<baseRefName>...HEAD` in the PR's clone. For your
own branch, add the lines of new untracked files: list them with
`git ls-files --others --exclude-standard`, then `wc -l` each. Leave out
generated files, lockfiles, vendored code and test fixtures.

Then ask the user, with `AskUserQuestion`, which review to run. Name the size
and any risky areas the diff touches: concurrency, timeouts or I/O, auth or
secrets, persistence or migrations, a public API, removed guards.

- **Fast:** you run every lens yourself in one pass. No subagents.
  Recommend it for a small diff that touches no risky area.
- **Thorough:** three `general-purpose` subagents, one per lens group,
  launched in one message so they run in parallel. Don't use `Explore`; it
  samples instead of auditing. Recommend it whenever a risky area is touched.
  - *Runtime:* security and runtime; concurrency and consistency.
  - *Design:* design; structure; conventions.
  - *Prose and tests:* prose; the tests lens.

  Leave out a group with nothing to judge: runtime needs executable code,
  prose and tests needs comments, docs or tests. Name the skipped group in
  the report.

A fast review ends with a one-line recommendation: run a thorough pass or
not, and why. Name the risky areas or the uncertain findings behind it.

Over about 2,000 changed lines, offer first to review per commit or per
directory.

Every subagent prompt carries:

- the absolute repo path and the exact diff command, with SHAs resolved;
- the problem statement, or "none confirmed";
- its lens sections, copied word for word from above, and the tests lens for
  the prose and tests pass;
- the nitpick bar and severity levels, copied word for word;
- the dependency-lookup and scope rules above;
- **read-only:** read files and run read-only git and `gh` commands only.
  Never edit, format, stage, commit, stash, check out, push, post, or run
  builds, tests or code generation;
- the return format: per finding, the lens, `path:line` on the new side of the
  diff, severity, a one-sentence hook, the evidence (the code path, or both
  concurrent paths), an explanation only if the hook can't carry it, and the
  fix. A clean lens returns "clean" and nothing else.

## Consolidation

Do this yourself before the user sees anything.

1. **Rerun failures.** If a pass failed or returned nothing usable, run its
   lenses yourself.
2. **Merge duplicates.** Keep the clearer framing of the two.
3. **Verify load-bearing claims.** Read the hunk yourself for every high
   finding, every race or deadlock (confirm both paths run concurrently and
   share the state), every claim that code is unused, and every Chesterton's
   fence. Drop what doesn't survive.
4. **Re-apply the nitpick bar** to the merged list. A low finding that passed
   alone often fails among a dozen.
5. **Note the clean lenses,** so the user sees the coverage even when the list
   is short.

## Finding gate

Open with one line: the count per severity and the clean lenses. Then take
the findings worst first, one at a time, as the decision gate describes. Each
carries `path:line`, severity, the problem, why it matters and the proposed
fix. The skill defines the options. Move on only when every finding is
decided.

After the gate, list the sidetracks and intent creep, one line each, for the
skill to handle.
