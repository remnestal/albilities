## Tests lens

Every test must earn its place by catching a bug no other test catches. The
questions below apply to a test you are about to write and to a test under
review. When writing, a test that fails one is cut or merged. When reviewing,
the failure is the finding.

### Questions per test

1. **Trace.** Which behaviour that a caller relies on does it check? "The
   function exists" is not one. Take behaviours from the spec, doc comment,
   issue or bug report, and describe each as "given X, it returns, emits or
   rejects Y".
2. **Partition.** Which input class or boundary does it cover that no sibling
   covers? Classes are valid, invalid, empty, nil and max. Boundaries are min-1,
   min, max and max+1. A sibling taking the same branch with another literal is
   a duplicate.
3. **Mutant.** Name the bug it catches: a flipped comparison, an off-by-one, a
   deleted statement, a zero-value return, a swallowed error. If it still
   passes with the line under test broken, it tests nothing.
4. **Oracle.** Where does the expected value come from? Use the first rung
   of this ladder that applies:
   - *specified*: the spec or contract states it;
   - *derived*: a round-trip or inverse, idempotence, an invariant, a relation
     between two inputs, or a slow reference implementation that is obviously
     right;
   - *implicit*: no crash, panic, leak or hang;
   - *human*: a golden file someone actually reads.

   If none applies, the test has no oracle. When writing, don't write it: ask
   the user where the expected value comes from. When reviewing, report the
   missing oracle as a finding.
5. **Refactor survival.** Does it still pass after a restructure that keeps
   behaviour? Asserting on call order, private state, or mock calls no caller
   observes makes it a change-detector.
6. **Delete.** If it were deleted, which bug would ship? "None" or "one test N
   already catches" means it goes.

A good test is behavioural, insensitive to structure, specific (its failure
points at the cause) and predictive (its passing means the code works).

### Leave untested on purpose

- Pure delegation, accessors, and wiring.
- Guarantees the language or framework already gives.
- Arid code such as logging.
- A partition another test covers.
- Anything whose only possible oracle is the implementation itself.

### Smells in existing tests

**No real logic under test:**
- Mocks asserting on other mocks, or the code under test stubbed out.
- The only assertion is "no error".
- No assertion at all, or one catch-all test for everything.
- Tests that only cover getters or framework behaviour.

**Weak oracle:**
- Expected values pasted from the output rather than worked out from the spec.
- The expected value computed with the same formula or helper as the code,
  which makes the test tautological.
- Golden files nobody reads.
- Only the error type checked where its message or content matters.

**Bloat:**
- Two tests covering one partition.
- Interactions over-specified.
- Sleeps, wall-clock time, or other non-determinism.
- One test checking several behaviours, so its failure can't say which broke.
- A test name that doesn't state the behaviour.

### The mutant question for a change

For each changed line of production code, ask which test fails if that line is
broken. If none does, the tests have a gap or the line is dead code. Report
which, and name the behaviour the missing test would check.
