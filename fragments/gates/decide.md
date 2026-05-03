## Decision gate

Use this for every finding, sidetrack or thread the user has to decide. Take
one item at a time, worst first. Open with one line: how many items there are,
and their severities.

**1. Show the item as formatted text, before any prompt:**

```
### 2/5 · Stale clone URL · medium

**Where:** `README.md:9`
**Problem:** The clone URL uses SSH, so it fails for anyone without a GitHub key.
**Why it matters:** A peer's first command errors out.
**Fix:** Use `git clone https://github.com/<owner>/<repo>.git`.
**Question:** Keep SSH as a second option for yourself?
```

- **Where:** `path:line`, plus the hunk or a short quote when it helps.
- **Problem:** what is wrong, in one or two plain sentences.
- **Why it matters:** the consequence of leaving it. One sentence.
- **Fix:** the concrete change. Give the actual new text or code when it is
  short; never "handle this properly".
- **Question:** only when something beyond the options needs deciding. Leave
  the line out otherwise.

**2. Then ask about that item alone** with `AskUserQuestion`:

- The question is short and names the item: "Fix 2/5, stale clone URL?".
  Don't repeat the explanation in it.
- The options come from the skill. Put the recommended one first and mark it
  "(Recommended)". Each option's description says only what choosing it does.
- Never put two items in one call.

**3. Act, then move on.** If the skill acts per item, act before the next;
otherwise act after the gate, as the skill says. When the answer is "Other",
answer the question in text, then ask about the same item again. A skipped
item is dropped, not softened.
