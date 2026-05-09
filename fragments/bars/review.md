## Severity

Give every deferred item exactly one level.

- **High:** must be fixed before merge. Wrong results, data loss, a crash or
  hang, a security hole, a reachable race or leak, or a broken public contract.
- **Medium:** should be fixed in this change. A real defect on a rare path, a
  missing test for behaviour callers rely on, a design choice that will cost
  more to change after merge, or a doc comment that now lies.
- **Low:** clears the bar but can wait. Clearer structure, a missing doc
  comment, a weak test, a convention drift. It never blocks a merge.

When unsure between two levels, pick the lower and say what would raise it.
