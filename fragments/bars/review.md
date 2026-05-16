## Nitpick bar

A finding must change behaviour, risk, or how easily someone can maintain the
code. Drop everything else before it reaches the user:

- formatting a formatter or linter would catch;
- naming you merely prefer;
- hypotheticals with no path to occurring;
- restyling code the change doesn't touch.

**"No path" means no path, not "hasn't failed yet."** A latent race, lock-order
inversion, or leaked goroutine, connection or file handle is a finding once two
callers can reach it, even with nothing observed failing. Name the paths that
reach it. If you can't, drop it.

The carve-out covers concurrency and leaks only. It is no licence for
speculating about topology: a read-after-write or ordering finding needs the
pattern visible in the code under review.

## Severity

Give every finding or deferred item exactly one level.

- **High:** must be fixed before merge. Wrong results, data loss, a crash or
  hang, a security hole, a reachable race or leak, or a broken public contract.
- **Medium:** should be fixed in this change. A real defect on a rare path, a
  missing test for behaviour callers rely on, a design choice that will cost
  more to change after merge, or a doc comment that now lies.
- **Low:** clears the bar but can wait. Clearer structure, a missing doc
  comment, a weak test, a convention drift. It never blocks a merge.

When unsure between two levels, pick the lower and say what would raise it.
