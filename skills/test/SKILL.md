---
name: test
description: Plan and write unit tests oracle-first, in any language, cutting tests that catch no bug. Use when adding tests or coverage for code, even when no test file exists yet, or when extending a test file for new or changed behaviour. Not for running tests, debugging a failing test, or reviewing tests in someone's diff.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *)
paths:
  - "**/*_test.go"
  - "**/*.test.*"
  - "**/*.spec.*"
  - "**/test_*.py"
  - "**/*_test.py"
  - "**/tests/**"
  - "**/test/**"
  - "**/__tests__/**"
---

# Writing tests

**Gate.** This skill is for writing new tests. If you are running tests,
debugging a failure, reading fixtures or reviewing, stop here.

**Plan before writing.** Work through the steps below before the first test.
When the plan has more than 5 tests, show it and get the user's go-ahead
before writing any. In plan mode the list goes in the plan.

**Every test names the bug it catches.** A test that would stay green with
the line under test broken is cut.

## 1. Read first

- Run `${CLAUDE_PLUGIN_ROOT}/bin/include-fragment lenses/tests` now.
  Its per-test questions are the gate in step 7, and it defines the oracle
  rungs, partition classes and what to leave untested.
- For Go, also run `${CLAUDE_PLUGIN_ROOT}/bin/include-fragment go/test`.

## 2. Requirements

List what callers rely on, from the spec, doc comment, issue or bug report.
List behaviours, not functions. Tag each must, should or incidental.
Incidental behaviour usually gets no test.

## 3. Behaviours

Write one sentence per observable effect: "given X, it returns, emits or
rejects Y". One function can have several behaviours, and one behaviour can
span functions.

## 4. Oracles

Pick the first oracle rung that applies to each behaviour. If none applies,
don't write the test. Ask the user what the correct result is.

## 5. Partitions and boundaries

Enumerate the input classes and boundaries for each behaviour. Plan one case
per partition, not one per literal.

## 6. Selection

Plan the cases. Also write down what you will deliberately not test, with a
one-line reason for each.

## 7. Devil's-advocate gate

Argue against each planned test using the lens's per-test questions. Cut a
test that fails one, or merge it into the test that already covers its
partition.

## 8. Present the plan

One line per test: the behaviour as its name, its partition, its oracle rung
and the bug it catches. Then the "not tested" list.

## 9. Write

- Name each test after the behaviour it checks.
- One behaviour per test, so a failure has one meaning.
- No sleeps or unseeded randomness. Inject the clock and the random source.
- When the tests touch existing ones, check those against the lens's smells
  too.
- After writing, break the line under test in your head and confirm the test
  fails. Any test that wouldn't goes back through the gate.
