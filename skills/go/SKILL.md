---
name: go
description: Rules for writing or changing Go code, including scaffolding a new package or module. Use when adding or editing functions, types, constructors, doc comments or tests in .go files. Not for reading, explaining or debugging Go without changing it, and not for reviewing someone else's diff.
allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *)
paths:
  - "**/*.go"
---

# Writing Go

**Gate.** This skill matters only when you are about to write or change Go.
If you are reading, explaining, tracing a stack trace or reviewing, stop here.

The rules apply to new or changed code. Violations in code you don't touch are
sidetracks: raise them, don't fix them in the same change.

Load each before writing the code it covers, once per session. One call can
load several: list them all after `bin/include-fragment`.

- **Adding or changing functions, types, constructors or packages:** run
  `${CLAUDE_PLUGIN_ROOT}/bin/include-fragment go/design go/doc`.
- **Writing or changing `_test.go` files:** run
  `${CLAUDE_PLUGIN_ROOT}/bin/include-fragment go/test`.
- **Network or file I/O, goroutines, timeouts, or anything with `Close`,
  `Shutdown` or a lifecycle:** run
  `${CLAUDE_PLUGIN_ROOT}/bin/include-fragment go/ctx-close`.
