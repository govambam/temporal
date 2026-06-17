---
title: Test reliability
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "**/*_test.go"
---

# Test reliability

Review changes to Go test files for testify-suite correctness and for patterns that
make tests hang or panic flakily on CI. Trace control flow across goroutines, channels,
and `Eventually`/`EventuallyWithT` blocks — most of these failures only appear under a
busy or slow test runner, so reason about the unlucky interleaving, not the happy path.

Report findings grouped by severity (🔴 Must fix / 🟡 Should fix / 🟢 Nit) with the
file and line, a one-line explanation of the failure mode, and the concrete fix.

### 🔴 Testify assertions inside a goroutine
Flag any testify assertion — `s.NoError`, `s.Equal`, `s.True`, `require.NoError`,
`require.Equal`, even `assert.NoError` — called inside a `go func() { ... }()` (or a
helper invoked as `go s.helper(...)`). If the goroutine outlives the test, a failed
assertion panics the whole binary with `panic: Fail in goroutine after TestXxx has
completed`. The fix is to move the assertion to the test goroutine, or send the
error/result back over a buffered channel and assert on it from the test goroutine.

### 🔴 Unguarded channel receive that can hang
Flag any `<-ch` (or `v := <-ch`) that is **not** inside a `select` with a
`ctx.Done()` (or timeout) case. If the sender never sends — because an earlier step
failed — the test hangs until the overall test timeout instead of failing fast with a
useful message. Require a `select { case v := <-ch: ...; case <-ctx.Done(): ... }`
fallback.

### 🟡 Run-once goroutine maintaining a precondition
When a goroutine is launched to maintain a precondition that a later assertion depends
on (e.g. keeping pollers active so a deployment version registers), flag it if it runs
the operation **once** and exits. A single attempt that fails transiently (network,
tight deadline, busy CI) exits silently, and the downstream `Eventually`/propagation
wait then hangs until its own deadline. It should loop until `ctx.Done()`. Be
specifically suspicious of `go s.someHelper(ctx, ...)` immediately followed by a wait
for something that helper was supposed to cause.

### 🟡 EventuallyWithT / Eventually timeout ordering
When an `Eventually` or `EventuallyWithT` waits on a condition driven by a background
goroutine, flag it if the background goroutine's own timeout/deadline is shorter than
the `Eventually` deadline. If the background op times out first, the condition can
never become true and the wait hangs until its full deadline. The background timeout
must be longer than the `Eventually` deadline.

### 🟡 Sleep / wall-clock used for ordering or synchronization
Flag `time.Sleep` or `time.Since(start) > threshold` used to enforce ordering or wait
for a condition (this is also linter-forbidden in this repo). Require channels,
`sync.WaitGroup`, or `EventuallyWithT` instead. Also flag `context.Background()` in a
test where `s.Context()` is available — the suite context carries the test timeout and
makes waits more stable.

### 🟡 require vs assert, and subtest/suite misuse
- Prefer `require` over `assert`: it is rarely useful to keep running a test after a
  failed assertion. Flag new `assert.*` / `s.Assert()...` where `require` is intended.
- Flag `s.T()` used inside a subtest — use the subtest's own `t` parameter.
- Flag single-value type assertions on errors (`err.(*MyErr)`) — they panic instead of
  failing the test on a type mismatch. Require `errors.As` / `errors.AsType` with a
  guarded return.

### 🟡 Silently discarded precondition errors
Flag `_, _ = f()` (or `_ = f()`) on a precondition operation whose failure would
invalidate the rest of the test. Surface the error (assert on it) or loop until it
succeeds, rather than swallowing it.

### 🟢 testifylint conventions
- Float comparisons: use `InDelta` / `InEpsilon`, not `Equal`.
- Error-type checks: use `require.ErrorAs` / `errors.AsType`, not a bare type assertion.
- Combine `require.Error` + `require.Contains` into a single `require.ErrorContains`.
- When an `Eventually` block needs assertions, use `EventuallyWithT` and that block's
  `t`.

### Permission to do nothing
If the changed test files contain none of these patterns, say so and report nothing.
Do not invent findings, do not flag pre-existing code outside the diff's scope, and do
not restate generic "add more tests" advice — only flag the concrete reliability
patterns above.
