# Changeset

Pin down exactly what is under review or being committed before judging it.
Run plain `git <cmd>` in the user's tree.

## Base and fork point

If `git rev-parse --verify HEAD` fails, the repo has no commits yet. Then
everything is pending work: review the untracked files and the index, and
skip the fork point, the commit list and history checks.

Pending work compares against `HEAD`, so skip the rest of this section for it.

1. `git symbolic-ref --short refs/remotes/origin/HEAD` gives `origin/<default>`.
2. If that fails, read the default branch with
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`, or
   `git remote show origin` and its `HEAD branch` line.
3. If both fail, use whichever of `origin/main` and `origin/master` exists.
4. Ask the user when the answer is ambiguous: both exist, an `upstream` remote
   exists (a fork), or the branch was cut from another feature branch.

Then run `git merge-base <base> HEAD` once. Write the SHA it prints, quoted,
wherever `<fork-sha>` appears below. Don't use `merge-base --fork-point`: it
depends on reflogs and fails on fresh clones.

## The diff

Pick the diff that matches the job:

- **Whole branch.** `git log --oneline --reverse "<fork-sha>"..HEAD` lists the
  commits. `git diff "<fork-sha>"` covers committed, staged and unstaged work.
  `<base>...HEAD` drops the working tree, so don't use it here.
- **Pending work.** `git status --short`, `git diff` and `git diff --staged`.

Read the diff in full, then the surrounding code for anything non-trivial.
The diff doesn't show what other callers expect or what local conventions are.

## Untracked files

A `??` entry from `git status --short` appears in no diff. List them with
`git ls-files --others --exclude-standard` and read each one. If a file looks
like part of the change, say so before going further: it is otherwise missing
from the review and from the commit.

## Debris

Flag leftovers the author did not mean to ship:

- stray prints and debug logging (`fmt.Println`, `console.log`, `dbg!`);
- skipped or focused tests (`t.Skip`, `.only`, `xit`, `@skip`) added to get
  green;
- `replace` directives in `go.mod` that point at a local path;
- hardcoded local endpoints, ports or credentials (`localhost`, `127.0.0.1`);
- commented-out code blocks;
- temporary files, such as scratch scripts or `.orig` and `.rej` files.
