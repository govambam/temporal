---
title: Test Reliability
model: claude-opus-4-6
effort: medium
input: full_diff
include:
  - "**/*_test.go"
---

# Test reliability

Review changed Go test files for patterns that make tests hang, flake, or panic the
whole test binary. These are concurrency-correctness issues specific to Go + testify
suites, not style preferences.

### 🔴 Must fix — no testify assertions from goroutines; no `s.T()` in subtests

Flag any testify assertion — `s.NoError`, `s.Equal`, `require.NoError`,
`require.X`, even `assert.X` — called inside a `go func()` / background goroutine. If
the goroutine outlives the test, the assertion panics the binary with
`panic: Fail in goroutine after TestXxx has completed`. Move the assertion to the
test goroutine, or send the error over a buffered channel and assert on the test
goroutine.

Also flag use of `s.T()` inside a subtest closure — use the subtest's own `t`
parameter instead. Suite-level helpers and assertion methods must not be reached from
a subtest via `s.T()`.

### 🔴 Must fix — guard channel receives with context cancellation

Flag a channel receive (`<-ch`) that is **not** inside a `select` with a
`ctx.Done()` / `s.Context().Done()` (or equivalent cancellation) case. A bare receive
hangs indefinitely if the sender never sends, turning a real failure into a test
timeout with no useful message. Wrap it:

```go
select {
case v := <-ch:
    // ...
case <-s.Context().Done():
    s.FailNow("timed out waiting for ...")
}
```

### 🟡 Should fix — no single-value error type assertions

Flag single-value type assertions on errors, e.g. `err.(*MyError)`. When the type
doesn't match this **panics** instead of failing the test cleanly. Use the
comma-ok form, or `errors.As` / `errors.AsType` with a guarded return.

## What not to flag

- Assertions on the main test goroutine.
- Channel receives already inside a `select` with a cancellation/timeout case.
- Comma-ok type assertions (`v, ok := x.(*T)`) — those are safe.
- Linter-enforced concerns that are already caught elsewhere (`time.Sleep`,
  `assert.X`, testify `Eventually`, `context.Background()` in tests) — don't restate
  them.

If a test file has none of these issues, **report nothing**.
