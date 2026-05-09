## Problem-statement gate

Before judging or planning the work, find four things in the sources the
skill names:

1. **Problem:** what is wrong or missing, in observable terms.
2. **Consumer:** who or what uses the change.
3. **Done-criterion:** the test, metric or behaviour that shows it is solved.
4. **Non-goals and constraints:** what is deliberately left out.

Always show all four as formatted text, one line each, in this order. When
the sources don't state a part, write the one you infer and end its line
with `(inferred)`. Never leave a part out.

```
**Problem:** `/export` times out on accounts with over 10k rows.
**Consumer:** the support team running exports for customers. (inferred)
**Done-criterion:** a 50k-row export finishes in under 30 seconds.
**Non-goals and constraints:** no change to the CSV format. (inferred)
```

If no part is inferred, go on. Otherwise ask once with `AskUserQuestion`
whether the inferred parts are right: **Confirm** keeps them, and **Correct**
takes the user's text for each part they change. One question covers all
inferred parts. An inferred part is a guess until the user confirms it, so
act on none of them before the answer.
