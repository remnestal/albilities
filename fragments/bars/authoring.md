## Authoring bar

These rules cover all text written for a person. The genre deltas below adjust
them per kind of text.

1. **Lead with the point.** The first sentence carries what the reader needs.
   Bad: "I was digging through the cache code and found something odd."
   Good: "The cache never evicts, so memory grows until the pod restarts."
2. **Report the result, leave out the journey.** Skip attempts, discussions
   and the order things happened in.
   Bad: "After trying a channel first, I ended up switching to a mutex."
   Good: "A mutex guards the counter."
3. **One idea per sentence, 25 words at most.**
   Bad: "The file is opened twice and the second handle is never closed, so it leaks."
   Good: "The second handle is never closed. Each call leaks a descriptor."
4. **Cut words that carry nothing.**
   Bad: "in order to perform validation of the input"
   Good: "to validate the input"
5. **Give the reason as its consequence.** Skip the general lesson behind it.
   Bad: "Slices share a backing array, so as a rule you should copy before..."
   Good: "This append overwrites the caller's data."
6. **Drop what the reader already has:** the diff, the file list, their question.
   Bad: "This PR touches three files: a.go, b.go and c.go."
   Good: (delete the sentence)
7. **Be concrete.** Name the exact symbol, value or error message.
   Bad: "Improve error handling."
   Good: "Wrap ErrNotFound with the missing key."
8. **Use active voice and everyday words.**
   Bad: "Retries are performed by the client."
   Good: "The client retries."
9. **Commit to claims.** Label a real guess once, as a guess.
   Bad: "This could perhaps potentially lead to some issues."
   Good: "This deadlocks when both requests hold a lock."
10. **Stop when done.** No recap, offer or sign-off.
    Bad: "Hope this helps! Happy to tweak anything."
    Good: (end)

### LLM tics to cut

- Em dashes. Use a period, comma, colon or parentheses.
- Lists of three written for rhythm.
- "Not X, but Y" and "It's not just X" framings.
- Trailing "-ing" clauses such as ", ensuring consistency".
- Inflated words: crucial, robust, seamless, leverage, delve, comprehensive.
- Filler openers: "It's worth noting", "Importantly", "Notably".
- Signposts such as "Here's what changed:".
- Restating the question before the answer.
- Hedges on plain facts.
- "Serves as" or "stands as" where "is" works.

### Genre deltas

**Commit.**
- Subject: Conventional Commits form, `type(scope): subject`. Imperative,
  lowercase, no trailing period.
- House style caps the subject at 50 characters, counting `type(scope): `.
  The sources set no such hard limit.
- Bad: `Fix(auth): Fixed the bug where the tokens were expiring too late.`
  Good: `fix(auth): reject tokens past their expiry`
- The body says why the change exists and what was wrong before. The diff
  shows how, so leave that out.

**PR description.**
- The first paragraph is the hook: what changes and why, readable on its own.
- Add one more paragraph only for a gotcha or constraint a reviewer must know.
- No provenance or journey. Link issues (`Closes #42`) instead of retelling them.

**Review comment.**
- One plain sentence that names the problem and its consequence.
  Bad: "Have you considered thread safety here? In Go, maps generally..."
  Good: "`seen` is written from two goroutines without a lock, so entries get lost."
- Prefix non-blocking comments with `nit:` or `optional:`. Unprefixed means blocking.
- Don't teach and don't over-explain. The author asks a follow-up if unclear.
- Praise goes in its own comment, never wrapped around a finding.

**Issue.**
- The repro outranks brevity. Give steps, expected, actual, the exact error
  text and versions, even when that runs long.
- Mark speculation about the cause as speculation, apart from what you observed.
