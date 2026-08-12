# Go context, timeouts and close behaviour

These rules apply to new or changed code. Problems in untouched code are a
sidetrack, not part of the change.

## Context and timeouts

- **Network and other blocking I/O needs a bound.** Each such call takes a
  `ctx` with a deadline, or a client or connection timeout. Flag
  `context.Background()` or `context.TODO()` on a request path, and
  `http.Client{}` with no `Timeout`.
- **Propagate the context to children.** Goroutines, downstream calls and
  queries take the caller's `ctx`. A fresh `context.Background()` inside a
  call chain drops cancellation and deadlines.
- **Shield critical-path children from the parent.** Work that must finish
  once started, such as a commit, a compensating write or a cleanup, uses
  `context.WithoutCancel(ctx)` plus its own timeout. Otherwise a cancelled
  request aborts it halfway.
- **Make timeouts agree.** A child timeout longer than the parent's remaining
  deadline never fires. A retry budget or safety timeout under a short parent
  is dead code. Find the effective bound and check it is the intended one.
- **Every `WithCancel` and `WithTimeout` has its `cancel` deferred,** in the
  scope that owns the work.

## Close behaviour

- **Idempotency and timing.** If a deferred `Close` is also the safety net,
  ask whether it should run as soon as the resource is done. Calling it early
  and deferring it is only safe when `Close` is idempotent, for example
  guarded by `sync.Once` or a closed flag.
- **Fallible close.** Decide whether the error is transient or critical, and
  whether it is returned, logged or suppressed. On write paths a close error
  can mean lost data. Return it with a named result:
  `defer func() { err = errors.Join(err, f.Close()) }()`. On read paths it can
  usually be dropped, and the code says so.
- **Deferral scope.** A `defer` inside a loop runs at function exit, not per
  iteration, so resources pile up. A `defer` in a helper that returns the
  resource closes it too early. Move the work into a function whose scope
  matches the resource's lifetime.
- **Panic safety.** Release runs on panic too, which `defer` guarantees and
  explicit calls on the happy path don't. A `Close` that can itself panic on
  nil or on a second call must not mask the original panic.
- **Close order.** Deferred calls run last in, first out. Shut down in
  dependency order: stop intake, let dependents finish or drain, then close
  the shared resources they use. Flag a resource closed while goroutines that
  use it may still be running, and dependents that learn of the close only by
  failing. They should be signalled first, through a cancelled `ctx`, a closed
  channel or a `Shutdown` call, and waited for.
- **Goroutine ownership.** Whoever starts a goroutine knows how it stops and
  waits for it (`sync.WaitGroup`, `errgroup`) before closing what it uses.
