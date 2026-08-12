# Go doc comments

These rules apply to new or changed code. A missing or malformed comment on
untouched code is a sidetrack, not part of the change.

## Exported symbols

Every exported symbol has a doc comment that starts with its name.

- **The first line is one high-level summary.** It says what the symbol does
  or returns, not how.
- **Further notes are separate blocks,** divided by an empty `//` line.
- **Caveats get their own block.** "Panics if...", "By default..." and
  "Default: ..." never trail the summary line.
- **Document the why and the non-obvious:** concurrency safety, zero values,
  ownership, error conditions, edge cases. Don't restate what the signature
  already says.
- **Cross-reference other symbols with doc links** such as [Client] or
  [io.Reader], so they render as links in godoc and gopls.
- **Write ASCII in prose:** `<=`, `>=`, `< 0`. No unicode comparison signs.

```go
// Retry executes f up to n times, returning the last error.
//
// Between attempts it waits according to the configured backoff.
// A zero n is treated as a single attempt.
//
// Retry is safe for concurrent use.
func Retry(n int, f func() error) error { ... }
```

## Packages

The package comment gives the purpose and typical usage in two or three
lines. Add a full example only when the API is non-obvious.

## Unexported symbols and inline comments

Comment an unexported symbol only when its logic is non-trivial. No decorative
banner or separator comments.

## Review signals

- an exported symbol with no comment, or one that doesn't start with its name;
- a caveat appended to the summary line instead of its own block;
- a comment that paraphrases the signature and says nothing else;
- a symbol name where a [Symbol] doc link belongs;
- unicode comparison signs, banners, or comments on trivial unexported code.
