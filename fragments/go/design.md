# Go design

These rules apply to new or changed code. A violation that already exists in
untouched code is a sidetrack, not part of the change.

## Construction and dependencies

- **Mandatory arguments go in the signature.** Optional configuration goes in
  a trailing `...Option`. Flag required settings that arrive as options, as
  setters, or as fields checked by a later `Validate()` or `Build()`.
- **Dependencies are passed in.** A function or constructor never builds its
  own collaborators and never reads them from package-level state. Flag
  `http.DefaultClient`, global loggers, `os.Getenv` deep in logic, and package
  `var`s that hold clients or config.
- **The consumer declares the interface.** Define the small interface next to
  the code that calls it, so no module has to import another to satisfy it.
  Flag interfaces exported beside their only implementation.
- **A module that warrants it ships a `mock/` sub-package** with test doubles.
  It is safe to import from external test suites and depends only on the
  parent's exported API.

## Method sieve: hidden temporal coupling

The smell: methods that must be called in a set order, with nothing in the
types enforcing it. A workflow step that modifies a struct is not a method if
it assumes an order.

Methods suit:
- accessors;
- resolvers and transformations that map inputs to outputs;
- order-independent attachments such as `Register`, `Handle` or `Use`;
- `WithX` methods that return a modified copy.

When steps have an order, take the first option that can express it:

1. **`New` completes setup.** The value is never half-built.
2. **A step returns a narrower type,** the only way to reach the next
   operations. This is the preferred replacement.
   ```go
   func (c *Client) Connect(ctx context.Context) (*Session, error)
   func (s *Session) Query(ctx context.Context, q string) (Rows, error)
   ```
3. **A callback scope** for acquire/release pairs, so the scoped value can't
   escape and release always runs (`pgx.BeginFunc`).
4. **Pure transformers** when the steps are data flow: each output is the next
   input, so the types fix the order.
5. **A full typestate chain** of one struct per state, only for a public API
   where misuse is costly. Inside your own code it is over-engineering.
6. **Lazy init with `sync.OnceValues`,** the discouraged fallback. Errors and
   panics are cached forever, so there is no retry. A recursive call
   deadlocks. The first caller's context controls the shared init, so one
   caller's cancellation breaks it for every caller.

**Not flagged:** scoped, documented lifecycle pairs (Start/Wait, Begin/Commit,
Flush/Close), iterator protocols (`Next`/`Scan`; prefer `iter.Seq` in new
code) and terminal methods.

**Review signals:**
- an exported `Init`, `Setup`, `Configure` or `Load` that other methods need,
  or a `Start` that isn't a lifecycle method;
- `ErrNotInitialized`, a "not initialized" panic, or a nil guard doing the
  same job;
- doc text such as "must be called before" or "call X first";
- `initialized`, `started` or `ready` flags checked at the top of methods;
- nil checks on dependencies `New` should require, or `SetDB`-style setters;
- exported fields or setters changed after `Start` or `Serve`;
- `sync.Once` standing in for a missing constructor;
- tests that must call methods in a fixed sequence before asserting anything.
